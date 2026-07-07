---
name: virentia-mutable
description: The @virentia/mutable deep-mutable store — mutableStore(initial), whose .value is a copy-on-write draft you mutate in place, with per-keypath reactivity. Reach for it ONLY when a store holds large or deeply nested state that is edited in place and often (editor/canvas/rich-text documents, big tables patched cell by cell, deep forms). For ordinary or flat state, or when you rely on value identity, a plain `store`/`reactive` + `computed` is the default — do not migrate existing stores to mutable without a concrete deep-editing reason. Assumes the `virentia` core mental model.
---

# @virentia/mutable

A store for `@virentia/core` whose `.value` is a **deeply mutable object** — `state.value.a.items.push(x)`, `state.value.count++`, `delete state.value.a.flag`. Read the **virentia** skill first: it is a normal store (per-scope value, transactional writes, observed by `computed`/`reaction`/`subscribe`/`map`), only mutated in place instead of replaced.

`pnpm add @virentia/mutable` (depends on `@virentia/core`).

## Use it only when it earns its place

This is a **specialized** store, not a default. Reach for it when the pain is **deep, in-place, frequent** editing:

- ✅ An editor/canvas/rich-text document, a big table patched cell by cell, a deeply nested form — where `doc.value.blocks[3].items[7].text = "…"` beats rebuilding that path by hand.
- ❌ Ordinary or flat state, or state you mostly **replace** — a plain `store` or `reactive` + `computed` is simpler and gives **stable value identity**.
- ❌ Don't migrate existing `store`s to `mutableStore` "for convenience". Introduce it for a specific structure that is genuinely painful to update immutably.

When unsure, use a plain `store`.

## Mental model

- `.value` is a **copy-on-write draft** (a `Proxy`). Mutating it copies **only the touched node** and threads the copy up its parents; untouched branches stay shared by reference. No `structuredClone`, no snapshots.
- **Ownership:** each scope remembers nodes it already copied and mutates them **in place** on later writes — so repeated edits to the same path are plain assignments (this is why it beats immer/mutative on repeated deep updates).
- **Commit at the transaction boundary** (immediately for a `scoped(...)` change, batched inside a reaction/effect), like any store write. Notification is **forced** (an in-place mutation may not change identity).
- **Leaves:** only plain objects and arrays are tracked deeply. `Date`, `RegExp`, `Map`, `Set`, and class instances are leaves — read raw, **replaced wholesale** (`state.value.when = new Date()`); mutating *into* a leaf (`state.value.when.setHours(0)`) is not tracked.

## API

```ts
import { mutableStore, seedMutableStore, unwrap } from "@virentia/mutable";

const doc = mutableStore({ blocks: [] as Block[], title: "" });
```

- `mutableStore(initial)` → `MutableStore<T>`: `.value` (get: mutable draft; **set: replace wholesale**), `.node`, `.subscribe(fn)`, `.map(fn)`.
- `seedMutableStore(scope, store, value)` — seed a scope's value (tests, SSR).
- `unwrap(value)` — the raw object behind a mutable proxy (for `===`, structuralClone, passing to non-Virentia code).

Mutate inside a scope, like any store — a reaction body, an effect, or `scoped(scope, …)`:

```ts
reaction({
  on: itemAdded,
  run(item) {
    doc.value.blocks[0].items.push(item); // in place — nested arrays/objects
  },
});
```

## Granular reactivity (per keypath)

A `computed`, `map`, or **automatic** reaction subscribes only to the keypaths it actually read, so mutating one branch re-runs only readers of that branch:

```ts
const count = computed(() => cart.value.items.length);
const coupon = computed(() => cart.value.coupon);

cart.value.items.push(item); // re-runs `count` only — `coupon` never read `items`
```

- Reading `cart.value.items[3].text` subscribes to `items`, `items[3]`, `items[3].text` (every prefix). Replacing an ancestor invalidates deep readers; a sibling edit (`items[4]`) does not.
- **Coarse by design** (re-fire on *every* commit): `store.subscribe(...)`, `useUnit(store)` on the whole value, and `unwrap(store.value)` — they read the entire value. In a component, subscribe to a **slice** with `store.map(sel)` / `computed` so it re-renders granularly.

## Habits / gotchas

- **Identity is not stable.** In-place mutation can leave `store.value`'s reference unchanged across a change. Never diff snapshots with `===`; depend on the parts you read, or use a plain `store` when you need value identity.
- **Your assigned object is not mutated.** `state.value.ref = external` then editing `state.value.ref.x` copies-on-write on descent — the object you passed in stays intact. Use `unwrap` if you need the live internal object.
- **Whole-value reads are coarse** — narrow with `.map`/`computed` (see above) rather than `useUnit(store)` when a consumer needs one slice.
- **Leaves replace, not mutate** — swap a whole `Date`/`Map`/`Set`; mutating into one is invisible to reactivity.
- **`.value =` replaces** the whole value (deferred to commit like a mutation); it does not merge.
- Built on `@virentia/core/internal` — it is a real graph node, so seeding, per-scope values, and cleanup work exactly like a normal store.
