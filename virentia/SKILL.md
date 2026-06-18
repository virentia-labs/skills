---
name: virentia
description: Foundational mental model and core API for the Virentia state manager (@virentia/core) — stores, events, effects, reactions, scopes, owners, transactions, lazy models. Use this whenever writing, reviewing, or debugging Virentia model code, or before reaching for any @virentia/* package (react, vue, forms, router, effector, inspector). Teaches the mindset, the API formulas, and the habits that keep business logic a model instead of scattered imperative code.
---

# Virentia — core model & mindset

Virentia is a state manager for apps with **complex business logic**. Its goal is not shorter writes — it is **preserving causality**: what happened, which rules ran, which async work started, and which state came out. Every state movement should have **a place and a name**.

If you find yourself writing `setX`, threading ids through handlers, or letting the UI set loading / catch errors / clear results, stop — that logic belongs in the model.

## The mental model (read this first)

A model is a plain factory function that wires six primitives. It does **not** know where it will render — it must run unchanged in React, Vue, a test, an SSR request, or background work.

| Primitive | Answers | Role |
|-----------|---------|------|
| **store** | "what must the model remember?" | memory (value lives in a *scope*, not globally) |
| **event** | "what meaningful thing happened?" | a named fact (`messageReceived`) or public intent (`open`) — never a setter |
| **effect** | "what external async work runs?" | request/timer/worker with a lifecycle (start, success, fail, pending) |
| **reaction** | "which rule connects things?" | a named rule: read stores, write stores, call effects |
| **scope** | "where do values live for THIS run?" | one concrete value map (app instance, test, request, cached screen) |
| **owner** | "when is this work detached?" | lifetime boundary for runtime-created models |

**Split responsibility is the whole point.** Stores remember. Events report. Effects do async. Reactions decide. Keeping these separate is what makes the model testable, reusable on the server, and changeable without regressions.

**Definitions vs values.** A `store`/`event`/`effect` is a stable *definition* (shape). The actual value lives in a `scope`. One model → many isolated states. This is the basic Virentia mechanism.

### Canonical model shape

```ts
import { event, reaction, store } from "@virentia/core";

export function createCounterModel() {
  const incremented = event<number>();   // a fact: count went up by N
  const reset = event<void>();            // a fact: counter was reset
  const count = store(0);                 // memory

  reaction({ on: incremented, run(amount) { count.value += amount; } });
  reaction({ on: reset, run() { count.value = 0; } });

  return { count, incremented, reset };
}
```

One event can drive several reactions, so the model never collapses into a pile of setters.

## Habits (do these by default)

1. **Name events in task language, not field language.** `submitted`, `messageReceived`, `open`, `reset` — not `setLoading`, `updateName`.
2. **Stores hold domain state; reactions write them.** Don't mutate stores from random call sites; route changes through events/reactions.
3. **One event, many rules.** When an event has several consequences, write several reactions, not one fat handler.
4. **Keep imperative code at boundaries and inside named rules.** Effect handlers are normal async functions; reaction bodies are normal code. The *structure* is declarative, the *bodies* can be imperative.
5. **Never store loading/error in the UI.** Use `effect.pending` and the effect's `fail`/`failData` units.
6. **Values live in a scope.** Reading/writing a store outside a reaction requires a scope frame (`scoped`/`allSettled`). Tests create a `scope()`.
7. **Runtime-created models get an owner** so reactions/subscriptions are cleaned up when the chat/tab/modal closes.

## API formulas

### store — memory
```ts
const count = store(0);
const profile = store({ name: "Ada", age: 36 });

scoped(scope, () => {
  count.value += 1;          // primitive: read/write .value
  profile.value = { ...profile.value, age: 37 };
});

const unsub = count.subscribe((value, scope) => {/* ... */});
```
Derived (read-only) stores — recompute lazily, notify only when the result changes:
```ts
const doubled = count.map(v => v * 2);
const positive = count.filter(v => v > 0);               // keeps prior value when predicate fails
const label = count.filterMap(v => v > 0 ? `#${v}` : undefined, undefined);
const visible = computed(() => users.value.filter(u => u.name.includes(query.value)));
```
Use `map`/`filter`/`filterMap` for single-source derivation; `computed(() => ...)` for multi-source. If the value depends on **events over time** (not just other stores), use a normal `store` + `reaction` instead.

### event — a named fact or intent
```ts
const queryChanged = event<string>();   // fact
const open = event<void>();             // public intent
scoped(scope, () => queryChanged("docs"));
const nonEmpty = queryChanged.filter(t => t.length > 0);   // events derive too
```
Payload carries only what reactions can't read from stores. Keep events small.

### effect — external async work with a lifecycle
```ts
const searchFx = effect(async (text: string, { signal }) => {
  const res = await fetch(`/api/search?q=${text}`, { signal });
  if (!res.ok) throw new Error("failed");
  return (await res.json()) as string[];
});
```
Exposed units (react to them like any event/store):
`started`, `done` (`{params,result}`), `doneData` (result), `fail` (`{params,error}`), `failData` (error), `finally`/`settled`, `pending` (`Store<boolean>`), `inFlight` (`Store<number>`), `abort(reason)`, `aborted`.
```ts
reaction({ on: searchFx.doneData, run(items) { results.value = items; } });
// pending/inFlight publish IMMEDIATELY (not transactional) — UI loading is instant.
```
The handler gets an `AbortSignal`; disposing the owner that created a call aborts it. Use effects for **anything async** — never `await` inside a reaction body for domain work.

### reaction — rules (two forms)
```ts
// Explicit `on` — when the TRIGGER carries meaning (an event/effect/lifecycle unit).
reaction({ on: queryChanged, run(text) { query.value = text; } });
reaction({ on: [saved, cancelled], run() { modalOpen.value = false; } });

