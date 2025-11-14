# Container Web Application Example

Este exemplo demonstra uma aplicação web containerizada com CI/CD completo incluindo multi-arch builds, security scanning e blue-green deployment.

## 📋 Sobre a Aplicação

- **Frontend**: React/Next.js
- **Backend**: Express.js API
- **Database**: PostgreSQL
- **Cache**: Redis
- **Containerização**: Docker multi-stage builds
- **Deployment**: AWS ECS + Azure Container Instances

## 🏗️ Arquitetura

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Load          │───▶│   Frontend      │───▶│   Backend       │
│   Balancer      │    │   Container     │    │   Container     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Container     │    │   Redis         │    │   PostgreSQL    │
│   Registry      │    │   Container     │    │   Container     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## 🚀 Workflow Configuration

```yaml
# .github/workflows/container-app.yml
name: 'Container Web Application CI/CD'

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME_FRONTEND: ${{ github.repository }}/frontend
  IMAGE_NAME_BACKEND: ${{ github.repository }}/backend

jobs:
  # Test Frontend
  test-frontend:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/ci-reusable.yml@main
    with:
      node-version: '18'
      working-directory: 'frontend'
      run-tests: true
      run-lint: true
      build-command: 'npm run build'

  # Test Backend
  test-backend:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/ci-reusable.yml@main
    with:
      node-version: '18'
      working-directory: 'backend'
      run-tests: true
      run-lint: true
      run-security-scan: true

  # Build Frontend Container
  build-frontend:
    needs: test-frontend
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/docker-build.yml@main
    with:
      image-name: ${{ env.IMAGE_NAME_FRONTEND }}
      context: './frontend'
      dockerfile: './frontend/Dockerfile'
      platforms: 'linux/amd64,linux/arm64'
      push-to-registry: true
      scan-image: true
    secrets:
      REGISTRY_USERNAME: ${{ github.actor }}
      REGISTRY_PASSWORD: ${{ secrets.GITHUB_TOKEN }}

  # Build Backend Container
  build-backend:
    needs: test-backend
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/docker-build.yml@main
    with:
      image-name: ${{ env.IMAGE_NAME_BACKEND }}
      context: './backend'
      dockerfile: './backend/Dockerfile'
      platforms: 'linux/amd64,linux/arm64'
      push-to-registry: true
      scan-image: true
    secrets:
      REGISTRY_USERNAME: ${{ github.actor }}
      REGISTRY_PASSWORD: ${{ secrets.GITHUB_TOKEN }}

  # Deploy to Development (AWS ECS)
  deploy-dev-aws:
    if: github.ref == 'refs/heads/develop'
    needs: [build-frontend, build-backend]
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/aws-deploy.yml@main
    with:
      environment: 'development'
      deploy-target: 'ecs'
      cluster-name: 'webapp-dev-cluster'
      service-name: 'webapp-dev-service'
      frontend-image: ${{ needs.build-frontend.outputs.image }}
      backend-image: ${{ needs.build-backend.outputs.image }}
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_DEV_ROLE_ARN }}

  # Deploy to Staging (Azure Container Instances)
  deploy-staging-azure:
    if: github.ref == 'refs/heads/develop'
    needs: [build-frontend, build-backend]
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/azure-deploy.yml@main
    with:
      environment: 'staging'
      deploy-target: 'aci'
      resource-group: 'webapp-staging-rg'
      container-group: 'webapp-staging'
      frontend-image: ${{ needs.build-frontend.outputs.image }}
      backend-image: ${{ needs.build-backend.outputs.image }}
    secrets:
      AZURE_CREDENTIALS: ${{ secrets.AZURE_CREDENTIALS }}

  # Blue-Green Deployment to Production
  deploy-prod:
    if: github.ref == 'refs/heads/main'
    needs: [build-frontend, build-backend]
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/blue-green-deploy.yml@main
    with:
      environment: 'production'
      deploy-target: 'ecs'
      cluster-name: 'webapp-prod-cluster'
      service-name: 'webapp-prod-service'
      frontend-image: ${{ needs.build-frontend.outputs.image }}
      backend-image: ${{ needs.build-backend.outputs.image }}
      health-check-url: 'https://webapp.mycompany.com/health'
      smoke-test-url: 'https://webapp.mycompany.com/api/health'
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_PROD_ROLE_ARN }}
```

