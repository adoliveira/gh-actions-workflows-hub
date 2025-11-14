# Serverless AWS Application Example

Este exemplo demonstra uma aplicação serverless completa na AWS usando Lambda, API Gateway, DynamoDB e S3.

## 📋 Sobre a Aplicação

- **Frontend**: React SPA hospedado no S3 + CloudFront
- **API**: AWS Lambda Functions + API Gateway
- **Database**: DynamoDB com DynamoDB Streams
- **Storage**: S3 para arquivos/uploads
- **Infrastructure**: AWS CDK + Terraform

## 🏗️ Arquitetura

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   CloudFront    │───▶│   S3 Bucket     │    │   API Gateway   │
│   (CDN)         │    │   (Frontend)    │    └─────────────────┘
└─────────────────┘    └─────────────────┘             │
                                                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Lambda        │◀───│   DynamoDB      │◀───│   Lambda        │
│   (Processor)   │    │   (Database)    │    │   (API)         │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │
         ▼                       ▼
┌─────────────────┐    ┌─────────────────┐
│   SQS/SNS       │    │   S3 Bucket     │
│   (Messaging)   │    │   (Files)       │
└─────────────────┘    └─────────────────┘
```

## 🚀 Workflow Configuration

```yaml
# .github/workflows/serverless.yml
name: 'Serverless Application CI/CD'

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  # Test Lambda Functions
  test-api:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/ci-reusable.yml@main
    with:
      node-version: '18'
      working-directory: 'lambda'
      run-tests: true
      run-lint: true

  # Build Frontend
  build-frontend:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/ci-reusable.yml@main
    with:
      node-version: '18'
      working-directory: 'frontend'
      build-command: 'npm run build'
      artifact-name: 'frontend-dist'
      artifact-path: 'frontend/dist'

  # Deploy Infrastructure
  deploy-infrastructure:
    needs: [test-api]
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/terraform-cicd.yml@main
    with:
      terraform-directory: 'infrastructure/terraform'
      environment: ${{ github.ref == 'refs/heads/main' && 'production' || 'development' }}
      cloud-provider: 'aws'
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_TERRAFORM_ROLE_ARN }}

  # Deploy Functions with CDK
  deploy-functions:
    needs: [test-api, deploy-infrastructure]
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/aws-cdk-deploy.yml@main
    with:
      cdk-directory: 'infrastructure/cdk'
      stack-name: 'serverless-api-stack'
      environment: ${{ github.ref == 'refs/heads/main' && 'production' || 'development' }}
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_CDK_ROLE_ARN }}

  # Deploy Frontend to S3
  deploy-frontend:
    needs: [build-frontend, deploy-infrastructure]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Download frontend build
        uses: actions/download-artifact@v3
        with:
          name: frontend-dist
          path: ./dist
      - name: Deploy to S3
        uses: adoliveira/gh-actions-workflows-hub/.github/actions/aws-deploy@main
        with:
          aws-role-arn: ${{ secrets.AWS_ROLE_ARN }}
          deploy-target: 's3'
          s3-bucket: ${{ vars.S3_BUCKET_NAME }}
          s3-path: './dist'
          cloudfront-distribution-id: ${{ vars.CLOUDFRONT_DISTRIBUTION_ID }}
```

## 📁 Estrutura do Projeto

```
serverless-aws/
├── .github/
│   └── workflows/
│       └── serverless.yml
├── lambda/
│   ├── handlers/
│   │   ├── users.js
│   │   ├── orders.js
│   │   └── files.js
│   ├── layers/
│   │   └── common/
│   ├── tests/
│   └── package.json
├── frontend/
│   ├── src/
│   ├── public/
│   ├── package.json
│   └── vite.config.js
├── infrastructure/
│   ├── terraform/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── cdk/
│       ├── lib/
│       ├── bin/
│       └── package.json
└── README.md
```

## ⚡ Lambda Functions

### API Handler Example
```javascript
// lambda/handlers/users.js
const AWS = require('aws-sdk');
const dynamodb = new AWS.DynamoDB.DocumentClient();

exports.handler = async (event) => {
  const { httpMethod, pathParameters, body } = event;
  
  try {
    switch (httpMethod) {
      case 'GET':
        return await getUsers(pathParameters?.id);
      case 'POST':
        return await createUser(JSON.parse(body));
      case 'PUT':
        return await updateUser(pathParameters.id, JSON.parse(body));
      case 'DELETE':
        return await deleteUser(pathParameters.id);
      default:
        return {
          statusCode: 405,
          body: JSON.stringify({ error: 'Method not allowed' })
        };
    }
  } catch (error) {
    return {
      statusCode: 500,
      body: JSON.stringify({ error: error.message })
    };
  }
};

