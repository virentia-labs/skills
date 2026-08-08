---
name: virentia-models
description: Domain entities with @virentia/core/models — model/staticModel declarations, f.* field schemas (TypeBox-backed, composable with per-element codecs), traits (requires/data, the one-declarer rule, Impl<>), collections (upsert add, merge, batches), relations (refs/children/inverse, delete policies, zero-annotation cycles), indexed queries with referential stability, unions with exhaustive match, json()/rebind/forwarding aliases, instance onCleanup, pooling, and the React/Vue useModel/component integration (keep flag). Use when an app works with lists of server entities (todos, orders, documents, feed items) or when reviewing models code. Assumes the `virentia` core mental model.
---

# @virentia/core/models

The entity layer over Virentia core: an entity is declared **once** — fields, relations, behavior — and loading from the server, searching, editing, and serializing back all use that one declaration. Instances live in **collections** (get-or-create per scope), not in stores you manage by hand.

Reach for models when the domain is *entities with ids that come from and return to a server*. For a non-entity subsystem (socket manager, player, cache) use a plain core `owner`. If an `owner` grows an id registry and a serializer, it wanted to be a model.

```ts
import { collection, f, staticModel } from "@virentia/core/models";

const Todo = staticModel({
  data: {
    title: f.string(),
    done: f.boolean(false).indexed(),
  },
});

const todos = collection(Todo);           // inside an active scope

todos.add([{ id: "1", title: "docs" }]);  // validates, creates/merges
todos.where(Todo.done.eq(false)).items;   // indexed, reactive, identity-stable
todos.get("1")!.json();                   // back to JSON through the same declaration
```

**Install note:** `@sinclair/typebox` is an **optional peer** of core — only apps importing `@virentia/core/models` install it (`pnpm add @virentia/core @sinclair/typebox`). A missing peer fails module resolution at the subpath import.

## Two kinds, one surface

| | `staticModel` | `model` |
|---|---|---|
| units from `setup` | built ONCE, shared | built per instance |
| instance cost | a scope + a value map (pooled) | stores + reactions + owner |
| `setup` args | `(self)` | `(self, props)` — props = the raw `add` input (non-field keys visible too) |
| per-instance closures | **forbidden** (dev-throw) | legal |
| typical count | hundreds–thousands (list rows, feed items) | a handful (screens, editors) |

Default to `staticModel`; switch to `model` only when an instance genuinely needs its own dynamically created units. Declaration form and usage surface are identical — switching later is a one-word change.

### staticModel constraints (the rules agents break most)

- `setup` runs once with **no instance existing** — the instance API on `self` (`self.id`, `self.json()`, `self.dispose()`, `self.rebind()`, `self.onCleanup()`) works **only inside unit bodies** (reaction/effect runs), where an ambient instance exists. In the setup body itself it throws "needs an instance context".
- **No per-instance closure state** — a mutable `let` in static `setup` leaks state between instances. The sanctioned per-instance slot is a **local field**: `inFlight: f.any(null).local()`.
- **Callbacks that fire outside model context** (`setInterval`, `socket.onmessage`, DOM listeners) have no ambient instance — `self` reads/writes would hit the wrong place. **Capture the facade first**: `const me = collection(Model).get(self.id)!` inside the unit body; `me.field.value` routes correctly from anywhere.
- Reactions declared in static `setup` are still per-instance: an own event (`await a.toggled()`) runs for that instance only; a reaction on an **external app store broadcasts** — one run per live instance, each in its own instance scope. Writes route by ownership: `self.field` lands in the instance, app stores land in the app scope (an instance can never fork an app store).

## Fields: `f.*` (schemas + codecs, composable)

Every factory carries a TypeBox schema (validation of JSON input) and, where needed, a **codec** (JSON value ↔ model value). Combinators compose BOTH recursively:

```ts
f.string() f.number() f.integer() f.boolean() f.null() f.any() f.unknown()
f.date()                     // Date in the model, ISO string in JSON (codec built in)
f.literal(v)  f.enum([...])  // string AND number literals
f.array(item)                // f.list is an alias; item codec applies per element
f.tuple([a, b])  f.union([...])  f.intersect([...])
f.record(value)  f.object(shapeOfFields | rawSchema)
f.recursive((self) => item)  // trees in one value
f.from(schema)               // any TypeBox schema as a leaf
f.arg(n)                     // generic placeholder for parameterized traits
```

- Any combinator item may be an `f.*` field **or** a raw TypeBox schema.
- `f.array(f.date())` → `Date[]` in the model, `string[]` in JSON — per element; `f.object({due: f.date()})` per key; `f.tuple` per position. Plain shapes allocate no codec.
- **Transforms inside `f.union`/`f.intersect`/`f.recursive` throw at declaration** (the branch to convert is undecidable) — put `.map` on the whole field.

Modifiers: default-as-argument (makes the JSON key optional), `.optional()` (null allowed both sides), `.indexed()` / `.indexed("ord")`, `.unique()` (secondary key: "find by slug AND uuid" is a second `.indexed().unique()` field, never a second id), `.local()`, `.meta({...})` (annotations merged into schema options — the channel for custom binary codecs), `fn<Sig>()` (trait behavior requirement, not a field).

