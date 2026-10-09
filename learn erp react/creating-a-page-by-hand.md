# Creating a React page by hand (junior guide)

This guide is for developers coming from a **PHP monolith + jQuery** background. It
explains how to add a new screen in `wizard_erp_ui` **without** needing to
understand every line inside `GridApp` or `GridContext`.

You will learn the **full cycle**:

1. ERP host page → React **mount**
2. **Page component** (title + body)
3. **Data** (grid loaded by id, or `useQuery` / `useSuspenseQuery`)
4. **Mutations** (save / POST via `useMutation`)
5. **Refresh** after success

Use **Replenishment** as the reference grid page and **Open Stock Counts** as the
reference “fetch JSON + cards + modal save” page.

**New to React?** Read [react-hooks-and-tanstack-junior.md](./react-hooks-and-tanstack-junior.md)
first (or in parallel). It explains components, hooks, context, Suspense, and
TanStack Query in depth with PHP/jQuery analogies. This doc assumes you are
comfortable with those ideas when wiring files.

**Tests:** When you add `service` / `api` / `actions`, follow
[writing-tests-junior.md](./writing-tests-junior.md) (replenishment `__tests__`
as the worked example).

**TypeScript:** New features should use `.ts` / `.tsx`. See
[typescript-react-junior.md](./typescript-react-junior.md) (props, hooks, Query,
grid rows — not a full TS course).

**Modals / forms:** Create & edit dialogs use `Modal` + `DynamicForm` (RHF inside).
See [modals-and-forms-junior.md](./modals-and-forms-junior.md).

---

## Table of contents

