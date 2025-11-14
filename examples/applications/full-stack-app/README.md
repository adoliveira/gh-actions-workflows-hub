# Full-Stack Application Example

Este exemplo demonstra um pipeline CI/CD completo para uma aplicação full-stack usando monorepo, com frontend React, backend Node.js/Python, e deployment multi-cloud.

## 📋 Sobre a Aplicação

- **Frontend**: React/Next.js com TypeScript
- **Backend**: Node.js API + Python microservices
- **Database**: PostgreSQL (primary) + MongoDB (analytics)
- **Cache**: Redis
- **Queue**: AWS SQS / Azure Service Bus
- **Storage**: AWS S3 / Azure Blob Storage
- **Deployment**: Multi-cloud (AWS + Azure)

## 🏗️ Arquitetura

```
                    ┌─────────────────┐
                    │   CDN/CloudFront│
                    │   (Frontend)    │
                    └─────────────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
   ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
   │  Load Balancer  │ │   API Gateway   │ │   Auth Service  │
   └─────────────────┘ └─────────────────┘ └─────────────────┘
            │                │                │
   ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
   │   App Server    │ │   API Service   │ │  Analytics      │
   │   (Node.js)     │ │   (Node.js)     │ │  (Python)       │
   └─────────────────┘ └─────────────────┘ └─────────────────┘
            │                │                │
   ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
   │   PostgreSQL    │ │     Redis       │ │    MongoDB      │
   │   (Primary DB)  │ │    (Cache)      │ │  (Analytics)    │
   └─────────────────┘ └─────────────────┘ └─────────────────┘
```

## 🚀 Monorepo Workflow

