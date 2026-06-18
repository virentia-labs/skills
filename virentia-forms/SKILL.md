---
name: virentia-forms
description: Build forms with @virentia/forms — fields, forms, shape & array fields, the two-channel error model (inner/outer), validation lifecycle & strategies, custom fields, field types, wizards, schema (zod) adapters, and React hooks (useField/useForm/useWizard). Use when creating or reviewing form state with Virentia. Assumes the `virentia` core mental model.
---

# @virentia/forms

Forms built on Virentia core: every field and form is a **model** whose state lives in stores, so it runs unchanged in tests/SSR. Read the **virentia** skill first. All mutating methods are **async** (`fill`, `reset`, `validate`, `submit`, …) — `await` them.

## Mental model

- A **field** owns a value + its errors + focus/validation state.
- A **form** composes fields by a schema and aggregates their values/errors.
- Errors live in **two channels**: `innerError` (from validators) and `outerError` (from the backend). The visible `error = outerError ?? innerError` — server errors don't erase local validation and vice-versa.
- Validation runs according to **strategies** (`change` | `blur` | `focus` | `submit` | `manual`) declared per field/form.

## Fields

```ts
import { createField } from "@virentia/forms";

const email = createField("", {
  validate: emailValidator,                 // one validator or an array (runs in order, stops at first error)
  validationStrategies: ["blur", "submit"],
});
```

Field stores (read via the React hook or `scoped`): `state` (value), `error`, `innerError`, `outerError`, `meta`, `isFocused`, `isValid`, `isValidationPending` (plural aliases `errors`/`innerErrors`/`outerErrors` exist too — the React hooks expose the plural names).
Field **async methods** (return a promise — `await` them): `fill(value)`, `reset()`, `validate()`, `setInnerErrors(e)`, `setOuterErrors(e)`, `clearInnerErrors()`, `clearOuterErrors()`. `read()` returns the current value (not a promise).
Field **event callables** (fire them, scope-bound — they run the matching reactions): `focus`, `blur`, `changeMeta`. These are events, **not** async methods.
Field events to react to: `changed`, `focused`, `blurred`, `validated`, `validationFailed`, `errorsChanged`.

### Shape fields — dynamic objects (unknown/changing keys)
```ts
const attrs = createShapeField({}, {
  createField: (key, value) => key.endsWith("Url") ? urlField(String(value)) : createField(value),
});
await attrs.fill({ avatarUrl: "...", theme: "dark" });   // fields created on demand
// add({key, field}), remove(key), replace({key, field}), clear(); fields: Store<Record<string, Field>>
```

### Array fields — ordered lists
```ts
const lines = createArrayField<LineValue>([], {
  createItem: (line, index) => createShapeField({ title: createField(line.title), qty: createField(line.qty) }),
});
await lines.push(value);   // unshift, insert(i,v), remove(i), pop(), replace(i,v), move(from,to), swap(a,b), clear()
// items: Store<ItemField[]>, length: Store<number>
```
Item errors mirror the array shape: `FieldError | readonly ItemErrors[]`. `move`/`swap` preserve each item's internal state.

## Forms

```ts
import { createForm } from "@virentia/forms";

const form = createForm({
  schema: { email: createField(""), password: createField("") },
  validation: (values) => null,             // form-level validator(s); runs after field validators
  validationStrategies: ["submit"],
});
```

Form stores: `values`/`value`, `errors` (outer ?? inner), `innerErrors`, `outerErrors`, `snapshot` (last accepted state), `isChanged` (values ≠ snapshot), `isValid`, `isValidationPending`.
Form methods (async): 
- `fill({ values?, errors? })` — patch values and/or **outer** errors (use for backend errors).
- `validate()` — run field + form validators; writes errors back into fields; does **not** touch snapshot.
- `submit()` — `validate()`, and **if valid** update snapshot + emit `submitted`.
- `reset()`, `clearInnerErrors()`, `clearOuterErrors()`, `forceUpdateSnapshot()`.
- `pick(selection)` — a projection over the **same** field instances (for wizard steps; not a copy).
Form events: `filled`, `changed`, `validated`, `validationFailed`, `submitted`.

### Submit + backend errors pattern
```ts
await form.submit();                                  // local validation gate
// on server rejection:
await form.fill({ errors: { email: "Already taken" } });  // → outerError, stays visible
await form.clearOuterErrors();                            // when the user starts fixing it
```

## Validation

A validator is a function `(value, ctx) => Errors | null` (or an `Effect` for async). `null`/`undefined` = valid.
```ts
const ctx; // { signal: AbortSignal, path: string[], read<T>(store): T }
```
- `ctx.signal` — pass to `fetch` for cancellable async validation; a newer run aborts the previous.
- `ctx.read(store)` — read another store **and** subscribe, so the field re-validates when it changes (cross-field rules).
- **Strategies** decide *when* validators run; a form's strategies apply to its child fields too. Default: only on explicit `validate()`/`submit()`.

