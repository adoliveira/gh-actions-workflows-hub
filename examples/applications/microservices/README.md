# Microservices Architecture Example

Este exemplo demonstra como implementar CI/CD para uma arquitetura de microsserviços usando os workflows hub.

## 📋 Sobre a Arquitetura

- **API Gateway**: Ponto único de entrada
- **User Service**: Gerenciamento de usuários (Node.js)
- **Order Service**: Processamento de pedidos (Python)
- **Payment Service**: Processamento de pagamentos (Java)
- **Notification Service**: Envio de notificações (Go)

## 🏗️ Arquitetura

```
                    ┌─────────────────┐
                    │   API Gateway   │
                    └─────────────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
   ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
   │  User Service   │ │  Order Service  │ │ Payment Service │
   │    (Node.js)    │ │    (Python)     │ │     (Java)      │
   └─────────────────┘ └─────────────────┘ └─────────────────┘
            │                │                │
            └────────────────┼────────────────┘
                             │
                   ┌─────────────────┐
                   │ Notification    │
                   │ Service (Go)    │
                   └─────────────────┘
                             │
                   ┌─────────────────┐
                   │   Message       │
                   │   Queue         │
                   └─────────────────┘
```

## 📁 Estrutura do Projeto

```
microservices/
├── .github/
│   └── workflows/
│       ├── user-service.yml
│       ├── order-service.yml
│       ├── payment-service.yml
│       ├── notification-service.yml
│       └── integration-tests.yml
├── services/
│   ├── user-service/
│   │   ├── src/
│   │   ├── tests/
│   │   ├── Dockerfile
│   │   └── package.json
│   ├── order-service/
│   │   ├── src/
│   │   ├── tests/
│   │   ├── Dockerfile
│   │   └── requirements.txt
│   ├── payment-service/
│   │   ├── src/
│   │   ├── tests/
│   │   ├── Dockerfile
│   │   └── pom.xml
│   └── notification-service/
│       ├── main.go
│       ├── Dockerfile
│       └── go.mod
├── infrastructure/
│   ├── terraform/
│   └── docker-compose.yml
├── tests/
│   └── integration/
└── README.md
```

## 🚀 Workflows

### User Service (Node.js)

```yaml
# .github/workflows/user-service.yml
name: 'User Service CI/CD'

on:
  push:
    branches: [ main, develop ]
    paths: [ 'services/user-service/**' ]
  pull_request:
    branches: [ main ]
    paths: [ 'services/user-service/**' ]

jobs:
  # Test and Build
  ci:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/ci-reusable.yml@main
    with:
      node-version: '18'
      package-manager: 'npm'
      working-directory: 'services/user-service'
      run-tests: true
      run-lint: true
      run-security-scan: true

  # Docker Build
  docker:
    needs: ci
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/docker-build.yml@main
    with:
      image-name: 'user-service'
      context: 'services/user-service'
      platforms: 'linux/amd64,linux/arm64'
      push-to-registry: true
    secrets:
      REGISTRY_USERNAME: ${{ secrets.GHCR_USERNAME }}
      REGISTRY_PASSWORD: ${{ secrets.GHCR_TOKEN }}

  # Deploy to Development
  deploy-dev:
    if: github.ref == 'refs/heads/develop'
    needs: [ci, docker]
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/aws-deploy.yml@main
    with:
      environment: 'development'
      service-name: 'user-service'
      deploy-target: 'ecs'
      image-tag: ${{ needs.docker.outputs.image-tag }}
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_DEV_ROLE_ARN }}

  # Deploy to Production
  deploy-prod:
    if: github.ref == 'refs/heads/main'
    needs: [ci, docker]
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/blue-green-deploy.yml@main
    with:
      environment: 'production'
      service-name: 'user-service'
      deploy-target: 'ecs'
      image-tag: ${{ needs.docker.outputs.image-tag }}
      health-check-url: 'https://api.myapp.com/user-service/health'
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_PROD_ROLE_ARN }}
```

### Order Service (Python)

```yaml
# .github/workflows/order-service.yml
name: 'Order Service CI/CD'

on:
  push:
    branches: [ main, develop ]
    paths: [ 'services/order-service/**' ]
  pull_request:
    branches: [ main ]
    paths: [ 'services/order-service/**' ]

jobs:
  # Test and Build
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          
      - name: Install dependencies
        working-directory: services/order-service
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install pytest pytest-cov flake8 black
          
      - name: Code formatting
        working-directory: services/order-service
        run: |
          black --check src/ tests/
          
      - name: Linting
        working-directory: services/order-service
        run: |
          flake8 src/ tests/
          
      - name: Run tests
        working-directory: services/order-service
        run: |
          pytest tests/ --cov=src --cov-report=xml
          
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: services/order-service/coverage.xml

  # Docker Build
  docker:
    needs: ci
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/docker-build.yml@main
    with:
      image-name: 'order-service'
      context: 'services/order-service'
      platforms: 'linux/amd64,linux/arm64'
      push-to-registry: true
    secrets:
      REGISTRY_USERNAME: ${{ secrets.GHCR_USERNAME }}
      REGISTRY_PASSWORD: ${{ secrets.GHCR_TOKEN }}

  # Deploy steps similar to user-service...
```

