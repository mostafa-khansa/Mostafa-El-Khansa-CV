# Writing tests (junior guide)

This guide teaches you how tests work in **wizard_erp_ui** so you can add them
when you build a page — not copy-paste without understanding.

Read together with:

- [creating-a-page-by-hand.md](./creating-a-page-by-hand.md) — which files to create
- [react-hooks-and-tanstack-junior.md](./react-hooks-and-tanstack-junior.md) — React basics
- [typescript-react-junior.md](./typescript-react-junior.md) — typing services, props, and tests

**Golden reference in the repo:** `src/modules/inventory/replenishment/__tests__/`
(three files that mirror `service`, `api`, `actions`, `mutations`).

---

## Table of contents

1. [Why we test (PHP developer view)](#1-why-we-test-php-developer-view)
2. [Tools in this project](#2-tools-in-this-project)
3. [Where tests live](#3-where-tests-live)
4. [What to test and what to skip](#4-what-to-test-and-what-to-skip)
5. [Running tests](#5-running-tests)
6. [Test file anatomy](#6-test-file-anatomy)
7. [The AAA pattern](#7-the-aaa-pattern)
8. [Layer 1 — Service / pure logic](#8-layer-1--service--pure-logic)
9. [Layer 2 — API (HTTP)](#9-layer-2--api-http)
10. [Layer 3 — Actions (toolbar clicks)](#10-layer-3--actions-toolbar-clicks)
11. [Layer 4 — Mutations config](#11-layer-4--mutations-config)
12. [Layer 5 — React components (when needed)](#12-layer-5--react-components-when-needed)
13. [Mocking explained](#13-mocking-explained)
14. [Matchers you will use daily](#14-matchers-you-will-use-daily)
15. [Write your first test — step by step](#15-write-your-first-test--step-by-step)
16. [When a test fails](#16-when-a-test-fails)
17. [Checklist for a new feature](#17-checklist-for-a-new-feature)

---

## 1. Why we test (PHP developer view)

In PHP you might use **PHPUnit**:

```php
$this->assertEquals(5, $service->mapQty($row));
```

Front-end tests do the same job for **JavaScript**:

- **Document behavior** — “when qty is 0, save throws Nothing to save.”
- **Catch regressions** — refactoring `pickMatched` does not break PO mapping.
- **Let you refactor safely** — especially when AI or you rename fields.

We do **not** try to run a real browser against the live ERP in unit tests. We
**fake** the grid and HTTP and test **your logic**.

---

## 2. Tools in this project

| Tool | Role |
| ---- | ---- |
| **Vitest** | Test runner (like PHPUnit). `describe`, `it`, `expect`, `vi`. |
| **jsdom** | Fake browser DOM in Node (for component tests). |
| **Testing Library** | Render React components and query like a user (`getByRole`, `getByText`). |
| **@testing-library/jest-dom** | Extra matchers (`toBeInTheDocument()`). |

Config: `vitest.config.js` (`globals: true` — you *can* omit imports, but this
repo **prefers explicit** `import { describe, it, expect, vi } from "vitest"`).

Setup for every test file: `src/test/setup.js` (auto cleanup after each test).

---

## 3. Where tests live

**Colocate** tests in a sibling `__tests__` folder:

```text
replenishment/
├── replenishment.service.js
├── replenishment.api.js
├── replenishment.actions.js
└── __tests__/
    ├── replenishment.actions.test.js   # often includes service + actions
    ├── replenishment.api.test.js
    └── replenishment.mutations.test.js
```

Naming: source `Foo.jsx` → `__tests__/Foo.test.jsx`.

---

## 4. What to test and what to skip

Team default (see `.cursor/rules/testing.mdc`):

| **Cover** | **Usually skip** |
| --------- | ---------------- |
| `*.service.js` — mapping, validation, grid loops | Thin list pages that only wire hooks |
| `*.api.js` — URL, body shape, errors | “Does `GridApp` render?” (too heavy) |
| `*.actions.js` — confirm, toast, mutate called | CSS class lists |
| `*.mutations.js` — `mutationFn` + `onSuccess` wiring | Full AG Grid integration |
| **Shared** UI (`Layout`, cross-feature components) | One-off cards/wrappers |

For your **first inventory page**, plan tests like replenishment:

1. Service + actions (one file is OK)
2. API
3. Mutations factory

Add a component test for `MyFeatureList.jsx` only if you ask for it or the page
has real UI logic (not just `<GridApp />`).

---

## 5. Running tests

| Goal | Command |
| ---- | ------- |
| Full suite | `npm test` |
| One file | `npx vitest run src/modules/inventory/replenishment/__tests__/replenishment.api.test.js` |
| One test by name | `npx vitest run -t "POSTs stringified rows"` |
| Watch while coding | `npm run test:watch` |

After writing tests: `npm run format` on the new file.

**Workflow:** write one `it(...)` → run that file → green → next case.

---

## 6. Test file anatomy

```js
import { afterEach, describe, expect, it, vi } from "vitest";
import { myFunction } from "../myFeature.service";

describe("myFunction", () => {
  it("does the happy path", () => {
    const result = myFunction({ id: 1 });
    expect(result).toBe(42);
  });

  it("throws when input is invalid", () => {
    expect(() => myFunction(null)).toThrow("required");
  });
});
```

| Piece | Meaning |
| ----- | ------- |
| `describe("name", () => { ... })` | Group related tests (like a PHPUnit test class). |
| `it("human readable behavior", () => { ... })` | One example / scenario. |
| `expect(actual).toEqual(expected)` | Assertion. |
| `vi.fn()` | Fake function; records calls. |
| `vi.mock("module")` | Replace imports with fakes. |
| `afterEach(() => { ... })` | Reset mocks between tests. |

Test names should read as **behavior**: `"throws when no rows have quantity to order"`,
not `"test collectRows"`.

---

## 7. The AAA pattern

Structure each test mentally as:

1. **Arrange** — set up inputs, fakes, `gridRef`, mocks.
2. **Act** — call the function or `await onClick()`.
3. **Assert** — `expect(...)` on return value, thrown error, or mock calls.

Example from replenishment (create PO):

```js
// Arrange
const createPurchaseOrder = vi.fn();
const [, , , createPo] = buildReplenishmentActions({
  gridRef: gridWithRows([poRow]),
  createPurchaseOrder,
});

// Act
await createPo.onClick();

// Assert
expect(createPurchaseOrder).toHaveBeenCalledTimes(1);
expect(createPurchaseOrder.mock.calls[0][0][0]).toMatchObject({
  product_id: 10,
  reorder_qty: 5,
});
```

---

## 8. Layer 1 — Service / pure logic

**Best place to start.** No React, no HTTP — call the function directly.

### Example: `toReplenishmentPoRow`

Maps one grid row → one POST object. Tests use **fixture rows** that look like
real cube data:

```js
function cubeReplenishmentRow(overrides = {}) {
  return {
    "g_aux_supplier_product.aux": 242,
    "current_inventory.product_id": 181,
    "current_inventory.qty_to_order": 5,
    ...overrides,
  };
}

it("maps grid cube ids to host payload fields", () => {
  expect(
    toReplenishmentPoRow(
      cubeReplenishmentRow({ "current_inventory.qty_to_order": 5 }),
    ),
  ).toEqual({
    supplier_product: 242,
    product_id: 181,
    meas: 51,
    // ...
    reorder_qty: 5,
  });
});
```

**What you are learning:**

- One `it` per **rule** (aliases, defaults, null handling).
- Use **realistic keys** from the grid (`current_inventory.product_id`), not only `product_id`.
- `toMatchObject` when you only care about part of the result.

### Example: functions that need `gridRef`

AG Grid is **not** mounted. You fake a minimal API:

```js
function gridWithRows(rows, extraApi = {}) {
  return {
    current: {
      api: {
        forEachNode: (visit) =>
          rows.forEach((data, i) => visit({ data, rowIndex: i })),
        ...extraApi,
      },
    },
  };
}
```

Then:

```js
const rows = [{ qty_to_order: 0, min_qty_sum: 5 }];
fillQtyToOrderFromField(gridWithRows(rows, { refreshCells: vi.fn() }), "min_qty_sum");
expect(rows[0].qty_to_order).toBe(5);
```

**PHP analogy:** inject a fake database connection that returns fixed rows —
same idea as `gridWithRows`.

### Edge cases worth testing for grid features

Copy replenishment’s checklist:

- Happy path with qty > 0
- Skip qty === 0
- Throw `"Nothing to save"`
- Throw validation (`Supplier is required for row : N`)
- Throw `"Grid is not ready"` when `api` missing
- Skip **pinned** / **group** rows (custom `forEachNode` in test)

---

## 9. Layer 2 — API (HTTP)

`*.api.js` should only talk to `requestJson` / HTTP client. Tests **never** hit
the real server.

Use helpers from `src/test/httpMock.js`:

```js
import {
  createHttpSpy,
  requestBodyParams,
  restoreHttpMock,
} from "@/test/httpMock";
import { saveReplenishmentToPo } from "../replenishment.api";

describe("saveReplenishmentToPo", () => {
  afterEach(() => {
    restoreHttpMock();
    vi.unstubAllGlobals();
  });

  it("POSTs stringified rows to the host action", async () => {
    const spy = createHttpSpy({ success: true, documents: [11] });
    const rows = [{ product_id: 99, reorder_qty: 5, meas: 3 }];

    await expect(saveReplenishmentToPo(rows)).resolves.toEqual({
      success: true,
      documents: [11],
      message: "Data saved successfully",
    });

    expect(spy.mock.calls[0][0].url).toContain(
      "inventory/replenishment/saveReplenishmentToPo",
    );
    expect(requestBodyParams(spy.mock.calls[0][0].data).get("rows")).toBe(
      JSON.stringify(rows),
    );
  });
});
```

**What you assert:**

1. Return value (including default `message` merged in api layer).
2. Correct **route** in URL.
3. **Body** shape (`rows` = `JSON.stringify(payload)` for replenishment).

**Error paths:**

```js
await expect(saveReplenishmentToPo([])).rejects.toThrow("Nothing to save");
```

Always `restoreHttpMock()` in `afterEach` so the next test does not see the wrong adapter.

---

## 10. Layer 3 — Actions (toolbar clicks)

`buildReplenishmentActions` returns plain objects `{ label, onClick }`. Tests call
`onClick` directly — **no React render**.

### Fake dependencies

| Dependency | Fake |
| ---------- | ---- |
| `createPurchaseOrder` | `vi.fn()` |
| `confirm` | `vi.fn().mockResolvedValue(true)` or `false` |
| `toast` | `vi.mock("sonner", () => ({ toast: { error: vi.fn() } }))` |
| `gridRef` | `gridWithRows([...])` |

### Scenarios to cover

1. Fill buttons change row data (sync `onClick()`).
2. Create PO calls `createPurchaseOrder` with mapped rows (`await createPo.onClick()`).
3. User cancels confirm → mutate **not** called.
4. Validation error → `toast.error`, confirm **not** called.
5. Grid not ready → toast, no confirm.

```js
it("does not create documents when confirm is cancelled", async () => {
  const createPurchaseOrder = vi.fn();
  const confirm = vi.fn().mockResolvedValue(false);
  const [, , , createPo] = buildReplenishmentActions({
    gridRef: gridWithRows([poRow]),
    confirm,
    createPurchaseOrder,
  });

  await createPo.onClick();

  expect(createPurchaseOrder).not.toHaveBeenCalled();
});
```

**Why test actions if service is tested?** Actions wire **order**: collect →
toast on error → confirm → mutate. That order is easy to break.

---

## 11. Layer 4 — Mutations config

`ReplenishmentMutations` is a small factory — no hook, no React.

```js
vi.mock("../replenishment.api", () => ({
  saveReplenishmentToPo: vi.fn(),
  saveReplenishmentToRequisition: vi.fn(),
}));

it("refreshes the grid after creating a purchase order", async () => {
  const { saveReplenishmentToPo } = await import("../replenishment.api");
  const refresh = vi.fn();

  const { createPurchaseOrder } = ReplenishmentMutations({ refresh });

  expect(createPurchaseOrder.mutationFn).toBe(saveReplenishmentToPo);
  expect(createPurchaseOrder.onSuccess).toBe(refresh);

  createPurchaseOrder.onSuccess({ documents: [11] });

  expect(refresh).toHaveBeenCalledTimes(1);
});
```

You are proving: **right API function** + **refresh runs on success**. You do
not need to run `useMutation` in this test.

---

## 12. Layer 5 — React components (when needed)

Use when testing **shared** UI or real interaction (clicks, labels).

```jsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import SectionError from "../SectionError";

it("calls reset when Try again is clicked", async () => {
  const user = userEvent.setup();
  const reset = vi.fn();

  render(<SectionError message="Load failed" reset={reset} />);

  await user.click(screen.getByRole("button", { name: /try again/i }));

  expect(reset).toHaveBeenCalledTimes(1);
});
```

**Rules:**

- Query by **role**, **label**, or **text** — not CSS classes.
- `userEvent` is **async**: `await user.click(...)`.
- Import `@/i18n` if the component uses translations (see `SectionError.test.jsx`).

### Context and providers

If a component needs `GridProvider`, wrap in tests:

```jsx
render(
  <GridProvider>
    <MyComponent />
  </GridProvider>,
);
```

Or mock the hook: `vi.mock("@/shared/context/GridContext", () => ({ useGrid: () => ({ gridRef: { current: {} } }) }))`.

Prefer testing **children logic** in service/actions when the list page is thin.

### AG Grid

Do not mount real AG Grid in unit tests. Mock `ag-grid-react` or test the code
that **prepares props** / callbacks separately (repo convention).

---

## 13. Mocking explained

### `vi.fn()`

Creates a spy:

```js
const save = vi.fn();
save([{ id: 1 }]);
expect(save).toHaveBeenCalledWith([{ id: 1 }]);
```

### `vi.fn().mockResolvedValue(false)`

For async `confirm()` that returns a Promise.

### `vi.mock("sonner")`

Replaces the whole module for every test in the file. Must be at **top level**
(before imports that use it — Vitest hoists mocks).

### `vi.clearAllMocks()` in `afterEach`

Clears call history between tests; keeps mock implementation.

### `vi.stubGlobal("wz", { ... })`

Fake `window.wz` host options. Pair with `vi.unstubAllGlobals()` in `afterEach`
when used with HTTP tests.

### What not to mock

Do not mock the function you are **testing**. Mock **boundaries**: HTTP, toast,
confirm dialog, grid API.

---

## 14. Matchers you will use daily

| Matcher | Use |
| ------- | --- |
| `expect(x).toBe(y)` | Primitives (`===`). |
| `expect(x).toEqual(y)` | Deep equality objects/arrays. |
| `expect(x).toMatchObject({ a: 1 })` | Subset of fields. |
| `expect(() => fn()).toThrow("message")` | Sync throw. |
| `await expect(promise).resolves.toEqual(...)` | Async success. |
| `await expect(promise).rejects.toThrow(...)` | Async failure. |
| `expect(fn).toHaveBeenCalledTimes(n)` | Mock call count. |
| `expect(fn).not.toHaveBeenCalled()` | Never called. |
| `expect(el).toBeInTheDocument()` | DOM (jest-dom). |

---

## 15. Write your first test — step by step

Assume you added `toMyFeatureRow(row)` in `myFeature.service.js`.

1. Create `src/modules/.../myFeature/__tests__/myFeature.service.test.js`.
2. Copy `cubeReplenishmentRow` pattern → `baseRow(overrides)`.
3. Write one happy-path `it` with `toEqual`.
4. Run: `npx vitest run path/to/myFeature.service.test.js`.
5. Add `it` for one alias field and one error case.
6. When save exists, add `myFeature.api.test.js` with `createHttpSpy`.
7. When actions exist, add `myFeature.actions.test.js` with `vi.fn()` mutates.

**Order of difficulty:** service → api → mutations → actions (actions often import service behavior).

---

## 16. When a test fails

| Message | What to do |
| ------- | ---------- |
| `expected X to be Y` | Read diff; fix code or fix wrong expectation. |
| `ReferenceError: wz is not defined` | Stub host global or use `httpMock` helpers. |
| `Cannot read properties of undefined (reading 'api')` | Your `gridRef` fake is incomplete — add `forEachNode`. |
| Test passes alone, fails in suite | Missing `afterEach` cleanup / `restoreHttpMock`. |
| `act` warnings | Often async state; use `await` on `userEvent` and `findBy*` queries. |

Decide deliberately: **bug in product** vs **outdated test**. Do not weaken assertions
to greenwash; fix the right side.

---

## 17. Checklist for a new feature

- [ ] `__tests__/` next to feature code
- [ ] Service: happy path + main validation errors + grid edge cases
- [ ] API: URL, body, empty input throws, `restoreHttpMock` in `afterEach`
- [ ] Actions: mutate called / not called on cancel and validation
- [ ] Mutations: `mutationFn` reference + `onSuccess` calls refresh
- [ ] `npx vitest run <your tests>` green
- [ ] `npm run format` on new files
- [ ] No real network, no real ERP

---

## Study path (1–2 days)

| Session | Read / do |
| ------- | --------- |
| **A** | This doc §1–7. Run `npx vitest run replenishment.api.test.js` and read every `it`. |
| **B** | Read `toReplenishmentPoRow` + `collectReplenishmentPoRows` tests. Draw how `gridWithRows` works. |
| **C** | Read `buildReplenishmentActions` tests (confirm, toast, cancel). |
| **D** | Write one new `it` in a test file (even a trivial extra case) and run it. |
| **E** | Add tests for your own `*.service.js` while building your first page. |

---

## See also

- `.cursor/rules/testing.mdc` — project rules (short)
- `.cursor/skills/write-tests/SKILL.md` — agent checklist for authors
- `.cursor/skills/run-tests/SKILL.md` — running and triaging CI failures
