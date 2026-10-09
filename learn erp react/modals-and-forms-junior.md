# Modals and forms (junior guide)

How to build **create/edit dialogs** in this repo using:

- **`Modal`** — layout chrome (title, close, draggable shell)
- **`DynamicForm`** — most ERP forms (fields defined in config, **React Hook Form** inside)
- **`useForm`** — only for simple custom forms (login, one-off UIs)

Read with:

- [creating-a-page-by-hand.md](./creating-a-page-by-hand.md) — opening a modal from a list page
- [react-hooks-and-tanstack-junior.md](./react-hooks-and-tanstack-junior.md) — `useState`, Suspense, mutations
- [typescript-react-junior.md](./typescript-react-junior.md) — typing modal props and save payloads

**Canonical examples:**

| Pattern | File |
| ------- | ---- |
| List page opens modal + mutation on save | `stockCount/CountList.jsx` + `CreateStockCountModal.jsx` |
| Create vs edit + Suspense + fetch by id | `CreateStockCountModal.jsx`, `AddDiscountModal.jsx` |
| Edit record from API (not grid row) | `AddDiscountModal.jsx` (`EditDiscountForm`) |
| Adjust registered form per open | `EditRateModal.jsx`, `CreateStockCountModal.jsx` |

---

## Table of contents

