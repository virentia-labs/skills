---
name: virentia-vue
description: Render Virentia models in Vue 3 with @virentia/vue — ScopeProvider, provideScope, useProvidedScope, useUnit, useModel, component, createModelCache, ModelContext. Use when wiring Virentia state into Vue components. Mirrors @virentia/react but adapts to Vue reactivity (stores come back as refs). Assumes the `virentia` core mental model.
---

# @virentia/vue

Connect core models to Vue 3 **at the rendering boundary only**. The API mirrors `@virentia/react`, adapted to Vue reactivity. Read the **virentia** skill first.

**Golden rule (differs from React):** in Vue, `useUnit`/`useModel` return **refs** for stores (use `.value` in `<script>`, auto-unwrap in `<template>`) and **callables bound to the scope** for events/effects. **Destructure** results so refs become top-level bindings that auto-unwrap in templates.

## Setup — ScopeProvider / provideScope

```vue
<script setup lang="ts">
import { scope } from "@virentia/core";
import { ScopeProvider } from "@virentia/vue";
const appScope = scope();
</script>

<template>
  <ScopeProvider :scope="appScope">
    <RouterView />
  </ScopeProvider>
</template>
```

Or, inside your own `setup`, call `provideScope(appScope)` instead of rendering the component. `useProvidedScope(): Scope` reads it (throws if absent) — use to pass the scope into `allSettled`, caches, adapters. (No optional variant — it always throws when missing.)

## useUnit — read stores (refs), bind callables

```vue
<script setup lang="ts">
const count = useUnit(counter.count);              // Readonly<Ref<number>>
const incremented = useUnit(counter.incremented);  // (n) => Promise<void>

// object shape — destructure so refs/callables are top-level bindings
const { value, save, pending } = useUnit({
  value: counter.count,
  save: saveFx,
  pending: saveFx.pending,   // effect.pending is a store → a ref
});

// array shape — annotate `as const`
const [c, inc] = useUnit([counter.count, counter.incremented] as const);
</script>

<template>
  <button :disabled="pending" @click="save(values)">{{ value }}</button>
</template>
```

In `<script>` read `count.value`; in `<template>` write `{{ count }}` (auto-unwraps). A ref nested in a non-destructured object does **not** auto-unwrap — that's why you destructure.

## useModel — create/prepare a model

Key difference from React: **props are passed as a getter or ref (`MaybeRefOrGetter`)**, so prop changes flow into `context.props`.

```vue
<script setup lang="ts">
import { useModel } from "@virentia/vue";
import { createCounterModel } from "./counter-model";

const props = defineProps<{ step: number }>();
const { count, clicked } = useModel(createCounterModel, () => props);  // getter form
</script>

<template>
  <button @click="clicked()">{{ count }}</button>
</template>
```

Also: `useModel(model)` to unwrap an existing model, and `useModel(factory, () => props, { cache, key })` for cached models. Creates the model once, fires `mounted`/`unmounted`, disposes on unmount unless cached.

## component — model + view component

The `view` is a separate Vue component that receives the prepared `model` prop (a `ReactiveModel`) plus the model props. Destructure `props.model` in its setup.

```ts
// counter.ts
import { component } from "@virentia/vue";
import { createCounterModel } from "./counter-model";
import CounterView from "./CounterView.vue";

export const Counter = component({ model: createCounterModel, view: CounterView });
```

```vue
<!-- CounterView.vue -->
<script setup lang="ts">
import type { ReactiveModel } from "@virentia/vue";
import type { CounterModel } from "./counter-model";

const props = defineProps<{ step: number; model: ReactiveModel<CounterModel> }>();
const { count, clicked } = props.model;   // count is a ref → auto-unwraps in template
</script>

<template>
  <button @click="clicked()">{{ count }}</button>
</template>
```

Cached variant: `component({ cache, key: (props) => props.id, model, view })`.

### Controlled / child models — `.create()`

```ts
export const Parent = component({
  model() {
    const counter = Counter.create({ step: 2 });   // call during the parent factory
    return { counter };
  },
  view: ParentView,
});
```

```vue
<!-- ParentView.vue -->
<script setup lang="ts">
import type { ReactiveModel } from "@virentia/vue";
import { Counter } from "./counter";
import type { ParentModel } from "./parent";
const props = defineProps<{ model: ReactiveModel<ParentModel> }>();
</script>
<template>
  <Counter :step="2" :model="model.counter" />
</template>
```

A controlled child keeps `props`/`mounted`/`unmounted`/`mounts` fresh while mounted but is **not disposed on unmount** — the parent owns it.

## createModelCache — survive unmount

Identical surface to React: `has/get/getInstance/delete/clear`, optional `scope` arg, models live until you `delete`/`clear`.

```ts
const chatCache = createModelCache<string, Props, ChatModel>();
// component({ cache: chatCache, key: (p) => p.chatId, model: createChatModel, view })
```

## ModelContext — what factories receive

```ts
interface ModelContext<Props, Key = undefined> {
  readonly scope: Scope;
  readonly owner: Owner;
  readonly props: StoreWritable<Props>;
  readonly mounted: EventCallable<void>;
  readonly unmounted: EventCallable<void>;
  readonly mounts: StoreWritable<number>;
  readonly key: Key;
}
```

Put lifecycle logic in the model (load on `mounted`, pause when `mounts` hits 0), not in Vue hooks.

## React → Vue cheat sheet

| | React | Vue |
|---|---|---|
| store from `useUnit` | plain value | `Readonly<Ref<T>>` (`.value` in script, auto-unwrap in template) |
| `useModel` props arg | plain object `useModel(f, props)` | getter/ref `useModel(f, () => props)` |
| view | inline `view({ model })` JSX | separate SFC receiving `model` prop |
| optional scope | `useOptionalProvidedScope()` | none — `useProvidedScope()` throws |
| events/effects | callbacks (same) | callbacks (same) |

Everything else — `component`, `.create()`, cache semantics, `ModelContext`, lifecycle — is identical.