### Payment Service (Java)

```yaml
# .github/workflows/payment-service.yml
name: 'Payment Service CI/CD'

on:
  push:
    branches: [ main, develop ]
    paths: [ 'services/payment-service/**' ]
  pull_request:
    branches: [ main ]
    paths: [ 'services/payment-service/**' ]

jobs:
  # Test and Build
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Java
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
          
      - name: Cache Maven dependencies
        uses: actions/cache@v3
        with:
          path: ~/.m2
          key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
          
      - name: Run tests
        working-directory: services/payment-service
        run: |
          mvn clean test
          
      - name: Build application
        working-directory: services/payment-service
        run: |
          mvn clean package -DskipTests
          
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: services/payment-service/target/site/jacoco/jacoco.xml

  # Docker Build
  docker:
    needs: ci
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/docker-build.yml@main
    with:
      image-name: 'payment-service'
      context: 'services/payment-service'
      platforms: 'linux/amd64,linux/arm64'
      push-to-registry: true
    secrets:
      REGISTRY_USERNAME: ${{ secrets.GHCR_USERNAME }}
      REGISTRY_PASSWORD: ${{ secrets.GHCR_TOKEN }}

  # Deploy steps...
```

### Notification Service (Go)

```yaml
# .github/workflows/notification-service.yml
name: 'Notification Service CI/CD'

on:
  push:
    branches: [ main, develop ]
    paths: [ 'services/notification-service/**' ]
  pull_request:
    branches: [ main ]
    paths: [ 'services/notification-service/**' ]

jobs:
  # Test and Build
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Go
        uses: actions/setup-go@v4
        with:
          go-version: '1.21'
          
      - name: Cache Go modules
        uses: actions/cache@v3
        with:
          path: ~/go/pkg/mod
          key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
          
      - name: Install dependencies
        working-directory: services/notification-service
        run: go mod download
        
      - name: Run tests
        working-directory: services/notification-service
        run: |
          go test -v -race -coverprofile=coverage.out ./...
          go tool cover -html=coverage.out -o coverage.html
          
      - name: Build application
        working-directory: services/notification-service
        run: |
          CGO_ENABLED=0 GOOS=linux go build -o notification-service main.go

  # Docker Build
  docker:
    needs: ci
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/docker-build.yml@main
    with:
      image-name: 'notification-service'
      context: 'services/notification-service'
      platforms: 'linux/amd64,linux/arm64'
      push-to-registry: true
    secrets:
      REGISTRY_USERNAME: ${{ secrets.GHCR_USERNAME }}
      REGISTRY_PASSWORD: ${{ secrets.GHCR_TOKEN }}

  # Deploy steps...
```

### Integration Tests

```yaml
# .github/workflows/integration-tests.yml
name: 'Integration Tests'

on:
  workflow_run:
    workflows: [
      "User Service CI/CD",
      "Order Service CI/CD", 
      "Payment Service CI/CD",
      "Notification Service CI/CD"
    ]
    types:
      - completed
    branches: [ main, develop ]

jobs:
  integration-tests:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
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
          
      - name: Install test dependencies
        working-directory: tests/integration
        run: npm install
        
      - name: Start services with docker-compose
        run: |
          docker-compose -f infrastructure/docker-compose.yml up -d
          sleep 30  # Wait for services to start
          
      - name: Run integration tests
        working-directory: tests/integration
        env:
          API_BASE_URL: http://localhost:8080
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379
        run: |
          npm test
          
      - name: Cleanup
        if: always()
        run: |
          docker-compose -f infrastructure/docker-compose.yml down
```

## 🐳 Docker Configurations

### User Service Dockerfile

```dockerfile
# services/user-service/Dockerfile
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

FROM node:18-alpine AS runtime

RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001

WORKDIR /app

COPY --from=builder /app/node_modules ./node_modules
COPY --chown=nodejs:nodejs . .

USER nodejs

EXPOSE 3000

CMD ["node", "src/index.js"]
```

### Order Service Dockerfile

```dockerfile
# services/order-service/Dockerfile
FROM python:3.11-slim AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.11-slim AS runtime

RUN adduser --disabled-password --gecos '' appuser

WORKDIR /app

COPY --from=builder /root/.local /home/appuser/.local
COPY --chown=appuser:appuser . .

USER appuser

ENV PATH=/home/appuser/.local/bin:$PATH

EXPOSE 8000

CMD ["python", "-m", "uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Payment Service Dockerfile

```dockerfile
# services/payment-service/Dockerfile
FROM openjdk:17-jdk-slim AS builder

WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN ./mvnw clean package -DskipTests

FROM openjdk:17-jre-slim AS runtime

RUN adduser --disabled-password --gecos '' appuser

WORKDIR /app

