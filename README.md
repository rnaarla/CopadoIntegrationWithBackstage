# Comprehensive Copado-Backstage Integration Guide

## 1. Initial Setup

### Environment Setup
```bash
# Install Node.js and Yarn if not already installed
node -v  # Should be >=14.x
yarn -v  # Should be >=1.x

# Create a new Backstage app if not existing
npx @backstage/create-app@latest
cd your-backstage-app

# Install dependencies
yarn install
```

### Required Credentials
1. Copado Admin Access
   - Login URL
   - Admin credentials
   - API access rights

2. Backstage Prerequisites
   - Admin access
   - Repository access
   - Development environment

## 2. Create Backstage Plugin

### Generate Plugin
```bash
# Create plugin scaffold
cd your-backstage-app
yarn new --select plugin
# Input name: backstage-plugin-copado

# Navigate to plugin directory
cd plugins/copado
```

### Plugin Structure
```plaintext
plugins/copado/
├── dev/
│   └── index.tsx               # Development harness
├── src/
│   ├── api/
│   │   ├── CopadoClient.ts    # API client
│   │   ├── types.ts           # TypeScript interfaces
│   │   └── index.ts           # API exports
│   ├── components/
│   │   ├── Pipeline/
│   │   │   ├── PipelineCard.tsx
│   │   │   └── PipelineList.tsx
│   │   ├── Deployment/
│   │   │   ├── DeploymentCard.tsx
│   │   │   └── DeploymentHistory.tsx
│   │   └── Dashboard/
│   │       └── CopadoDashboard.tsx
│   ├── hooks/
│   │   └── useCopadoApi.ts
│   ├── routes.ts
│   └── index.ts
└── package.json
```

### Implementation Files

```typescript
// src/api/types.ts
export interface CopadoConfig {
  baseUrl: string;
  clientId: string;
  clientSecret: string;
}

export interface CopadoPipeline {
  id: string;
  name: string;
  status: 'running' | 'success' | 'failed' | 'pending';
  lastRun: string;
  environment: string;
  branch: string;
  promotedBy: string;
  lastModified: string;
}

export interface CopadoDeployment {
  id: string;
  pipelineId: string;
  status: string;
  startTime: string;
  endTime: string;
  environment: string;
  promotedBy: string;
  changes: Array<{
    component: string;
    type: string;
    description: string;
  }>;
}
```

## 3. Configure Copado Authentication

### Copado Setup
1. Navigate to Copado Setup > Security Controls > OAuth
2. Create new Connected App:
```plaintext
Name: Backstage Integration
Callback URL: https://your-backstage-url/api/auth/copado/handler
OAuth Scopes:
- api
- refresh_token
- id
```

### Backstage Configuration
```yaml
# app-config.yaml
auth:
  providers:
    copado:
      development:
        clientId: ${COPADO_CLIENT_ID}
        clientSecret: ${COPADO_CLIENT_SECRET}
        callbackUrl: http://localhost:3000/api/auth/copado/handler

proxy:
  endpoints:
    '/copado/api':
      target: https://your-copado-instance.com/api
      headers:
        Authorization: ${COPADO_AUTH_TOKEN}

copado:
  baseUrl: ${COPADO_BASE_URL}
```

## 4. Implement API Integration

### API Client Implementation
```typescript
// src/api/CopadoClient.ts
import { createApiRef } from '@backstage/core-plugin-api';
import { CopadoConfig, CopadoPipeline, CopadoDeployment } from './types';

export const copadoApiRef = createApiRef<CopadoApi>({
  id: 'plugin.copado.service',
});

export class CopadoClient implements CopadoApi {
  private readonly baseUrl: string;
  private readonly authToken: string;

  constructor(options: CopadoConfig) {
    this.baseUrl = options.baseUrl;
    this.authToken = this.getAuthToken(options);
  }

  private async getAuthToken(config: CopadoConfig): Promise<string> {
    // Implement OAuth flow
    const response = await fetch(`${config.baseUrl}/oauth2/token`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
      },
      body: new URLSearchParams({
        grant_type: 'client_credentials',
        client_id: config.clientId,
        client_secret: config.clientSecret,
      }),
    });

    const data = await response.json();
    return data.access_token;
  }

  async getPipelines(): Promise<Array<CopadoPipeline>> {
    const response = await fetch(`${this.baseUrl}/pipelines`, {
      headers: {
        Authorization: `Bearer ${this.authToken}`,
      },
    });
    return await response.json();
  }

  async getDeployments(pipelineId: string): Promise<Array<CopadoDeployment>> {
    const response = await fetch(
      `${this.baseUrl}/pipelines/${pipelineId}/deployments`,
      {
        headers: {
          Authorization: `Bearer ${this.authToken}`,
        },
      },
    );
    return await response.json();
  }
}
```