**Bindings (the codec IS the field declaration).** Object-form `data` binds JSON keys by field name. Function form binds explicitly and adds **locality**:

```ts
data: (p: Props<{ name: string; is_done?: boolean }>) => ({
  title: f.string(p.name),               // JSON key "name"
  done: f.boolean(p.is_done.or(false)),  // optional with default
  draft: f.string(""),                   // NO p.* binding → LOCAL: not in add, not in json(), not in Dto
}),
```

`p.key.map(parse, serialize)` / `.inOnly()`. **A user `.map` takes over the whole conversion** — it receives the raw JSON value, owns both directions, and schema validation of that key is skipped (the map owns its wire shape). A one-way map without `.inOnly()` drops the field from `json()` with a warning.

Reserved names (declaration-time error): `id key alive json toJSON dispose rebind onCleanup create name kind fields relations inverses behaviors traits staticMembers setup locals fieldOrigins`. Yes — **`name` is reserved**.

## Traits

Part interface, part abstract class; identity = the object reference (no string names anywhere in models).

```ts
const Selectable = trait({
  requires: { selected: f.boolean(), canSelect: fn<() => boolean>() },
  setup(self) {
    const toggle = event<void>();
    reaction({ on: toggle, run: () => { if (self.canSelect()) self.selected.value = !self.selected.value; } });
    return { toggle };
  },
});
```

- **One-declarer rule:** any number may *require* a name; exactly one participant *declares* it (trait data, model data, or a setup return). Collisions, undeclared field requirements, and unimplemented behaviors are errors.
- Everything a trait declares/returns flows into `self` and the instance — no `use(Trait, x)` indirection exists.
- `Impl<typeof Selectable>` — the instance-side constraint for generic code.
- Parameterized: `f.arg(0)` in trait data + application `Keyed(f.string())`; the applied trait keeps the base identity (matters for unions).

## Collections & instances

