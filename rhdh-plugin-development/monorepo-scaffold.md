# Monorepo Scaffolding

## Root `package.json`

```json
{
  "name": "scope-plugin-my-plugin",
  "version": "0.1.0",
  "private": true,
  "description": "A My Plugin suite for Red Hat Developer Hub",
  "license": "Apache-2.0",
  "repository": {
    "type": "git",
    "url": "https://github.com/org/rhdh-plugin-my-plugin.git"
  },
  "workspaces": ["packages/*"],
  "scripts": {
    "build": "turbo run build",
    "build:all": "turbo run build --force",
    "clean": "turbo run clean && rimraf node_modules",
    "lint": "turbo run lint",
    "lint:fix": "turbo run lint:fix",
    "test": "turbo run test",
    "typecheck": "turbo run typecheck",
    "export-dynamic": "turbo run export-dynamic",
    "package:backend": "cd packages/my-plugin-backend && npm run package",
    "package:frontend": "cd packages/my-plugin && npm run package",
    "package:all": "npm run package:backend && npm run package:frontend"
  },
  "devDependencies": {
    "@types/node": "^20.11.0",
    "prettier": "^3.2.0",
    "rimraf": "^5.0.5",
    "turbo": "^2.0.0",
    "typescript": "~5.3.3"
  },
  "packageManager": "yarn@4.1.0",
  "engines": {
    "node": ">=18.0.0"
  }
}
```

## `turbo.json`

```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["tsconfig.json"],
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"],
      "cache": true
    },
    "export-dynamic": {
      "dependsOn": ["build"],
      "outputs": ["dist-dynamic/**"],
      "cache": false
    },
    "lint": {
      "outputs": [],
      "cache": true
    },
    "lint:fix": {
      "outputs": [],
      "cache": false
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": [],
      "cache": true
    },
    "typecheck": {
      "dependsOn": ["^build"],
      "outputs": [],
      "cache": true
    },
    "clean": {
      "cache": false
    }
  }
}
```

## Root `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2021",
    "module": "CommonJS",
    "lib": ["ES2021"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "noEmit": true
  },
  "exclude": ["node_modules"]
}
```

## Backend Package `tsconfig.json`

```json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src",
    "noEmit": false,
    "composite": true
  },
  "include": ["src"],
  "references": [
    { "path": "../my-plugin-common" }
  ]
}
```

## Frontend Package `tsconfig.json`

```json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src",
    "noEmit": false,
    "composite": true,
    "jsx": "react-jsx",
    "moduleResolution": "Bundler",
    "lib": ["ES2021", "DOM", "DOM.Iterable"]
  },
  "include": ["src"],
  "references": [
    { "path": "../my-plugin-common" }
  ]
}
```

## Common Package `tsconfig.json`

```json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src",
    "noEmit": false,
    "composite": true
  },
  "include": ["src"]
}
```

## Common Package Structure

```
packages/my-plugin-common/
├── package.json
├── tsconfig.json
└── src/
    ├── index.ts          # re-exports everything
    ├── types.ts          # shared TypeScript interfaces
    ├── schemas.ts        # Zod validation schemas
    ├── utils.ts          # shared utility functions
    └── permissions.ts    # permission definitions
```

### `src/index.ts`

```typescript
export * from './types';
export * from './schemas';
export * from './utils';
export * from './permissions';
```

### `src/types.ts`

```typescript
export interface MyItem {
  id: string;
  displayName: string;
  description?: string;
  status: 'active' | 'inactive';
  metadata: Record<string, unknown>;
  createdAt: string;
  updatedAt: string;
}
```

### `src/schemas.ts`

```typescript
import { z } from 'zod';

export const myItemSchema = z.object({
  id: z.string().min(1).max(255),
  displayName: z.string().min(1).max(255),
  description: z.string().optional(),
  status: z.enum(['active', 'inactive']).default('active'),
  metadata: z.record(z.unknown()).default({}),
});

export type MyItemInput = z.infer<typeof myItemSchema>;

export const entityRefSchema = z
  .string()
  .min(1)
  .regex(/^[a-z]+:[a-z0-9-]+\/[a-z0-9-_.]+$/i, 'Invalid entity reference format');
```

## Directory Layout Summary

```
my-plugin/
├── .cursorrules                          # AI agent rules
├── .gitignore
├── .prettierrc
├── README.md
├── package.json                          # root workspace
├── turbo.json
├── tsconfig.json                         # solution-style root
├── packages/
│   ├── my-plugin/                        # frontend-plugin
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── index.ts
│   │       ├── plugin.ts
│   │       ├── routes.ts
│   │       ├── api/
│   │       ├── components/
│   │       ├── hooks/
│   │       └── pages/
│   ├── my-plugin-backend/                # backend-plugin
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── migrations/
│   │   │   └── YYYYMMDDHHMMSS_init.js
│   │   └── src/
│   │       ├── index.ts
│   │       ├── plugin.ts
│   │       ├── extensions/
│   │       ├── database/
│   │       ├── service/
│   │       └── engine/                   # business logic (optional)
│   ├── my-plugin-common/                 # shared library
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── index.ts
│   │       ├── types.ts
│   │       ├── schemas.ts
│   │       ├── utils.ts
│   │       └── permissions.ts
│   └── my-plugin-backend-module-xxx/     # backend module(s)
│       ├── package.json
│       ├── tsconfig.json
│       └── src/
│           ├── index.ts
│           ├── module.ts
│           └── XxxImpl.ts
├── docs/
│   └── configuration.md
└── examples/
    ├── app-config.my-plugin.yaml
    └── rhdh-helm-values.yaml
```

## `.gitignore`

```
node_modules/
dist/
dist-dynamic/
*.tsbuildinfo
*.tgz
.turbo/
coverage/
```

## `.prettierrc`

```json
{
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "arrowParens": "avoid"
}
```

## Yarn 4 Setup

```bash
corepack enable
corepack prepare yarn@4.1.0 --activate
yarn init -2
yarn set version 4.1.0
```

Ensure `.yarnrc.yml` has:
```yaml
nodeLinker: node-modules
```
