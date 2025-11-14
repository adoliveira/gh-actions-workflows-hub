# Simple Node.js Application Example

Este exemplo demonstra um pipeline CI/CD básico para uma aplicação Node.js simples usando os workflows hub.

## 📋 Sobre a Aplicação

- **Framework**: Express.js
- **Testes**: Jest + Supertest
- **Linting**: ESLint + Prettier
- **Deploy**: AWS Lambda + API Gateway
- **Database**: DynamoDB (opcional)

## 🏗️ Arquitetura

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Developer     │───▶│   GitHub        │───▶│   AWS Lambda    │
│   git push      │    │   Actions       │    │   Function      │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │   API Gateway   │
                       └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │   DynamoDB      │
                       │   (opcional)    │
                       └─────────────────┘
```

## 📁 Estrutura do Projeto

```
simple-node-app/
├── .github/
│   └── workflows/
│       └── ci-cd.yml
├── src/
│   ├── app.js
│   ├── routes/
│   │   ├── health.js
│   │   └── users.js
│   └── utils/
│       └── database.js
├── tests/
│   ├── unit/
│   │   └── routes.test.js
│   └── integration/
│       └── app.test.js
├── package.json
├── .eslintrc.js
├── .prettierrc
├── jest.config.js
├── serverless.yml
└── README.md
```

## 🚀 Quick Start

### 1. Configure o Workflow

```yaml
# .github/workflows/ci-cd.yml
name: 'Node.js CI/CD Pipeline'

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  # CI Pipeline
  ci:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/ci-reusable.yml@main
    with:
      node-version: '18'
      package-manager: 'npm'
      run-tests: true
      run-lint: true
      run-security-scan: true
      upload-coverage: true
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  # Deploy to Development
  deploy-dev:
    if: github.ref == 'refs/heads/develop'
    needs: ci
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/aws-deploy.yml@main
    with:
      environment: 'development'
      aws-region: 'us-east-1'
      deploy-target: 'lambda'
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_DEV_ROLE_ARN }}

  # Deploy to Production
  deploy-prod:
    if: github.ref == 'refs/heads/main'
    needs: ci
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/aws-deploy.yml@main
    with:
      environment: 'production'
      aws-region: 'us-east-1'
      deploy-target: 'lambda'
      require-approval: true
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_PROD_ROLE_ARN }}
```

### 2. Aplicação Express.js

**package.json:**
```json
{
  "name": "simple-node-app",
  "version": "1.0.0",
  "description": "Simple Node.js application with CI/CD pipeline",
  "main": "src/app.js",
  "scripts": {
    "start": "node src/app.js",
    "dev": "nodemon src/app.js",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "lint": "eslint src/ tests/",
    "lint:fix": "eslint src/ tests/ --fix",
    "format": "prettier --write src/ tests/",
    "build": "echo 'No build step required'",
    "deploy": "serverless deploy"
  },
  "dependencies": {
    "express": "^4.18.0",
    "cors": "^2.8.5",
    "helmet": "^7.1.0",
    "aws-sdk": "^2.1490.0",
    "serverless-http": "^3.2.0"
  },
  "devDependencies": {
    "jest": "^29.7.0",
    "supertest": "^6.3.0",
    "nodemon": "^3.0.0",
    "eslint": "^8.52.0",
    "prettier": "^3.0.0",
    "serverless": "^3.38.0",
    "serverless-offline": "^13.2.0"
  }
}
```

**src/app.js:**
```javascript
const express = require('express');
const cors = require('cors');
const helmet = require('helmet');
const serverless = require('serverless-http');

const healthRoutes = require('./routes/health');
const userRoutes = require('./routes/users');

const app = express();

// Middleware
app.use(helmet());
app.use(cors());
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Routes
app.use('/health', healthRoutes);
app.use('/api/users', userRoutes);

// Root endpoint
app.get('/', (req, res) => {
  res.json({
    message: 'Simple Node.js Application',
    version: process.env.npm_package_version || '1.0.0',
    environment: process.env.NODE_ENV || 'development',
    timestamp: new Date().toISOString()
  });
});

// Error handling middleware
app.use((err, req, res, next) => {
  console.error('Error:', err);
  res.status(500).json({
    error: 'Internal Server Error',
    message: process.env.NODE_ENV === 'development' ? err.message : 'Something went wrong'
  });
});

// 404 handler
app.use('*', (req, res) => {
  res.status(404).json({
    error: 'Not Found',
    message: `Route ${req.originalUrl} not found`
  });
});

// For serverless deployment
module.exports.handler = serverless(app);

// For local development
if (require.main === module) {
  const PORT = process.env.PORT || 3000;
  app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
  });
}