```yaml
# .github/workflows/full-stack.yml
name: 'Full-Stack Application CI/CD'

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  # Detect Changes
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      frontend-changed: ${{ steps.changes.outputs.frontend }}
      backend-changed: ${{ steps.changes.outputs.backend }}
      analytics-changed: ${{ steps.changes.outputs.analytics }}
      infrastructure-changed: ${{ steps.changes.outputs.infrastructure }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v2
        id: changes
        with:
          filters: |
            frontend:
              - 'apps/frontend/**'
            backend:
              - 'apps/backend/**'
            analytics:
              - 'apps/analytics/**'
            infrastructure:
              - 'infrastructure/**'

  # Frontend Pipeline
  frontend:
    if: needs.detect-changes.outputs.frontend-changed == 'true'
    needs: detect-changes
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/ci-reusable.yml@main
    with:
      node-version: '18'
      working-directory: 'apps/frontend'
      package-manager: 'pnpm'
      run-tests: true
      run-lint: true
      run-type-check: true
      build-command: 'pnpm build'
      artifact-name: 'frontend-dist'
      artifact-path: 'apps/frontend/dist'

  # Backend Pipeline
  backend:
    if: needs.detect-changes.outputs.backend-changed == 'true'
    needs: detect-changes
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/ci-reusable.yml@main
    with:
      node-version: '18'
      working-directory: 'apps/backend'
      package-manager: 'pnpm'
      run-tests: true
      run-lint: true
      run-security-scan: true
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  # Analytics Service Pipeline  
  analytics:
    if: needs.detect-changes.outputs.analytics-changed == 'true'
    needs: detect-changes
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          cache: 'pip'
      
      - name: Install dependencies
        working-directory: apps/analytics
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-cov black flake8

      - name: Code formatting
        working-directory: apps/analytics
        run: black --check .

      - name: Linting
        working-directory: apps/analytics
        run: flake8 .

      - name: Run tests
        working-directory: apps/analytics
        run: pytest --cov=. --cov-report=xml

  # Build Docker Images
  build-images:
    needs: [frontend, backend, analytics]
    if: |
      always() && 
      (needs.frontend.result == 'success' || needs.frontend.result == 'skipped') &&
      (needs.backend.result == 'success' || needs.backend.result == 'skipped') &&
      (needs.analytics.result == 'success' || needs.analytics.result == 'skipped')
    strategy:
      matrix:
        service: [frontend, backend, analytics]
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/docker-build.yml@main
    with:
      image-name: ${{ github.repository }}/${{ matrix.service }}
      context: ./apps/${{ matrix.service }}
      platforms: 'linux/amd64,linux/arm64'
      push-to-registry: true
      cache-from: type=gha
      cache-to: type=gha,mode=max
    secrets:
      REGISTRY_USERNAME: ${{ github.actor }}
      REGISTRY_PASSWORD: ${{ secrets.GITHUB_TOKEN }}

  # Deploy Infrastructure
  deploy-infrastructure:
    if: needs.detect-changes.outputs.infrastructure-changed == 'true'
    needs: detect-changes
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/terraform-cicd.yml@main
    with:
      terraform-directory: 'infrastructure/terraform'
      environment: ${{ github.ref == 'refs/heads/main' && 'production' || 'development' }}
      cloud-provider: 'aws'
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_TERRAFORM_ROLE_ARN }}

  # Integration Tests
  integration-tests:
    needs: [build-images, deploy-infrastructure]
    if: always() && needs.build-images.result == 'success'
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'pnpm'
          cache-dependency-path: pnpm-lock.yaml

      - name: Install dependencies
        run: pnpm install

      - name: Run integration tests
        working-directory: tests/integration
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379
          API_BASE_URL: http://localhost:3001
        run: pnpm test

  # E2E Tests
  e2e-tests:
    needs: [integration-tests]
    if: needs.integration-tests.result == 'success'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install

      - name: Install Playwright
        working-directory: tests/e2e
        run: pnpm playwright install

      - name: Run E2E tests
        working-directory: tests/e2e
        run: pnpm test

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: e2e-test-results
          path: tests/e2e/test-results/

  # Deploy to Development
  deploy-dev:
    if: github.ref == 'refs/heads/develop'
    needs: [build-images, integration-tests]
    strategy:
      matrix:
        cloud: [aws, azure]
        service: [frontend, backend, analytics]
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/multi-env-deploy.yml@main
    with:
      environment: 'development'
      cloud-provider: ${{ matrix.cloud }}
      service-name: ${{ matrix.service }}
      deploy-target: ${{ matrix.cloud == 'aws' && 'ecs' || 'aci' }}
      image: ghcr.io/${{ github.repository }}/${{ matrix.service }}:${{ github.sha }}
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_DEV_ROLE_ARN }}
      AZURE_CREDENTIALS: ${{ secrets.AZURE_DEV_CREDENTIALS }}

  # Deploy to Production (Blue-Green)
  deploy-prod:
    if: github.ref == 'refs/heads/main'
    needs: [build-images, e2e-tests]
    strategy:
      matrix:
        service: [frontend, backend, analytics]
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/blue-green-deploy.yml@main
    with:
      environment: 'production'
      service-name: ${{ matrix.service }}
      deploy-target: 'ecs'
      image: ghcr.io/${{ github.repository }}/${{ matrix.service }}:${{ github.sha }}
      health-check-url: https://api.myapp.com/${{ matrix.service }}/health
      canary-percentage: 10
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_PROD_ROLE_ARN }}
```

## 📁 Estrutura do Monorepo

```
full-stack-app/
├── .github/
│   └── workflows/
│       └── full-stack.yml
├── apps/
│   ├── frontend/              # Next.js/React app
│   │   ├── src/
│   │   ├── public/
│   │   ├── package.json
│   │   ├── next.config.js
│   │   └── Dockerfile
│   ├── backend/               # Node.js API
│   │   ├── src/
│   │   ├── tests/
│   │   ├── package.json
│   │   └── Dockerfile
│   └── analytics/             # Python analytics service
│       ├── src/
│       ├── tests/
│       ├── requirements.txt
│       └── Dockerfile
├── packages/
│   ├── shared/                # Shared utilities
│   ├── types/                 # TypeScript type definitions
│   └── config/                # Shared configuration
├── infrastructure/
│   ├── terraform/             # Infrastructure as code
│   ├── docker-compose.yml     # Local development
│   └── kubernetes/            # K8s manifests
├── tests/
│   ├── integration/           # Integration tests
│   └── e2e/                   # End-to-end tests
├── docs/                      # Documentation
├── scripts/                   # Build and utility scripts
├── pnpm-workspace.yaml        # PNPM workspace config
├── package.json               # Root package.json
└── README.md
```

