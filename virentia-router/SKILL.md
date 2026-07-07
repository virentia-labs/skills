---
name: virentia-router
description: Routing with @virentia/router — typed path templates, routes (params/opened/closed/beforeOpen), router + history adapter, navigation, query tracking, virtual/chained/grouped routes, and React (RouterProvider, route views, Outlet, Link, hooks) + React Native (stack/bottom-tabs) integration. Also covers migrating from argon-router. Use when adding or reviewing routing in a Virentia app. Assumes the `virentia` core mental model.
---

# @virentia/router

A scope-based router: routes are **models** (their state lives in stores), navigation is an event run in a scope, rendering is the React/RN layer's job. Read the **virentia** skill first.

**Key habit:** opening a route is async and scope-bound — `await scoped(scope, () => route.open(...))`. There is no synchronous global navigation.

## Path templates (@virentia/router-paths)

```ts
"/users/:id"            // { id: string }
"/posts/:id<number>"    // { id: number }
"/docs/:tab<overview|api>"  // union
"/users/:id?"           // optional
"/files/:path+"         // ≥1 segments (array); :path* = ≥0
```
Params are inferred into the route's type. Use at module level.

## Routes (core model)

```ts
import { route } from "@virentia/router";

const profileRoute = route({ path: "/users/:id<number>" });        // path route
const settingsRoute = route({ parent: profileRoute });             // nested (pathless or path)
const guarded = route({ path: "/admin", beforeOpen: [checkAuthFx] }); // guard/preload
```

Route shape: `params: Store<Params>`, `isOpened: Store<boolean>`, `isPending: Store<boolean>`, events `open` (call it to navigate), `opened`, `openedOnClient`, `openedOnServer`, `closed`.

These are core stores, so outside React you read them through **`.value` inside a scope**: `scoped(scope, () => profileRoute.params.value.id)`, `appRouter.query.value`, `appRouter.activeRoutes.value[0]`. In components use `useUnit` instead (it unwraps to a plain value).

```ts
await scoped(appScope, () =>
  profileRoute.open({ params: { id: 42 }, query: { tab: "posts" }, replace: false })
);
```

- **`beforeOpen`** on `route` is an **array** of guards/preloaders (each an effect, event, or function), run once per activation — auth, data preload, redirects (`await otherRoute.open({ replace: true })`). Each receives `{ params, query }`. Guards run only for **external** (history-driven) activations; a programmatic `route.open`/`navigate` runs its guards once up front, and the resulting URL echo (a `programmatic` origin) skips them — so a guard never fires twice for one navigation. (`route` requires the array form; the single-unit-or-array form is `chainRoute`-only.)
- **Parent routes** open with their child; nesting is model-only — rendering nesting is done with `Outlet`.

## Router + history

```ts
import { router, historyAdapter } from "@virentia/router";
import { createBrowserHistory } from "history";

const appRouter = router({ routes: [profileRoute, settingsRoute /*, nestedRouter */] });

await scoped(appScope, () => appRouter.setHistory(historyAdapter(createBrowserHistory())));
```
Router shape: stores `path`, `query`, `history`, `activeRoutes`; events `navigate`, `back`, `forward`, `setHistory`, `dispose`. The **app owns** the history object and hands it in (use `queryAdapter` for hosted widgets, `createMemoryHistory` for RN/tests). Nested routers inherit the parent's history; `base` scopes a child router's path space. (`routerControls()` is the lower-level shared-history primitive `router` builds on.)

## Navigation

```ts
route.open({ params, query?, replace? })                  // preferred — typed, opens parents too
appRouter.navigate({ path?, query?, replace? })           // raw path/query update
appRouter.back(); appRouter.forward();                    // history nav
await scoped(scope, () => route.open(payload));           // tests / SSR loaders
```

## Query tracking — dialogs, filters, tabs in the URL

```ts
const inviteDialog = appRouter.trackQuery({
  forRoutes: [teamRoute],                 // optional; default = all routes
  parameters: { safeParse: (q) => ("dialog" in q ? { success: true, data: q } : { success: false }) },
});
inviteDialog.entered;  // Event<T> — query matched
inviteDialog.exited;   // Event<void> — no longer matches
inviteDialog.enter(data);                 // write the query
inviteDialog.exit({ ignoreParams: [] });  // clear (optionally keep some params)
```
`parameters` accepts **any** `{ safeParse(query) }` object (zod-compatible but not required).

