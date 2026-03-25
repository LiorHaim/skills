# Backend Plugin Patterns

## Plugin Entry Point (`src/plugin.ts`)

```typescript
import {
  coreServices,
  createBackendPlugin,
} from '@backstage/backend-plugin-api';
import { createRouter } from './service/router';
import { MyDatabase } from './database/MyDatabase';

export const myPlugin = createBackendPlugin({
  pluginId: 'my-plugin',
  register(env) {
    // 1. Register extension points BEFORE registerInit
    const items: MyItem[] = [];
    env.registerExtensionPoint(myExtensionPoint, {
      addItem(item: MyItem) {
        items.push(item);
      },
    });

    // 2. Register init
    env.registerInit({
      deps: {
        config: coreServices.rootConfig,
        logger: coreServices.logger,
        database: coreServices.database,
        scheduler: coreServices.scheduler,
        httpRouter: coreServices.httpRouter,
        auth: coreServices.auth,
        httpAuth: coreServices.httpAuth,
        discovery: coreServices.discovery,
        permissions: coreServices.permissions,
      },
      async init({
        config, logger, database, scheduler,
        httpRouter, auth, httpAuth, discovery, permissions,
      }) {
        const pluginLogger = logger.child({ plugin: 'my-plugin' });

        // Database
        const db = await MyDatabase.create(database, pluginLogger);
        await db.runMigrations();

        // Services
        const myService = new MyService(db, pluginLogger);

        // Scheduled tasks
        await scheduler.scheduleTask({
          id: 'my-plugin-task',
          frequency: { minutes: 30 },
          timeout: { minutes: 10 },
          initialDelay: { seconds: 30 },
          fn: async () => { /* ... */ },
        });

        // Router
        const router = createRouter({
          db, logger: pluginLogger, auth, httpAuth, permissions, discovery,
        });
        httpRouter.use(router);

        // Auth policies
        httpRouter.addAuthPolicy({
          path: '/health',
          allow: 'unauthenticated',
        });
        httpRouter.addAuthPolicy({
          path: '/.well-known/backstage/permissions/metadata',
          allow: 'unauthenticated',
        });
      },
    });
  },
});
```

## Plugin Index (`src/index.ts`)

```typescript
export { myPlugin as default } from './plugin';
export { myExtensionPoint } from './extensions/myExtensionPoint';
export type { MyItem } from './extensions/myExtensionPoint';
```

## Extension Point (`src/extensions/myExtensionPoint.ts`)

```typescript
import { createExtensionPoint } from '@backstage/backend-plugin-api';

export interface MyItem {
  id: string;
  displayName: string;
  process(context: MyContext): Promise<MyResult[]>;
  appliesTo?(entity: Entity): boolean;
}

export interface MyExtensionPoint {
  addItem(item: MyItem): void;
}

export const myExtensionPoint = createExtensionPoint<MyExtensionPoint>({
  id: 'my-plugin.items',
});
```

## Backend Module (`src/module.ts`)

Modules extend plugins via extension points. In RHDH dynamic plugins, the extension
point must be redeclared locally with the **same ID string**.

```typescript
import {
  createBackendModule,
  createExtensionPoint,
  coreServices,
} from '@backstage/backend-plugin-api';

// Local extension point reference (same ID as parent plugin)
const myExtensionPoint = createExtensionPoint<{
  addItem(item: MyItem): void;
}>({
  id: 'my-plugin.items', // MUST match the parent plugin's extension point ID
});

export const myPluginModuleSource = createBackendModule({
  pluginId: 'my-plugin',    // must match parent plugin's pluginId
  moduleId: 'source-name',
  register(reg) {
    reg.registerInit({
      deps: {
        items: myExtensionPoint,
        config: coreServices.rootConfig,
        logger: coreServices.logger,
      },
      async init({ items, config, logger }) {
        const moduleLogger = logger.child({ module: 'my-plugin-source' });

        // Read config, validate, create implementation
        const impl = new MySourceImpl({ config, logger: moduleLogger });
        items.addItem(impl);

        moduleLogger.info('Registered source module');
      },
    });
  },
});
```

Module index:
```typescript
export { myPluginModuleSource as default } from './module';
```

## Database Class (`src/database/MyDatabase.ts`)

```typescript
import type { Knex } from 'knex';
import type { DatabaseService, LoggerService } from '@backstage/backend-plugin-api';
import { resolvePackagePath } from '@backstage/backend-plugin-api';

export class MyDatabase {
  private readonly isPostgres: boolean;

  private constructor(
    private readonly db: Knex,
    private readonly logger: LoggerService,
  ) {
    const client = (db.client as { config?: { client?: string } })?.config?.client ?? '';
    this.isPostgres = client === 'pg' || client === 'postgres' || client === 'postgresql';
  }

  static async create(
    database: DatabaseService,
    logger: LoggerService,
  ): Promise<MyDatabase> {
    const client = await database.getClient();
    return new MyDatabase(client, logger);
  }

  async runMigrations(): Promise<void> {
    const migrationsDir = resolvePackagePath(
      '@scope/plugin-my-plugin-backend',
      'migrations',
    );
    await this.db.migrate.latest({ directory: migrationsDir });
  }

  // Date helper for Postgres vs SQLite compatibility
  private toISOString(value: unknown): string {
    if (!value) return new Date().toISOString();
    if (typeof value === 'string') return value;
    if (value instanceof Date) return value.toISOString();
    return new Date(value as string | number).toISOString();
  }

  // Query pattern
  async getItems(filter?: { status?: string }): Promise<MyItem[]> {
    let query = this.db('myplugin_items').select('*');
    if (filter?.status) {
      query = query.where('status', filter.status);
    }
    const rows = await query;
    return rows.map(row => this.mapRow(row));
  }

  // Upsert pattern (Postgres)
  async upsertItem(item: MyItem): Promise<void> {
    if (this.isPostgres) {
      await this.db('myplugin_items')
        .insert({ /* ... */ })
        .onConflict('item_id')
        .merge({ /* updated fields */ });
    } else {
      const existing = await this.db('myplugin_items')
        .where('item_id', item.id).first();
      if (existing) {
        await this.db('myplugin_items')
          .where('item_id', item.id).update({ /* ... */ });
      } else {
        await this.db('myplugin_items').insert({ /* ... */ });
      }
    }
  }
}
```