## 🔧 Workspace Configuration

### PNPM Workspace
```yaml
# pnpm-workspace.yaml
packages:
  - 'apps/*'
  - 'packages/*'
  - 'tests/*'
```

### Root Package.json
```json
{
  "name": "full-stack-app",
  "private": true,
  "workspaces": [
    "apps/*",
    "packages/*",
    "tests/*"
  ],
  "scripts": {
    "dev": "pnpm run --parallel dev",
    "build": "pnpm run --recursive build",
    "test": "pnpm run --recursive test",
    "lint": "pnpm run --recursive lint",
    "type-check": "pnpm run --recursive type-check",
    "docker:build": "docker-compose build",
    "docker:up": "docker-compose up",
    "docker:down": "docker-compose down"
  },
  "devDependencies": {
    "@typescript-eslint/eslint-plugin": "^6.0.0",
    "@typescript-eslint/parser": "^6.0.0",
    "eslint": "^8.0.0",
    "prettier": "^3.0.0",
    "typescript": "^5.0.0",
    "husky": "^8.0.0",
    "lint-staged": "^14.0.0"
  }
}
```

## 🎯 Frontend Application

### Next.js Configuration
```typescript
// apps/frontend/next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    appDir: true,
  },
  env: {
    API_BASE_URL: process.env.API_BASE_URL || 'http://localhost:3001',
    ANALYTICS_URL: process.env.ANALYTICS_URL || 'http://localhost:5000',
  },
  async rewrites() {
    return [
      {
        source: '/api/:path*',
        destination: `${process.env.API_BASE_URL}/api/:path*`,
      },
    ];
  },
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          {
            key: 'X-Content-Type-Options',
            value: 'nosniff',
          },
          {
            key: 'X-Frame-Options',
            value: 'DENY',
          },
          {
            key: 'X-XSS-Protection',
            value: '1; mode=block',
          },
        ],
      },
    ];
  },
  output: 'standalone',
};

module.exports = nextConfig;
```

### React Component with Shared Types
```typescript
// apps/frontend/src/components/UserDashboard.tsx
import { User, ApiResponse } from '@full-stack-app/types';
import { useApi } from '@full-stack-app/shared/hooks';
import { Card, Button } from '@full-stack-app/ui';

interface UserDashboardProps {
  userId: string;
}

export function UserDashboard({ userId }: UserDashboardProps) {
  const { data: user, loading, error } = useApi<User>(`/users/${userId}`);
  const { data: analytics } = useApi<any>(`/analytics/users/${userId}`);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <div className="space-y-6">
      <Card>
        <h1 className="text-2xl font-bold">{user?.name}</h1>
        <p className="text-gray-600">{user?.email}</p>
      </Card>
      
      {analytics && (
        <Card>
          <h2 className="text-xl font-semibold">Analytics</h2>
          <div className="grid grid-cols-3 gap-4 mt-4">
            <div>
              <span className="text-2xl font-bold">{analytics.totalOrders}</span>
              <p className="text-sm text-gray-600">Total Orders</p>
            </div>
            <div>
              <span className="text-2xl font-bold">${analytics.totalSpent}</span>
              <p className="text-sm text-gray-600">Total Spent</p>
            </div>
            <div>
              <span className="text-2xl font-bold">{analytics.lastLoginDays}</span>
              <p className="text-sm text-gray-600">Days Since Last Login</p>
            </div>
          </div>
        </Card>
      )}
    </div>
  );
}
```

## 🔧 Backend Services

