---
name: virentia-net
description: Declarative remote-data layer with @virentia/net — query() and mutation() that ARE Virentia effects (not farfetched-style event bags), trigger() bindings, the concurrency (takeLatest/takeFirst/takeEvery/queue) / retry / timeout / debounce / fallback / tap operators, optimistic updates + invalidation for mutations, TanStack Query / Apollo executor adapters (net-core/tanstack, net-core/apollo), and overrideDefaults for global/per-scope defaults. Covers the framework bindings too: @virentia/net-react and @virentia/net-vue (useQuery/useMutation). Use when adding or reviewing data fetching / server-state in a Virentia app. Assumes the `virentia` core mental model (and virentia-react/virentia-vue for the bindings).
---

# @virentia/net-core

The "farfetched" of this stack: describe fetching as a model, outside the UI. Read the **virentia** skill first — net is built on core effects, stores, events, reactions, scopes.

**The one idea that changes everything: a query/mutation IS an effect.** `query()`/`mutation()` return a real `@virentia/core` effect. You do **not** get a status object with `start`/`finished.*`/`$status`; you get the effect's own surface plus two small stores. Don't reach for effector/farfetched patterns.

```ts
import { query, mutation, trigger, concurrency, retry } from "@virentia/net-core";
```

Install: `pnpm add @virentia/net-core @virentia/core`.

## query

```ts
const userQuery = query({
  params: ({ id }: { id: string }) => ({ id }),          // optional: call-shape → handler-shape
  handler: async ({ id }, { signal }) => {               // ctx is { signal, scope }
    const res = await fetch(`/api/users/${id}`, { signal });
    return (await res.json()) as User;
  },
  use: [concurrency({ strategy: "takeLatest" }), retry({ times: 3, delay: 300 })],
});
```

It's an effect — **call it to run**, and read its units. Net adds `data`/`stale`/`refresh`/`reset`:

- `userQuery.pending` (`Store<boolean>`), `userQuery.inFlight` (`Store<number>`)
- `userQuery.doneData` (`Event<Data>`), `userQuery.failData` (`Event<Error>`), `userQuery.finally`
- `userQuery.abort()`, per-call `signal`
- `userQuery.data` (`Store<Data | null>`, latest success), `userQuery.error` (`Store<Err | null>`, cleared on success), `userQuery.stale` (`Store<boolean>`)
- `userQuery.refresh` (re-run last params in the scope), `userQuery.reset` (clear + abort)

**No `$` on stores** — this is Virentia, not effector: `userQuery.data`, not `$data`.

Config: `handler` (required), `params?`, `use?`, `executor?` (swap engine, defaults to running `handler`), `key?` (lane key), `initialData?`, `trigger?`, `name?`. Without `params`, call-shape == handler-shape.

## Reading & driving (per scope, like every Virentia model)

```ts
import { scope, scoped } from "@virentia/core";
const app = scope();
await scoped(app, () => userQuery({ id: "42" })); // run in a test/SSR
scoped(app, () => userQuery.data.value);                            // read outside components
```

In React/Vue, read units with `useUnit` — or the net hooks below. Separate scopes keep independent `data`/`pending`.

## trigger

Run a query/mutation when any Virentia unit fires. It's a `reaction` that calls the effect, with mapping/filter/owner-cleanup handled:

```ts
trigger(userQuery, {
  on: userRoute.opened,                              // event | effect unit | store | array
  params: () => ({ id: userRoute.params.value.id }), // may ignore payload, read reactive state
  filter: (p) => Boolean(p),                          // optional guard
});
```

`params` may be a **zero-arg reactive read** (ignore payload, read stores). Many triggers per query allowed. Returns an unsubscribe; auto-cleans inside an `owner`. Inline `config.trigger` is the same, applied at creation.

## Operators (the `use: []` middleware)

Onion order — first operator is outermost; scheduler stage (`concurrency`) always wraps executor stage (`retry`), regardless of array order.

**`concurrency({ strategy, key? })`** — the result-taking strategy. Per-scope, per-lane, using its own abort controllers (a newer run cancels only the older one, never the whole effect):
- `takeLatest` (default): abort previous, keep newest (search-as-you-type).
- `takeFirst`: in-flight call is shared (dedup) — handler runs once.
- `takeEvery`: no coordination.
- `queue`: serialize.
- `key: (params) => …` → independent lanes (per-id).

**`retry({ times, delay, when })`** — re-run on failure, abort-aware. `times` = retries after the first failure (default 3). `delay` may be `(attempt, error) => ms` for backoff. `when(error, attempt)` vetoes. Skips/aborts are never retried.

**`timeout(ms)`** — races a deadline, rejects with `TimeoutError`, and aborts the run (works even if the handler ignores the signal). Executor stage.

**`debounce({ wait })`** — pre-delay; a true debounce when paired with `takeLatest` (`use: [concurrency({ strategy: "takeLatest" }), debounce({ wait: 300 })]`). Scheduler stage.

**`fallback(value | (error, params) => value)`** — recover a failure with a value instead of failing; put it **before** `retry` so it catches after retries exhaust. Skips/aborts pass through.

**`tap({ onStart, onSuccess, onError, onSettled })`** — observe a run (analytics/logging) without changing the result; wrap store writes in `scoped(ctx.scope, …)`.