module.exports = app;
```

**src/routes/health.js:**
```javascript
const express = require('express');
const router = express.Router();

// Health check endpoint
router.get('/', (req, res) => {
  const healthCheck = {
    uptime: process.uptime(),
    message: 'OK',
    timestamp: Date.now(),
    environment: process.env.NODE_ENV || 'development'
  };

  try {
    res.status(200).json(healthCheck);
  } catch (error) {
    healthCheck.message = error.message;
    res.status(503).json(healthCheck);
  }
});

// Detailed health check
router.get('/detailed', async (req, res) => {
  const healthCheck = {
    status: 'healthy',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    environment: process.env.NODE_ENV || 'development',
    version: process.env.npm_package_version || '1.0.0',
    memory: process.memoryUsage(),
    cpu: process.cpuUsage()
  };

  // Add database health check if using DynamoDB
  try {
    if (process.env.DYNAMODB_TABLE) {
      const db = require('../utils/database');
      await db.healthCheck();
      healthCheck.database = { status: 'connected' };
    }
    res.status(200).json(healthCheck);
  } catch (error) {
    healthCheck.status = 'unhealthy';
    healthCheck.database = { status: 'disconnected', error: error.message };
    res.status(503).json(healthCheck);
  }
});

module.exports = router;
```

**src/routes/users.js:**
```javascript
const express = require('express');
const router = express.Router();
const database = require('../utils/database');

// Mock users data (replace with real database)
let users = [
  { id: '1', name: 'John Doe', email: 'john@example.com', createdAt: new Date().toISOString() },
  { id: '2', name: 'Jane Smith', email: 'jane@example.com', createdAt: new Date().toISOString() }
];

// Get all users
router.get('/', async (req, res) => {
  try {
    // If DynamoDB is configured, use it
    if (process.env.DYNAMODB_TABLE) {
      const dbUsers = await database.getAllUsers();
      return res.json({ users: dbUsers });
    }
    
    // Otherwise return mock data
    res.json({ users });
  } catch (error) {
    console.error('Error fetching users:', error);
    res.status(500).json({ error: 'Failed to fetch users' });
  }
});

// Get user by ID
router.get('/:id', async (req, res) => {
  try {
    const { id } = req.params;
    
    if (process.env.DYNAMODB_TABLE) {
      const user = await database.getUser(id);
      if (!user) {
        return res.status(404).json({ error: 'User not found' });
      }
      return res.json({ user });
    }
    
    const user = users.find(u => u.id === id);
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    res.json({ user });
  } catch (error) {
    console.error('Error fetching user:', error);
    res.status(500).json({ error: 'Failed to fetch user' });
  }
});

// Create new user
router.post('/', async (req, res) => {
  try {
    const { name, email } = req.body;
    
    if (!name || !email) {
      return res.status(400).json({ error: 'Name and email are required' });
    }
    
    const newUser = {
      id: Date.now().toString(),
      name,
      email,
      createdAt: new Date().toISOString()
    };
    
    if (process.env.DYNAMODB_TABLE) {
      await database.createUser(newUser);
    } else {
      users.push(newUser);
    }
    
    res.status(201).json({ user: newUser, message: 'User created successfully' });
  } catch (error) {
    console.error('Error creating user:', error);
    res.status(500).json({ error: 'Failed to create user' });
  }
});

// Update user
router.put('/:id', async (req, res) => {
  try {
    const { id } = req.params;
    const { name, email } = req.body;
    
    if (process.env.DYNAMODB_TABLE) {
      const updatedUser = await database.updateUser(id, { name, email });
      if (!updatedUser) {
        return res.status(404).json({ error: 'User not found' });
      }
      return res.json({ user: updatedUser, message: 'User updated successfully' });
    }
    
    const userIndex = users.findIndex(u => u.id === id);
    if (userIndex === -1) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    users[userIndex] = { ...users[userIndex], name, email };
    res.json({ user: users[userIndex], message: 'User updated successfully' });
  } catch (error) {
    console.error('Error updating user:', error);
    res.status(500).json({ error: 'Failed to update user' });
  }
});

// Delete user
router.delete('/:id', async (req, res) => {
  try {
    const { id } = req.params;
    
    if (process.env.DYNAMODB_TABLE) {
      const deleted = await database.deleteUser(id);
      if (!deleted) {
        return res.status(404).json({ error: 'User not found' });
      }
      return res.json({ message: 'User deleted successfully' });
    }
    
    const userIndex = users.findIndex(u => u.id === id);
    if (userIndex === -1) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    users.splice(userIndex, 1);
    res.json({ message: 'User deleted successfully' });
  } catch (error) {
    console.error('Error deleting user:', error);
    res.status(500).json({ error: 'Failed to delete user' });
  }
});

