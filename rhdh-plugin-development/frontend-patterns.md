# Frontend Plugin Patterns

## Plugin Entry Point (`src/plugin.ts`)

```typescript
import {
  createPlugin,
  createApiFactory,
  discoveryApiRef,
  fetchApiRef,
  createRoutableExtension,
  createComponentExtension,
} from '@backstage/core-plugin-api';
import { myApiRef, MyClient } from './api';
import { myRouteRef, myDashboardRouteRef } from './routes';

export const myPlugin = createPlugin({
  id: 'my-plugin',
  routes: {
    root: myRouteRef,
    dashboard: myDashboardRouteRef,
  },
  apis: [
    createApiFactory({
      api: myApiRef,
      deps: {
        discoveryApi: discoveryApiRef,
        fetchApi: fetchApiRef,
      },
      factory: ({ discoveryApi, fetchApi }) =>
        new MyClient({ discoveryApi, fetchApi }),
    }),
  ],
});

// Full page — use createRoutableExtension
export const EntityMyPluginContent = myPlugin.provide(
  createRoutableExtension({
    name: 'EntityMyPluginContent',
    component: () =>
      import('./components/EntityMyPluginContent').then(
        m => m.EntityMyPluginContent,
      ),
    mountPoint: myRouteRef,
  }),
);

// Dashboard page
export const MyDashboardPage = myPlugin.provide(
  createRoutableExtension({
    name: 'MyDashboardPage',
    component: () =>
      import('./pages/MyDashboardPage').then(m => m.MyDashboardPage),
    mountPoint: myDashboardRouteRef,
  }),
);

// Card/widget — use createComponentExtension
export const EntityMyPluginCard = myPlugin.provide(
  createComponentExtension({
    name: 'EntityMyPluginCard',
    component: {
      lazy: () =>
        import('./components/EntityMyPluginCard').then(
          m => m.EntityMyPluginCard,
        ),
    },
  }),
);

// Sidebar item with permission gating
export const MySidebarItem = myPlugin.provide(
  createComponentExtension({
    name: 'MySidebarItem',
    component: {
      lazy: () =>
        import('./components/MySidebarItem').then(m => m.MySidebarItem),
    },
  }),
);
```

## Routes (`src/routes.ts`)

```typescript
import { createRouteRef } from '@backstage/core-plugin-api';

export const myRouteRef = createRouteRef({
  id: 'my-plugin',
});

export const myDashboardRouteRef = createRouteRef({
  id: 'my-plugin-dashboard',
});
```

## API Interface (`src/api/MyApi.ts`)

```typescript
import { createApiRef } from '@backstage/core-plugin-api';
import type { MyItem, MyOverview } from '@scope/plugin-my-plugin-common';

export interface MyApi {
  getItems(): Promise<MyItem[]>;
  getOverview(): Promise<MyOverview>;
  refreshData(entityRef: string): Promise<{ message: string }>;
}

export const myApiRef = createApiRef<MyApi>({
  id: 'plugin.my-plugin.api',
});
```

## API Client (`src/api/MyClient.ts`)

```typescript
import type { DiscoveryApi, FetchApi } from '@backstage/core-plugin-api';
import type { MyApi } from './MyApi';
import type { MyItem, MyOverview } from '@scope/plugin-my-plugin-common';

export class MyClient implements MyApi {
  private readonly discoveryApi: DiscoveryApi;
  private readonly fetchApi: FetchApi;

  constructor(options: { discoveryApi: DiscoveryApi; fetchApi: FetchApi }) {
    this.discoveryApi = options.discoveryApi;
    this.fetchApi = options.fetchApi;
  }

  private async getBaseUrl(): Promise<string> {
    return await this.discoveryApi.getBaseUrl('my-plugin');
  }

  private async fetch<T>(path: string, init?: RequestInit): Promise<T> {
    const baseUrl = await this.getBaseUrl();
    const response = await this.fetchApi.fetch(`${baseUrl}${path}`, init);

    if (!response.ok) {
      const error = await response.json().catch(() => ({
        error: response.statusText,
      }));
      throw new Error(error.error || `API request failed: ${response.status}`);
    }

    return response.json();
  }

  async getItems(): Promise<MyItem[]> {
    const response = await this.fetch<{ items: MyItem[] }>('/items');
    return response.items;
  }

  async getOverview(): Promise<MyOverview> {
    return this.fetch('/overview');
  }

  async refreshData(entityRef: string): Promise<{ message: string }> {
    return this.fetch(
      `/refresh?entityRef=${encodeURIComponent(entityRef)}`,
      { method: 'POST' },
    );
  }
}
```