Ordering: scheduler ops (`concurrency`, `debounce`) wrap executor ops (`fallback`, `retry`, `timeout`); within a stage, array order (first = outermost).

## mutation

Same effect-with-operators as `query`, plus write-side extras:

```ts
const addItem = mutation({
  handler: async (name: string) => api.add(name),
  optimistic: {
    update: (name) => { items.value = [...items.value, name]; },  // on start (in scope)
    rollback: (name) => { items.value = items.value.filter((i) => i !== name); }, // once, on final fail
  },
  invalidates: [todosQuery],  // on success, refresh these (re-run last params)
});
```

`optimistic` wraps *outside* retry → update applies once, rollback fires once. Default overlap is uncoordinated (each submit its own run); add `concurrency` if you want serialize/dedup. No `cache()` for mutations.

## Adapters (out of the box, surface unchanged)

Swap the **executor** (innermost engine) to back a query with TanStack Query or Apollo — `data`/`pending`/triggers/operators keep working. Optional subpath exports (clients read lazily, so they can be per-scope deps):

```ts
import { tanstackExecutor } from "@virentia/net-core/tanstack";
import { apolloExecutor } from "@virentia/net-core/apollo";

query({ handler, executor: tanstackExecutor(() => queryClient, { queryKey: ({ id }) => ["user", id] }) });
query({ executor: apolloExecutor(() => apolloClient, { document: USER, variables: (p) => ({ id: p.id }) }) }); // no handler
```

tanstack routes your `handler` through `QueryClient.fetchQuery` (its cache/dedup); apollo IS the fetch (document-based, `handler` omitted). Net's abort signal is forwarded either way. Clients are accepted structurally, so a real `QueryClient`/`ApolloClient` fits with no hard dep. An **`Executor` is just a function** `(params, ctx) => Promise<Data>` (ctx has `signal`/`scope`/`handler`) — the innermost link of the chain; you rarely write one by hand.

## overrideDefaults (global + scoped)

Set the defaults every query/mutation inherits — a default executor or operators — globally or per scope. Resolved at execution time inside the run's scope, so `{ scope }` overrides stay isolated (tests, SSR). Returns a revert fn.

```ts
import { overrideDefaults, query } from "@virentia/net-core";

overrideDefaults(query, { executor: tanstackExecutor(() => queryClient) });          // global
const revert = overrideDefaults(query, { use: [retry({ times: 5 })] }, { scope: testScope }); // scoped
```

Precedence: built-in < global < scoped < the query's own `executor`/`use`. First arg is the factory (`query`/`mutation`) — each keyed separately.

## Framework bindings

A query/mutation is an effect, so bindings are thin sugar over `useUnit` — they read the per-scope `data`/`error`/`pending`/`stale` stores and bind `run`/`mutate`/`refetch`/`reset` to the provided scope. State is per-scope; provide it with the framework's `ScopeProvider`.

**React — `@virentia/net-react`** (`pnpm add @virentia/net-react @virentia/net-core @virentia/react @virentia/core react`):
```tsx
import { ScopeProvider, useQuery, useMutation } from "@virentia/net-react";

function User({ id }: { id: string }) {
  const { data, error, pending, stale, run, refetch, reset } = useQuery(userQuery);
  useEffect(() => { run({ id }); }, [id]);           // or drive it with a net trigger
  // ...
}
// useMutation(m) → { data, error, pending, mutate, reset }
// wrap the tree: <ScopeProvider scope={appScope}>…</ScopeProvider>  (re-exported)
```

**Vue — `@virentia/net-vue`** (mirrors React; stores come back as refs):
```ts
import { useQuery, provideScope } from "@virentia/net-vue";
const { data, error, pending, stale, run, refetch, reset } = useQuery(userQuery); // *.value in script
// provideScope(appScope) in an ancestor's setup (or the ScopeProvider component). Re-exported.
```

Binding habits: read via the hooks (not manual `doneData`/`failData` subscriptions); prefer a net **trigger** over an in-component effect/watch when the run follows app events; one provided scope per app/test/request. For units the hooks don't expose (e.g. `inFlight`), use `useUnit(query.inFlight)` (re-exported).

## Skips

An intentionally-not-executed run (superseded `takeLatest`, cache hit, closed barrier later) surfaces a `SkipSignal`, not a real error — recognize with `isSkip(error)`. Handy when subscribing to `failData` to ignore cancellations.

## Habits / anti-patterns

- ✅ `query.data` / `query.pending` / `query.failData`. ❌ inventing `$data`, `$status`, `start`, `finished.success` — it's an effect; use the effect surface.
- ✅ cancellation via `handler`'s `ctx.signal` passed to `fetch`. ❌ ignoring the signal (breaks takeLatest/abort).
- ✅ per-id concurrency via `concurrency({ key })`. ❌ one global lane when ids are independent.
- ✅ trigger with a reactive `params: () => …` to read route/store state at fire time. ❌ threading ids through payloads you don't have.
- ✅ read/drive through a `scope` in tests (`scoped`). ❌ reading `.value` with no scope.

## Not yet shipped

`cache()` and barriers (`createBarrier`/`applyBarrier`) are planned — don't reference them as existing API. (`query`/`mutation`, `trigger`, `concurrency`/`retry`, the TanStack/Apollo adapters, and `overrideDefaults` all ship today.)
