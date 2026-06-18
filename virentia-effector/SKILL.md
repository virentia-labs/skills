---
name: virentia-effector
description: Interoperate with real Effector using @virentia/effector — associate (pair a Virentia scope with a forked Effector scope), fool (bridge a unit so it works on both sides), ensureAssociation. Use when integrating Virentia models into an existing Effector codebase, gradually migrating, or running both in SSR. Assumes the `virentia` core mental model and basic Effector (createEvent/createStore/createEffect, fork, sample, allSettled).
---

# @virentia/effector

Bridges **real Effector** (`effector ^23`) units to/from Virentia core units so both graphs run in the same logical run. It does **not** reimplement Effector — it wires the two scope systems together. Read the **virentia** skill first.

## When to use

- An existing Effector app that wants to adopt Virentia models incrementally.
- A Virentia model that must drive, or be driven by, Effector units (`sample`, effects).
- SSR where both a Virentia scope and an Effector `fork()` scope must share one request run.

## The two pieces

### associate — pair the scopes (do this once per run)
```ts
import { associate } from "@virentia/effector";
import { fork } from "effector";
import { scope as virentiaScope } from "@virentia/core";

const vScope = virentiaScope();
const eScope = fork();
associate({ virentia: vScope, effector: eScope });   // global weak-map link; lives while both scopes are reachable
```
Idempotent (same pair → same association). No `dispose()` — it's released when the scopes are GC'd. `ensureAssociation({ virentia?, effector? })` looks up an existing pair and throws if missing (use for invariants in bridged handlers).

### fool — make a unit speak both languages
```ts
import { fool } from "@virentia/effector";
```
`fool(unit)` returns a **new** bridge unit (the original is untouched) usable on **both** sides — in Effector `sample({ clock, source, target })`/`allSettled`, and in Virentia `reaction({ on })`/`scoped`. Works for Event, Store, Effect on either side.

Scope is resolved from the **current run**:
- Virentia → Effector: call the fool unit inside `scoped(vScope, …)`; the bridge finds the associated Effector scope and launches there.
- Effector → Virentia: trigger inside `allSettled(unit, { scope: eScope })`; the bridge finds the associated Virentia scope and runs the reaction there.
- No scope in context → it throws (no implicit/global scope is ever created).

## Patterns

**Virentia event → Effector graph**
```ts
const checkoutRequested = fool(event<{ orderId: string }>());
sample({ clock: checkoutRequested, source: $session, fn: (s, o) => ({ ...o, token: s.token }), target: billingStartedFx });
await scoped(vScope, () => checkoutRequested({ orderId: "x" }));
```

**Effector event → Virentia reaction**
```ts
const routeOpened = fool(createEvent<string>());
reaction({ on: routeOpened, run(route) { currentRoute.value = route; } });
await allSettled(routeOpened, { scope: eScope, params: "/users/1" });
```

**SSR — one request, both scopes**
```ts
const vScope = virentiaScope();
const eScope = fork();
associate({ virentia: vScope, effector: eScope });
await scoped(vScope, () => allSettled(appStarted, { scope: eScope, params: request }));
```

## Unit mapping & habits

| Virentia | Effector |
|---|---|
| `event<T>()` | `createEvent<T>()` |
| `store<T>(init)` | `createStore<T>(init)` |
| `effect(handler)` | `createEffect(handler)` |

- Always `associate` the two scopes **before** triggering any fooled unit.
- `fool` once per unit and reuse the result; don't fool the same unit repeatedly inside hot paths.
- Keep new domain logic in Virentia models; use `fool`/`associate` only at the seam.
- For devtools across the seam, see the **virentia-inspector** skill (`connectEffector`).