## 📁 Estrutura do Projeto

```
container-webapp/
├── .github/
│   └── workflows/
│       └── container-app.yml
├── frontend/
│   ├── src/
│   ├── public/
│   ├── package.json
│   ├── Dockerfile
│   └── nginx.conf
├── backend/
│   ├── src/
│   ├── tests/
│   ├── package.json
│   └── Dockerfile
├── infrastructure/
│   ├── aws/
│   │   ├── ecs-cluster.tf
│   │   ├── load-balancer.tf
│   │   └── rds.tf
│   └── azure/
│       ├── container-instances.tf
│       └── database.tf
├── docker-compose.yml
├── docker-compose.prod.yml
└── README.md
```

## 🐳 Docker Configurations

### Frontend Dockerfile (Multi-stage)

```dockerfile
# frontend/Dockerfile
# Stage 1: Build
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Copy source code and build
COPY . .
RUN npm run build

# Stage 2: Production
FROM nginx:alpine AS production

# Install security updates
RUN apk update && apk upgrade && apk add --no-cache dumb-init

# Copy custom nginx config
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy built application
COPY --from=builder /app/dist /usr/share/nginx/html

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001

# Change ownership
RUN chown -R nextjs:nodejs /usr/share/nginx/html && \
    chown -R nextjs:nodejs /var/cache/nginx && \
    chown -R nextjs:nodejs /var/log/nginx && \
    chown -R nextjs:nodejs /etc/nginx/conf.d

# Switch to non-root user
USER nextjs

EXPOSE 80

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:80/health || exit 1

CMD ["nginx", "-g", "daemon off;"]
```

### Backend Dockerfile (Multi-stage)

```dockerfile
# backend/Dockerfile
# Stage 1: Build
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Stage 2: Development (for local development)
FROM node:18-alpine AS development

WORKDIR /app

# Install development dependencies
COPY package*.json ./
RUN npm ci

# Copy source code
COPY . .

CMD ["npm", "run", "dev"]

# Stage 3: Production
FROM node:18-alpine AS production

# Install security updates and dumb-init
RUN apk update && apk upgrade && apk add --no-cache dumb-init curl

WORKDIR /app

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Copy dependencies from builder
COPY --from=builder /app/node_modules ./node_modules
COPY --chown=nodejs:nodejs . .

# Switch to non-root user
USER nodejs

EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# Use dumb-init to handle signals properly
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "src/server.js"]
```

## ⚙️ Nginx Configuration

```nginx
# frontend/nginx.conf
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';" always;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }

    # API proxy
    location /api/ {
        proxy_pass http://backend:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # SPA fallback
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

## 🏗️ Infrastructure as Code

### AWS ECS Configuration

```hcl
# infrastructure/aws/ecs-cluster.tf
resource "aws_ecs_cluster" "webapp" {
  name = "${var.environment}-webapp-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }

  tags = {
    Environment = var.environment
    Project     = "webapp"
  }
}

resource "aws_ecs_service" "webapp" {
  name            = "${var.environment}-webapp-service"
  cluster         = aws_ecs_cluster.webapp.id
  task_definition = aws_ecs_task_definition.webapp.arn
  desired_count   = var.environment == "production" ? 3 : 1

  deployment_configuration {
    maximum_percent         = 200
    minimum_healthy_percent = 100
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.webapp.arn
    container_name   = "frontend"
    container_port   = 80
  }

  depends_on = [aws_lb_listener.webapp]

  tags = {
    Environment = var.environment
    Project     = "webapp"
  }
}

