# Virentia skills

Agent Skills for the [Virentia](https://movpushmov.dev/virentia) state manager. Each folder is a skill (a `SKILL.md` with `name`/`description` frontmatter + guidance) in the standard skills.sh / Claude Agent Skills format. Loaded by name+description; the full body loads when a skill matches the task.

## Skills

| Skill | Use it for |
|-------|-----------|
| [`virentia`](virentia/SKILL.md) | **Start here.** The mental model + `@virentia/core` API (stores, events, effects, reactions, scopes, owners, transactions, lazy models). Every other skill builds on it. |
| [`virentia-react`](virentia-react/SKILL.md) | Rendering models in React (`ScopeProvider`, `useUnit`, `useModel`, `component`, caches). |
| [`virentia-vue`](virentia-vue/SKILL.md) | Rendering models in Vue 3 (mirrors React; stores come back as refs). |
| [`virentia-forms`](virentia-forms/SKILL.md) | `@virentia/forms` — fields, forms, validation, error channels, wizards, zod adapters, React hooks. |
| [`virentia-router`](virentia-router/SKILL.md) | `@virentia/router` — routes, history, navigation, query tracking, React/RN views; argon-router migration. |
| [`virentia-net`](virentia-net/SKILL.md) | The net remote-data layer, end to end: `@virentia/net-core` (`query`/`mutation` as effects, `trigger`, concurrency/retry/timeout/debounce/fallback/tap operators, TanStack/Apollo adapters, `overrideDefaults`) **plus** the framework bindings — `@virentia/net-react` & `@virentia/net-vue` (`useQuery`/`useMutation`). |
| [`virentia-storage`](virentia-storage/SKILL.md) | `@virentia/storage-core` — persist stores into boxes (local, session, query, memory, custom) with two-way sync. |
| [`virentia-mutable`](virentia-mutable/SKILL.md) | `@virentia/mutable` — a deep-mutable store (`mutableStore`) with per-keypath reactivity. Specialized: reach for it only when state is large or deeply nested and edited in place (editors, big tables, deep forms); a plain `store` + `computed` is the default. |
| [`virentia-effector`](virentia-effector/SKILL.md) | `@virentia/effector` — bridge real Effector units (`associate`, `fool`). |
| [`virentia-inspector`](virentia-inspector/SKILL.md) | `@virentia/inspector` — devtools bridge, `connectEffector`, unit naming. |

## How they relate

`virentia` is foundational — read it before any package skill. The package skills assume that mental model and only cover their own surface. `virentia-react`/`virentia-vue` share the same model/component/cache semantics; `virentia-forms` and `virentia-router` are core models with React (and RN) bindings; `virentia-storage` persists any core store into a storage backend; `virentia-mutable` is a specialized store variant for deep in-place editing (not a default — reach for it only when the state shape calls for it); `virentia-effector` and `virentia-inspector` are integration seams.