`entered`/`exited` are also split by **origin**, so you can react to router-driven and location-driven changes separately: `enteredExternally`/`exitedExternally` (URL changed from the outside — initial load, back/forward, manual URL) vs `enteredProgrammatically`/`exitedProgrammatically` (the router changed the query — `enter`/`exit`, `route.open`, or `navigate`). `entered`/`exited` still fire in both cases; origin is classified structurally (the router recognizes the history echo of a URL it just wrote), not threaded through payloads.

## Virtual, grouped & chained routes

```ts
const modal = virtualRoute<Payload>();                    // manual open/close, no URL
const section = group([usersRoute, teamsRoute]);          // opened while ANY input route is open
const afterAuth = chainRoute({              // wait for an async gate before "really" opening
  route: dashboardRoute,
  beforeOpen: ensureSessionFx,             // single unit or array (chainRoute-specific)
  openOn: ensureSessionFx.doneData,        // REQUIRED — fires the chained open; omit it and it never opens
  cancelOn: [logoutRequested],             // fires close + the route's `cancelled` event
});
```

## React integration (@virentia/router-react)

```tsx
<ScopeProvider scope={appScope}>
  <RouterProvider router={appRouter} history={historyAdapter(createBrowserHistory())}>
    <App />
  </RouterProvider>
</ScopeProvider>
```

Views:
```tsx
const profileView = routeView({ route: profileRoute, view: ProfilePage, layout: AppLayout });
const Routes = routesView({ routes: [profileView, /* ... */], otherwise: NotFound });

// lazy page
lazyRouteView({ route: profileRoute, view: () => import("./ProfilePage"), fallback: Skeleton });

// nested via Outlet (parent route is itself a wrapper)
routeView({ route: parentRoute, view: () => <Layout><Outlet/></Layout>, children: [childView] });
// shared layout for siblings: withLayout(Layout, [viewA, viewB])
```
Inside a view, read params with `useUnit(route.params)`.

Links & hooks:
```tsx
<Link to={profileRoute} params={{ id: 42 }} query={{ tab: "posts" }} replace>Profile</Link>

useRouter();                       // the Router
useIsOpened(route);                // boolean
useOpenedViews(routeViews);        // ordered parent→child
const { path, open } = useLink(profileRoute, { id: 42 });  // for custom clickable elements
```

## React Native (@virentia/router-react-native)

Same providers, but use `createMemoryHistory` and wrap in `NavigationContainer`.
```ts
const { Navigator } = stackNavigator({ router: appRouter, routes: [routeView({ route: homeRoute, view: HomeScreen })], initialRouteName: "/home" });
const { Navigator } = bottomTabsNavigator({
  router: appRouter,
  routes: [{ route: homeRoute, view: HomeScreen }, { route: profileRoute, view: ProfileScreen, openPayload: { params: { id: "42" } } }],
});
```
A tab press calls `route.open` with its `openPayload`.

## Migrating from argon-router

- Packages: `@argon-router/{paths,core,react}` → `@virentia/router{-paths,,-react}`.
- Drop the `$` prefixes: `route.$params/$isOpened/$isPending` → `route.params/isOpened/isPending`; `router.$path/$query/...` → `router.path/query/...`.
- `createEvent/createStore/createEffect` → `event/store/effect`; `Provider` → `ScopeProvider` (+ `RouterProvider`).
- Navigation becomes async + scope-bound: `route.open({...})` → `await scoped(scope, () => route.open({...}))`.
- The app now **owns** history: `createBrowserHistory()` → `appRouter.setHistory(historyAdapter(...))`.
- `trackQuery` is schema-agnostic (any `safeParse`), not zod-locked, and splits `entered`/`exited` by origin (`*Externally`/`*Programmatically`).
- `beforeOpen` runs once — a programmatic open's URL echo is recognized as `programmatic` origin and skips guards, so no double-fire.
- Factory names are bare nouns (no `create` prefix): `route`, `router`, `virtualRoute`, `routerControls`, `routeView`, `routesView`, `lazyRouteView`, `stackNavigator`, `bottomTabsNavigator`.

## Habits / gotchas

- Always open routes through a scope (`scoped`); never assume a global current scope.
- Prefer `route.open` over `router.navigate` — it's typed and opens parent routes.
- Read route state via stores (`useUnit(route.params)` / `route.isOpened`) — don't parse the URL yourself.
- Put auth/preload in `beforeOpen`, not in components.