## Custom fields & field types

Return a `FieldContract` from a factory; expose nested fields via `readFields()` so the form can collect their errors. Wrap with `defineField(...)`.
```ts
function createMoneyField(initial: Money) {
  const amount = createField(initial.amount);
  const currency = createField(initial.currency);
  return defineField({
    kind: "money",
    state: computed(() => ({ amount: amount.state.value, currency: currency.state.value })),
    readFields: () => ({ amount, currency }),
    fill: async (v) => { await Promise.all([amount.fill(v.amount), currency.fill(v.currency)]); },
    reset: async () => { await Promise.all([amount.reset(), currency.reset()]); },
  });
}
```
**Field types** are reusable, extendable factories:
```ts
const primitive = fieldType({ create: createField });
export const emailField = primitive.extend({
  kind: "email",
  create: (base, initial = "") => base(initial, { validate: [zodFieldValidator(z.string().email())] }),
});
```

## Wizards (multi-step)

Two shapes:
- **Shared root** — all steps edit one final object; steps are `form.pick(...)` projections:
  ```ts
  const wizard = createWizardForm({
    schema: { email, password, plan },
    steps: (form) => [
      step("account", { form: form.pick({ email: true, password: true }) }),
      step("plan", { form: form.pick({ plan: true }) }),
    ],
  });
  ```
- **Independent steps** — each step is its own form/model: `createWizard({ steps: [step("import", { form: importForm }), ...] })`.

Wizard stores: `steps`, `visibleSteps` (filtered by step `when`), `currentId`, `currentIndex`, `currentStep`, `currentForm`, `visitedIds`, `completedIds`, `canGoBack`, `canGoNext`.
Wizard methods (async): `next()` (validate current, advance if valid), `back()`, `goTo(id)` (validates intermediate steps when moving forward), `complete()` (validate all visible → emit `completed`) — these return `Promise<boolean>`; `reset()` returns `Promise<void>`.

## React integration — `@virentia/forms-react` (separate package)

Hooks live in **`@virentia/forms-react`** (not a subpath of `@virentia/forms`). Wrap the tree in `<ScopeProvider scope={...}>` (from `@virentia/react`). Hooks return scoped snapshots + scoped handlers.
```tsx
import { useField, useForm, useWizard, useWizardForm } from "@virentia/forms-react";

const f = useField(field);     // { value, errors, innerErrors, outerErrors, isValid, isValidationPending, view, fill, reset, validate, setInnerErrors, setOuterErrors, clearInnerErrors, clearOuterErrors }
const fm = useForm(form);      // { values, errors, snapshot, isChanged, isValid, submit, fill, reset, validate, forceUpdateSnapshot, ... }
const wz = useWizard(wizard);  // { currentForm, currentIndex, canGoBack, canGoNext, next, back, goTo, complete, ... }  // useWizardForm === useWizard
```
The hook result exposes the **plural** `errors` and does **not** include `blur`/`focus`/`error` (singular) — those are field events. Bind a field event with `useUnit` from `@virentia/react`:
```tsx
import { useUnit } from "@virentia/react";
import { useField } from "@virentia/forms-react";

function EmailInput({ field }) {
  const { value, errors, fill } = useField(field);
  const blur = useUnit(field.blur);   // field.blur is an event → bind it to the scope
  return <input value={value} onChange={(e) => fill(e.target.value)} onBlur={() => blur()} aria-invalid={!!errors} />;
}
```

## Schema adapters (zod)

```ts
import { zodFieldValidator, zodFormValidator } from "@virentia/forms-zod"; // separate package; zodValidator === zodFormValidator

createField("", { validate: zodFieldValidator(z.string().email("Invalid email")) });
createForm({ schema, validation: zodFormValidator(z.object({ email: z.string().email() })) });
```
The form adapter maps zod issues onto the nested form shape (writes child inner errors by path); the field adapter returns the first issue message (or `null` when valid). Pass a factory `(ctx) => schema` to read stores for reactive rules.

## Habits / gotchas

- **Three packages:** core models from `@virentia/forms`, React hooks from `@virentia/forms-react`, zod adapters from `@virentia/forms-zod` (each a bare package, no subpaths).
- `await` every mutating method; they return promises. `focus`/`blur`/`changeMeta` are events you fire, not async methods.
- Backend errors → `outerError` (`fill({ errors })`); validators → `innerError`. Don't cross the streams.
- `submit()` only advances snapshot when valid; `validate()` never touches snapshot.
- Custom/composite fields **must** expose children via `readFields()` or the form won't see their errors.
- Wizard steps over a shared root use `pick` — they edit the same instances, so validation is consistent.