1. [Mental model: PHP vs this app](#1-mental-model-php-vs-this-app)
2. [What happens when the page loads](#2-what-happens-when-the-page-loads)
3. [Choose your page type](#3-choose-your-page-type)
4. [Path A — Grid page (most inventory lists)](#4-path-a--grid-page-most-inventory-lists)
5. [Path B — Custom UI + fetch + mutation](#5-path-b--custom-ui--fetch--mutation)
6. [TanStack Query cheat sheet](#6-tanstack-query-cheat-sheet)
7. [Checklist before you say “done”](#7-checklist-before-you-say-done)
8. [How to verify by hand](#8-how-to-verify-by-hand)
9. [Common mistakes](#9-common-mistakes)
10. [Using AI without losing understanding](#10-using-ai-without-losing-understanding)

---

## 1. Mental model: PHP vs this app

| Old (PHP + jQuery) | New (this repo) |
| ------------------ | ---------------- |
| One `.php` view with HTML + inline script | PHP still renders a **host div**; React renders into it |
| `$('#save').click(...)` | `onClick` in `*.actions.js` or `usePageHeader({ actions })` |
| `$.post('/save', data)` | `*.api.js` + `useMutation` |
| DataTables / server-rendered grid | `<GridApp properties={{ id: N }} />` (grid loads itself) |
| `location.reload()` | `reloadGrid({ reason: "mutation" })` or `queryClient.invalidateQueries` |

React here is mostly **glue**. Business rules that touch grid rows belong in
`*.service.js`, not inside JSX.

---

## 2. What happens when the page loads

```text
ERP HTML
  └── <div id="replenishment-list"></div>
        └── bootstrap.js finds registry entry (selector matches id)
              └── MountProviders (QueryClient, Modal, Grid, PageChrome, …)
                    └── <ReplenishmentList />
                          ├── usePageHeader({ title, actions })  → grey header bar
                          └── <GridApp properties={...} />       → AG Grid (grid id 137)
```

Important files (you will touch these when adding a page):

| File | You change it when… |
| ---- | ------------------- |
| `src/mounts/registry.jsx` | Wiring React to a new `#div-id` on the ERP page |
| `src/modules/<area>/<feature>/*.jsx` | Your page UI |
| `src/modules/<area>/<feature>/*.routes.js` | Menu links and ERP paths (optional but usual) |
| `src/modules/<area>/index.routes.js` | Merging route keys into inventory (or other area) |

You do **not** wrap pages in `QueryClientProvider` yourself — `MountProviders` does
that for every normal mount.

See also: [page-chrome.md](./page-chrome.md) for header buttons, filters, pills.

---

## 3. Choose your page type

| Type | When to use | Example in repo |
| ---- | ----------- | ---------------- |
| **A. Grid page** | ERP already has a grid definition (numeric `id`). User edits rows and you POST row payloads. | `replenishment/ReplenishmentList.jsx` |
| **B. Custom UI** | You render cards, forms, or tables yourself; you fetch JSON from explicit API functions. | `stockCount/OpenCounts.jsx` |
| **A + B hybrid** | Grid + extra header fetches (supplier dropdown, etc.) | `supplierPrices/SupplierPricesBySupplierList.jsx` |

If you are unsure: ask for the **grid id** from the backend team. If there is a
grid id, start with **Path A**.

---

## 4. Path A — Grid page (most inventory lists)

Goal: same shape as **Replenishment** — title, optional toolbar actions, grid,
save mutations.

### Step 0 — Gather facts (write this down before coding)

Fill in a short spec:

```text
ERP path:        inventory/myFeature/index
Mount div id:    my-feature-list
Grid id:         137
Editable cols:   qty_to_order
Toolbar:         Fill Min, Fill Max, Save to PO
POST routes:     inventory/myFeature/saveToPo
Confirm save?     yes
Reload after save? yes (grid)
```

### Step 1 — Create the feature folder

Under `src/modules/<area>/<feature>/` (example: `inventory/myFeature/`):

```text
myFeature/
├── constant.js              # grid id, field names
├── myFeature.routes.js      # path + gridId for menus
├── MyFeatureList.jsx        # page entry (default export)
├── myFeature.actions.js     # header Actions menu (optional)
├── myFeature.service.js     # read/write grid rows (optional)
├── myFeature.api.js         # HTTP POST/GET (optional)
└── myFeature.mutations.js   # useMutation config (optional)
```

**Naming (repo convention):** the folder gives context; files are short role names
(`List.jsx`, `actions.js`, not `MyFeatureActionsMenu.jsx`).

### Step 2 — `constant.js`

This is like hard-coding grid settings you used to pass from PHP.

```js
/** My feature list (`inventory/myFeature/index`, grid 999). */
export const QTY_FIELD = "qty_to_order";

export const MY_FEATURE_GRID_PROPERTIES = {
  id: 999, // ← real grid id from ERP
  virtual: true, // common for large editable grids
  editableColumns: [QTY_FIELD],
};
```

- `id` — ERP grid definition number.
- `virtual: true` — virtual scrolling; follow replenishment for editable inventory grids.
- `editableColumns` — only these columns are editable in React; omit or `[]` for read-only.

### Step 3 — `MyFeatureList.jsx` (minimal grid-only first)

**Rule:** get the grid on screen before adding save logic.

```jsx
import { memo, useMemo } from "react";
import { usePageHeader } from "@/shared/context/PageChromeContext";
import GridApp from "@/shared/grid/components/GridApp";
import { MY_FEATURE_GRID_PROPERTIES } from "./constant";

const MyFeatureGrid = memo(function MyFeatureGrid({ properties }) {
  return (
    <GridApp
      autoLoad={false}
      filterPlacement="header"
      properties={properties}
    />
  );
});

const MyFeatureList = () => {
  const gridProperties = useMemo(() => MY_FEATURE_GRID_PROPERTIES, []);

  usePageHeader({
    title: "My Feature",
  });

  return (
    <div className="flex min-h-0 flex-col overflow-hidden bg-wz-surface !font-ibm-plex-sans">
      <MyFeatureGrid properties={gridProperties} />
    </div>
  );
};

export default MyFeatureList;
```

**What each piece means:**

| Piece | Purpose |
| ----- | ------- |
| `usePageHeader` | Sets title and toolbar on the grey bar (not inside your div). |
| `useMemo(() => PROPS, [])` | Keeps the same `properties` object reference across renders. |
| `memo(MyFeatureGrid)` | Avoids re-rendering the heavy grid when parent re-renders for unrelated reasons. |
| `filterPlacement="header"` | Puts filters in the page header (see page-chrome.md). |
| `autoLoad={false}` | Replenishment pattern; grid load timing is controlled by `GridApp` / host — match sibling pages in the same area. |

**Checkpoint:** register mount (Step 7) and open the ERP URL. You should see title + grid.

### Step 4 — `myFeature.routes.js` + menu merge

```js
/** @typedef {import("@/routes/types").RouteDef} RouteDef */

/** @type {Record<string, RouteDef>} */
export const myFeatureRoutes = {
  myFeatureGrid: {
    label: "My Feature",
    path: "inventory/myFeature/index",
    gridId: 999,
    option: "some_host_option_if_needed",
  },
};
```

In `src/modules/inventory/index.routes.js` (or the right area file):

```js
import { myFeatureRoutes } from "./myFeature/myFeature.routes";

export const inventoryRoutes = {
  // ...existing spreads
  ...myFeatureRoutes,
};
```

Add the route **key** (`myFeatureGrid`) to the appropriate `links: [...]` array in
`inventoryMenuModules` if the sidebar should show it.

### Step 5 — `myFeature.service.js` (grid logic, no React)

Put anything that loops rows or maps a row → API payload here.

**Patterns to copy from replenishment:**

- `visitBodyRows(gridRef, callback)` — `gridRef.current.api.forEachNode`
- `fillQtyToOrderFromField` — bulk update a column
- `collectReplenishmentPoRows` — validate + build array for POST
- `toReplenishmentPoRow` — one grid row → one JSON object for PHP

Why no React? You can unit test this file with a fake `gridRef` (see
`replenishment/__tests__/replenishment.actions.test.js`).

**`gridRef` explained:** Think of it as jQuery’s `$('#grid')` handle, stored in
React context via `useGrid()`. The grid component assigns `gridRef.current` when
AG Grid is ready.

### Step 6 — `myFeature.actions.js` (toolbar clicks)

Returns a plain array for `usePageHeader({ actions: [...] })`:

```js
import { toast } from "sonner";
import { collectMyFeatureRows, fillQtyFromField } from "./myFeature.service";

export function buildMyFeatureActions({
  gridRef,
  confirm,
  saveToPo,
}) {
  return [
    {
      label: "Fill to Min Qty",
      onClick: () => fillQtyFromField(gridRef, "min_qty_sum"),
    },
    {
      label: "Save to PO",
      onClick: async () => {
        let rows;
        try {
          rows = collectMyFeatureRows(gridRef);
        } catch (err) {
          toast.error(err?.message || "Save failed");
          return;
        }

        const accepted = confirm
          ? await confirm({
              title: "Create purchase order?",
              description: "Rows with qty > 0 will be included.",
              confirmLabel: "Create",
              cancelLabel: "Cancel",
            })
          : true;
        if (!accepted) return;

        saveToPo?.(rows);
      },
    },
  ];
}
```

**Tricks:**

- `async () => { ... await confirm(...) }` — wait for user Yes/No.
- `saveToPo?.(rows)` — only call if the list passed a save function (tests may omit it).
- `try/catch` around **collect** — validation errors become toasts, not uncaught exceptions.

### Step 7 — `myFeature.api.js` + `myFeature.mutations.js`

**API** — only HTTP (like a thin PHP controller client):

```js
import { requestJson } from "@/lib/apiClient";

const FORM_HEADERS = {
  "Content-Type": "application/x-www-form-urlencoded; charset=UTF-8",
};

async function postRows(route, rows) {
  const body = new URLSearchParams();
  body.set("rows", JSON.stringify(rows));
  return requestJson(route, {
    method: "POST",
    headers: FORM_HEADERS,
    body,
  });
}

export function saveMyFeatureToPo(rows) {
  return postRows("inventory/myFeature/saveToPo", rows);
}
```

**Mutations** — options object for `useMutation` (not a hook itself):

```js
import { saveMyFeatureToPo } from "./myFeature.api";

export function MyFeatureMutations({ refresh } = {}) {
  return {
    saveToPo: {
      mutationFn: saveMyFeatureToPo,
      onSuccess: refresh,
    },
  };
}
```

### Step 8 — Wire fetch/save in `MyFeatureList.jsx`

```jsx
import { useMutation } from "@tanstack/react-query";
import { useConfirmDialog } from "@/shared/context/ConfirmDialogContext";
import { useGrid } from "@/shared/context/GridContext";
import { buildMyFeatureActions } from "./myFeature.actions";
import { MyFeatureMutations } from "./myFeature.mutations";

const MyFeatureList = () => {
  const { confirm } = useConfirmDialog();
  const { gridRef, reloadGrid } = useGrid();
  const gridProperties = useMemo(() => MY_FEATURE_GRID_PROPERTIES, []);

  const mutations = MyFeatureMutations({
    refresh: () => reloadGrid?.({ reason: "mutation" }),
  });
  const { mutate: saveToPo } = useMutation(mutations.saveToPo);

  const actions = useMemo(
    () =>
      buildMyFeatureActions({
        gridRef,
        confirm,
        saveToPo,
      }),
    [confirm, gridRef, saveToPo],
  );

  usePageHeader({
    title: "My Feature",
    actions,
  });

  // ... same return as before
};
```

**Cycle on Save:**

```text
User → actions onClick → service collects rows → confirm dialog
  → mutate(rows) → api POST → onSuccess → reloadGrid
```

You do not call `reloadGrid` inside `actions.js` — keep refresh in `mutations`
`onSuccess` so every save path behaves the same.

### Step 9 — Register the mount in `registry.jsx`

1. Import your list component at the top of `src/mounts/registry.jsx`.
2. Add an entry:

```js
{
  id: "my-feature-list",
  selector: "#my-feature-list",
  remountKey: "remountMyFeatureList",
  render: () => <MyFeatureList />,
},
```

3. ERP view must include: `<div id="my-feature-list"></div>` (backend / monolith).

`remountKey` exposes `window.remountMyFeatureList()` for host scripts to force a
fresh React tree — match naming of sibling mounts.

### Step 10 — Format, lint, tests

```bash
npm run format
npm run lint
```

Add or extend tests for `*.service.js` and `*.actions.js` (pattern:
`replenishment/__tests__/`). You do not need component tests for the first page
unless asked.

---

## 5. Path B — Custom UI + fetch + mutation

Use when there is **no** grid id — you fetch JSON and render your own UI.

Reference: `src/modules/inventory/stockCount/OpenCounts.jsx`.

### File layout (typical)

```text
myFeature/
├── MyFeaturePage.jsx       # entry: header, Suspense, modal state
├── components/             # presentational pieces
├── api/
│   ├── myFeature.api.js    # requestJson wrappers
│   ├── myFeature.queries.js # queryKey + queryFn objects
│   └── myFeature.queryKeys.js
└── myFeature.service.js    # URLs, host navigation, non-HTTP helpers
```

### Step 1 — API function

```js
// api/myFeature.api.js
import { requestJson } from "@/lib/apiClient";

export const myFeatureApi = {
  getList: () => requestJson("inventory/myFeature/getList"),
  save: (payload) =>
    requestJson("inventory/myFeature/save", {
      method: "POST",
      body: JSON.stringify(payload),
    }),
};
```

Match content-type and body shape to what PHP expects (form-encoded vs JSON —
replenishment uses form + `rows` JSON string).

### Step 2 — Query keys + query factory

```js
// api/myFeature.queryKeys.js
export const myFeatureKeys = {
  all: ["myFeature"],
  list: () => [...myFeatureKeys.all, "list"],
};

// api/myFeature.queries.js
import { myFeatureApi } from "./myFeature.api";
import { myFeatureKeys } from "./myFeature.queryKeys";

export function myFeatureListQuery() {
  return {
    queryKey: myFeatureKeys.list(),
    queryFn: myFeatureApi.getList,
  };
}
```

**Why query keys?** TanStack Query caches by key. After a save, you
`invalidateQueries({ queryKey: myFeatureKeys.list() })` so the list refetches.

### Step 3 — Read data with `useSuspenseQuery` + `Suspense`

Repo rule: prefer **suspense** for list UIs — skeleton lives in `fallback`, not
`if (loading)` inside the list.

```jsx
import { Suspense } from "react";
import { useSuspenseQuery } from "@tanstack/react-query";
import { myFeatureListQuery } from "./api/myFeature.queries";

function MyFeatureListInner() {
  const { data } = useSuspenseQuery(myFeatureListQuery());
  const items = data ?? [];
  return (
    <ul>
      {items.map((row) => (
        <li key={row.id}>{row.name}</li>
      ))}
    </ul>
  );
}

export default function MyFeaturePage() {
  usePageHeader({ title: "My Feature" });

  return (
    <div className="bg-wz-surface !font-ibm-plex-sans">
      <Suspense fallback={<p>Loading…</p>}>
        <MyFeatureListInner />
      </Suspense>
    </div>
  );
}
```

For **optional** fetches (only when an id exists), `useQuery({ enabled: !!id })`
is fine — see `CountDifferenceList.jsx`.

### Step 4 — Save with `useMutation`

```jsx
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { myFeatureApi } from "./api/myFeature.api";
import { myFeatureKeys } from "./api/myFeature.queryKeys";

function MyFeaturePage() {
  const queryClient = useQueryClient();

  const saveMutation = useMutation({
    mutationFn: myFeatureApi.save,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: myFeatureKeys.list() });
    },
  });

  const handleSave = (values) => {
    saveMutation.mutate(values);
  };

  // pass handleSave to a form or modal
}
```

OpenCounts also uses `invalidateStockCount()` from a small cache helper — follow
the feature you copy if it already has `*.cache.js`.

### Step 5 — Local UI state

Use `useState` for things that are **only UI**:

- modal open/closed
- which row is being edited
- draft form values before submit

Do **not** duplicate server list data in state if Query already holds it — invalidate
and refetch instead.

---

## 6. TanStack Query cheat sheet

Full explanations: [react-hooks-and-tanstack-junior.md §10](./react-hooks-and-tanstack-junior.md#10-tanstack-query-from-zero).

| Hook | Use for |
| ---- | ------- |
| `useSuspenseQuery(options)` | Load data for display; parent has `<Suspense fallback={…}>`. |
| `useQuery(options)` | Load when `enabled: false` or partial/optional data. |
| `useMutation(options)` | POST/PUT/delete; call `mutate(data)` from a button. |
| `useQueryClient()` | `invalidateQueries` after save to refresh lists. |

| Option | Meaning |
| ------ | ------- |
| `queryKey` | Cache identity — must be stable and unique per dataset. |
| `queryFn` | Async function that returns data (usually calls `*.api.js`). |
| `mutationFn` | Async function that performs the write. |
| `onSuccess` | Run after save — reload grid or invalidate queries. |

Global errors often surface via `queryClient` toasts — local validation still uses
`toast.error` in actions (replenishment pattern).

---

## 7. Checklist before you say “done”

- [ ] ERP page has the correct mount `id` matching `registry.jsx` `selector`
- [ ] `MY_FEATURE_GRID_PROPERTIES.id` matches ERP grid (Path A)
- [ ] `usePageHeader` sets `title` (and `actions` if needed)
- [ ] Path A: `useGrid()` provides `gridRef` to actions/service
- [ ] Save payload matches PHP (field names, `rows` JSON encoding)
- [ ] Success path refreshes data (`reloadGrid` or `invalidateQueries`)
- [ ] `npm run format` and `npm run lint` pass
- [ ] Service/actions tests added or updated for non-trivial logic

---

## 8. How to verify by hand

1. **Mount:** Open ERP URL — blank area means wrong `#id` or missing registry entry.
2. **Grid:** Columns appear — wrong `id` in `constant.js` or host option blocking grid.
3. **Actions:** Button in header — missing `actions` in `usePageHeader` or empty `build*Actions`.
4. **Save:** Network tab — POST hits correct route; response 200; grid reloads or list updates.
5. **Validation:** Row with missing supplier (or your rule) — toast, no POST.

Trace one click in DevTools: **Sources** → open `myFeature.actions.js` → set
breakpoint on `onClick`.

---

## 9. Common mistakes

| Mistake | Fix |
| ------- | --- |
| Grid logic inside JSX body | Move to `*.service.js`; call from `onClick` only |
| Forgot `useGrid()` | Actions cannot read rows |
| `reloadGrid` in actions | Put refresh in mutation `onSuccess` |
| New object every render for `properties` | `useMemo(() => CONST, [])` |
| Skipping confirm for destructive saves | Use `useConfirmDialog()` like replenishment |
| Wrong POST body shape | Copy an existing endpoint in the same module area |
| `useSuspenseQuery` without `Suspense` parent | Wrap list child in `<Suspense fallback={…}>` |

---

## 10. Using AI without losing understanding

Before merging AI output, require:

1. **File list** with one-line responsibility each.
2. **Flow diagram** from click to POST to refresh.
3. **Row mapping** — one example grid row → one POST object.

You should be able to answer without looking at code:

- What triggers the mutation?
- Where is data read from?
- What runs on success?

If you cannot, ask for explanation — not more code.

---

## Quick reference — Replenishment file map

| File | Role |
| ---- | ---- |
| `ReplenishmentList.jsx` | Hooks: header, grid, mutations |
| `replenishment.actions.js` | Toolbar menu |
| `replenishment.service.js` | Grid row read/write + map to PO row |
| `replenishment.api.js` | HTTP POST |
| `replenishment.mutations.js` | `mutationFn` + `onSuccess: refresh` |
| `constant.js` | Grid 137, editable column |
| `replenishment.routes.js` | Menu + ERP path |
| `registry.jsx` | `#replenishment-list` → component |

---

## Suggested first exercise

1. Copy only **Steps 1–3** and **Step 9** for a **read-only** grid you already have in ERP.
2. Confirm it loads.
3. Add **one** action that calls `toast.success("works")`.
4. Add **one** service function that updates a column (copy `fillQtyToOrderFromField`).
5. Add **one** save endpoint with mutation + reload.

After that, you are not memorizing React — you are repeating a **recipe** this
codebase already uses everywhere.