// Automatic dependency tracking — when the rule naturally FOLLOWS state.
reaction(() => { canSubmit.value = online.value && query.value.trim().length > 2; });
```
- Explicit `{ on, run }`: runs **only** when a listed unit fires; receives its payload; does **not** run at creation.
- Auto `reaction(fn)` / `{ run }`: runs once immediately, re-runs when any store it read changes; dependencies refresh each run (branch-aware).
- `reaction(...)` returns `{ stop() }`. Detach with `.stop()`, or register inside an `owner` to dispose together.

Pick explicit when the event is the reason; pick auto when "this value should always reflect that state."

### scope — where values live & boundaries
```ts
const appScope = scope();                              // fresh value map
const test = scope({ values: new Map([[count, 10]]) }); // seed values

// allSettled — explicit boundary; waits for async graph work to settle. Best for tests/SSR/loaders.
await allSettled(model.incremented, { scope: appScope, payload: 2 });

// scoped — short frame so plain code can read/write stores (incl. after await).
scoped(appScope, () => { count.value += 1; });
const run = scoped(appScope);          // reusable runner
await run(() => searchFx("docs"));

// scoped().wrap — capture a scope and reopen it when an EXTERNAL library calls you later.
socket.on("message", scoped(appScope).wrap((msg) => { messages.value = [...messages.value, msg]; }));

getCurrentScope();                     // Scope | null (active scope in context)
```
Rule of thumb: **`allSettled` at deliberate boundaries** (you have the scope and a unit), **`scoped` for plain code touching stores**, **`scoped().wrap` for callbacks handed to other libraries**.

### owner — lifetime & cleanup
```ts
const model = owner((dispose, owner) => {
  const tick = event<void>();
  const seconds = store(0);
  reaction({ on: tick, run() { seconds.value += 1; } });

  const id = setInterval(() => {/* call tick in scope */}, 1000);
  onCleanup(() => clearInterval(id));   // runs on dispose

  return { seconds, tick, dispose };
});

model.dispose();   // detaches reactions/subscriptions, aborts in-flight effects, runs onCleanup
```
Also: `getOwner()`, `withOwner(owner, fn)` (register cleanup on an existing owner), and `using model = owner(...)` via `Symbol.dispose` where supported. Any model that appears/disappears at runtime (modal, chat, tab, preview) should be created in an owner.

### lazyModel — code-split a model
```ts
const chat = lazyModel(() => import("./chat.model").then(m => m.createChatModel()));
reaction({ on: chat.opened, run(p) { currentId.value = p.chatId; } });  // subscribe before load
await allSettled(chat.opened, { scope: appScope, payload: { chatId: "support" } });
chat.pending;  // Store<boolean>, true while the module imports
```
You may subscribe to its events/effect-lifecycle units before load; store **reads** stay synchronous, so read lazy stores only after the model has loaded.

## Transactions — when writes become visible

- Synchronous unit calls share **one transaction**; writes update a draft, reads see the draft, commit happens once when the outer sync call finishes → subscribers notified once with the final value.
- **`await` is a transaction boundary.** Pending writes commit before the continuation; a fresh transaction starts after. Drafts are not kept alive across `await`.
- Effect `pending`/`inFlight` are runtime signals → they publish **immediately**, bypassing domain batching.
- **Sibling reactions on the same source have non-deterministic order.** Never encode business logic in registration order — instead use explicit nested calls (run now, in JS order) or react to the *committed* store value.

```ts
reaction({ on: opened, run() { first(); second(); } });
// first() runs fully (child drain) before second() — like normal JS calls.
```

## Decision guide

- Need to remember a value? → `store`. Derived from other stores? → `.map`/`computed`.
- Something happened / a command? → `event` (fact or intent).
- Async (fetch, timer, worker)? → `effect`; show loading via `effect.pending`, errors via `effect.fail`.
- "When X, do Y"? → `reaction({ on: X, run })`. "Y should always mirror state"? → `reaction(() => ...)`.
- Need an isolated run (test/SSR/widget)? → `scope()` + `allSettled`.
- Model lives only while a UI piece exists? → `owner` (+ the React/Vue `component`/`useModel` helpers, which create the owner for you).
- Big/rarely-used model? → `lazyModel`.

## Anti-patterns

- ❌ `setCount`/`updateName` events → ✅ `incremented`/`nameChanged`.
- ❌ `await fetch()` inside a reaction for domain work → ✅ an `effect` + reaction on `doneData`/`failData`.
- ❌ loading/error state stored in the component → ✅ `effect.pending` / `effect.fail`.
- ❌ global mutable value shared across instances → ✅ a `store` read in a `scope`.
- ❌ relying on which sibling reaction runs first → ✅ explicit nested calls or react to committed state.
- ❌ reading/writing `.value` in plain code with no scope → ✅ wrap in `scoped`/`allSettled`.

## Package map (use the matching skill)

- Rendering: **virentia-react**, **virentia-vue** (`ScopeProvider`, `useUnit`, `useModel`, `component`).
- Forms: **virentia-forms** (`createForm`, fields, validation, wizards).
- Routing: **virentia-router** (`createRoute`, `createRouter`, query tracking, React/RN views).
- Existing Effector code: **virentia-effector** (`associate`, `fool`).
- Devtools: **virentia-inspector** (`installVirentiaDevtools`, `connectEffector`).

Most app code should speak only stores/events/effects/reactions/scopes/owners. Drop to the kernel (`createNode`, `ctx.launch`) only when writing adapters, devtools, or a new primitive.