module.exports = router;
```

**src/utils/database.js:**
```javascript
const AWS = require('aws-sdk');

// Configure AWS SDK
const dynamodb = new AWS.DynamoDB.DocumentClient({
  region: process.env.AWS_REGION || 'us-east-1'
});

const TABLE_NAME = process.env.DYNAMODB_TABLE || 'simple-node-app-users';

class Database {
  async healthCheck() {
    try {
      const params = {
        TableName: TABLE_NAME,
        Limit: 1
      };
      
      await dynamodb.scan(params).promise();
      return true;
    } catch (error) {
      throw new Error(`Database health check failed: ${error.message}`);
    }
  }

  async getAllUsers() {
    try {
      const params = {
        TableName: TABLE_NAME
      };
      
      const result = await dynamodb.scan(params).promise();
      return result.Items || [];
    } catch (error) {
      throw new Error(`Failed to get users: ${error.message}`);
    }
  }

  async getUser(id) {
    try {
      const params = {
        TableName: TABLE_NAME,
        Key: { id }
      };
      
      const result = await dynamodb.get(params).promise();
      return result.Item;
    } catch (error) {
      throw new Error(`Failed to get user: ${error.message}`);
    }
  }

  async createUser(user) {
    try {
      const params = {
        TableName: TABLE_NAME,
        Item: user,
        ConditionExpression: 'attribute_not_exists(id)'
      };
      
      await dynamodb.put(params).promise();
      return user;
    } catch (error) {
      throw new Error(`Failed to create user: ${error.message}`);
    }
  }

  async updateUser(id, updates) {
    try {
      const params = {
        TableName: TABLE_NAME,
        Key: { id },
        UpdateExpression: 'SET #name = :name, email = :email, updatedAt = :updatedAt',
        ExpressionAttributeNames: {
          '#name': 'name'
        },
        ExpressionAttributeValues: {
          ':name': updates.name,
          ':email': updates.email,
          ':updatedAt': new Date().toISOString()
        },
        ReturnValues: 'ALL_NEW'
      };
      
      const result = await dynamodb.update(params).promise();
      return result.Attributes;
    } catch (error) {
      throw new Error(`Failed to update user: ${error.message}`);
    }
  }

  async deleteUser(id) {
    try {
      const params = {
        TableName: TABLE_NAME,
        Key: { id },
        ReturnValues: 'ALL_OLD'
      };
      
      const result = await dynamodb.delete(params).promise();
      return result.Attributes;
    } catch (error) {
      throw new Error(`Failed to delete user: ${error.message}`);
    }
  }
}

module.exports = new Database();
```

### 3. Configuração de Testes

**jest.config.js:**
```javascript
module.exports = {
  testEnvironment: 'node',
  collectCoverageFrom: [
    'src/**/*.js',
    '!src/app.js',
    '!**/node_modules/**'
  ],
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html'],
  testMatch: [
    '**/tests/**/*.test.js'
  ],
  setupFilesAfterEnv: ['<rootDir>/tests/setup.js']
};
```

**tests/setup.js:**
```javascript
// Global test setup
process.env.NODE_ENV = 'test';
process.env.DYNAMODB_TABLE = 'test-users-table';
```

**tests/unit/routes.test.js:**
```javascript
const request = require('supertest');
const app = require('../../src/app');

describe('Routes Tests', () => {
  describe('GET /', () => {
    it('should return application info', async () => {
      const response = await request(app)
        .get('/')
        .expect(200);
      
      expect(response.body).toMatchObject({
        message: 'Simple Node.js Application',
        environment: 'test'
      });
      expect(response.body.timestamp).toBeDefined();
    });
  });

  describe('GET /health', () => {
    it('should return health status', async () => {
      const response = await request(app)
        .get('/health')
        .expect(200);
      
      expect(response.body).toMatchObject({
        message: 'OK',
        environment: 'test'
      });
      expect(response.body.uptime).toBeDefined();
      expect(response.body.timestamp).toBeDefined();
    });
  });

  describe('GET /health/detailed', () => {
    it('should return detailed health status', async () => {
      const response = await request(app)
        .get('/health/detailed')
        .expect(200);
      
      expect(response.body.status).toBe('healthy');
      expect(response.body.memory).toBeDefined();
      expect(response.body.cpu).toBeDefined();
    });
  });

  describe('Users API', () => {
    it('should get all users', async () => {
      const response = await request(app)
        .get('/api/users')
        .expect(200);
      
      expect(response.body.users).toBeDefined();
      expect(Array.isArray(response.body.users)).toBe(true);
    });

    it('should create a new user', async () => {
      const newUser = {
        name: 'Test User',
        email: 'test@example.com'
      };

      const response = await request(app)
        .post('/api/users')
        .send(newUser)
        .expect(201);
      
      expect(response.body.user.name).toBe(newUser.name);
      expect(response.body.user.email).toBe(newUser.email);
      expect(response.body.user.id).toBeDefined();
    });

    it('should validate required fields', async () => {
      const response = await request(app)
        .post('/api/users')
        .send({})
        .expect(400);
      
      expect(response.body.error).toBe('Name and email are required');
    });
  });

  describe('404 Handler', () => {
    it('should return 404 for unknown routes', async () => {
      const response = await request(app)
        .get('/unknown-route')
        .expect(404);
      
      expect(response.body.error).toBe('Not Found');
    });
  });
});
```

### 4. Serverless Configuration

**serverless.yml:**
```yaml
service: simple-node-app

