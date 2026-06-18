---
name: virentia-inspector
description: Wire up Virentia devtools with @virentia/inspector — installVirentiaDevtools for Virentia apps, connectEffector (from @virentia/inspector/effector) for Effector apps, naming units for a readable graph, and the graph/timeline the inspector surfaces. Use when setting up the inspector, debugging the reactive graph, or making units show up with good names. Assumes the `virentia` core mental model.
---

# @virentia/inspector

A devtools bridge + UI that visualizes the reactive graph (nodes = units, edges = reactive/ownership links), records a timeline of unit calls, lets you trigger units, and supports breakpoints. Works for Virentia apps and, over the wire, for Effector apps.

## Run the inspector UI

```bash
pnpm add -D @virentia/inspector
pnpm exec virentia-inspector        # serves the UI + WebSocket relay (default http://127.0.0.1:5174)
```
Flags: `--port`, `--host`, `--open`.

## Connect a Virentia app

The app-side bridge and the naming helpers live in **`@virentia/core/devtools`** (the `@virentia/inspector` package is the CLI/UI + the `/effector` subpath).

```ts
import { installVirentiaDevtools } from "@virentia/core/devtools";

if (import.meta.env.DEV) {
  const devtools = installVirentiaDevtools({
    appName: "MyApp",
    autoOpen: true,
    channel: "myapp-local",   // must match the inspector's ?channel=
  });
}
```
Bridge methods: `sendGraph()` (refresh the graph), `setBreakpoints(nodeIds)`, `snapshot()`, `open()`, `dispose()` (use in tests/HMR teardown).

## Connect an Effector app (over the wire)

```ts
import { connectEffector } from "@virentia/inspector/effector";

if (import.meta.env.DEV) {
  const conn = connectEffector({
    appName: "MyEffectorApp",
    units: [model.itemAdded, model.itemCount, model.loadPriceFx],  // root units to wire the graph
    scopes: [{ scope: appScope, name: "checkout tab" }],           // forked scopes to watch
    autoOpen: true,
    channel: "myapp-local",
  });
}
```
`connectEffector` **requires you to hand it the units (and scopes)** you want surfaced — there's no global auto-discovery. Add more later with `conn.addUnits([...])` / `conn.addScope(scope, name)`. (Effector limitations vs Virentia: per-step duration shows as 0, breakpoints are echo-only.)

## Name your units (do this for a usable graph)

At creation, pass a name via the unit's devtools options/name argument:
```ts
const itemAdded = event<{ id: string }>("cart.itemAdded");        // event takes a name string (or {name})
const itemCount = store(0, undefined, { name: "cart.itemCount" }); // store: devtools opts is the THIRD arg (2nd is skipToken)
const loadPriceFx = effect(async (id) => fetchPrice(id), { name: "cart.loadPriceFx" }); // effect takes name string or {name}
reaction({ name: "cart.apply", on: itemAdded, run() { itemCount.value += 1; } });        // reaction config takes name
```
Or name retroactively (devtools helpers):
```ts
import { nameUnit, nameScope, describeUnit } from "@virentia/core/devtools";
nameUnit(itemCount, "cart.itemCount");
nameScope(appScope, "checkout tab");
describeUnit(loadPriceFx, { name: "cart.loadPriceFx", description: "Loads current price" });
```
Use a consistent `feature.unit` naming scheme — the graph and timeline read by these names.

## What the UI gives you

- **Graph canvas** — solid edges = reactive (A can trigger B), dashed = ownership (effect lifecycle subunits). Right-click a node → **Trigger** (runs real model code with a JSON payload). "Show isolated" reveals named-but-disconnected units.
- **Timeline** — records every unit call when recording is on: name, type, scope, duration, payload/result preview; failed/stopped chains flagged.
- **Scope selector** — inspect state per scope (app instance, request, cached screen).
- **Breakpoints** — pause a chain after a selected node (Virentia only).

## Habits / gotchas

- Guard the install behind `import.meta.env.DEV` (or your dev flag); call `dispose()` on HMR teardown to avoid duplicate bridges.
- The app's `channel` must match the inspector window's channel.
- Name units as you build — an unnamed graph is hard to read; unnamed nodes only show via "Show isolated".
- For Effector apps you must explicitly pass `units`/`scopes` to `connectEffector`; nothing appears otherwise.