- `collection(Model)` — get-or-create per scope; scopes hold independent populations. `Model.create(props)` = sugar for the active scope's collection.
- **`add` is an upsert**: unknown id creates (validating required keys + schemas BEFORE any write), known id merges (present keys win, absent untouched), `{replace: true}` resets absent optionals to defaults and refuses to null required keys. Arrays are atomic (validated up front; mid-apply failure rolls back this batch's creations). `add` matches **real ids only** — never forwarding aliases.
- Instance surface: fields as stores (`t.title.value`), members as callables, plus `id`, `key`, `alive`, `json()`, `dispose()`, `rebind(newId)`, `onCleanup(fn)`.
- `t.alive` is **reactive and never throws** — a computed over it degrades cleanly after dispose. Reads of fields on a disposed instance throw `entity was disposed — disposed at <site>` (the dispose site is captured).
- **Pooling:** disposed static instance scopes are reused. `t.key` = slot+generation — survives `rebind`, never repeats across reuse; stale handles throw instead of reading the next occupant. Use `t.key` as the UI list key, never `t.id`.

### `onCleanup` — external resources per instance

`instance.onCleanup(fn)` ties a socket/marker/listener/unsubscribe to the instance's lifetime. Returns an unregister fn. Runs at dispose **after the restrict gate** (an aborted dispose runs nothing) and **before teardown** (fields still readable). A throwing cleanup is reported, never blocks disposal. Works: from unit bodies via `self.onCleanup` (static — ambient instance), right in dynamic `setup`, and from outside on the facade. Patterns:

- Static latest-wins abort: controller in a **local field** slot; supersede aborts previous; `const release = self.onCleanup(() => controller.abort())` + `release()` in `finally` so registrations don't pile up per run.
- Writes to `self` **after `await`** in a reaction land on the right instance — scope survives suspension per instance.
- Dynamic `model`: unit subscriptions made in `setup` are owner-attached and cleaned automatically; `onCleanup` is for non-unit resources (AbortController, WebSocket).

## Relations

```ts
refs.one(target)  refs.many(target)          // association — stores ids, serializes ids
children.one(target)  children.many(target)  // composition — created THROUGH the parent, cascade on dispose, serialize EMBEDDED
inverse(() => Model.field)                   // storage-less view over the owning side; link/unlink, never serializes
```

- Target forms: value | thunk `() => Model` | `Self`. Thunks resolve at first navigation → **mutual cycles need zero annotations** and are fully typed (`post.author.value?.login.value`).
- Inverse cardinality is derived: `children.*` and `refs.one(...).unique()` invert to a single `value`; otherwise a query-like list. many-to-many = `refs.many` + `inverse`; an edge with data is an explicit model with two `refs.one`.
- Delete policies on `refs`: default `nullify`; `.policy("restrict")` aborts dispose while referenced; `"orphan"` leaves ids (reads → null). Children need no policy — they die with the parent. Direct `collection(Child).add` throws for child-owned models.
- A relation may target a **union** (value or thunk). Instance writes store (variant, id) pairs — precise under cross-variant id collisions (policies/rebind touch only the right variant). JSON keeps plain ids; an id found in TWO variants fails at read with a clear error.
- Load order never matters: an fk before its target reads null, then resolves; a children array reconciles (known ids merge, absent are disposed, array order authoritative).

## Queries & indexes

Descriptors live on the model: `Todo.done.eq(false)`, ops `eq neq gt gte lt lte between startsWith includes`, sort `.asc/.desc`; a plain function is a residual scan. Terminals: `items ids count first select(set-of-field-values) set(mass write) remove()(mass dispose) toArray`, iterable. Descriptors are model-bound — a foreign descriptor throws, not "empty result".

- `.indexed()` → eq rides a hash bucket; `.indexed("ord")` → ranges/sort ride a sorted view (rebuilt lazily: a thousand writes per frame = one rebuild). Maintained on write — never stale.
- Terminals are **reactive reads**: computeds/reactions over `count/items/ids` re-run when the result can change.
- **Referential stability:** plans are interned by shape, values are bind parameters. A chain rebuilt every render returns the SAME arrays while the result is unchanged — no deps arrays, no memoization, and UI re-renders only when the set changes.
- `collection.get(id)` is a **reactive single-instance view**: re-resolves on add/dispose/rebind inside computeds/reactions, resolves forwarding aliases, returns null (never throws mid-read).

## Unions

```ts
const FeedItem = union(Post, Ad, Story).by((json) => "likes" in json ? Post : "budget" in json ? Ad : Story);
const feed = collection(FeedItem);   // a VIEW — instances live in variant collections, both directions visible
```

- `by` runs for **unknown ids only** (full objects — shape discrimination is sound). A known id merges into its variant WITHOUT `by`, so partial updates don't need discriminating keys. A full dto of another variant with a taken id = "variants do not migrate" error. **No `.by` → no `add`** (types AND runtime).
- **Common members**: a field is shared iff EVERY variant carries it **from the same trait by reference** — name coincidence gives nothing (enforced in types and at runtime). Common predicates translate per variant and ride each variant's own indexes; sort merges (k-way).
- `match([Post, p => p.likes.gte(100)], [Ad, a => a.active.eq(true)], [Story, () => true])` — **exhaustive by model references** (missing variant = compile error naming it + runtime error); branch callback sees ITS variant's descriptors; returns predicate or boolean literal. Chains with where/sort/take.
- Items are typed as the union of variant instances; narrow with `in` (`if ("likes" in item)`)

## JSON in/out, ids, aliases

- `json()`/`toJSON`: bound fields under wire keys through out-transforms; refs as ids; children embedded; local fields/inverses/members never.
- **Temporary ids never serialize** — an instance created without an id gets `~tmpN`, usable everywhere locally, omitted from `json()` (the server assigns the real one).
- Optimistic flow: `add({...})` → navigate with temp id → `api.save(t.json())` → `t.rebind(saved.id)`. Rebind rewrites every fk via reverse indexes and leaves a **forwarding alias** old→new: `get(oldId)` (and `useModel(todos.get(oldId))`) keeps resolving. Aliases are lookup-only (never in indexes/queries/json), die with the instance, chain flat, and a real id always wins; `add` with an id equal to a live alias takes the spelling over (dev warning). There is no `addAlias` — an alias is always the trace of a rename.

## React / Vue bindings (no new hooks)

- `useModel(query)` — chain built inline per render is free (interned plans); re-renders **only when the result set changes**. Row content must subscribe itself: `<TodoRow>` with `useModel(item)` inside — a list-level `t.title.value` goes stale on rename by design. Row key = `t.key`.
- `useModel(todos.get(id))` — entity view: follows own writes/rebind/dispose, null-safe. Vue returns refs (`active.value.items`).
- `useModel(Screen, props, { keep })` / `component({ model: Screen, view, keep })` — the definition is a props→instance factory through its collection: `add`-upsert, props re-merge (present keys win), unmount disposes. `keep` flips only the end-of-life owner: remount finds the per-scope singleton (no id; **several live = dev error**, not a silent pick) or the instance by `props.id`. No cache/key options — the collection IS the cache. `component.create()` = controlled instance, the view never disposes it.

## Sharp edges checklist

- `name` is a reserved field name; so are `relations`, `inverses`, `onCleanup`, the whole instance API.
- Static setup body: no instance API on `self`, no mutable closures — local fields are the per-instance slot.
- Timers/socket callbacks: capture the facade (`collection(M).get(self.id)!`), never use `self` there.
- `.map` on a field disables schema validation for that key — the map owns its wire shape.
- `f.union`/`intersect`/`recursive` reject items with codecs (dates, maps) — `.map` the whole field.
- Mass dispose is `query.remove()` on a DERIVED query; `collection.remove(id)` is targeted; `collection.remove()` without id is a no-op.
- Inverse linking is `link`/`unlink` (`add`/`remove` on inverse views don't exist — `remove` on queries means mass-dispose).
- `keep` without id + several live instances throws; pass the entity id from the route.
- TypeBox is an optional peer — importing the subpath without it installed fails resolution.
