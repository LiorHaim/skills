# Dynamic Plugin Packaging for RHDH

## Overview

RHDH uses **Scalprum** for frontend federation and the **New Backend System** for backend
plugin discovery. Plugins can be deployed in two ways:

- **`.tgz` archives** — hosted on an npm registry or HTTP server (RHDH 1.8+).
- **OCI artifacts** — pushed to a container registry like Quay.io or ghcr.io (RHDH 1.9+, recommended).

Both methods are configured via Helm values or `dynamic-plugins.yaml`.

## Package.json Configuration

### Backend Plugin

```json
{
  "name": "@scope/plugin-my-plugin-backend",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": ["dist", "dist-dynamic", "migrations"],
  "backstage": {
    "role": "backend-plugin",
    "pluginId": "my-plugin",
    "pluginPackages": [
      "@scope/plugin-my-plugin",
      "@scope/plugin-my-plugin-backend",
      "@scope/plugin-my-plugin-common"
    ]
  },
  "scripts": {
    "build": "tsc --build && cp -r migrations dist/",
    "export-dynamic": "npx @red-hat-developer-hub/cli@latest plugin export",
    "package": "npm run build && npm run export-dynamic && npm pack ./dist-dynamic --pack-destination ."
  }
}
```

### Backend Module

```json
{
  "name": "@scope/plugin-my-plugin-backend-module-source",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": ["dist", "dist-dynamic"],
  "backstage": {
    "role": "backend-plugin-module",
    "pluginId": "my-plugin",
    "pluginPackages": [
      "@scope/plugin-my-plugin-backend"
    ]
  },
  "scripts": {
    "build": "tsc --build",
    "export-dynamic": "npx @red-hat-developer-hub/cli@latest plugin export",
    "package": "npm run build && npm run export-dynamic && npm pack ./dist-dynamic --pack-destination ."
  }
}
```

### Frontend Plugin

```json
{
  "name": "@scope/plugin-my-plugin",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": ["dist", "dist-dynamic"],
  "backstage": {
    "role": "frontend-plugin",
    "pluginId": "my-plugin",
    "pluginPackages": [
      "@scope/plugin-my-plugin",
      "@scope/plugin-my-plugin-backend",
      "@scope/plugin-my-plugin-common"
    ]
  },
  "scalprum": {
    "name": "scope.plugin-my-plugin",
    "exposedModules": {
      "PluginRoot": "./src/index.ts"
    }
  },
  "sideEffects": false,
  "scripts": {
    "build": "tsc --build",
    "export-dynamic": "npx @red-hat-developer-hub/cli@latest plugin export",
    "package": "npm run build && npm run export-dynamic && npm pack ./dist-dynamic --pack-destination ."
  }
}
```

**Scalprum naming rule:** Replace `@` and `/` in npm scope → dot-separated.
`@scope/plugin-my-plugin` → `scope.plugin-my-plugin`.

### Common Package

The common package has no `backstage.role` — it's a plain library:

```json
{
  "name": "@scope/plugin-my-plugin-common",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": ["dist"],
  "scripts": {
    "build": "tsc --build"
  }
}
```

## Build & Export Pipeline

```bash
# Build all packages (respects dependency order)
yarn build        # turbo run build

# Export dynamic plugin artifacts for one package
cd packages/my-plugin-backend
npm run export-dynamic     # creates dist-dynamic/

# Package as .tgz
npm pack ./dist-dynamic --pack-destination .
# Produces: scope-plugin-my-plugin-backend-dynamic-0.1.0.tgz
```

Root-level convenience scripts:
```json
{
  "package:backend": "cd packages/my-plugin-backend && npm run package",
  "package:frontend": "cd packages/my-plugin && npm run package",
  "package:all": "npm run package:backend && npm run package:frontend"
}
```

## OCI Registry Packaging (RHDH 1.9+)

Starting with RHDH 1.9, the recommended deployment method is OCI artifacts pushed to
a container registry. This replaces the need for hosting `.tgz` files on an HTTP server.

### Package and Push

```bash
# Set variables
export REGISTRY=quay.io/my-org
export PLUGIN_NAME=plugin-my-plugin-backend
export VERSION=$(node -p "require('./package.json').version")

# Build + export dynamic
npm run build && npm run export-dynamic

# Package as OCI and push to registry
npx @red-hat-developer-hub/cli@latest plugin package \
  --tag ${REGISTRY}/${PLUGIN_NAME}:${VERSION}

# Push (if not auto-pushed by the CLI)
podman push ${REGISTRY}/${PLUGIN_NAME}:${VERSION}
```

### OCI References in Config

