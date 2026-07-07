---
name: virentia-storage
description: Persist Virentia stores with @virentia/storage-core — the storage boxes (local, session, query, memory, custom), the persist binding (hydrate + two-way sync), per-scope binding, owners, and custom serialization. Use when a store must survive reloads, sync across tabs, live in the URL, or be saved anywhere. Assumes the `virentia` core mental model.
---

# @virentia/storage-core

Persist a Virentia store into a **box** — `localStorage`, `sessionStorage`, the URL query string, memory, or your own — with two-way sync. Read the **virentia** skill first. State stays a model: a `store` remembers a value, `persist` mirrors that value into a box for one `scope`. The model never learns where it is stored.

`@virentia/core` is a peer dependency: `pnpm add @virentia/storage-core @virentia/core`.

## Mental model

- A **box** is a tiny backend (`get`/`set`/`remove`/optional `watch`). Values crossing it are already **deserialized** — a string-backed box (Web Storage, URL) serializes internally; `memory` keeps references.
- **`persist`** binds one **writable store** to one box for one **scope**. Persistence is inherently per-scope: one browser has one `localStorage`, so a binding pairs exactly one scope with the box.
- Sync is **two-way**: hydrate on start, write on store change, pull external changes back in (when the box has `watch`). A `busy` guard breaks the write↔watch loop.

## Boxes

```ts
import { local, session, query, memory, custom } from "@virentia/storage-core";
```

| Box | Backend | Survives | `watch` (external sync) |
|-----|---------|----------|-------------------------|
| `local(opts?)` | `localStorage` | reloads, restarts | other tabs (`storage` event) |
| `session(opts?)` | `sessionStorage` | reloads (per tab) | — |
| `query(opts?)` | URL `?key=value` | URL / history | back/forward (`popstate`) |
| `memory(seed?)` | in-process `Map` | the session | same process |
| `custom(impl)` | anything you supply | up to you | up to you |

- **SSR-safe:** `local`/`session`/`query` probe their environment and fall back to `memory()` when `window`/Web Storage is missing (server, worker) or blocked (private mode). The same model code runs on the server unguarded; nothing persists there, which is correct for a per-request run.
- **`query` writes don't echo:** it uses `history.replaceState`/`pushState`, which don't fire `popstate`, so a write never re-enters as an external change. `watch` catches back/forward only. Use `query({ history: "push" })` when each change is its own history entry.
- **`custom` is the extension point** the built-ins are built on:
  ```ts
  const cookies = custom({
    get: (key) => readCookie(key),
    set: (key, value) => writeCookie(key, value),
    remove: (key) => deleteCookie(key),
    // optional: watch(key, listener) => () => void
  });
  ```

## persist

```ts
import { scope, scoped, store } from "@virentia/core";
import { local, persist } from "@virentia/storage-core";

const theme = store<"light" | "dark">("light");
const app = scope();

scoped(app, () => {
  persist({ source: theme, key: "theme", storage: local() });
});
```

What it does, in order:
1. **hydrate** — if the box has `key`, seed the store from it; else seed the box from the store's current value.
2. **store → box** — write on every committed change **in the bound scope** (a change in another scope is ignored).
3. **box → store** — pull external changes back in when the box supports `watch`.

### Options

```ts
persist({
  source,       // StoreWritable<T> — REQUIRED, must be writable (not a .map/computed store)
  key,          // string
  storage,      // StorageBox
  scope,        // Scope — defaults to the current active scope
  serialize,    // (value: T) => unknown — value → stored form
  deserialize,  // (raw: unknown) => T — stored form → value
});
```

- **Scope:** defaults to `getCurrentScope()`, so call `persist` inside `scoped(scope, …)`. Pass `{ scope }` explicitly in setup code that holds the scope but isn't in a frame. **No scope and no `{ scope }` → it throws.**
- **Returns `stop()`** — detaches both directions. Inside an `owner`, the binding also tears down on dispose (so a modal/tab/preview model stops persisting when it goes away).
- **Custom serialization** covers values a box can't round-trip (a `Date`), running on top of the box's own serialization:
  ```ts
  persist({
    source: lastSeen, key: "lastSeen", storage: local(),
    serialize: (d) => d.toISOString(),
    deserialize: (raw) => new Date(raw as string),
  });
  ```

## Serializers

String-backed boxes take a `serializer` (`{ read(raw), write(value) }`):
- `local`/`session` default to **`jsonSerializer`** (strict JSON).
- `query` defaults to **`querySerializer`** — strings pass through verbatim for readable URLs, everything else is JSON; on read it tries JSON then falls back to the raw string (`?q=docs` → `"docs"`, `?page=2` → `2`).

A box that can't parse a stored string returns `undefined` from `get` rather than throwing.

## Habits / gotchas

- **`source` must be writable** — a derived store (`.map`, `computed`) can't be persisted; persist the source store instead.
- **Bind per scope.** Wrap `persist` in `scoped(scope, …)` or pass `{ scope }`. It never infers a global scope.
- **Register under an `owner`** for runtime models so the binding is cleaned up; otherwise keep the returned `stop()` and call it yourself.
- **URL state → `query`.** Bind search/filter/page stores to `query()` to make them shareable and back/forward-aware. Prefer `replace` (default) so persisting UI state doesn't spam the back button.
- **Don't reach for `custom` first** — the four built-ins cover reloads, per-tab, URL, and memory. Use `custom` for cookies, an `AsyncStorage`-like sync bridge, or a test double.
- **The box owns serialization; `serialize`/`deserialize` are the store-side transform.** Don't double-encode (e.g. `JSON.stringify` in `serialize` on top of `jsonSerializer`).
