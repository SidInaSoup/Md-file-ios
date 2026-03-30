# SlideGen Frontend — Performance & Production-Readiness Audit

## 📊 Bundle Size Snapshot

| Chunk | Size (minified) | Size (gzip) | Notes |
|---|---|---|---|
| **index (main)** | **792 kB** | **242 kB** | ⚠️ Exceeds 500 kB limit. Contains Konva, react-konva, all pages, utilities |
| AdminTokenDashboard | 381 kB | 112 kB | Recharts is bundled here (lazy-loaded ✅) |
| CSS | 77 kB | 12 kB | Tailwind |
| Admin pages (3 lazy) | ~24 kB total | ~8 kB | Properly lazy-loaded ✅ |

> [!CAUTION]
> The main bundle is **792 kB** minified (242 kB gzipped). Vite itself warns about `>500 kB` chunks. Every user — even those just viewing the DeckList — pays the full cost of loading Konva + SlideStage + GeneratedDeckView.

---

## 🔴 Critical Issues

### 1. Missing Lazy-Loading for Heavy Pages

**Files:** [App.tsx](file:///home/siddarth/general/pptGen/slidegen-frontend/src/App.tsx#L4-L8)

Three **heavy pages** are eagerly imported and bundled into the main chunk:

```tsx
import DeckList from './pages/DeckList'           // 21 KB source
import GenerateView from './pages/GenerateView'    // 67 KB source
import GeneratedDeckView from './pages/GeneratedDeckView' // 105 KB source — the biggest file
```

Meanwhile the relatively small admin pages *are* properly lazy-loaded. The priorities are inverted.

**Impact:** Users loading `/` (DeckList) still download and parse the entire GeneratedDeckView + SlideStage + Konva runtime.

**Fix:** Lazy-load all three:
```tsx
const DeckList = lazy(() => import('./pages/DeckList'))
const GenerateView = lazy(() => import('./pages/GenerateView'))
const GeneratedDeckView = lazy(() => import('./pages/GeneratedDeckView'))
```

---

### 2. Zustand Is Imported But **Never Used** on Critical Paths

**Files:** [store/ui.ts](file:///home/siddarth/general/pptGen/slidegen-frontend/src/store/ui.ts), [store/tokenUsageStore.ts](file:///home/siddarth/general/pptGen/slidegen-frontend/src/store/tokenUsageStore.ts)

- `useUI` (from `store/ui.ts`) is **exported but never imported** anywhere in the codebase. Zero consumers.
- `useTokenUsageStore` is only consumed from `AdminTokenDashboard`, `AdminTokenExplorer`, and `TokenUsageFilterBar` — all admin-only pages that are already lazy-loaded.

**Impact:** Zustand itself is tiny (~2 kB), so it's not a bundle-size disaster. But `useUI` is dead code that should be removed. The store module is tree-shaken for non-admin builds, so this is a **low severity** issue for bundle size — but it's a code hygiene concern.

**Fix:** Delete `store/ui.ts` entirely since no component uses `useUI`. Zustand can stay as a dependency for the admin stores.

---

### 3. Recharts — 381 kB Chunk Connected to Admin Only

**File:** `AdminTokenDashboard.tsx` (only consumer of `recharts`)

Recharts is already isolated to a lazy-loaded admin page, so it doesn't affect the critical path. ✅ No action needed.

---

### 4. Full `Konva` Import in GeneratedDeckView (not SlideStage)

**File:** [GeneratedDeckView.tsx:4](file:///home/siddarth/general/pptGen/slidegen-frontend/src/pages/GeneratedDeckView.tsx#L4)

```tsx
import Konva from 'konva'
```

GeneratedDeckView imports the full Konva library at the top, but only uses the type `Konva.Stage` for the `stageRef`:
```tsx
const stageRef = useRef<Konva.Stage | null>(null)
```

This forces the entire Konva runtime into the module scope even though `SlideStage.tsx` already imports it. Vite deduplicates modules, so this doesn't double the cost — but it **prevents code-splitting** GeneratedDeckView away from Konva.

**Fix:** Use `import type` to avoid pulling in the runtime:
```tsx
import type Konva from 'konva'
```

Same issue in [useSlideThumbnail.ts:2](file:///home/siddarth/general/pptGen/slidegen-frontend/src/utils/useSlideThumbnail.ts#L2) — this one *does* use Konva's runtime (`stage.toDataURL()`, `stage.findOne()`), so it legitimately needs the import. But it should be co-split with SlideStage.

---

## 🟠 High-Severity Issues

### 5. GeneratedDeckView.tsx Is a 2,506-line Monolith

**File:** [GeneratedDeckView.tsx](file:///home/siddarth/general/pptGen/slidegen-frontend/src/pages/GeneratedDeckView.tsx) — **105 KB source, 2,506 lines**

This single file contains:
- `normalizeBlueprintResponse` — data normalization (lines 42–68)
- `getBox`, `pickText`, `normalizeAssetUrl`, `pickImageUrl`, `bulletsToText`, `blueprintToRenderableElement` — element conversion helpers (lines 70–267)
- `applyPatchToBlueprintElement` — patch logic (lines 269–291)
- `BlueprintToolbar` — **entire toolbar component** with 400+ lines of JSX (lines 297–703)
- `SlidePreviewThumbnail` — thumbnail component (lines 708–817)
- `SlidePreviewPanel` — slide panel component (lines 820–861)
- `SlideCarousel` — carousel with pre-rendering logic (lines 874–1219)
- `GeneratedDeckView` main component — 1,200+ lines (lines 1223–2505)

**Performance Impact:**
- React must parse and execute 105 KB of JS on page load
- Every state change in the main component potentially triggers re-evaluation of helper closures
- The toolbar re-renders on every parent state change (no `React.memo`)

**Fix:** Extract at minimum:
1. `BlueprintToolbar` → `components/BlueprintToolbar.tsx`
2. `SlidePreviewPanel` + `SlidePreviewThumbnail` → `components/SlidePreviewPanel.tsx`
3. `SlideCarousel` → `components/SlideCarousel.tsx`
4. Helper functions → `utils/blueprint.ts`

---

### 6. `Math.random()` Called During Render in Thumbnail Fallback

**File:** [GeneratedDeckView.tsx:754-799](file:///home/siddarth/general/pptGen/slidegen-frontend/src/pages/GeneratedDeckView.tsx#L754-L799)

```tsx
style={{
  height: `${Math.min(20, 8 + Math.random() * 12)}%`,
  width: `${50 + Math.random() * 40}%`,
  ...
}}
```

`Math.random()` in JSX style props means **every render produces different style objects**, which:
1. Defeats React's reconciliation — the style objects are never equal, so DOM updates happen every time
2. Causes visual flicker if the component re-renders
3. Prevents memoization

**Fix:** Pre-compute random dimensions in a `useMemo` keyed on a stable identifier, or use deterministic sizing based on element properties.

---

### 7. Console.log Pollution — 19+ Debug Statements in Production Code

**Files:** `GeneratedDeckView.tsx` (10 statements), `SlideStage.tsx` (8 statements), `lib/api.ts` (1 statement)

Examples of hot-path console.logs that fire on every interaction:
```tsx
// SlideStage.tsx:687 — fires on EVERY editor state change
console.log('[SlideStage editor]', editor)

// SlideStage.tsx:780 — fires on EVERY canvas click
console.log('[STAGE click]', { id: ..., name: ..., class: ... })

// SlideStage.tsx:1958 — fires on EVERY TABLE RENDER
console.log('[TableBox] el.style:', el.style, 'vs:', vs, ...)

// GeneratedDeckView.tsx:1668 — fires when canUndo recalculates (via useMemo!)
console.log('[Undo] trigger:', undoRedoTrigger, ...)
```

**Impact:** Console.log with complex objects triggers serialization costs. The click/editor logs fire on mouse interactions and cause observable slowness in the DevTools-open scenario. The `useMemo` ones defeat the purpose of memoization since the side effect runs every time.

**Fix:** Remove all debug console.logs, or wrap in `import.meta.env.DEV` guards.

---

### 8. Console.log Inside `useMemo` — Side Effect That Runs on Every Recalculation

**File:** [GeneratedDeckView.tsx:1667-1675](file:///home/siddarth/general/pptGen/slidegen-frontend/src/pages/GeneratedDeckView.tsx#L1667-L1675)

```tsx
const canUndo = useMemo(() => {
    console.log('[Undo] trigger:', undoRedoTrigger, 'stackLen:', undoStackRef.current.length)
    return undoStackRef.current.length > 0
}, [undoRedoTrigger])

const canRedo = useMemo(() => {
    console.log('[Redo] trigger:', undoRedoTrigger, 'stackLen:', redoStackRef.current.length)
    return redoStackRef.current.length > 0
}, [undoRedoTrigger])
```

Side effects inside `useMemo` are explicitly forbidden by React. Strict mode double-invocation will run these logs twice. Not just noise — it's a correctness violation.

---

### 9. Inline `<style>` Tags in JSX — Duplicated on Every Render

**File:** [GeneratedDeckView.tsx:1126-1135](file:///home/siddarth/general/pptGen/slidegen-frontend/src/pages/GeneratedDeckView.tsx#L1126-L1135) and [GeneratedDeckView.tsx:2475-2502](file:///home/siddarth/general/pptGen/slidegen-frontend/src/pages/GeneratedDeckView.tsx#L2475-L2502)

```tsx
<style>{`
  @keyframes slideInFromRight { ... }
  @keyframes slideInFromLeft { ... }
`}</style>
```

Two separate `<style>` blocks with CSS keyframes are embedded in JSX. These:
1. Get re-inserted into the DOM on every mount
2. Duplicate keyframe definitions that could be in the CSS file
3. The `SlideCarousel` style block is **inside a conditionally-rendered component**, so it mounts/unmounts

**Fix:** Move keyframe definitions to `index.css` or `App.css`.

---

## 🟡 Medium-Severity Issues

### 10. Unstable Callback References Cause Unnecessary Re-renders

**File:** [GeneratedDeckView.tsx:2166-2167](file:///home/siddarth/general/pptGen/slidegen-frontend/src/pages/GeneratedDeckView.tsx#L2116-L2128)

```tsx
onChange={trackedOnChange}              // defined in component body, new ref each render
onDelete={onDelete}                      // async function, new ref each render
onCreateText={onCreateText}              // async function, new ref each render
```

None of these are wrapped in `useCallback`. Every render of `GeneratedDeckView` creates new function references, which causes `SlideCarousel` → `SlideStage` to re-render even when nothing changed.

**Fix:** Wrap `trackedOnChange`, `onDelete`, `onCreateText`, `onCreateImage` in `useCallback`.

---

### 11. `SlideStage.tsx` Is Another Monolith — 2,852 Lines (93 KB)

**File:** [SlideStage.tsx](file:///home/siddarth/general/pptGen/slidegen-frontend/src/components/SlideStage.tsx) — 2,852 lines

Contains `TextBox`, `ImageBox`, `ShapeBox`, `TableBox`, and all their editing logic inline. All table manipulation helpers (`insertRow`, `insertCol`, `deleteRows`, `deleteCols`, `clearCells`, `patchCell`) are defined in the same file.

**Performance Impact:** Same as issue #5 — everything is one giant parse/eval unit. Component extraction would enable React.memo boundaries.

---

### 12. Sorting Inside Render Without Memoization

**File:** [SlideStage.tsx:849-851](file:///home/siddarth/general/pptGen/slidegen-frontend/src/components/SlideStage.tsx#L849-L851)

```tsx
{elements
  .slice()
  .sort((a, b) => (a.z_index ?? 0) - (b.z_index ?? 0))
  .map((el) => ...
```

`.slice().sort()` runs on **every render** of SlideStage — including scroll, selection, hover, and editor state changes. For a 20-element slide, this is negligible, but it's easily memoizable.

**Fix:** `useMemo(() => [...elements].sort(...), [elements])`

---

### 13. Offscreen Pre-Renderer Mounts Full SlideStage Instances

**File:** [GeneratedDeckView.tsx:1196-1216](file:///home/siddarth/general/pptGen/slidegen-frontend/src/pages/GeneratedDeckView.tsx#L1196-L1216)

The pre-render pipeline mounts a **full** `<SlideStage>` off-screen at position `left: -9999` to capture thumbnails. This means:
- Full Konva Stage instantiation per pre-rendered slide
- Full element tree rendering through react-konva
- 700ms timer per slide × N slides of serial work

For a 20-slide deck, that's ~14 seconds of background work before all thumbnails are ready.

**Fix consideration:** Use `stage.toDataURL()` from a single persistent off-screen stage, re-using Konva layer content. Or generate server-side thumbnails.

---

### 14. Missing `React.memo` on Expensive Sub-Components

None of the inner components (`BlueprintToolbar`, `SlidePreviewThumbnail`, `SlidePreviewPanel`, `SlideCarousel`) use `React.memo`. Since they receive callback props that change on every render (issue #10), they re-render on every parent state change.

---

### 15. Autosave Timer Never Flushes on Unmount

**File:** [GeneratedDeckView.tsx:1586-1593](file:///home/siddarth/general/pptGen/slidegen-frontend/src/pages/GeneratedDeckView.tsx#L1586-L1593)

```tsx
useEffect(() => {
    const timer = setInterval(() => {
        if (dirtyElementsRef.current.size > 0) {
            saveDirtyItems()
        }
    }, 30000)
    return () => clearInterval(timer)
}, [themeId])
```

If the user navigates away before the 30-second interval fires, dirty changes are lost. The cleanup only clears the interval — it doesn't flush pending saves.

**Fix:** Add `saveDirtyItems()` to the cleanup function, or use `beforeunload`.

---

### 16. `saveDirtyItems` Has Stale Closure Over `themeId`

**File:** [GeneratedDeckView.tsx:1562-1583](file:///home/siddarth/general/pptGen/slidegen-frontend/src/pages/GeneratedDeckView.tsx#L1562-L1583)

`saveDirtyItems` references `themeId` from its closure scope but isn't wrapped in `useCallback` with `themeId` as dependency. The `useEffect` at line 1586 depends on `[themeId]` but references `.current` values — meaning a `themeId` change recreates the interval but `saveDirtyItems` may still capture an old `themeId`.

---

## 🟢 Low-Severity / Code Quality Issues

### 17. Duplicate Click-Outside Handlers

**File:** [GeneratedDeckView.tsx:373-391](file:///home/siddarth/general/pptGen/slidegen-frontend/src/pages/GeneratedDeckView.tsx#L373-L391)

Two identical `useEffect` blocks attach `mousedown` listeners for click-outside (one for color picker, one for SmartArt menu). These could be a reusable hook.

### 18. `DeckView.tsx` Is Dead Code

**File:** [DeckView.tsx](file:///home/siddarth/general/pptGen/slidegen-frontend/src/pages/DeckView.tsx) — 27 KB

The import is commented out in App.tsx (`// import DeckView from './pages/DeckView'`) and the route is also commented out. But the file still exists (27 KB). Vite's tree-shaking should exclude it, but it's unnecessary source code.

### 19. `nanoid` Dependency — Never Imported

`nanoid` is in `package.json` but no file in `src/` imports it. Dead dependency.

### 20. `lib/api.ts` Debug Log in Production

**File:** [lib/api.ts:86](file:///home/siddarth/general/pptGen/slidegen-frontend/src/lib/api.ts#L86)

```tsx
console.log('[DEBUG] listThemes raw data:', data);
```

This fires every time themes are fetched — which happens on the DeckList landing page.

### 21. `eslint-disable` Comments in Generated Code

Multiple `// eslint-disable-next-line no-console` annotations, indicating awareness of console.log issues but choosing to keep them.

### 22. No Production Error Boundary Messaging

The `ErrorBoundary` component is minimal (979 bytes). In production, unhandled errors in the SlideStage canvas or Konva rendering will show the default fallback with no actionable recovery.

---

## 📋 Recommended Priority Order

| Priority | Issue | Effort | Impact on Load Time |
|---|---|---|---|
| **P0** | #1 Lazy-load DeckList, GenerateView, GeneratedDeckView | 10 min | 🟢 ~40% reduction in main bundle |
| **P0** | #4 Change `import Konva` to `import type Konva` in GeneratedDeckView | 2 min | 🟢 Enables proper code-splitting |
| **P0** | #7 Remove all console.log statements | 15 min | 🟢 Eliminates runtime serialization overhead |
| **P1** | #5 Extract components from GeneratedDeckView | 2-3 hrs | 🟡 Enables React.memo, reduces parse cost |
| **P1** | #6 Fix Math.random() in render path | 10 min | 🟡 Prevents unnecessary DOM thrashing |
| **P1** | #9 Move keyframes to CSS file | 10 min | 🟡 Prevents style re-injection |
| **P1** | #10 Wrap callbacks in useCallback | 30 min | 🟡 Prevents cascading re-renders |
| **P1** | #15 Flush dirty saves on unmount | 10 min | 🔵 Data loss prevention |
| **P2** | #2 Remove dead `useUI` store | 5 min | 🟡 Code hygiene |
| **P2** | #11 Extract sub-components from SlideStage | 3-4 hrs | 🟡 Maintainability + perf |
| **P2** | #12 Memoize element sorting | 5 min | 🟡 Minor render optimization |
| **P2** | #14 Add React.memo to sub-components | 20 min | 🟡 Re-render reduction |
| **P3** | #18 Delete dead DeckView.tsx | 2 min | 🔵 Hygiene |
| **P3** | #19 Remove unused nanoid dep | 2 min | 🔵 Hygiene |
| **P3** | #17 Extract click-outside hook | 15 min | 🔵 DRY |

---

## Vite Config Recommendations

Add manual chunk splitting to `vite.config.ts`:

```ts
export default defineConfig({
  plugins: [react()],
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'konva': ['konva', 'react-konva'],
          'react-vendor': ['react', 'react-dom', 'react-router-dom'],
          'tanstack': ['@tanstack/react-query'],
        }
      }
    }
  },
  // ... proxy config
})
```

This would split the 792 kB chunk into:
- `konva` (~300 kB) — only loaded when SlideStage is needed
- `react-vendor` (~200 kB) — cached across deploys
- `tanstack` (~50 kB) — cached separately
- `index` (~240 kB) — your application code

---

Let me know which issues you'd like me to fix first.
