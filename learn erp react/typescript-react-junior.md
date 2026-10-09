# TypeScript for React in this repo (practical junior guide)

You do **not** need a full TypeScript course to work here. You need enough to:

- Write new feature files as `.ts` / `.tsx`
- Type **props**, **API payloads**, and **hook return values**
- Read errors from `npm run type-check`
- Migrate mentally from JSDoc (`replenishment`) to types

The project uses **`strict: true`** (`tsconfig.json`) but **`allowJs: true`** — old
`.js` files still work; **new code should be TypeScript**.

Companion docs:

- [creating-a-page-by-hand.md](./creating-a-page-by-hand.md)
- [react-hooks-and-tanstack-junior.md](./react-hooks-and-tanstack-junior.md)
- [writing-tests-junior.md](./writing-tests-junior.md)

---

## Table of contents

1. [What TypeScript adds (PHP analogy)](#1-what-typescript-adds-php-analogy)
2. [File extensions and commands](#2-file-extensions-and-commands)
3. [Types you use every day](#3-types-you-use-every-day)
4. [Objects: `interface` vs `type`](#4-objects-interface-vs-type)
5. [Functions](#5-functions)
6. [React components and props](#6-react-components-and-props)
7. [Hooks with types](#7-hooks-with-types)
8. [TanStack Query + mutations](#8-tanstack-query--mutations)
9. [Grid rows and ERP data (`unknown`)](#9-grid-rows-and-erp-data-unknown)
10. [API layer: DTOs and Zod (team pattern)](#10-api-layer-dtos-and-zod-team-pattern)
11. [Imports: `import type`](#11-imports-import-type)
12. [Typing a replenishment-style feature](#12-typing-a-replenishment-style-feature)
13. [Tests in TypeScript](#13-tests-in-typescript)
14. [Common errors and fixes](#14-common-errors-and-fixes)
15. [What to learn later (skip for now)](#15-what-to-learn-later-skip-for-now)

---

## 1. What TypeScript adds (PHP analogy)

| PHP | TypeScript |
| --- | ---------- |
| Docblock `@param int $id` | `id: number` in function signature |
| `array` shape in comments | `interface Row { product_id: number }` |
| Runtime type errors in production | Many caught in editor / `type-check` **before** deploy |
| PHPUnit | Still Vitest — types help tests too |

TypeScript is **erased at build** — the browser still runs JavaScript. Types are for
**you and the compiler**, not for PHP-style runtime validation (unless you add Zod).

---

## 2. File extensions and commands

| Extension | Use |
| --------- | --- |
| `.tsx` | React components (JSX) |
| `.ts` | Services, API, mutations, types, tests without JSX |
| `.js` / `.jsx` | Legacy — do not add new feature files here |

| Command | Purpose |
| ------- | ------- |
| `npm run type-check` | Whole `src/` — run before PR if you touched TS |
| Editor | Red squiggles = same checker as CI |

`noImplicitAny: true` — if TypeScript cannot infer a type, you must write it (no
silent `any`).

---

## 3. Types you use every day

```ts
let count: number = 0;
let name: string = "Replenishment";
let done: boolean = false;
let ids: number[] = [1, 2, 3];
let maybeId: number | null = null; // number OR null
let maybeName: string | undefined = undefined; // optional field from API
```

**Union** `A | B` — value is one of those (like “int or null”).

**Optional property** — may be missing:

```ts
interface Options {
  title: string;
  gridId?: number; // same as gridId: number | undefined
}
```

**Literal types** — exact strings:

```ts
type SourceField = "min_qty_sum" | "max_qty_sum" | "suggested_qty";
```

Useful for `fillQtyToOrderFromField(gridRef, sourceField)`.

---

## 4. Objects: `interface` vs `type`

Both describe object shapes. In this repo:

- **`interface`** — domain entities, grid params (see `ReceivedNotInvoicedGridParams.ts`).
- **`type`** — unions, aliases, `z.infer<...>` from Zod.

```ts
// interface — common for feature params
export interface ReplenishmentPoRow {
  supplier_product: number | null;
  product_id: number | null;
  meas: number | string | null;
  g_color: number | null;
  g_size: number | null;
  g_cupsize: number | null;
  reorder_qty: number;
}

// type — union or alias
export type GridRef = React.RefObject<{ api?: GridApiLike } | null>;
```

Rule of thumb: start with **`interface`** for row/DTO objects; use **`type`** when
you combine or infer.

---

## 5. Functions

```ts
// Named args object (like replenishment.actions)
export function buildReplenishmentActions(args: {
  gridRef: GridRef;
  confirm?: (opts: ConfirmOptions) => Promise<boolean>;
  createPurchaseOrder?: (rows: ReplenishmentPoRow[]) => void;
}) {
  return [{ label: "Save", onClick: () => {} }];
}

// Return type inferred, or write explicitly:
function toReplenishmentPoRow(row: GridRow): ReplenishmentPoRow {
  return { /* ... */ };
}
```

**Async** — return type is `Promise<T>`:

```ts
async function saveToPo(rows: ReplenishmentPoRow[]): Promise<SaveResponse> {
  return requestJson(...);
}
```

**Void** — returns nothing meaningful:

```ts
function refresh(): void {
  reloadGrid?.({ reason: "mutation" });
}
```

---

## 6. React components and props

### Function component (this repo’s style)

```tsx
type ReplenishmentGridProps = {
  properties: ReplenishmentGridProperties;
};

const ReplenishmentGrid = memo(function ReplenishmentGrid({
  properties,
}: ReplenishmentGridProps) {
  return (
    <GridApp
      autoLoad={false}
      filterPlacement="header"
      properties={properties}
    />
  );
});
```

You can inline props in the parameter list (see `LabeledDropdownSelect.tsx`) — fine
for small components; extract `type XxxProps` when it grows.

### `children`

```tsx
type CardProps = {
  title: string;
  children: React.ReactNode;
};
```

### Events (rare in grid pages; common in forms)

```tsx
function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
  setQuery(e.target.value);
}

<button type="button" onClick={() => save()}>
```

For inventory grid pages you often **do not** type DOM events — actions live in
`*.actions.ts`.

### Default export

Same as JS: `export default ReplenishmentList;`

---

## 7. Hooks with types

### `useState`

TypeScript infers from the initial value:

```ts
const [open, setOpen] = useState(false); // boolean
const [draftId, setDraftId] = useState<false | null | number>(false);
```

Use explicit generic when the initial value does not show the full type:

```ts
const [rows, setRows] = useState<ReplenishmentPoRow[]>([]);
```

### `useRef`

```ts
const gridRef = useRef<{ api?: GridApiLike } | null>(null);
```

In practice you often use **`useGrid()`** from context — types are on the context
already; you pass `gridRef` through.

### `useMemo` / `useCallback`

Usually **inferred**. Annotate only if the compiler complains:

```ts
const actions = useMemo(
  () => buildReplenishmentActions({ gridRef, confirm, createPurchaseOrder }),
  [gridRef, confirm, createPurchaseOrder],
);
```

### Custom hooks

Return type can be inferred or explicit:

```ts
export function useHydratedProductToDeliverGridParams(): ProductToDeliverGridParams {
  // ...
}
```

---

## 8. TanStack Query + mutations

### Query options object

Pattern in TS modules (`meas.queries.ts`, `stockCount.queries.js`):

```ts
export function openStockCountsQuery() {
  return {
    queryKey: stockCountKeys.open(),
    queryFn: stockCountApi.getOpenCounts,
  };
}
```

With `useSuspenseQuery`:

```tsx
const { data } = useSuspenseQuery(openStockCountsQuery());
// data type comes from queryFn return type if api is typed
```

If `requestJson` returns `unknown`, narrow after fetch or type the API function:

```ts
async function getOpenCounts(): Promise<OpenCountCard[]> {
  const result = await requestJson("...");
  return result as OpenCountCard[]; // prefer Zod parse instead of bare cast
}
```

### `useMutation`

```ts
const { mutate: createPurchaseOrder } = useMutation({
  mutationFn: saveReplenishmentToPo,
  onSuccess: () => reloadGrid?.({ reason: "mutation" }),
});
```

Type `mutationFn` argument from API:

```ts
// replenishment.api.ts
export function saveReplenishmentToPo(
  rows: ReplenishmentPoRow[],
): Promise<SaveReplenishmentResponse> {
  return postReplenishmentRows("inventory/replenishment/saveReplenishmentToPo", rows);
}
```

### Mutation factory (replenishment.mutations)

```ts
type ReplenishmentMutationsOptions = {
  refresh?: () => void;
};

export function ReplenishmentMutations({ refresh }: ReplenishmentMutationsOptions = {}) {
  return {
    createPurchaseOrder: {
      mutationFn: saveReplenishmentToPo,
      onSuccess: refresh,
    },
  };
}
```

---

## 9. Grid rows and ERP data (`unknown`)

Cube / grid rows have **many column name variants** (`current_inventory.product_id`,
short aliases, etc.). You often do not model every column.

**Pragmatic approach in this codebase:**

```ts
/** One row from AG Grid / cube — not every column is listed. */
type GridRow = Record<string, unknown>;
```

Mapper functions take `GridRow` (or `object`) and return a **strict** PO row:

```ts
export function toReplenishmentPoRow(row: GridRow | null): ReplenishmentPoRow {
  // pickMatched, rowNumber helpers stay JS-style; return typed object
}
```

Avoid `any` — use `unknown` and narrow:

```ts
function asNumber(value: unknown): number | null {
  if (typeof value === "number" && Number.isFinite(value)) return value;
  if (typeof value === "string" && value !== "") return Number(value);
  return null;
}
```

This matches how `currency.api.ts` uses `Record<string, unknown>` for host rows.

---

## 10. API layer: DTOs and Zod (team pattern)

Newer TS APIs validate JSON with **Zod**, then export a type:

```ts
import { z } from "zod";

export const GetBaseCurrenciesResponseDtoSchema = z.object({
  success: z.literal(true),
  data: z.array(CurrencyDtoSchema),
});

export type GetBaseCurrenciesResponseDto = z.infer<
  typeof GetBaseCurrenciesResponseDtoSchema
>;
```

Usage after fetch:

```ts
const raw = await requestJson(...);
const parsed = GetBaseCurrenciesResponseDtoSchema.parse(raw);
// parsed is typed and validated
```

**You do not need Zod on day one** for a simple POST that sends rows you built
yourself. Add Zod when parsing **untrusted** host JSON into entities.

Folder convention in TS features:

```text
api/
├── myFeature.api.ts
└── dtos/
    └── getSomething/
        └── getSomethingResponse.dto.ts
```

---

## 11. Imports: `import type`

Types-only imports — erased at compile, clearer for bundlers:

```ts
import type { ReplenishmentPoRow } from "./replenishment.types";
import type { DateString } from "@/utils/dateString/dateString";
```

Value + type from same module:

```ts
import { saveReplenishmentToPo } from "./replenishment.api";
import type { SaveReplenishmentResponse } from "./replenishment.api";
```

---

## 12. Typing a replenishment-style feature

Suggested file split when converting from JS to TS:

| File | Types to add |
| ---- | ------------ |
| `constant.ts` | `ReplenishmentGridProperties` interface |
| `replenishment.types.ts` | `ReplenishmentPoRow`, `GridRow`, `SaveReplenishmentResponse` |
| `replenishment.service.ts` | Function args/returns use those types |
| `replenishment.api.ts` | `rows: ReplenishmentPoRow[]`, typed `Promise<...>` |
| `replenishment.actions.ts` | `BuildReplenishmentActionsArgs` interface |
| `replenishment.mutations.ts` | `ReplenishmentMutationsOptions` |
| `ReplenishmentList.tsx` | Minimal — hooks infer mutate types from `mutationFn` |
| `replenishment.routes.ts` | `Record<string, RouteDef>` — see below |

**Routes:** Today `RouteDef` lives as JSDoc in `src/routes/types.js`. In a `.ts`
routes file you can import the typedef via:

```ts
import type { RouteDef } from "@/routes/types";
```

(If the import fails, use a local `satisfies` object or duplicate a minimal
`RouteDef` interface until routes are migrated — ask the team which pattern they
standardized on.)

**Grid properties** — mirror `constant.js`:

```ts
export interface ReplenishmentGridProperties {
  id: number;
  virtual: boolean;
  editableColumns: string[];
}

export const REPLENISHMENT_GRID_PROPERTIES: ReplenishmentGridProperties = {
  id: 137,
  virtual: true,
  editableColumns: [QTY_TO_ORDER_FIELD],
};
```

---

## 13. Tests in TypeScript

Same Vitest API; file `*.test.ts` / `*.test.tsx`:

```ts
import { describe, expect, it, vi } from "vitest";
import type { ReplenishmentPoRow } from "../replenishment.types";

const poRow: ReplenishmentPoRow = {
  supplier_product: 1,
  product_id: 10,
  meas: 1,
  g_color: null,
  g_size: null,
  g_cupsize: null,
  reorder_qty: 5,
};
```

Fixtures like `cubeReplenishmentRow` stay `GridRow` with overrides:

```ts
function cubeReplenishmentRow(overrides: GridRow = {}): GridRow {
  return { "current_inventory.product_id": 181, ...overrides };
}
```

See [writing-tests-junior.md](./writing-tests-junior.md).

---

## 14. Common errors and fixes

| Error | Meaning | Fix |
| ----- | ------- | --- |
| `Object is possibly 'null'` | You used `.` on something nullable | `if (!x) return;` or `x?.prop` |
| `Property 'foo' does not exist` | Wrong shape | Fix interface or use narrowing |
| `Type 'string' is not assignable to type 'number'` | ID came as string from grid | Parse or widen type `number \| string` |
| `Parameter 'x' implicitly has an 'any' type` | No type in strict mode | Add type or generic |
| `Cannot use JSX unless '--jsx' is set` | File should be `.tsx` | Rename file |
| Module has no exported member | Wrong import name | Check export default vs named |

**Do not “fix” by slapping `any` everywhere** — use `unknown` + small helpers, or
`GridRow` for cube data.

**`as SomeType`** — only when you are sure (or right after Zod `.parse`).

---

## 15. What to learn later (skip for now)

- Generics deep dive (`<T extends ...>`)
- `satisfies` operator
- Conditional types, mapped types
- Declaring modules for untyped packages
- React 19 / Server Components (this app is client-mounted ERP UI)

---

## Minimum checklist for your first TS page

- [ ] New files are `.ts` / `.tsx`, not `.js`
- [ ] One `*.types.ts` (or `entities/`) for row + response shapes
- [ ] Service functions have typed parameters and return values
- [ ] API `mutationFn` arguments match service output
- [ ] Components: `XxxProps` type on props
- [ ] `npm run type-check` passes
- [ ] `npm run lint` + tests still green

---

## Suggested learning path (1 day with React doc)

| Block | Activity |
| ----- | -------- |
| 1 | Read §3–5; type a fake `ReplenishmentPoRow` on paper |
| 2 | Read §6–7; add `Props` to a tiny component |
| 3 | Read §8–9; type one `saveX(rows: Row[])` API function |
| 4 | Run `npm run type-check`; fix one real error in a TS file in `src/` |
| 5 | Rename **one** new feature file to `.ts` from the page guide and type it |

---

## See also

- `tsconfig.json` — `strict`, `allowJs`, path alias `@/*`
- `src/modules/inventory/inventory/receivedNotInvoiced/types/` — grid params interfaces
- `src/modules/accounting/currency/api/` — Zod DTOs + API
- `src/shared/components/ui/*.tsx` — shadcn components (typed props)