resource "aws_ecs_task_definition" "webapp" {
  family                   = "${var.environment}-webapp"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = 512
  memory                   = 1024
  execution_role_arn       = aws_iam_role.ecs_task_execution.arn
  task_role_arn           = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([
    {
      name  = "frontend"
      image = "${var.frontend_image_url}:${var.image_tag}"
      
      portMappings = [
        {
          containerPort = 80
          protocol      = "tcp"
        }
      ]

      environment = [
        {
          name  = "NODE_ENV"
          value = var.environment
        }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          awslogs-group         = aws_cloudwatch_log_group.webapp.name
          awslogs-region        = var.aws_region
          awslogs-stream-prefix = "frontend"
        }
      }

      healthCheck = {
        command     = ["CMD-SHELL", "curl -f http://localhost:80/health || exit 1"]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 60
      }
    },
    {
      name  = "backend"
      image = "${var.backend_image_url}:${var.image_tag}"
      
      portMappings = [
        {
          containerPort = 3000
          protocol      = "tcp"
        }
      ]

      environment = [
        {
          name  = "NODE_ENV"
          value = var.environment
        },
        {
          name  = "DATABASE_URL"
          value = "postgresql://${aws_db_instance.webapp.username}:${var.db_password}@${aws_db_instance.webapp.endpoint}:5432/${aws_db_instance.webapp.db_name}"
        },
        {
          name  = "REDIS_URL"
          value = "redis://${aws_elasticache_replication_group.webapp.configuration_endpoint_address}:6379"
        }
      ]

      secrets = [
        {
          name      = "JWT_SECRET"
          valueFrom = aws_secretsmanager_secret.jwt_secret.arn
        }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          awslogs-group         = aws_cloudwatch_log_group.webapp.name
          awslogs-region        = var.aws_region
          awslogs-stream-prefix = "backend"
        }
      }

      healthCheck = {
        command     = ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 60
      }
    }
  ])

  tags = {
    Environment = var.environment
    Project     = "webapp"
  }
}
```

### Azure Container Instances

```hcl
# infrastructure/azure/container-instances.tf
resource "azurerm_container_group" "webapp" {
  name                = "${var.environment}-webapp"
  location            = var.location
  resource_group_name = azurerm_resource_group.webapp.name
  ip_address_type     = "Public"
  dns_name_label      = "${var.environment}-webapp-${random_string.dns_suffix.result}"
  os_type             = "Linux"

  container {
    name   = "frontend"
    image  = "${var.frontend_image_url}:${var.image_tag}"
    cpu    = "0.5"
    memory = "1.0"

    ports {
      port     = 80
      protocol = "TCP"
    }

    environment_variables = {
      NODE_ENV = var.environment
    }

    liveness_probe {
      http_get {
        path   = "/health"
        port   = 80
        scheme = "Http"
      }
      initial_delay_seconds = 60
      period_seconds        = 30
      timeout_seconds       = 5
      failure_threshold     = 3
    }

    readiness_probe {
      http_get {
        path   = "/health"
        port   = 80
        scheme = "Http"
      }
      initial_delay_seconds = 30
      period_seconds        = 10
      timeout_seconds       = 3
      failure_threshold     = 3
    }
  }

  container {
    name   = "backend"
    image  = "${var.backend_image_url}:${var.image_tag}"
    cpu    = "0.5"
    memory = "1.0"

    ports {
      port     = 3000
      protocol = "TCP"
    }

    environment_variables = {
      NODE_ENV     = var.environment
      DATABASE_URL = "postgresql://${azurerm_postgresql_server.webapp.administrator_login}:${var.db_password}@${azurerm_postgresql_server.webapp.fqdn}:5432/${azurerm_postgresql_database.webapp.name}"
      REDIS_URL    = "redis://${azurerm_redis_cache.webapp.hostname}:${azurerm_redis_cache.webapp.ssl_port}"
    }

    secure_environment_variables = {
      JWT_SECRET = var.jwt_secret
    }
  }

  tags = {
    Environment = var.environment
    Project     = "webapp"
  }
}
```

## 🐳 Docker Compose for Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  frontend:
    build:
      context: ./frontend
      target: development
    ports:
      - "3000:3000"
    volumes:
      - ./frontend:/app
      - /app/node_modules
    environment:
      - CHOKIDAR_USEPOLLING=true
    depends_on:
      - backend

  backend:
    build:
      context: ./backend
      target: development
    ports:
      - "3001:3000"
    volumes:
      - ./backend:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://webapp:webapp@postgres:5432/webapp
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=dev-secret-key
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: webapp
      POSTGRES_PASSWORD: webapp
      POSTGRES_DB: webapp
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./backend/db/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U webapp"]
      interval: 10s
      timeout: 5s
      retries: 5

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

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - frontend
      - backend

volumes:
  postgres_data:
  redis_data:
```