```yaml
plugins:
  # Single-plugin package (no !path suffix needed in 1.9+)
  - package: oci://quay.io/my-org/plugin-my-plugin-backend:0.1.0
    disabled: false

  # Multi-plugin package (specify path)
  - package: oci://quay.io/my-org/plugin-my-plugin-backend:0.1.0!backstage-plugin-my-plugin-backend
    disabled: false

  # Use {{inherit}} for version from the RHDH release (Red Hat plugins only)
  - package: 'oci://registry.access.redhat.com/rhdh/some-plugin:{{inherit}}'
    disabled: false
```

## RHDH Helm Deployment

### Helm values (`rhdh-helm-values.yaml`)

```yaml
global:
  dynamic:
    includes:
      - dynamic-plugins.default.yaml
    plugins:
      # === Option A: OCI registry (RHDH 1.9+, recommended) ===
      - package: oci://quay.io/my-org/plugin-my-plugin-backend:0.1.0
        disabled: false

      - package: oci://quay.io/my-org/plugin-my-plugin-backend-module-source:0.1.0
        disabled: false

      - package: oci://quay.io/my-org/plugin-my-plugin:0.1.0
        disabled: false
        pluginConfig:
          # ... (same pluginConfig as below)

      # === Option B: .tgz over HTTP (RHDH 1.8+) ===
      # Backend plugin
      - package: >-
          https://registry.example.com/@scope/plugin-my-plugin-backend-dynamic/-/plugin-my-plugin-backend-dynamic-0.1.0.tgz
        disabled: false
        # integrity: sha512-...   # optional SRI hash

      # Backend module
      - package: >-
          https://registry.example.com/@scope/plugin-my-plugin-backend-module-source-dynamic/-/plugin-my-plugin-backend-module-source-dynamic-0.1.0.tgz
        disabled: false

      # Frontend plugin
      - package: >-
          https://registry.example.com/@scope/plugin-my-plugin-dynamic/-/plugin-my-plugin-dynamic-0.1.0.tgz
        disabled: false
        pluginConfig:
          dynamicPlugins:
            frontend:
              scope.plugin-my-plugin:    # matches scalprum.name
                mountPoints:
                  - mountPoint: entity.page.my-plugin/cards
                    importName: EntityMyPluginCard
                    config:
                      layout:
                        gridColumnEnd: 'span 6'
                  - mountPoint: entity.page.my-plugin/context
                    importName: EntityMyPluginContent
                dynamicRoutes:
                  - path: /my-plugin
                    importName: MyDashboardPage
                    menuItem:
                      icon: Assessment
                      text: My Plugin
                sidebarItems:
                  - mountPoint: sidebar.nav
                    importName: MySidebarItem
```

### App Config (`app-config.my-plugin.yaml`)

```yaml
my-plugin:
  schedule:
    frequency:
      minutes: 30
    timeout:
      minutes: 10

  # Plugin-specific configuration
  someOption: value

backend:
  database:
    client: pg
    connection:
      host: ${POSTGRES_HOST}
      port: ${POSTGRES_PORT}
      user: ${POSTGRES_USER}
      password: ${POSTGRES_PASSWORD}
```

## Critical RHDH Dynamic Plugin Rules

1. **No cross-plugin imports at runtime.** Modules cannot `import` from
   `@scope/plugin-my-plugin-backend` directly. Extension points are matched
   by their **ID string**, so modules must redeclare the extension point
   locally with `createExtensionPoint({ id: 'same-id-string' })`.

2. **`src/index.ts` must have a default export.** Backend plugins export the
   plugin instance; modules export the module instance.

3. **`dist-dynamic/` is the artifact.** After `export-dynamic`, this directory
   contains the rewritten bundle. `npm pack` creates the `.tgz` from it.

4. **`files` array matters.** Backend plugins must include `migrations`.
   All dynamic packages include `dist` and `dist-dynamic`.

5. **`sideEffects: false`** should be set on frontend plugins for proper
   tree-shaking by Scalprum.

6. **Scalprum name** must match `pluginConfig.dynamicPlugins.frontend.*` key
   in Helm values exactly.

7. **Backend migrations** must be copied to `dist/` during build so
   `resolvePackagePath` can find them at runtime.

8. **OCI `!path` suffix (1.9+):** Not required if the OCI package contains
   only one plugin. Required for multi-plugin packages.

9. **`{{inherit}}` tag (1.9+):** Use when referencing Red Hat-provided OCI
   images to automatically inherit the version matching the RHDH release.

10. **Extensions Catalog Index (1.9+):** Helm upgrades to 1.9 require
    additional volume mounts (`extensions-catalog`) and init container env
    vars (`CATALOG_INDEX_IMAGE`, `CATALOG_ENTITIES_EXTRACT_DIR`). If you
    override `initContainers`, `extraVolumes`, or `extraVolumeMounts` in
    custom values, you must manually merge the new defaults.
