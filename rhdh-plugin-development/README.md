# RHDH Plugin Development Skill

An AI agent skill for building full-stack plugins for **Red Hat Developer Hub (RHDH)** and **Backstage** using the New Backend System (v1.8+).

## What This Skill Covers

- **Backend Plugins** — `createBackendPlugin`, extension points, services, scheduled tasks
- **Frontend Plugins** — `createPlugin`, routable/component extensions, API client pattern
- **Backend Modules** — `createBackendModule`, extension point wiring for dynamic plugins
- **Database** — Knex/Postgres patterns, migrations, Postgres vs SQLite compatibility
- **Permissions** — `createPermission`, route-level authorization, permission-gated UI
- **Dynamic Plugin Packaging** — rhdh-cli export, Scalprum federation, Helm deployment
- **Monorepo Scaffolding** — Yarn 4 workspaces, Turbo, tsconfig, package.json templates

## Files

| File | Lines | Purpose |
|------|-------|---------|
| `SKILL.md` | 133 | Main entry point — core rules and quick-reference table |
| `backend-patterns.md` | 379 | Backend plugin, modules, database, router, permissions |
| `frontend-patterns.md` | 327 | Frontend plugin, API client, hooks, MUI v4 patterns |
| `dynamic-plugin-packaging.md` | 228 | RHDH packaging, Scalprum, Helm, 7 critical dynamic plugin rules |
| `monorepo-scaffold.md` | 312 | Full project scaffolding templates |

## Installation

Copy to your Cursor personal skills directory:

```bash
cp -r rhdh-plugin-development ~/.cursor/skills/
```

Or clone this repo and symlink:

```bash
git clone https://github.com/LiorHaim/skills.git
ln -s $(pwd)/skills/rhdh-plugin-development ~/.cursor/skills/rhdh-plugin-development
```

## Built From

These patterns are battle-tested from real RHDH plugin development, including:

- New Backend System plugin architecture (`createBackendPlugin`)
- Extension point patterns for modular fact retrievers
- Dynamic plugin constraints (cross-plugin import restrictions, ID-based extension point matching)
- Knex migration patterns with Postgres/SQLite dual compatibility
- Scalprum-based frontend federation for RHDH
- Permission-gated routes and UI components

Combined with the latest official [Backstage](https://backstage.io/docs) and [RHDH](https://developers.redhat.com/rhdh) documentation.

## Companion `.cursorrules`

For project-level rules, add this to your plugin project's `.cursorrules`:

```markdown
# Role
You are a Principal RHDH Solutions Architect specializing in the New Backend System (v1.8+),
Dynamic Plugins, and Software Quality Engineering.

# Constraints
- Target Platform: RHDH v1.8+ using createBackendPlugin
- Packaging: Compatible with rhdh-cli for Dynamic Plugin export
- peerDependencies: All @backstage/* packages (host-provided)
- dependencies: Logic libraries (zod, etc.) that must be bundled
- Persistence: coreServices.database (Knex/Postgres), no in-memory storage
- Dynamic Plugins: Extension points matched by ID string, not JS identity
```

## License

Apache-2.0