## Plugin Index (`src/index.ts`)

```typescript
export {
  myPlugin,
  EntityMyPluginContent,
  EntityMyPluginCard,
  MyDashboardPage,
  MySidebarItem,
} from './plugin';

export { myApiRef } from './api';
export type { MyApi } from './api';
export { myRouteRef, myDashboardRouteRef } from './routes';

// Re-export common types for consumer convenience
export type {
  MyItem,
  MyOverview,
} from '@scope/plugin-my-plugin-common';
```

## Component Directory Structure

```
src/
├── api/
│   ├── index.ts              # re-exports MyApi, MyClient, myApiRef
│   ├── MyApi.ts              # interface + createApiRef
│   └── MyClient.ts           # implementation
├── components/
│   ├── EntityMyPluginContent/
│   │   ├── index.ts
│   │   └── EntityMyPluginContent.tsx
│   ├── EntityMyPluginCard/
│   │   ├── index.ts
│   │   └── EntityMyPluginCard.tsx
│   └── MySidebarItem/
│       ├── index.ts
│       └── MySidebarItem.tsx
├── hooks/
│   ├── useMyData.ts
│   └── useMyApi.ts
├── pages/
│   ├── MyDashboardPage/
│   │   ├── index.ts
│   │   └── MyDashboardPage.tsx
│   └── ...
├── plugin.ts
├── routes.ts
└── index.ts
```

## Hooks Pattern

```typescript
import { useApi } from '@backstage/core-plugin-api';
import { useEntity } from '@backstage/plugin-catalog-react';
import { useAsync } from 'react-use';
import { myApiRef } from '../api';

export function useMyEntityData() {
  const api = useApi(myApiRef);
  const { entity } = useEntity();
  const entityRef = `${entity.kind}:${entity.metadata.namespace ?? 'default'}/${entity.metadata.name}`;

  const { value, loading, error } = useAsync(
    () => api.getItems(),
    [entityRef],
  );

  return { data: value, loading, error };
}
```

## Permission-Gated UI

```typescript
import { usePermission } from '@backstage/plugin-permission-react';
import { myPluginReadPermission } from '@scope/plugin-my-plugin-common';

export const MyProtectedComponent = () => {
  const { allowed, loading } = usePermission({
    permission: myPluginReadPermission,
  });

  if (loading) return <Progress />;
  if (!allowed) return null; // or a "no access" message

  return <MyActualContent />;
};
```

## Entity Page Integration

Usage in the consuming app (or via dynamic plugins):

```tsx
// Entity tab
<EntityLayout.Route path="/my-plugin" title="My Plugin">
  <EntityMyPluginContent />
</EntityLayout.Route>

// Overview card
<Grid item md={6}>
  <EntityMyPluginCard />
</Grid>

// Standalone route
<Route path="/my-plugin" element={<MyDashboardPage />} />
```

## MUI v4 Component Patterns (RHDH)

RHDH currently uses MUI v4. Import from:

```typescript
import { makeStyles, Theme } from '@material-ui/core/styles';
import {
  Box, Card, CardContent, Typography, Chip, Grid,
  Table, TableBody, TableCell, TableHead, TableRow,
  LinearProgress, Tooltip, IconButton, Tabs, Tab,
} from '@material-ui/core';
import { Alert, Skeleton } from '@material-ui/lab';
```

Use `makeStyles` for styling:

```typescript
const useStyles = makeStyles((theme: Theme) => ({
  root: {
    padding: theme.spacing(2),
  },
  card: {
    marginBottom: theme.spacing(2),
    borderRadius: theme.shape.borderRadius,
  },
}));
```

## Backstage Core Components

Prefer Backstage's built-in components when available:

```typescript
import {
  Content,
  ContentHeader,
  Header,
  HeaderLabel,
  Page,
  Progress,
  Table,
  StatusOK,
  StatusError,
  StatusWarning,
  InfoCard,
  Link,
  EmptyState,
  ErrorPanel,
} from '@backstage/core-components';
```