### Node.js API Server
```typescript
// apps/backend/src/server.ts
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import { createProxyMiddleware } from 'http-proxy-middleware';
import { config } from '@full-stack-app/config';
import { logger } from '@full-stack-app/shared/logger';
import { userRoutes } from './routes/users';
import { orderRoutes } from './routes/orders';
import { authMiddleware } from './middleware/auth';

const app = express();
const port = config.api.port || 3001;

// Middleware
app.use(helmet());
app.use(cors());
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Logging middleware
app.use((req, res, next) => {
  logger.info(`${req.method} ${req.path}`, {
    userAgent: req.get('User-Agent'),
    ip: req.ip,
  });
  next();
});

// Health check
app.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    version: process.env.npm_package_version,
  });
});

// Proxy to analytics service
app.use('/analytics', createProxyMiddleware({
  target: config.analytics.url,
  changeOrigin: true,
  pathRewrite: {
    '^/analytics': '',
  },
}));

// API routes
app.use('/api/users', authMiddleware, userRoutes);
app.use('/api/orders', authMiddleware, orderRoutes);

// Error handling
app.use((err: any, req: express.Request, res: express.Response, next: express.NextFunction) => {
  logger.error('Unhandled error:', err);
  res.status(500).json({
    error: 'Internal Server Error',
    message: process.env.NODE_ENV === 'development' ? err.message : 'Something went wrong',
  });
});

app.listen(port, () => {
  logger.info(`Server running on port ${port}`);
});

export default app;
```

### Python Analytics Service
```python
# apps/analytics/src/main.py
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from sqlalchemy.orm import Session
from typing import List, Optional
import logging

from .database import SessionLocal, engine
from .models import UserAnalytics, OrderAnalytics
from .schemas import UserAnalyticsResponse, OrderAnalyticsResponse
from .services import AnalyticsService

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(
    title="Analytics Service",
    description="Analytics microservice for user and order data",
    version="1.0.0"
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "service": "analytics",
        "version": "1.0.0"
    }

@app.get("/users/{user_id}", response_model=UserAnalyticsResponse)
async def get_user_analytics(
    user_id: str,
    db: Session = Depends(get_db)
):
    try:
        service = AnalyticsService(db)
        analytics = service.get_user_analytics(user_id)
        
        if not analytics:
            raise HTTPException(status_code=404, detail="User analytics not found")
        
        return analytics
    except Exception as e:
        logger.error(f"Error getting user analytics: {e}")
        raise HTTPException(status_code=500, detail="Internal server error")

@app.post("/users/{user_id}/events")
async def track_user_event(
    user_id: str,
    event_data: dict,
    db: Session = Depends(get_db)
):
    try:
        service = AnalyticsService(db)
        service.track_user_event(user_id, event_data)
        return {"message": "Event tracked successfully"}
    except Exception as e:
        logger.error(f"Error tracking event: {e}")
        raise HTTPException(status_code=500, detail="Internal server error")

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=5000)
```

## 🧪 Shared Packages

### Shared Types Package
```typescript
// packages/types/src/index.ts
export interface User {
  id: string;
  name: string;
  email: string;
  createdAt: string;
  updatedAt?: string;
}

export interface Order {
  id: string;
  userId: string;
  items: OrderItem[];
  total: number;
  status: OrderStatus;
  createdAt: string;
}

export interface OrderItem {
  id: string;
  productId: string;
  quantity: number;
  price: number;
}

export type OrderStatus = 'pending' | 'processing' | 'shipped' | 'delivered' | 'cancelled';

export interface ApiResponse<T> {
  data: T;
  message?: string;
  error?: string;
}

export interface PaginatedResponse<T> {
  items: T[];
  total: number;
  page: number;
  pageSize: number;
}

export interface AnalyticsData {
  totalOrders: number;
  totalSpent: number;
  lastLoginDays: number;
  topProducts: string[];
}
```