## 5. Create UI Components

### Dashboard Component
```typescript
// src/components/Dashboard/CopadoDashboard.tsx
import React from 'react';
import { Grid } from '@material-ui/core';
import { Header, Page } from '@backstage/core-components';
import { PipelineList } from '../Pipeline/PipelineList';
import { DeploymentHistory } from '../Deployment/DeploymentHistory';

export const CopadoDashboard = () => {
  return (
    <Page themeId="tool">
      <Header title="Copado Dashboard" subtitle="DevOps Pipeline Overview" />
      <Grid container spacing={3}>
        <Grid item xs={12} md={6}>
          <PipelineList />
        </Grid>
        <Grid item xs={12} md={6}>
          <DeploymentHistory />
        </Grid>
      </Grid>
    </Page>
  );
};
```

### Pipeline Components
```typescript
// src/components/Pipeline/PipelineCard.tsx
import React from 'react';
import { Card, CardHeader, CardContent } from '@material-ui/core';
import { StatusOK, StatusError, StatusRunning } from '@backstage/core-components';

export const PipelineCard: React.FC<{ pipeline: CopadoPipeline }> = ({
  pipeline,
}) => {
  const getStatusComponent = (status: string) => {
    switch (status) {
      case 'success':
        return <StatusOK />;
      case 'failed':
        return <StatusError />;
      case 'running':
        return <StatusRunning />;
      default:
        return null;
    }
  };

  return (
    <Card>
      <CardHeader 
        title={pipeline.name}
        action={getStatusComponent(pipeline.status)}
      />
      <CardContent>
        <Grid container spacing={2}>
          <Grid item xs={6}>
            <Typography variant="body2">Branch: {pipeline.branch}</Typography>
          </Grid>
          <Grid item xs={6}>
            <Typography variant="body2">
              Last Run: {new Date(pipeline.lastRun).toLocaleString()}
            </Typography>
          </Grid>
        </Grid>
      </CardContent>
    </Card>
  );
};
```

## 6. Register Plugin in Backstage

### Plugin Registration
```typescript
// packages/app/src/plugins.ts
export { plugin as CopadoPlugin } from '@internal/plugin-copado';

// packages/app/src/components/catalog/EntityPage.tsx
import { EntityCopadoContent } from '@internal/plugin-copado';

const serviceEntityPage = (
  <EntityLayout>
    <EntityLayout.Route
      path="/copado"
      title="Copado"
      element={<EntityCopadoContent />}
    />
  </EntityLayout>
);
```

### Entity Integration
```yaml
# catalog-info.yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: example-service
  annotations:
    copado.com/project-id: your-project-id
    copado.com/pipeline-id: your-pipeline-id
spec:
  type: service
  lifecycle: production
  owner: team-a
```

## 7. Test and Deploy

### Testing Setup
```typescript
// src/components/__tests__/CopadoDashboard.test.tsx
import React from 'react';
import { render, screen } from '@testing-library/react';
import { CopadoDashboard } from '../Dashboard/CopadoDashboard';
import { TestApiProvider } from '@backstage/test-utils';
import { copadoApiRef } from '../../api';

describe('CopadoDashboard', () => {
  it('renders dashboard components', async () => {
    render(
      <TestApiProvider apis={[[copadoApiRef, mockCopadoApi]]}>
        <CopadoDashboard />
      </TestApiProvider>
    );

    expect(screen.getByText('Copado Dashboard')).toBeInTheDocument();
    expect(await screen.findByTestId('pipeline-list')).toBeInTheDocument();
  });
});
```

### Deployment Steps
```bash
# Build plugin
cd plugins/copado
yarn build

# Build Backstage app
cd ../..
yarn build

# Run tests
yarn test

# Start in development
yarn dev

# Deploy to production
yarn build-image
docker push your-registry/backstage-app:latest
```

### Monitoring Setup
```typescript
// src/api/CopadoClient.ts
import { Logger } from '@backstage/core-plugin-api';

export class CopadoClient {
  private readonly logger: Logger;

  constructor(options: CopadoConfig, logger: Logger) {
    this.logger = logger;
  }

  async getPipelines(): Promise<Array<CopadoPipeline>> {
    try {
      const response = await fetch(...);
      this.logger.info('Successfully fetched pipelines');
      return await response.json();
    } catch (error) {
      this.logger.error('Failed to fetch pipelines', error);
      throw error;
    }
  }
}
```
