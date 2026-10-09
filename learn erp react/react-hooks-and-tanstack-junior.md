# React, hooks, and TanStack Query (complete junior guide)

Read this **alongside** [creating-a-page-by-hand.md](./creating-a-page-by-hand.md). That
doc tells you **which files to create**. This doc tells you **what React and TanStack
are doing** so you are not copying hooks blindly.

You do **not** need to memorize everything. Use this as a reference while you build
your first page. Come back to each section when you see that hook in real code.

---

## Table of contents

1. [What React actually is](#1-what-react-actually-is)
2. [JSX and components](#2-jsx-and-components)
3. [Props, state, and re-renders](#3-props-state-and-re-renders)
4. [Events and functions](#4-events-and-functions)
5. [Modules: import and export](#5-modules-import-and-export)
6. [Hooks — rules and mental model](#6-hooks--rules-and-mental-model)
7. [Hook reference (used in this repo)](#7-hook-reference-used-in-this-repo)
8. [Context — shared data without prop drilling](#8-context--shared-data-without-prop-drilling)
9. [Suspense and loading UI](#9-suspense-and-loading-ui)
10. [TanStack Query from zero](#10-tanstack-query-from-zero)
11. [Read ReplenishmentList line by line](#11-read-replenishmentlist-line-by-line)
12. [Practice questions](#12-practice-questions)

---

## 1. What React actually is

### PHP page mental model

On each request, PHP builds HTML and sends it. The browser shows it. If you want
to change the page, you usually **request again** or use jQuery to **change the DOM**.

### React mental model

React keeps a **description** of the UI in memory (a tree of components). When
something important changes (usually **state**), React **runs your components again**
and figures out what changed in the real DOM.

You mostly write:

```text
UI = f(data)
```

Not: “find `#row-5` and set `.innerHTML`.”

### Component

A **component** is a function (in this project, always a function) that returns
**JSX** — markup that looks like HTML inside JavaScript.

```jsx
function Hello() {
  return <p>Hello</p>;
}
```

`Hello` is a component. React calls `Hello()` when it needs to show that UI.

### Where React mounts in this ERP

The monolith prints an empty div. `bootstrap.js` calls `createRoot(div)` and
renders your page component inside `MountProviders`. From that moment, **React owns
everything inside that div**.

---

## 2. JSX and components

### JSX is not HTML

```jsx
<div className="flex">Title</div>
```

- `className` not `class` (because `class` is a JS keyword).
- You can embed JavaScript with `{expression}`:

```jsx
const title = "Replenishment";
return <h1>{title}</h1>;
```

### One parent per return

```jsx
// OK — one wrapper
return (
  <div>
    <GridApp />
  </div>
);

// Also OK — fragment, no extra DOM node
return (
  <>
    <GridApp />
  </>
);
```

### Components are capitalized

`<GridApp />` — React treats this as a custom component.  
`<div />` — built-in HTML element.

### `children`

When you write:

```jsx
<PageChrome>{children}</PageChrome>
```

`children` is whatever JSX was placed **between** the tags. Layout components use
this pattern everywhere.

---

## 3. Props, state, and re-renders

### Props = inputs from parent (read-only)

Like arguments to a PHP function or template variables you pass in:

```jsx
function ReplenishmentGrid({ properties }) {
  return <GridApp properties={properties} />;
}
```

Parent passes `properties={gridProperties}`. The child must **not** mutate props
(React expects props to be treated as immutable).

**PHP analogy:** `$this->render('grid', ['id' => 137])` — the view receives data;
it should not change the controller’s array in place.

### State = memory inside one component instance

When **state** changes, React **re-runs** that component function and updates the DOM.

```jsx
const [open, setOpen] = useState(false);
```

- `open` — current value (`false` = modal closed).
- `setOpen` — function to update it: `setOpen(true)`.

**PHP analogy:** there is no perfect match on the server. Closest: a session flag
`$_SESSION['modal_open']` that you read on each request — except React can update
**without** a full page reload.

### Re-render (simplified)

1. Something calls `setState` / `setOpen` / mutation success / context update.
2. React marks components that depend on that data as needing an update.
3. React calls those component functions again.
4. React compares new JSX to old and patches the DOM.

You do **not** call “re-render” yourself. You **update state or cache**, React renders.

### When beginners get confused

| Symothesis | Reality |
| ---------- | ------- |
| “The whole app reloads” | Usually only a subtree re-renders |
| “I must manually update the grid cell” | Often you mutate row data + tell grid API to refresh cells (see service layer) |
| “useMemo stops all re-renders” | It only caches a **computed value** between renders |

---

## 4. Events and functions

### jQuery

```js
$("#save").on("click", function () {
  save();
});
```

### React

```jsx
<button type="button" onClick={() => save()}>
  Save
</button>
```

You pass a **function** to `onClick`. React calls it when the user clicks.

### Why `() => save()` instead of `onClick={save}`?

Both can work. Use an arrow when you need to **pass arguments**:

```jsx
onClick={() => fillQtyToOrderFromField(gridRef, "min_qty_sum")}
```

That creates a new small function on each render. For header actions built in
`useMemo`, the action list is rebuilt only when dependencies change — that is fine.

### `async` handlers

```js
onClick: async () => {
  const ok = await confirm({ ... });
  if (!ok) return;
  saveToPo(rows);
},
```

`await` pauses **this function** until the user answers the dialog. The browser
stays responsive; you are not blocking the whole page.

---

## 5. Modules: import and export

Each file is a **module**. You connect files with `import` / `export`.

```js
// constant.js
export const MY_GRID_PROPERTIES = { id: 137 };

// MyFeatureList.jsx
import { MY_GRID_PROPERTIES } from "./constant";
```

- **Named export:** `export function buildActions` → `import { buildActions } from "./actions"`.
- **Default export:** `export default MyFeatureList` → `import MyFeatureList from "./MyFeatureList"`.

Path `./constant` means “same folder”. `@/` is a project alias for `src/`.

**Rule for your pages:** keep **one default export** per component file (repo convention).

---

## 6. Hooks — rules and mental model

### What is a hook?

A function whose name starts with `use` that connects you to React features:
state, context, refs, memoization, etc.

Examples: `useState`, `useEffect`, `useGrid` (custom hook wrapping context).

### The two rules (simplified)

1. Only call hooks at the **top level** of a component or custom hook — not inside
   `if`, loops, or nested functions.
2. Only call hooks from **React function components** or **other hooks**.

**Why:** React relies on **call order** being the same every render. An `if` would
skip a hook sometimes and break that.

```jsx
// BAD
if (show) {
  const [x, setX] = useState(0);
}

// GOOD
const [x, setX] = useState(0);
if (!show) return null;
```

### Custom hooks

`useGrid()`, `usePageHeader()`, `useConfirmDialog()` are **custom hooks** defined in
this repo. They are thin wrappers around `useContext` + helpers. You use them like
built-in hooks.

---

## 7. Hook reference (used in this repo)

### `useState`

**Purpose:** UI-only memory (modal open, selected tab, draft id).

```jsx
const [creating, setCreating] = useState(false);
setCreating(true);
```

**Functional update** (when new state depends on old):

```jsx
setItems((current) => [...current, newItem]);
```

Use this inside `useCallback` when you do not want to list `items` as a dependency.

---

### `useEffect`

**Purpose:** Run side effects **after** paint — sync with something outside React.

Examples in this codebase:

- Prefetch host options when page mounts (`OpenCounts`).
- Register `usePageHeader` config (inside `PageChromeContext` implementation).

```jsx
useEffect(() => {
  prefetchHostOptions("count_codes");
}, []);
```

`[]` = run once when component mounts (like `$(document).ready` once).

**Do not use `useEffect` for:**

- Saving on button click → use the click handler.
- Deriving display values from props → compute during render.
- Loading list data → prefer TanStack Query.

**PHP analogy:** `useEffect(..., [])` ≈ one-time init in a view script.  
`useEffect(..., [id])` ≈ “when `$id` changes, refetch related stuff” — but for data,
Query does this better.

---

### `useRef`

**Purpose:** A box `{ current: value }` that **persists** across renders and
**does not** cause re-render when you change it.

```jsx
const gridRef = useRef(null);
// later: gridRef.current.api.forEachNode(...)
```

**PHP/jQuery analogy:** storing `var $grid = $('#grid')` in a closure — same DOM
handle every time, but React’s ref is the official way.

**Grid in this app:** `useGrid()` gives you a `gridRef` from context; the grid
component sets `gridRef.current` when AG Grid is ready.

---

### `useMemo`

**Purpose:** Cache the result of an **expensive or referential** computation until
dependencies change.

```jsx
const actions = useMemo(
  () => buildReplenishmentActions({ gridRef, confirm, createPurchaseOrder }),
  [gridRef, confirm, createPurchaseOrder],
);
```

Without `useMemo`, `buildReplenishmentActions(...)` would run every render and
return a **new array**. Downstream code might think “actions changed” every time.

```jsx
const gridProperties = useMemo(() => REPLENISHMENT_GRID_PROPERTIES, []);
```

Here the value is constant; `useMemo` keeps the **same object reference** forever.

**You can skip `useMemo` on your first page** until lint or bugs push you to add it.
Replenishment uses it to match team performance patterns.

---

### `useCallback`

**Purpose:** Cache a **function** reference between renders.

```jsx
const openCreate = useCallback(() => setDraftId(null), []);
```

Used when passing callbacks to memoized children or when a hook’s dependency list
needs a stable function. OpenCounts uses it for modal open/close/save handlers.

**First page:** optional; learn `useState` + `useMutation` first.

---

### `memo` (not a hook, but related)

```jsx
const ReplenishmentGrid = memo(function ReplenishmentGrid({ properties }) {
  return <GridApp properties={properties} />;
});
```

If parent re-renders but `properties` is the same reference, React **skips**
re-rendering `ReplenishmentGrid`. Helps heavy children like `GridApp`.

---

### `useContext` (via custom hooks)

You rarely write `useContext` yourself here. You call:

| Hook | Gets you |
| ---- | -------- |
| `useGrid()` | `gridRef`, `reloadGrid`, grid state actions |
| `usePageHeader(config)` | Sets title / actions in the header |
| `useConfirmDialog()` | `confirm({ title, ... })` → Promise boolean |

Under the hood: a **Provider** higher in the tree (`MountProviders`) holds values;
your page **consumes** them.

**PHP analogy:** Laravel’s `View::share('grid', $grid)` — but scoped to the React
tree under the mount, not the whole PHP app.

---

## 8. Context — shared data without prop drilling

### Problem

`gridRef` is created in `GridProvider`. `ReplenishmentList` needs it.
`buildReplenishmentActions` needs it. You do not want to pass props through ten layers.

### Solution

```text
GridProvider (holds gridRef)
  └── PageChrome
        └── ReplenishmentList  → useGrid() reads gridRef
```

### Important

Context updates **re-render consumers**. This repo splits grid **state** vs **actions**
in `GridContext` partly so not every consumer re-renders on every cell change. You
only need to know: **`useGrid()` is how your page talks to the grid.**

---

## 9. Suspense and loading UI

### Old jQuery style

```js
$("#list").html("Loading...");
$.get("/api/list", function (data) {
  render(data);
});
```

### This repo’s preferred style

Parent shows a **fallback** (skeleton). Child assumes data is ready.

```jsx
<Suspense fallback={<OpenCountsFallback />}>
  <OpenCountsList />
</Suspense>

function OpenCountsList() {
  const { data } = useSuspenseQuery(openStockCountsQuery());
  // no if (loading) here
}
```

While `useSuspenseQuery` is waiting, React **throws a promise** internally (conceptually),
walks up to the nearest `<Suspense>`, and shows `fallback`.

**Rule in this project:** do not put big `if (isLoading)` trees in list components;
use Suspense fallback instead. See `.cursor/rules/tanstack-query.mdc`.

---

## 10. TanStack Query from zero

TanStack Query (React Query) is a **client-side cache for server data**. It answers:

- When do I fetch?
- Where do I store the result?
- When is data “stale”?
- How do I refresh after save?
- How do I avoid duplicate in-flight requests?

You already have `QueryClientProvider` in `MountProviders` — you only use hooks inside
your page.

### Why not `useEffect` + `fetch`?

You can, but you repeat the same problems on every screen:

| Problem | Query helps |
| ------- | ----------- |
| Double fetch in StrictMode dev | Dedupes by `queryKey` |
| No cache — tab back refetches everything | Cache + `staleTime` |
| Loading/error flags everywhere | `useSuspenseQuery` + global error handling |
| Forgot to refresh list after save | `invalidateQueries` in `onSuccess` |

### Core vocabulary

| Term | Meaning |
| ---- | ------- |
| **query** | Read data (GET-like) |
| **mutation** | Write data (POST-like) |
| **queryKey** | Unique id for a piece of server data (array) |
| **queryFn** | Async function that returns the data |
| **mutationFn** | Async function that performs the write |
| **cache** | In-memory store of query results |
| **invalidate** | Mark cache stale → refetch on next need |

### queryKey example

```js
export const stockCountKeys = {
  open: () => ["stockCount", "open"],
};

export function openStockCountsQuery() {
  return {
    queryKey: stockCountKeys.open(),
    queryFn: stockCountApi.getOpenCounts,
  };
}
```

Think: **cache folder path**. Same key → same cached list.

This repo also uses special key prefixes (`cs::`, `cp::`) for session/page cache —
see comments in `src/lib/queryClient.ts`. Feature keys in `stockCount.queryKeys.js`
follow local conventions; copy the feature you clone.

### `useSuspenseQuery` (reads)

```jsx
const { data } = useSuspenseQuery(openStockCountsQuery());
```

- **Suspends** until `queryFn` resolves (parent must have `<Suspense>`).
- On error, can hit global `QueryCache` handlers (toasts).

### `useQuery` (reads, optional)

```jsx
const { data } = useQuery({
  ...someQuery(),
  enabled: Boolean(countId),
});
```

Use when fetch should **not** run yet (no id) or you are not using Suspense on
that branch.

### `useMutation` (writes)

Does **not** use `queryKey` the same way. You call it when the user acts.

```jsx
const { mutate: saveToPo, isPending } = useMutation({
  mutationFn: saveReplenishmentToPo,
  onSuccess: () => reloadGrid?.({ reason: "mutation" }),
});
```

Lifecycle:

```text
idle → mutate(rows) called → pending → success or error → idle
```

- **`mutate(rows)`** — fire and forget; `onSuccess` runs if OK.
- **`mutateAsync(rows)`** — returns a Promise; use in `async` form submit handlers.

Replenishment passes `mutate` into actions as `createPurchaseOrder`.

### Factory pattern: `ReplenishmentMutations`

```js
export function ReplenishmentMutations({ refresh } = {}) {
  return {
    createPurchaseOrder: {
      mutationFn: saveReplenishmentToPo,
      onSuccess: refresh,
    },
  };
}
```

Then:

```jsx
const { mutate: createPurchaseOrder } = useMutation(
  mutations.createPurchaseOrder,
);
```

Keeps `ReplenishmentList.jsx` short and tests easy.

### After save: two refresh styles in this app

| Page type | Refresh |
| --------- | ------- |
| Grid (replenishment) | `reloadGrid({ reason: "mutation" })` |
| Custom list (open counts) | `invalidateStockCount()` or `queryClient.invalidateQueries({ queryKey: ... })` |

Both mean: **show fresh server data**. Grid reload talks to AG Grid host; Query
invalidation refetches JSON lists.

### Errors and toasts

`src/lib/queryClient.ts` configures global handlers on `QueryCache` and `MutationCache`.
Many failures auto-toast. **Validation before save** (missing supplier, empty rows)
still belongs in `*.service.js` + `toast.error` in actions — replenishment does that
in `try/catch` around `collectReplenishmentPoRows`.

### Mental diagram

```text
                    ┌─────────────────┐
                    │  QueryClient    │
                    │  (cache)        │
                    └────────┬────────┘
                             │
     useSuspenseQuery ───────┼─────── useMutation
           │                 │              │
           ▼                 ▼              ▼
      queryFn GET       same cache     mutationFn POST
           │                            onSuccess → invalidate / reloadGrid
           ▼
      component shows data
```

---

## 11. Read ReplenishmentList line by line

File: `src/modules/inventory/replenishment/ReplenishmentList.jsx`

```jsx
import { memo, useMemo } from "react";
import { useMutation } from "@tanstack/react-query";
```

- React tools + TanStack write hook.

```jsx
import { usePageHeader } from "@/shared/context/PageChromeContext";
import { useConfirmDialog } from "@/shared/context/ConfirmDialogContext";
import { useGrid } from "@/shared/context/GridContext";
```

- Custom hooks = context for header, confirm dialog, grid handle.

```jsx
const ReplenishmentGrid = memo(function ReplenishmentGrid({ properties }) {
  return (
    <GridApp
      autoLoad={false}
      filterPlacement="header"
      properties={properties}
    />
  );
});
```

- Small wrapper so `GridApp` does not re-render unless `properties` changes.

```jsx
const ReplenishmentList = () => {
  const { confirm } = useConfirmDialog();
  const { gridRef, reloadGrid } = useGrid();
```

- `confirm` — async Yes/No.
- `gridRef` — access rows via AG Grid API in service/actions.
- `reloadGrid` — after successful save.

```jsx
  const gridProperties = useMemo(() => REPLENISHMENT_GRID_PROPERTIES, []);
```

- Stable prop object for grid id 137, virtual, editable column.

```jsx
  const mutations = ReplenishmentMutations({
    refresh: () => reloadGrid?.({ reason: "mutation" }),
  });
  const { mutate: createPurchaseOrder } = useMutation(
    mutations.createPurchaseOrder,
  );
```

- Configure POST + “on success, reload grid”.
- Rename `mutate` to readable `createPurchaseOrder` for actions file.

```jsx
  const actions = useMemo(
    () =>
      buildReplenishmentActions({
        gridRef,
        confirm,
        createPurchaseOrder,
        createSupplierRequisition,
      }),
    [confirm, createPurchaseOrder, createSupplierRequisition, gridRef],
  );
```

- Build toolbar once per meaningful change (not every parent re-render).

```jsx
  usePageHeader({
    title: "Replenishment",
    actions,
  });
```

- Side effect: register header for this page (implementation uses effect + context).

```jsx
  return (
    <div className="...">
      <ReplenishmentGrid properties={gridProperties} />
    </div>
  );
};
```

- Body is only the grid; chrome is outside via `usePageHeader`.

**What this file does NOT do:** loop rows, POST to server, or validate — that is
`replenishment.service.js`, `replenishment.actions.js`, `replenishment.api.js`.

---

## 12. Practice questions

Answer in your own words before checking the doc:

1. What is the difference between **props** and **state**?
2. Why is `gridRef` not stored in `useState`?
3. What runs first when user clicks “Create Purchase Order” — `mutationFn` or
   `collectReplenishmentPoRows`?
4. What is the job of `queryKey`?
5. Why does OpenCounts wrap the list in `<Suspense>`?
6. Where should “Supplier is required” validation live — JSX, actions, service, or api?

**Answers (short):**

1. Props come from parent and should not be mutated; state is owned by the component and triggers re-render when updated.
2. Changing a ref does not need to re-render the whole page; the grid API is imperative.
3. `collectReplenishmentPoRows` in the action’s `onClick`; `mutationFn` only after confirm.
4. Identifies cached server data; invalidation targets the same key.
5. Because `useSuspenseQuery` suspends until data loads; fallback shows skeleton.
6. **Service** (throw Error); **actions** catch and `toast.error`.

---

## Suggested learning order (fill gaps over a week)

| Day | Focus |
| --- | ----- |
| 1 | Sections 1–4 + build grid-only page (no mutations) |
| 2 | Section 7 (`useState`, `useRef`, `useGrid`) + one button + `toast` |
| 3 | Section 10 mutations + replenishment save path |
| 4 | Section 9 + read `OpenCounts.jsx` |
| 5 | Section 7 (`useMemo`, `memo`) — understand replenishment as written |
| 6 | `useEffect` only where you see it in cloned feature |
| 7 | Practice questions + write spec for your own screen |

---

## See also

- [creating-a-page-by-hand.md](./creating-a-page-by-hand.md) — file checklist and Path A / B
- [writing-tests-junior.md](./writing-tests-junior.md) — Vitest, mocks, replenishment test walkthrough
- [typescript-react-junior.md](./typescript-react-junior.md) — enough TS for new React pages
- [modals-and-forms-junior.md](./modals-and-forms-junior.md) — Modal, DynamicForm, RHF
- [page-chrome.md](./page-chrome.md) — header actions, filters, pills
- [grid-pills.md](./grid-pills.md) — grid view pills in the header