### Shared Configuration
```typescript
// packages/config/src/index.ts
import { z } from 'zod';

const configSchema = z.object({
  api: z.object({
    port: z.number().default(3001),
    baseUrl: z.string().default('http://localhost:3001'),
  }),
  analytics: z.object({
    url: z.string().default('http://localhost:5000'),
  }),
  database: z.object({
    url: z.string(),
    ssl: z.boolean().default(false),
  }),
  redis: z.object({
    url: z.string().default('redis://localhost:6379'),
  }),
  aws: z.object({
    region: z.string().default('us-east-1'),
    s3Bucket: z.string().optional(),
    sqsQueueUrl: z.string().optional(),
  }),
  azure: z.object({
    storageAccount: z.string().optional(),
    serviceBusNamespace: z.string().optional(),
  }),
});

export const config = configSchema.parse({
  api: {
    port: parseInt(process.env.API_PORT || '3001'),
    baseUrl: process.env.API_BASE_URL || 'http://localhost:3001',
  },
  analytics: {
    url: process.env.ANALYTICS_URL || 'http://localhost:5000',
  },
  database: {
    url: process.env.DATABASE_URL!,
    ssl: process.env.NODE_ENV === 'production',
  },
  redis: {
    url: process.env.REDIS_URL || 'redis://localhost:6379',
  },
  aws: {
    region: process.env.AWS_REGION || 'us-east-1',
    s3Bucket: process.env.AWS_S3_BUCKET,
    sqsQueueUrl: process.env.AWS_SQS_QUEUE_URL,
  },
  azure: {
    storageAccount: process.env.AZURE_STORAGE_ACCOUNT,
    serviceBusNamespace: process.env.AZURE_SERVICE_BUS_NAMESPACE,
  },
});

export type Config = z.infer<typeof configSchema>;
```

## 🧪 Testing Strategy

### Integration Tests
```typescript
// tests/integration/api.test.ts
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import request from 'supertest';
import { app } from '../../apps/backend/src/server';
import { setupTestDatabase, cleanupTestDatabase } from './helpers/database';

describe('API Integration Tests', () => {
  beforeAll(async () => {
    await setupTestDatabase();
  });

  afterAll(async () => {
    await cleanupTestDatabase();
  });

  describe('Users API', () => {
    it('should create and retrieve user', async () => {
      const userData = {
        name: 'Test User',
        email: 'test@example.com',
      };

      // Create user
      const createResponse = await request(app)
        .post('/api/users')
        .send(userData)
        .expect(201);

      expect(createResponse.body.data.name).toBe(userData.name);
      expect(createResponse.body.data.email).toBe(userData.email);

      // Get user
      const userId = createResponse.body.data.id;
      const getResponse = await request(app)
        .get(`/api/users/${userId}`)
        .expect(200);

      expect(getResponse.body.data.id).toBe(userId);
      expect(getResponse.body.data.name).toBe(userData.name);
    });
  });

  describe('Orders API', () => {
    it('should create order for user', async () => {
      // First create a user
      const userResponse = await request(app)
        .post('/api/users')
        .send({ name: 'Test User', email: 'test@example.com' });

      const userId = userResponse.body.data.id;

      // Create order
      const orderData = {
        userId,
        items: [
          { productId: 'prod1', quantity: 2, price: 10.99 },
          { productId: 'prod2', quantity: 1, price: 25.50 },
        ],
      };

      const orderResponse = await request(app)
        .post('/api/orders')
        .send(orderData)
        .expect(201);

      expect(orderResponse.body.data.userId).toBe(userId);
      expect(orderResponse.body.data.items).toHaveLength(2);
      expect(orderResponse.body.data.total).toBe(47.48);
    });
  });
});
```

### E2E Tests with Playwright
```typescript
// tests/e2e/user-journey.spec.ts
import { test, expect } from '@playwright/test';

test.describe('User Journey', () => {
  test('complete user registration and order flow', async ({ page }) => {
    // Navigate to registration page
    await page.goto('/register');
    
    // Fill registration form
    await page.fill('[data-testid=name-input]', 'John Doe');
    await page.fill('[data-testid=email-input]', 'john@example.com');
    await page.fill('[data-testid=password-input]', 'securePassword123');
    await page.click('[data-testid=register-button]');
    
    // Verify redirect to dashboard
    await expect(page).toHaveURL('/dashboard');
    await expect(page.locator('[data-testid=welcome-message]')).toContainText('Welcome, John');
    
    // Navigate to products
    await page.click('[data-testid=products-link]');
    await expect(page).toHaveURL('/products');
    
    // Add product to cart
    await page.click('[data-testid=product-1] [data-testid=add-to-cart]');
    await expect(page.locator('[data-testid=cart-count]')).toContainText('1');
    
    // Go to checkout
    await page.click('[data-testid=cart-link]');
    await page.click('[data-testid=checkout-button]');
    
    // Complete order
    await page.fill('[data-testid=shipping-address]', '123 Main St');
    await page.fill('[data-testid=credit-card]', '4111111111111111');
    await page.click('[data-testid=place-order-button]');
    
    // Verify order confirmation
    await expect(page.locator('[data-testid=order-success]')).toBeVisible();
  });
});
```