COPY --from=builder /app/target/*.jar app.jar
COPY --chown=appuser:appuser . .

USER appuser

EXPOSE 8080

CMD ["java", "-jar", "app.jar"]
```

### Notification Service Dockerfile

```dockerfile
# services/notification-service/Dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o notification-service main.go

FROM alpine:latest AS runtime

RUN adduser -D -s /bin/sh appuser

WORKDIR /app

COPY --from=builder /app/notification-service .
COPY --chown=appuser:appuser . .

USER appuser

EXPOSE 8080

CMD ["./notification-service"]
```

## 🔧 Infrastructure as Code

### Docker Compose (Local Development)

```yaml
# infrastructure/docker-compose.yml
version: '3.8'

services:
  api-gateway:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - user-service
      - order-service
      - payment-service
      - notification-service

  user-service:
    image: ghcr.io/your-org/user-service:latest
    environment:
      - NODE_ENV=development
      - DB_HOST=postgres
      - REDIS_HOST=redis
    depends_on:
      - postgres
      - redis

  order-service:
    image: ghcr.io/your-org/order-service:latest
    environment:
      - ENVIRONMENT=development
      - DATABASE_URL=postgresql://user:pass@postgres:5432/orders
      - REDIS_URL=redis://redis:6379
    depends_on:
      - postgres
      - redis

  payment-service:
    image: ghcr.io/your-org/payment-service:latest
    environment:
      - SPRING_PROFILES_ACTIVE=development
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/payments
      - SPRING_REDIS_HOST=redis
    depends_on:
      - postgres
      - redis

  notification-service:
    image: ghcr.io/your-org/notification-service:latest
    environment:
      - ENV=development
      - REDIS_ADDR=redis:6379
      - QUEUE_URL=redis://redis:6379
    depends_on:
      - redis

  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: microservices
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

## 📊 Service Discovery & Health Checks

### Health Check Configuration

```yaml
# Each service implements standard health endpoints
# GET /health - Basic health check
# GET /health/ready - Readiness check
# GET /health/live - Liveness check

# Example health check response:
{
  "status": "healthy",
  "timestamp": "2024-01-01T00:00:00Z",
  "uptime": 3600,
  "version": "1.0.0",
  "dependencies": {
    "database": "healthy",
    "cache": "healthy",
    "message_queue": "healthy"
  }
}
```

### Service Registry Pattern

```yaml
# services/user-service/k8s-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
        version: v1
    spec:
      containers:
      - name: user-service
        image: ghcr.io/your-org/user-service:latest
        ports:
        - containerPort: 3000
        livenessProbe:
          httpGet:
            path: /health/live
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
```

## 🔐 Security & Configuration

### Secrets Management

```yaml
# Each service uses environment-specific secrets
# Development: GitHub Secrets
# Production: AWS Secrets Manager / Azure Key Vault

environment:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: url
  - name: JWT_SECRET
    valueFrom:
      secretKeyRef:
        name: jwt-secret
        key: secret
```

### Inter-Service Communication

```yaml
# API Gateway routing configuration
# nginx.conf
upstream user_service {
    server user-service:3000;
}

upstream order_service {
    server order-service:8000;
}

server {
    listen 80;
    
    location /api/users {
        proxy_pass http://user_service;
    }
    
    location /api/orders {
        proxy_pass http://order_service;
    }
}
```

## 📈 Monitoring & Observability

### Metrics Collection

```yaml
# Each service exposes metrics at /metrics endpoint
# Prometheus configuration for scraping
scrape_configs:
  - job_name: 'microservices'
    static_configs:
      - targets: [
          'user-service:3000',
          'order-service:8000',
          'payment-service:8080',
          'notification-service:8080'
        ]
    metrics_path: '/metrics'
```

### Distributed Tracing

```yaml
# OpenTelemetry configuration
# Each service includes tracing middleware
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"
      - "14268:14268"
    environment:
      COLLECTOR_OTLP_ENABLED: true
```

## 🧪 Testing Strategy

### Unit Tests
- Each service has comprehensive unit test coverage (>80%)
- Mock external dependencies
- Test business logic in isolation

### Integration Tests
- Test service-to-service communication
- Test database interactions
- Test message queue interactions

### Contract Tests
- Use Pact for consumer-driven contract testing
- Ensure API compatibility between services

### End-to-End Tests
- Test complete user journeys
- Use tools like Playwright or Cypress
- Run against staging environment

## 📚 Best Practices

### 1. Service Design
- Single Responsibility Principle
- Database per service
- Stateless services
- Event-driven communication

### 2. Deployment Strategy
- Blue-Green deployments for zero downtime
- Canary deployments for risk mitigation
- Feature flags for gradual rollouts

### 3. Monitoring
- Centralized logging with correlation IDs
- Distributed tracing across services
- Circuit breakers for resilience

### 4. Security
- API Gateway for authentication/authorization
- Service-to-service mTLS
- Secrets management
- Network policies

---

💡 **Dica**: Start small with fewer services and evolve your architecture as complexity grows!