async function getUsers(id) {
  if (id) {
    const result = await dynamodb.get({
      TableName: process.env.USERS_TABLE,
      Key: { id }
    }).promise();
    
    return {
      statusCode: result.Item ? 200 : 404,
      body: JSON.stringify(result.Item || { error: 'User not found' })
    };
  }
  
  const result = await dynamodb.scan({
    TableName: process.env.USERS_TABLE
  }).promise();
  
  return {
    statusCode: 200,
    body: JSON.stringify({ users: result.Items })
  };
}
```

## 🏗️ Infrastructure as Code

### Terraform Configuration
```hcl
# infrastructure/terraform/main.tf
resource "aws_s3_bucket" "frontend" {
  bucket = "${var.project_name}-${var.environment}-frontend"
  
  tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}

resource "aws_s3_bucket_website_configuration" "frontend" {
  bucket = aws_s3_bucket.frontend.id
  
  index_document {
    suffix = "index.html"
  }
  
  error_document {
    key = "error.html"
  }
}

resource "aws_cloudfront_distribution" "frontend" {
  origin {
    domain_name = aws_s3_bucket.frontend.bucket_regional_domain_name
    origin_id   = "S3-${aws_s3_bucket.frontend.id}"
    
    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.frontend.cloudfront_access_identity_path
    }
  }
  
  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"
  
  default_cache_behavior {
    allowed_methods        = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "S3-${aws_s3_bucket.frontend.id}"
    compress               = true
    viewer_protocol_policy = "redirect-to-https"
    
    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }
  }
  
  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }
  
  viewer_certificate {
    cloudfront_default_certificate = true
  }
  
  tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}
```

### CDK Stack
```typescript
// infrastructure/cdk/lib/api-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as apigateway from 'aws-cdk-lib/aws-apigateway';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import { Construct } from 'constructs';

export class ApiStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);
    
    // DynamoDB Table
    const usersTable = new dynamodb.Table(this, 'UsersTable', {
      tableName: `users-${this.node.tryGetContext('environment')}`,
      partitionKey: { name: 'id', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      removalPolicy: cdk.RemovalPolicy.DESTROY
    });
    
    // Lambda Layer
    const commonLayer = new lambda.LayerVersion(this, 'CommonLayer', {
      code: lambda.Code.fromAsset('lambda/layers/common'),
      compatibleRuntimes: [lambda.Runtime.NODEJS_18_X],
      description: 'Common utilities for Lambda functions'
    });
    
    // Lambda Function
    const usersFunction = new lambda.Function(this, 'UsersFunction', {
      runtime: lambda.Runtime.NODEJS_18_X,
      handler: 'handlers/users.handler',
      code: lambda.Code.fromAsset('lambda'),
      layers: [commonLayer],
      environment: {
        USERS_TABLE: usersTable.tableName,
        NODE_ENV: this.node.tryGetContext('environment')
      },
      timeout: cdk.Duration.seconds(30)
    });
    
    // Grant permissions
    usersTable.grantReadWriteData(usersFunction);
    
    // API Gateway
    const api = new apigateway.RestApi(this, 'ServerlessApi', {
      restApiName: `serverless-api-${this.node.tryGetContext('environment')}`,
      description: 'Serverless API Gateway'
    });
    
    const usersResource = api.root.addResource('users');
    const userResource = usersResource.addResource('{id}');
    
    // Integrate Lambda with API Gateway
    const usersIntegration = new apigateway.LambdaIntegration(usersFunction);
    
    usersResource.addMethod('GET', usersIntegration);
    usersResource.addMethod('POST', usersIntegration);
    userResource.addMethod('GET', usersIntegration);
    userResource.addMethod('PUT', usersIntegration);
    userResource.addMethod('DELETE', usersIntegration);
    
    // Outputs
    new cdk.CfnOutput(this, 'ApiUrl', {
      value: api.url,
      description: 'API Gateway URL'
    });
    
    new cdk.CfnOutput(this, 'UsersTableName', {
      value: usersTable.tableName,
      description: 'DynamoDB Users Table Name'
    });
  }
}
```

## ⚛️ Frontend Application

### React Component Example
```jsx
// frontend/src/components/UserManager.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_BASE_URL = import.meta.env.VITE_API_URL || 'https://api.example.com';