1. [Mental model (PHP analogy)](#1-mental-model-php-analogy)
2. [Modal component](#2-modal-component)
3. [Opening a modal from a page](#3-opening-a-modal-from-a-page)
4. [React Hook Form in one page](#4-react-hook-form-in-one-page)
5. [DynamicForm — what it is](#5-dynamicform--what-it-is)
6. [Registered forms: `FORM_NAMES` and `getFormConfig`](#6-registered-forms-form_names-and-getformconfig)
7. [Field types and common field options](#7-field-types-and-common-field-options)
8. [Create flow (step by step)](#8-create-flow-step-by-step)
9. [Edit flow (fetch by id — required rule)](#9-edit-flow-fetch-by-id--required-rule)
10. [Save, delete, and mutations](#10-save-delete-and-mutations)
11. [Custom fields and extending a form](#11-custom-fields-and-extending-a-form)
12. [When NOT to use DynamicForm](#12-when-not-to-use-dynamicform)
13. [Adding a new form to the registry](#13-adding-a-new-form-to-the-registry)
14. [Testing](#14-testing)
15. [Checklist](#15-checklist)

---

## 1. Mental model (PHP analogy)

| Old (PHP + jQuery) | This repo |
| ------------------ | --------- |
| Bootstrap modal HTML in view | `<Modal title="…" onClose={…}>{children}</Modal>` |
| Form fields in `.php` | Field list in `form.config.js` (registry) or inline config |
| `$_POST` validation | Zod resolver built from field `required` / `validate` (`buildSchema.js`) |
| `$.post('save', serialize())` | `save` function passed to `getFormConfig` → often `useMutation` |
| Reload list after save | Parent `onSave` → `invalidateQueries` / `reloadGrid` |

**DynamicForm owns React Hook Form for you.** You usually do **not** call `useForm` in the modal — you pass a **config** object.

---

## 2. Modal component

Path: `src/shared/components/Layout/Modal.jsx`

```jsx
import Modal from "@/shared/components/Layout/Modal";

<Modal
  title="Physical count"
  onClose={() => setOpen(false)}
  classname="w-[min(480px,92vw)]"
>
  {/* form or Suspense + form */}
</Modal>
```

| Prop | Purpose |
| ---- | ------- |
| `title` | Header text (draggable bar) |
| `onClose` | X button and when user dismisses — **you** clear parent state |
| `classname` | Width / max width (Tailwind), e.g. `w-[500px]` |
| `children` | Form body (scrollable area, max ~72vh height) |

Details:

- Renders with **`createPortal`** into `document.body` (above the page).
- Uses **`ModalContext`** so stacked modals only show the top one (`useIsTopModal`).
- Do **not** hand-roll a full-screen overlay for feature work — use this `Modal`.

---

## 3. Opening a modal from a page

Pattern from **Open Stock Counts** / **Count list**:

```jsx
const [draftId, setDraftId] = useState(false);
// false = closed · null = create · number = edit id

usePageHeader({
  primaryAction: {
    label: "New count",
    onClick: () => setDraftId(null),
  },
});

return (
  <>
    {/* list content */}
    {draftId !== false && (
      <CreateStockCountModal
        countId={editing ? draftId : undefined}
        onClose={() => setDraftId(false)}
        onSave={handleCreateOrEdit}
      />
    )}
  </>
);
```

**Rules:**

- Modal open state lives in the **parent page** (`useState`), not in global context (unless using a dedicated modal host — rare for features).
- Pass **`onClose`** to reset state.
- Pass **`onSave`** async handler that runs mutation + closes or refreshes list.

You do **not** use `usePageHeader` inside the modal for the main save button — **DynamicForm** footer has Save / Cancel.

---

## 4. React Hook Form in one page

**React Hook Form (RHF)** keeps form values, validation, and submit handling without re-rendering every keystroke on the whole tree.

Concepts:

| API | Role |
| --- | ---- |
| `useForm({ defaultValues })` | Create form instance |
| `register("fieldName")` | Wire simple inputs |
| `Controller` | Wire custom components (DynamicForm uses this in `FieldRenderer`) |
| `handleSubmit(fn)` | Run `fn(values)` only if valid |
| `formState.errors` | Validation messages |

In **DynamicForm**, RHF is **internal** (`DynamicForm.jsx`):

```js
const form = useForm({ resolver, defaultValues });
const submit = form.handleSubmit((values) => persist(values));
```

You only need raw RHF for **small** forms (see `LoginMount.jsx`: `register`, `handleSubmit`, `errors`).

---

## 5. DynamicForm — what it is

Path: `src/shared/components/DynamicForm/DynamicForm.jsx`

A **config-driven** form:

1. Reads `config.fields` (array of field definitions).
2. Builds Zod schema + default values.
3. Renders fields via `FieldRenderer` + grid layout (`cols`).
4. Footer: Save, optional Save & New, Delete, Clear, Cancel — based on config.

```jsx
import { DynamicForm, getFormConfig, FORM_NAMES } from "@/shared/components/DynamicForm";

const config = getFormConfig(FORM_NAMES.STOCK_COUNT, {
  save: async (values) => { /* POST */ },
  initialValues: { /* optional */ },
});

<DynamicForm
  config={config}
  onSuccess={onClose}
  onCancel={onClose}
  className="p-3"
/>
```

| Prop | Purpose |
| ---- | ------- |
| `config` | From `getFormConfig` (fields + `save` / `delete` / labels) |
| `onSuccess` | Called after successful save (usually `onClose`) |
| `onCancel` | Cancel button |
| `onSubmit` | Override save handler (rare; prefer `config.save`) |
| `hideFooter` | Hide built-in footer (rare) |

**`key` on DynamicForm:** When switching create ↔ edit, use a stable `key` so RHF resets (see `AddDiscountModal`).

Errors from `save()` are **not** shown inside DynamicForm; failed mutations typically toast via **`MutationCache`** in `queryClient.ts`. Validation errors show on fields.

---

## 6. Registered forms: `FORM_NAMES` and `getFormConfig`

All shared form **definitions** live in:

`src/shared/components/DynamicForm/form.config.js`

```js
import { FORM_NAMES, getFormConfig } from "@/shared/components/DynamicForm";

getFormConfig(FORM_NAMES.STOCK_COUNT, {
  save: mySaveFn,
  saveAndNew: mySaveFn,      // optional — shows "Save & New"
  delete: myDeleteFn,        // optional — shows Delete + confirm
  initialValues: { code: "1" },
  submitLabel: "Save",
  deleteConfirmTitle: "Delete?",
  deleteConfirmDescription: "This cannot be undone.",
  clearable: false,          // hide "Clear form"
  actions: [],               // extra footer buttons
});
```

`getFormConfig`:

1. Looks up `forms[name]` in the registry.
2. Clones fields and merges **`initialValues`** into field defaults.
3. Attaches your **`save` / `delete`** functions to the returned config.

List registered names: `listFormNames()` or read `FORM_NAMES` in `form.config.js`.

---

## 7. Field types and common field options

From `fieldTypes.js`:

| `FIELD_TYPES` | Use |
| ------------- | --- |
| `TEXT` | Single line |
| `TEXTAREA` | Multi line |
| `NUMBER` | Numeric input |
| `DATE` | Date picker (display `dd/mm/yyyy`) |
| `SELECT` | Searchable select |
| `MULTI_SELECT` | Multiple values |
| `RADIO` / `SWITCH` / `CHECKBOX` | Booleans / choices |
| `CUSTOM` | You supply `render: (props) => <YourComponent />` |

Example field (from stock count registry):

```js
{
  name: "thedate",
  type: FIELD_TYPES.DATE,
  label: "Count date",
  required: true,
  cols: 12,
  validation: {
    message: "Count date is required",
    invalidMessage: "Count date must be a valid date (dd/mm/yyyy)",
  },
},
```

Common options:

| Option | Purpose |
| ------ | ------- |
| `name` | Key in form values (POST payload) |
| `label` / `placeholder` | UI |
| `required` | Zod + UI |
| `cols` | Grid width 1–12 (or function `(values) => 6`) |
| `defaultValue` | Initial value |
| `disabled` | Read-only |
| `hidden` | `true` or `when("otherField", (v) => …)` |
| `options` | Static array or `sessionOptions("warehouses")` |
| `validate` | `(value) => error string \| undefined` |
| `onChange` | Patch other fields when this changes |
| `clearable` | Field-level clear button (default true) |

**Conditional visibility:**

```js
import { when } from "@/shared/components/DynamicForm";

hidden: when("products_to_count", (value) => !isSpecificProducts(value)),
```

`when` reads current form values and returns boolean.

---

## 8. Create flow (step by step)

**Goal:** “New …” button → modal → defaults → save → close → refresh list.

### 1. Parent page

- `useState` for open/closed (and optional id for edit).
- `useMutation` with `onSuccess` → invalidate cache or `reloadGrid`.
- `primaryAction` or button sets state open.

### 2. Modal shell

```jsx
<Modal title="Add item" onClose={onClose} classname="w-[480px]">
  <Suspense fallback={<FormSkeleton className="p-3" />}>
    <CreateItemForm onClose={onClose} onSave={onSave} />
  </Suspense>
</Modal>
```

Use **Suspense** when the form needs **defaults from the server** (`useSuspenseQuery`).

### 3. Inner form component

```jsx
function CreateItemForm({ onClose, onSave }) {
  const { data: defaults } = useSuspenseQuery(itemFormDefaultsQuery());

  const config = useMemo(
    () =>
      getFormConfig(FORM_NAMES.MY_FORM, {
        save: onSave,
        initialValues: defaults,
      }),
    [defaults, onSave],
  );

  return (
    <DynamicForm config={config} onSuccess={onClose} onCancel={onClose} />
  );
}
```

Stock count: `CreateStockCountForm` + `stockCountFormDefaultsQuery()`.

### 4. Save handler (parent or modal)

Often wrapped in mutation:

```jsx
const saveMutation = useMutation({
  mutationFn: (payload) => saveItem(payload),
  onSuccess: () => {
    setOpen(false);
    queryClient.invalidateQueries({ queryKey: itemKeys.list() });
  },
});

const handleSave = useCallback(
  async (values) => {
    await saveMutation.mutateAsync(values);
  },
  [saveMutation],
);
```

Pass `handleSave` to modal as `onSave`. DynamicForm calls `config.save(values)` on submit.

---

## 9. Edit flow (fetch by id — required rule)

**Team rule** (see `.cursor/rules/dynamic-form-prefill.mdc`):

- Do **not** prefill edit forms from **grid/list row** objects.
- Pass **only id** → `useSuspenseQuery` → map **API response** → `initialValues`.

```jsx
function EditItemForm({ id, onClose, onSave }) {
  const { data } = useSuspenseQuery(itemByIdQuery(id));

  const config = useMemo(
    () =>
      getFormConfig(FORM_NAMES.MY_FORM, {
        save: (values) => onSave(values),
        initialValues: mapApiToFormValues(data),
      }),
    [data, onSave],
  );

  return (
    <DynamicForm config={config} onSuccess={onClose} onCancel={onClose} />
  );
}

// Modal
{editId != null ? (
  <Suspense fallback={<FormSkeleton />}>
    <EditItemForm id={editId} onClose={onClose} onSave={onSave} />
  </Suspense>
) : (
  <CreateItemForm … />
)}
```

**Stock count edit:** `EditStockCountForm` loads `stockCountHeaderQuery(countId)` + criteria query, then `toStockCountFormValues(record)`.

**Discount edit:** `EditDiscountForm` uses `discountQuery(record)` then `mapRecordToFormValues(recordData)` on **API data**, not the grid row.

**Exceptions:** pure create, settings forms, or passing a **parent id** for “add child under parent” — not a full row as edit source.

---

## 10. Save, delete, and mutations

### Where `save` lives

Pass into `getFormConfig`:

```js
save: async (values) => {
  await mutateAsync(values);
},
```

DynamicForm runs validation first, then `save(values)`.

### Delete

```js
getFormConfig(FORM_NAMES.X, {
  save: persist,
  delete: async () => {
    await deleteAsync();
  },
  deleteConfirmTitle: "…",
  deleteConfirmDescription: "…",
});
```

Delete uses the same confirm dialog pattern as the rest of the app.

### Save & New

```js
saveAndNew: persist,  // only on create flows usually
```

Resets form after save instead of calling `onSuccess`.

### Wiring mutations (AddDiscountModal style)

```jsx
const { mutateAsync } = useMutation({
  mutationFn: (values) => saveDiscount(values, record),
});

const persist = useCallback(
  async (values) => {
    const result = await mutateAsync(values);
    await onSave?.(values, result);
  },
  [mutateAsync, onSave],
);
```

Parent `onSave` can invalidate queries; modal `onSuccess={onClose}` closes UI.

---

## 11. Custom fields and extending a form

### Per-open field tweaks (no registry edit)

After `getFormConfig`, map `config.fields`:

```js
const config = getFormConfig(FORM_NAMES.EDIT_CURRENCY_RATE, { … });
config.fields = config.fields.map((field) =>
  field.name === "value_2" ? { ...field, hidden: true } : field,
);
```

Stock count inserts a **CUSTOM** criteria block:

```js
{
  name: "criteria",
  type: FIELD_TYPES.CUSTOM,
  render: (props) => <Criteria {...props} />,
  validate: (value) => (hasSelection(value) ? undefined : "Add at least one criteria"),
}
```

### Custom hook for config (`useStockCountFormConfig`)

When logic is large, extract `useMemo` that returns full config (see `CreateStockCountModal.jsx`).

---

## 12. When NOT to use DynamicForm

| Situation | Use instead |
| --------- | ----------- |
| Login, 2–3 raw inputs | `useForm` + manual JSX (`LoginMount.jsx`) |
| Highly bespoke layout | RHF + your components + `Modal` |
| Grid inline edit | Grid editors, not modal form |
| One-off dialog with no fields registry | RHF or plain state |

Default for ERP **create/edit records**: **DynamicForm** + registry entry.

---

## 13. Adding a new form to the registry

Only when the form is **reused** or matches existing ERP forms.

1. Add key to **`FORM_NAMES`** in `form.config.js`.
2. Add entry to **`forms`** object with `fields: [ … ]` (or `fields: () => [ … ]` if dynamic).
3. Use **`sessionOptions('option_key')`** or services for select options (see existing forms).
4. Open from modal with `getFormConfig(FORM_NAMES.YOUR_FORM, { save, initialValues })`.
5. Add **`DynamicForm.test.jsx`** or service tests if you add non-trivial `validate` / `onChange` logic.

Keep field `name` values aligned with what your **API save** expects.

---

## 14. Testing

| Layer | What to test |
| ----- | ------------ |
| Save API / service | Payload mapping, validation (`writing-tests-junior.md`) |
| `mapApiToFormValues` | Pure function unit tests |
| DynamicForm registry | `form.config.test.js`, `DynamicForm.test.jsx` for tricky fields |
| Modal component | Usually skip — test save handler and config builder |

Do not mount full DynamicForm in every feature test; test **`getFormConfig` + save callback** with a fake `save` if needed.

---

## 15. Checklist

**Modal**

- [ ] Parent owns open state; `onClose` clears it
- [ ] `classname` sets sensible width
- [ ] `Suspense` + `FormSkeleton` if inner form uses `useSuspenseQuery`

**Create**

- [ ] Defaults from API or `{}` via `initialValues`
- [ ] `config.save` → mutation → parent refresh
- [ ] `onSuccess={onClose}` on DynamicForm

**Edit**

- [ ] Open with **id only**
- [ ] Fetch record with `useSuspenseQuery`
- [ ] `initialValues` from **API mapper**, not grid row
- [ ] `key` on DynamicForm when switching records

**Quality**

- [ ] Delete/save confirm copy is clear
- [ ] `npm run lint` / `type-check` / tests for save mapping

---

## End-to-end diagram

```text
Page (useState open)
  └── Modal (title, onClose)
        └── Suspense (FormSkeleton)
              └── Form inner
                    ├── useSuspenseQuery (edit/create defaults)
                    ├── useMemo → getFormConfig(FORM_NAMES.*, { save, initialValues })
                    └── DynamicForm
                          ├── RHF + Zod validate
                          └── Save → config.save(values) → useMutation → API
                                └── onSuccess → onClose + invalidate / reloadGrid
```

---

## See also

- [page-chrome.md](./page-chrome.md) — `primaryAction` to open modals
- `.cursor/rules/dynamic-form-prefill.mdc` — edit prefill rule
- `src/shared/components/DynamicForm/__tests__/DynamicForm.test.jsx` — form behavior tests
- `src/shared/components/Skeletons` — `FormSkeleton` for modal fallback
