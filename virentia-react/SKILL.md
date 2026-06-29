---
name: virentia-react
description: Render Virentia models in React with @virentia/react — ScopeProvider, useProvidedScope, useUnit, useModel, component, createModelCache, ModelContext, controlled & cached models. Use when wiring Virentia state into React components. Assumes the `virentia` core mental model (stores/events/effects/reactions/scopes/owners).
---

# @virentia/react

Connect core models to React **at the rendering boundary only**. Domain logic stays in `@virentia/core` models; this package picks a scope, reads stores, binds callables, and manages model lifetime. Read the **virentia** skill first.

**Golden rule:** in React, `useUnit`/`useModel` unwrap stores to **plain values** and events/effects to **callbacks bound to the provided scope**. (Vue differs — it returns refs.)

## Setup — ScopeProvider

Every hook needs a scope above it. Provide one per tree that shares state; nest another for isolation.

```tsx
import { scope } from "@virentia/core";
import { ScopeProvider } from "@virentia/react";

const appScope = scope();

export function App() {
  return (
    <ScopeProvider scope={appScope}>
      <Routes />
    </ScopeProvider>
  );
}
```

`useProvidedScope(): Scope` reads it (throws if absent) — use when you must pass the scope into boundary helpers (`allSettled`, caches, adapters):

```tsx
const scope = useProvidedScope();
const onClick = () => allSettled(saved, { scope });
```

## useUnit — read stores, bind callables

Stores → values (component re-renders on change in that scope). Events/effects → scope-bound callbacks. `effect.pending` is a store → a value.

```tsx
const count = useUnit(counter.count);              // number
const incremented = useUnit(counter.incremented);  // (n) => Promise<void>

// object shape — several units at once
const { count, save, pending } = useUnit({
  count: counter.count,
  save: saveFx,
  pending: saveFx.pending,
});

// array shape
const [value, increment] = useUnit([counter.count, counter.incremented]);

return <button disabled={pending} onClick={() => increment(1)}>{count}</button>;
```

Use `useUnit` for simple components that need a handful of units.

## useModel — create/prepare a model for one component

```tsx
// 1) unwrap an existing model object
const vm = useModel(model);                          // stores→values, events→callbacks

// 2) create from a factory + props (props passed as a PLAIN OBJECT in React)
function Counter(props: { step: number }) {
  const m = useModel(createCounterModel, props);
  return <button onClick={() => m.clicked()}>{m.count}</button>;
}

// 3) cached: survives unmount, reused by key in the scope
const m = useModel(createChatModel, props, { cache: chatCache, key: props.chatId });
```

`useModel` creates the model **once** per component instance, keeps `context.props` fresh, fires `mounted`/`unmounted`, and disposes on unmount **unless** cached. The factory receives `ModelContext` (see below).

## component — model + view in one declaration

Use when the model is owned by the view. The `view` gets the unwrapped `model` plus the original props.

```tsx
export const Counter = component({
  model: createCounterModel,                 // ({ props }: ModelContext<{step:number}>) => {...}
  view({ model, step }) {
    return <button onClick={() => model.clicked()}>{model.count}</button>;
  },
});

// cached component
export const ChatPanel = component({
  cache: chatCache,
  key: (props: { chatId: string }) => props.chatId,
  model: createChatModel,
  view({ model }) { return <div>{model.messages.items.length}</div>; },
});
```

Use `component` when the model belongs to the view; use `useModel` to compose a model inside an existing component.

### Controlled / child models — `.create()`

A returned component has `.create(props)`, letting a **parent model own a child model**. Call it while the parent model factory runs (it captures the parent's scope).

```tsx
export const Parent = component({
  model() {
    const counter = Counter.create({ step: 2 });   // parent owns it
    return { counter };
  },
  view({ model }) {
    return <Counter step={2} model={model.counter} />;  // pass via `model` prop
  },
});
```

When a controlled `model` is passed in, React still keeps `props`/`mounted`/`unmounted`/`mounts` fresh, but **unmounting the child does not dispose it** — the parent owns its lifetime.

## createModelCache — keep models alive across unmounts

For chats, editor tabs, media players, previews, detail screens that should reopen with prior state.

```tsx
const chatCache = createModelCache<string, Props, ChatModel>();

chatCache.has(key, scope?);          // boolean
chatCache.get(key, scope?);          // Model | undefined
chatCache.getInstance(key, scope?);  // ModelInstance | undefined
chatCache.delete(key, scope?);       // dispose one (true if existed)
chatCache.clear(scope?);             // dispose all (in scope, or everywhere)
```

The `scope` arg is optional; without it the cache searches every scope it knows. **Caching is a lifetime decision** — the model lives until you `delete`/`clear`. Deleting disposes the owner (reactions + cleanup detach).

## ModelContext — what factories receive

```ts
interface ModelContext<Props, Key = undefined> {
  readonly scope: Scope;
  readonly owner: Owner;
  readonly props: ReactiveWritable<Props>; // reactive — read latest props without recreating the model
  readonly mounted: EventCallable<void>;  // fires on mount
  readonly unmounted: EventCallable<void>;// fires on unmount
  readonly mounts: StoreWritable<number>; // live mounted-view count (useful for cached models)
  readonly key: Key;                      // cache key, else undefined
}
```

Use lifecycle units **inside the model** (load on `mounted`, pause when `mounts` hits 0) so the behavior is testable as model code — not as React effects in disguise.

```tsx
function createChatModel({ props, mounted, key }: ModelContext<Props, string>) {
  const messages = store({ items: [] as string[] });
  const loadFx = effect(async (id: string) => fetchMessages(id));
  reaction({ on: mounted, run() { void loadFx(props.chatId); } });
  return { loading: loadFx.pending, messages };
}
```

## Habits / gotchas

- Provide a scope once at the top; read it with `useProvidedScope` when crossing into core helpers.
- Prefer `component`/`useModel` over hand-rolling `useEffect` lifecycles — the model owns the behavior.
- Cached models don't auto-dispose; clean them up deliberately.
- Props in React are a **plain object** (`useModel(factory, props)`), unlike Vue's getter form.
- `.create()` must run during the parent model factory, not in render.