## 🧪 Testing Strategy

### Frontend Tests
```javascript
// frontend/src/components/__tests__/UserList.test.jsx
import { render, screen, waitFor } from '@testing-library/react';
import { UserList } from '../UserList';
import { QueryClient, QueryClientProvider } from 'react-query';

const createTestQueryClient = () => new QueryClient({
  defaultOptions: {
    queries: { retry: false },
    mutations: { retry: false },
  },
});

const renderWithQueryClient = (component) => {
  const testQueryClient = createTestQueryClient();
  return render(
    <QueryClientProvider client={testQueryClient}>
      {component}
    </QueryClientProvider>
  );
};

describe('UserList', () => {
  it('renders user list', async () => {
    renderWithQueryClient(<UserList />);
    
    await waitFor(() => {
      expect(screen.getByText('Users')).toBeInTheDocument();
    });
  });
});
```

### Backend Tests
```javascript
// backend/tests/integration/users.test.js
const request = require('supertest');
const app = require('../../src/app');

describe('Users API', () => {
  beforeEach(async () => {
    await setupTestDatabase();
  });

  afterEach(async () => {
    await cleanupTestDatabase();
  });

  describe('GET /api/users', () => {
    it('should return all users', async () => {
      const response = await request(app)
        .get('/api/users')
        .expect(200);

      expect(response.body).toHaveProperty('users');
      expect(Array.isArray(response.body.users)).toBe(true);
    });
  });

  describe('POST /api/users', () => {
    it('should create a new user', async () => {
      const userData = {
        name: 'Test User',
        email: 'test@example.com'
      };

      const response = await request(app)
        .post('/api/users')
        .send(userData)
        .expect(201);

      expect(response.body.user.name).toBe(userData.name);
      expect(response.body.user.email).toBe(userData.email);
    });
  });
});
```

## 📊 Security & Monitoring

### Security Scanning
```yaml
# Integrated in docker-build workflow
security_scan:
  runs-on: ubuntu-latest
  steps:
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
        format: 'sarif'
        output: 'trivy-results.sarif'

    - name: Upload Trivy scan results to GitHub Security tab
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'
```

### Application Monitoring
```javascript
// backend/src/middleware/monitoring.js
const prometheus = require('prom-client');

// Create metrics
const httpRequestDuration = new prometheus.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.1, 0.3, 0.5, 0.7, 1, 3, 5, 7, 10]
});

const httpRequestsTotal = new prometheus.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code']
});

module.exports = (req, res, next) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    const route = req.route ? req.route.path : req.path;
    
    httpRequestDuration
      .labels(req.method, route, res.statusCode)
      .observe(duration);
    
    httpRequestsTotal
      .labels(req.method, route, res.statusCode)
      .inc();
  });
  
  next();
};
```

## 🚀 Deployment Strategies

### Blue-Green Deployment
1. Deploy new version (Green) alongside current (Blue)
2. Run health checks and smoke tests
3. Switch traffic from Blue to Green
4. Monitor for issues
5. Keep Blue as rollback option

### Canary Deployment
1. Deploy new version to subset of infrastructure
2. Route small percentage of traffic to new version
3. Monitor metrics and user feedback
4. Gradually increase traffic percentage
5. Complete rollout if successful

## 📚 Best Practices

### 1. Container Security
- Use non-root users in containers
- Scan images for vulnerabilities
- Use minimal base images
- Keep dependencies updated

### 2. Performance
- Implement multi-stage builds
- Use appropriate resource limits
- Configure proper health checks
- Optimize image layers

### 3. Monitoring
- Centralized logging
- Application metrics
- Infrastructure monitoring
- Alerting on critical issues

### 4. Deployment
- Automated testing at all levels
- Gradual rollout strategies
- Quick rollback capabilities
- Infrastructure as Code

---

💡 **Dica**: Containerize early but don't over-engineer. Start simple and add complexity as your application scales!