## Migration File (`migrations/YYYYMMDDHHMMSS_init.js`)

```javascript
/**
 * @param { import("knex").Knex } knex
 * @returns { Promise<void> }
 */
exports.up = async function up(knex) {
  await knex.schema.createTable('myplugin_items', table => {
    table.uuid('id').primary().defaultTo(knex.fn.uuid());
    table.string('item_id', 255).notNullable().unique();
    table.string('display_name', 255).notNullable();
    table.text('description');
    table.string('status', 50).notNullable().defaultTo('active');
    table.jsonb('metadata').notNullable().defaultTo('{}');
    table.boolean('is_active').notNullable().defaultTo(true);
    table.timestamp('created_at', { useTz: true }).notNullable().defaultTo(knex.fn.now());
    table.timestamp('updated_at', { useTz: true }).notNullable().defaultTo(knex.fn.now());
  });

  await knex.schema.alterTable('myplugin_items', table => {
    table.index('status', 'idx_myplugin_items_status');
    table.index('is_active', 'idx_myplugin_items_active');
  });
};

/**
 * @param { import("knex").Knex } knex
 * @returns { Promise<void> }
 */
exports.down = async function down(knex) {
  await knex.schema.dropTableIfExists('myplugin_items');
};
```

**Migration rules:**
- File name: `YYYYMMDDHHMMSS_description.js` (plain JS, not TS).
- Always provide both `up` and `down`.
- Prefix table names with plugin ID to avoid collisions.
- Use `{ useTz: true }` for timestamps.
- Use `knex.fn.uuid()` for primary keys.
- `jsonb` for structured data; `text[]` for simple arrays.
- Create indexes in separate `alterTable` calls.

## Router (`src/service/router.ts`)

```typescript
import { Router } from 'express';
import type { Request, Response } from 'express';
import type {
  LoggerService, AuthService, HttpAuthService, PermissionsService,
} from '@backstage/backend-plugin-api';
import { AuthorizeResult } from '@backstage/plugin-permission-common';
import { myReadPermission } from '@scope/plugin-my-plugin-common';

interface RouterOptions {
  db: MyDatabase;
  logger: LoggerService;
  auth: AuthService;
  httpAuth: HttpAuthService;
  permissions: PermissionsService;
}

export function createRouter(options: RouterOptions): Router {
  const { db, logger, httpAuth, permissions } = options;
  const router = Router();

  // Health check (unauthenticated)
  router.get('/health', (_req, res) => {
    res.json({ status: 'ok' });
  });

  // Permission-gated endpoint
  router.get('/items', async (req: Request, res: Response) => {
    try {
      const credentials = await httpAuth.credentials(req);
      const decision = await permissions.authorize(
        [{ permission: myReadPermission }],
        { credentials },
      );
      if (decision[0].result === AuthorizeResult.DENY) {
        res.status(403).json({ error: 'Forbidden' });
        return;
      }

      const items = await db.getItems();
      res.json({ items });
    } catch (error) {
      logger.error(`Failed to get items: ${error}`);
      res.status(500).json({ error: 'Internal server error' });
    }
  });

  // Zod validation pattern
  router.post('/items', async (req: Request, res: Response) => {
    try {
      const credentials = await httpAuth.credentials(req);
      const parsed = myItemSchema.safeParse(req.body);
      if (!parsed.success) {
        res.status(400).json({ error: parsed.error.message });
        return;
      }
      await db.upsertItem(parsed.data);
      res.status(201).json({ created: true });
    } catch (error) {
      logger.error(`Failed to create item: ${error}`);
      res.status(500).json({ error: 'Internal server error' });
    }
  });

  return router;
}
```

## Permissions (`common/src/permissions.ts`)

```typescript
import { createPermission } from '@backstage/plugin-permission-common';

export const myPluginReadPermission = createPermission({
  name: 'my-plugin.read',
  attributes: { action: 'read' },
});

export const myPluginWritePermission = createPermission({
  name: 'my-plugin.write',
  attributes: { action: 'update' },
});

export const myPluginPermissions = [
  myPluginReadPermission,
  myPluginWritePermission,
];
```

## Available Core Services

| Service | Ref | Purpose |
|---------|-----|---------|
| Config | `coreServices.rootConfig` | Read app-config values |
| Logger | `coreServices.logger` | Scoped logging |
| Database | `coreServices.database` | Knex client with plugin isolation |
| Scheduler | `coreServices.scheduler` | Cron-like task scheduling |
| HTTP Router | `coreServices.httpRouter` | Express router registration |
| Auth | `coreServices.auth` | Backend-to-backend auth tokens |
| HTTP Auth | `coreServices.httpAuth` | Extract credentials from requests |
| Discovery | `coreServices.discovery` | Discover other plugin base URLs |
| Permissions | `coreServices.permissions` | Authorization decisions |
| Identity | `coreServices.identity` | Resolve caller identity |
| Root Logger | `coreServices.rootLogger` | Unscoped logger (rarely needed) |