provider:
  name: aws
  runtime: nodejs18.x
  region: ${opt:region, 'us-east-1'}
  stage: ${opt:stage, 'dev'}
  environment:
    NODE_ENV: ${self:provider.stage}
    DYNAMODB_TABLE: ${self:service}-${self:provider.stage}-users
  iamRoleStatements:
    - Effect: Allow
      Action:
        - dynamodb:Query
        - dynamodb:Scan
        - dynamodb:GetItem
        - dynamodb:PutItem
        - dynamodb:UpdateItem
        - dynamodb:DeleteItem
      Resource: "arn:aws:dynamodb:${self:provider.region}:*:table/${self:provider.environment.DYNAMODB_TABLE}"

functions:
  app:
    handler: src/app.handler
    events:
      - http:
          path: /
          method: ANY
          cors: true
      - http:
          path: /{proxy+}
          method: ANY
          cors: true

resources:
  Resources:
    UsersTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: ${self:provider.environment.DYNAMODB_TABLE}
        AttributeDefinitions:
          - AttributeName: id
            AttributeType: S
        KeySchema:
          - AttributeName: id
            KeyType: HASH
        BillingMode: PAY_PER_REQUEST
        Tags:
          - Key: Environment
            Value: ${self:provider.stage}
          - Key: Service
            Value: ${self:service}

plugins:
  - serverless-offline

custom:
  serverless-offline:
    httpPort: 3000
```

### 5. Configuração de Linting

**.eslintrc.js:**
```javascript
module.exports = {
  env: {
    node: true,
    es2021: true,
    jest: true
  },
  extends: [
    'eslint:recommended'
  ],
  parserOptions: {
    ecmaVersion: 12
  },
  rules: {
    'no-console': 'warn',
    'no-unused-vars': 'error',
    'semi': ['error', 'always'],
    'quotes': ['error', 'single'],
    'indent': ['error', 2],
    'comma-dangle': ['error', 'never']
  }
};
```

**.prettierrc:**
```json
{
  "semi": true,
  "trailingComma": "none",
  "singleQuote": true,
  "printWidth": 100,
  "tabWidth": 2
}
```

## 🔧 Como Usar

### 1. Setup Local

```bash
# Clone o projeto
git clone <your-repo>
cd simple-node-app

# Instale dependências
npm install

# Execute localmente
npm run dev

# Execute testes
npm test
```

### 2. Deploy Manual

```bash
# Deploy para desenvolvimento
npm run deploy -- --stage dev

# Deploy para produção
npm run deploy -- --stage prod
```

### 3. Configurar Secrets no GitHub

- `AWS_DEV_ROLE_ARN`: Role ARN para desenvolvimento
- `AWS_PROD_ROLE_ARN`: Role ARN para produção
- `SONAR_TOKEN`: Token do SonarCloud (opcional)

## 📊 Monitoring

### CloudWatch Logs
- Logs da aplicação ficam em `/aws/lambda/simple-node-app-{stage}-app`
- Configure retention period apropriado

### Health Checks
- `GET /health` - Health check básico
- `GET /health/detailed` - Health check detalhado com métricas

### API Documentation
```bash
# Exemplos de uso da API
curl https://your-api.execute-api.us-east-1.amazonaws.com/dev/

# Listar usuários
curl https://your-api.execute-api.us-east-1.amazonaws.com/dev/api/users

# Criar usuário
curl -X POST https://your-api.execute-api.us-east-1.amazonaws.com/dev/api/users \
  -H "Content-Type: application/json" \
  -d '{"name": "John Doe", "email": "john@example.com"}'
```

## 📚 Próximos Passos

1. Adicione autenticação com AWS Cognito
2. Implemente rate limiting
3. Configure monitoring com X-Ray
4. Adicione validação de schema com Joi
5. Implemente cache com Redis/ElastiCache

---

💡 **Dica**: Este exemplo serve como base para aplicações Node.js simples. Customize conforme suas necessidades!