## 🚀 Deployment & Infrastructure

### Docker Compose for Development
```yaml
# docker-compose.yml
version: '3.8'

services:
  frontend:
    build:
      context: ./apps/frontend
      target: development
    ports:
      - "3000:3000"
    volumes:
      - ./apps/frontend:/app
      - /app/node_modules
    environment:
      - NEXT_PUBLIC_API_URL=http://localhost:3001
    depends_on:
      - backend

  backend:
    build:
      context: ./apps/backend
      target: development
    ports:
      - "3001:3001"
    volumes:
      - ./apps/backend:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://app:password@postgres:5432/fullstack_app
      - REDIS_URL=redis://redis:6379
      - ANALYTICS_URL=http://analytics:5000
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  analytics:
    build:
      context: ./apps/analytics
    ports:
      - "5000:5000"
    environment:
      - DATABASE_URL=mongodb://mongo:27017/analytics
      - REDIS_URL=redis://redis:6379
    depends_on:
      - mongo
      - redis

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: password
      POSTGRES_DB: fullstack_app
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 10s
      timeout: 5s
      retries: 5

  mongo:
    image: mongo:7
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

volumes:
  postgres_data:
  mongo_data:
  redis_data:
```

## 📊 Monitoring & Observability

### Application Metrics
```typescript
// packages/shared/src/monitoring.ts
import prometheus from 'prom-client';

export const register = new prometheus.Registry();

export const httpRequestDuration = new prometheus.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code', 'service'],
  buckets: [0.1, 0.3, 0.5, 0.7, 1, 3, 5, 7, 10],
  registers: [register],
});

export const httpRequestsTotal = new prometheus.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code', 'service'],
  registers: [register],
});

export const activeConnections = new prometheus.Gauge({
  name: 'active_connections_total',
  help: 'Total number of active connections',
  labelNames: ['service'],
  registers: [register],
});

// Middleware for Express
export const metricsMiddleware = (serviceName: string) => {
  return (req: any, res: any, next: any) => {
    const start = Date.now();
    
    res.on('finish', () => {
      const duration = (Date.now() - start) / 1000;
      const route = req.route ? req.route.path : req.path;
      
      httpRequestDuration
        .labels(req.method, route, res.statusCode, serviceName)
        .observe(duration);
      
      httpRequestsTotal
        .labels(req.method, route, res.statusCode, serviceName)
        .inc();
    });
    
    next();
  };
};
```

## 📚 Best Practices

### 1. Monorepo Management
- Use workspace dependencies effectively
- Implement consistent tooling across packages
- Set up proper build dependencies
- Use shared configurations

### 2. Code Quality
- Enforce consistent code style with Prettier
- Use ESLint for code quality
- Implement TypeScript strict mode
- Set up pre-commit hooks with Husky

### 3. Testing Strategy
- Unit tests for individual packages
- Integration tests for API endpoints
- E2E tests for critical user journeys
- Performance tests for key scenarios

### 4. Deployment
- Progressive deployment strategies
- Environment-specific configurations
- Infrastructure as Code
- Monitoring and alerting

### 5. Security
- Regular dependency updates
- Security scanning in CI/CD
- Secrets management
- API rate limiting and authentication

---

💡 **Dica**: Monorepos são poderosos mas complexos. Comece simples e evolua a estrutura conforme o projeto cresce!