export function UserManager() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetchUsers();
  }, []);

  const fetchUsers = async () => {
    try {
      setLoading(true);
      const response = await axios.get(`${API_BASE_URL}/users`);
      setUsers(response.data.users);
    } catch (err) {
      setError('Failed to fetch users');
      console.error('Error:', err);
    } finally {
      setLoading(false);
    }
  };

  const createUser = async (userData) => {
    try {
      const response = await axios.post(`${API_BASE_URL}/users`, userData);
      setUsers([...users, response.data.user]);
    } catch (err) {
      setError('Failed to create user');
      console.error('Error:', err);
    }
  };

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <div>
      <h2>User Management</h2>
      <div>
        {users.map(user => (
          <div key={user.id}>
            <h3>{user.name}</h3>
            <p>{user.email}</p>
          </div>
        ))}
      </div>
    </div>
  );
}
```

## 🧪 Testing

### Lambda Function Tests
```javascript
// lambda/tests/users.test.js
const { handler } = require('../handlers/users');

// Mock AWS SDK
jest.mock('aws-sdk', () => ({
  DynamoDB: {
    DocumentClient: jest.fn(() => ({
      get: jest.fn().mockReturnValue({
        promise: jest.fn().mockResolvedValue({
          Item: { id: '1', name: 'Test User', email: 'test@example.com' }
        })
      }),
      scan: jest.fn().mockReturnValue({
        promise: jest.fn().mockResolvedValue({
          Items: [{ id: '1', name: 'Test User', email: 'test@example.com' }]
        })
      })
    }))
  }
}));

describe('Users Handler', () => {
  beforeEach(() => {
    process.env.USERS_TABLE = 'test-users-table';
  });

  it('should get all users', async () => {
    const event = {
      httpMethod: 'GET',
      pathParameters: null
    };

    const result = await handler(event);
    
    expect(result.statusCode).toBe(200);
    expect(JSON.parse(result.body).users).toHaveLength(1);
  });

  it('should get user by id', async () => {
    const event = {
      httpMethod: 'GET',
      pathParameters: { id: '1' }
    };

    const result = await handler(event);
    
    expect(result.statusCode).toBe(200);
    expect(JSON.parse(result.body).id).toBe('1');
  });
});
```

## 📊 Monitoring & Logging

### CloudWatch Configuration
```yaml
# Add to CDK stack
const logGroup = new logs.LogGroup(this, 'ApiLogGroup', {
  logGroupName: `/aws/lambda/serverless-api-${environment}`,
  retention: logs.RetentionDays.ONE_WEEK,
  removalPolicy: cdk.RemovalPolicy.DESTROY
});

// X-Ray tracing
usersFunction.addEnvironment('_X_AMZN_TRACE_ID', 'Root=1-5e6b2131-1234567890abcdef');

// CloudWatch Alarms
new cloudwatch.Alarm(this, 'FunctionErrorAlarm', {
  metric: usersFunction.metricErrors(),
  threshold: 5,
  evaluationPeriods: 2,
  treatMissingData: cloudwatch.TreatMissingData.NOT_BREACHING
});
```

## 🚀 Deployment Strategies

### Environment-specific Configurations
```json
// infrastructure/cdk/cdk.json
{
  "context": {
    "development": {
      "environment": "dev",
      "region": "us-east-1",
      "retention": 7
    },
    "production": {
      "environment": "prod", 
      "region": "us-east-1",
      "retention": 30
    }
  }
}
```

## 📚 Best Practices

### 1. Lambda Optimization
- Keep functions small and focused
- Use Lambda layers for shared code
- Optimize cold start times
- Implement proper error handling

### 2. Security
- Use IAM roles with least privilege
- Enable API Gateway throttling
- Implement authentication/authorization
- Use AWS Secrets Manager for sensitive data

### 3. Cost Optimization
- Use DynamoDB on-demand billing
- Implement S3 lifecycle policies
- Monitor and optimize Lambda duration
- Use CloudFront for static content

### 4. Monitoring
- Enable X-Ray tracing
- Set up CloudWatch alarms
- Implement structured logging
- Monitor business metrics

---

💡 **Dica**: Start with simple functions and gradually add complexity. Serverless scales automatically but costs can grow quickly without proper monitoring!