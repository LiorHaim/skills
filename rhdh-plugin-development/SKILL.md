---
name: rhdh-plugin-development
description: >-
  Build full-stack plugins for Red Hat Developer Hub (RHDH) and Backstage using
  the New Backend System (v1.8-1.9+). Covers backend plugins (createBackendPlugin),
  frontend plugins (createPlugin), backend modules (createBackendModule),
  extension points, dynamic plugin packaging with rhdh-cli, Knex/Postgres
  database patterns, permissions, monorepo scaffolding, and Scalprum/Helm
  configuration. Use when creating, scaffolding, or developing any Backstage or
  RHDH plugin, module, or extension point.
---

# RHDH / Backstage Plugin Development

## Target Platform

- **RHDH v1.8–1.9+** using the **New Backend System** (`createBackendPlugin`).
- RHDH 1.9 is based on **Backstage 1.45.3**.
- **Dynamic Plugins** packaged with `@red-hat-developer-hub/cli` (rhdh-cli).
- Deployment via `.tgz` archives or **OCI registry** (`oci://` references, RHDH 1.9+).
- All `@backstage/*` packages are **peerDependencies** (provided by the host).
- Logic libraries (e.g., `zod`, `json-rules-engine`) go in **dependencies** (bundled).

## Monorepo Structure

Every plugin suite is a Yarn 4 workspaces monorepo with Turbo:

```
my-plugin/
├── package.json          # root: workspaces, turbo, devDeps
├── turbo.json
├── tsconfig.json         # solution-style root, noEmit: true
├── packages/
│   ├── my-plugin/                      # frontend-plugin
│   ├── my-plugin-backend/              # backend-plugin
│   ├── my-plugin-common/               # shared types/schemas/permissions
│   └── my-plugin-backend-module-xxx/   # backend-plugin-module(s)
├── docs/
├── examples/             # app-config examples, Helm values
└── README.md
```

For scaffolding details, see [monorepo-scaffold.md](monorepo-scaffold.md).

## Quick Reference by Task

| Task | Reference |
|------|-----------|
| Backend plugin, extension points, database, router | [backend-patterns.md](backend-patterns.md) |
| Frontend plugin, API client, components | [frontend-patterns.md](frontend-patterns.md) |
| Dynamic plugin export, Scalprum, Helm config | [dynamic-plugin-packaging.md](dynamic-plugin-packaging.md) |
| Project scaffolding, package.json, tsconfig | [monorepo-scaffold.md](monorepo-scaffold.md) |

## Core Rules

### Dependency Management

```
peerDependencies (host provides):
  @backstage/*               — all backstage packages
  react, react-dom           — frontend only
  react-router-dom           — frontend only
  @material-ui/*             — frontend only (MUI v4 for RHDH compatibility)
  knex                       — backend only

dependencies (bundled):
  workspace packages         — @scope/plugin-*-common (workspace:^)
  logic libraries            — zod, json-rules-engine, lodash, etc.
  external SDK clients       — @gitbeaker/rest, @kubernetes/client-node, etc.

devDependencies:
  same @backstage/* peers    — pinned for local build/typecheck
  @types/*                   — type definitions
  typescript, rimraf         — build tooling
```

### Package Naming Convention

```
@scope/plugin-{name}                          → frontend-plugin
@scope/plugin-{name}-backend                  → backend-plugin
@scope/plugin-{name}-common                   → shared types (no backstage role)
@scope/plugin-{name}-backend-module-{source}  → backend-plugin-module
```

### Backend Plugin Lifecycle

1. `register(env)` — register extension points, then `registerInit`
2. Extension point callbacks execute (modules add their contributions)
3. `init()` runs — wire database, services, router, scheduled tasks

### Database Rules

- Use `coreServices.database` → `DatabaseService.getClient()` → `Knex`.
- Migrations in `migrations/` dir, plain `.js` files (Knex format).
- Copy migrations to dist: `"build": "tsc --build && cp -r migrations dist/"`.
- Use `resolvePackagePath('@scope/plugin-name-backend', 'migrations')` at runtime.
- Prefix all table names with plugin ID (e.g., `scorecard_facts`).
- Handle Postgres vs SQLite differences with `isPostgres` detection.
- Use `{ useTz: true }` for all timestamp columns.

### Frontend Rules

- Use `createPlugin` from `@backstage/core-plugin-api` (not `createFrontendPlugin`).
- Register API factories in `apis: [createApiFactory({ ... })]`.
- Use `createRoutableExtension` for full pages, `createComponentExtension` for cards/widgets.
- Lazy-load all components: `component: () => import('./Component').then(m => m.Component)`.
- API client uses `discoveryApi.getBaseUrl('pluginId')` + `fetchApi.fetch`.

### Dynamic Plugin Rules (RHDH)

- Modules **cannot** import from the parent plugin at runtime.
- Extension points are matched by **ID string**, not JS object identity.
- Each module must declare its own local `createExtensionPoint` with the **same ID**.
- Frontend needs `scalprum` block in package.json for Scalprum federation.
- Backend needs `backstage.role` set to `backend-plugin` or `backend-plugin-module`.
- Export with: `npx @red-hat-developer-hub/cli@latest plugin export`.
- Package as `.tgz`: `npm pack ./dist-dynamic --pack-destination .`.
- Package as OCI (1.9+): `npx @red-hat-developer-hub/cli@latest plugin package --tag <registry>/<name>:<version>`.
- Deploy via `oci://` references in `dynamic-plugins.yaml` (1.9+). The `!path` suffix is optional for single-plugin packages.

### Permissions

- Define permissions in the `-common` package using `createPermission`.
- Export a combined array for easy registration.
- Check permissions in routes via `permissions.authorize(...)` + `AuthorizeResult`.
- Mark unauthenticated paths with `httpRouter.addAuthPolicy`.
- Always expose `/.well-known/backstage/permissions/metadata` as unauthenticated.

### Code Style

- **Frontend:** React + Material UI v4 (MUI v4 for RHDH compatibility).
- **Backend:** Express router, Zod validation from common package.
- **Tone:** Professional, CNCF/Open Source standards.
- **No in-memory storage** for production data — always Knex/Postgres.
- Use `type` imports for types: `import type { ... } from '...'`.
- Default export from `src/index.ts` is the plugin/module instance.
