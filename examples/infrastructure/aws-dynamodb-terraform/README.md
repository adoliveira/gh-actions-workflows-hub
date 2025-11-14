# AWS DynamoDB with Terraform Example

Este exemplo demonstra como provisionar tabelas DynamoDB na AWS usando Terraform com o workflow hub.

## 📋 Recursos Provisionados

- **DynamoDB Tables**: Tabelas com diferentes padrões de acesso
- **Global Secondary Indexes (GSI)**: Índices para queries alternativas
- **Auto Scaling**: Escalabilidade automática baseada em utilização
- **Backup & Recovery**: Backup automático e point-in-time recovery
- **Encryption**: Encryption at rest com KMS
- **Monitoring**: CloudWatch alarms e métricas

## 🏗️ Arquitetura

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Application   │───▶│  DynamoDB       │───▶│   CloudWatch    │
└─────────────────┘    │  Table          │    │   Monitoring    │
                       │  ┌───────────┐  │    └─────────────────┘
                       │  │    GSI    │  │
                       │  └───────────┘  │    ┌─────────────────┐
                       │  ┌───────────┐  │───▶│   Auto Backup   │
                       │  │Auto Scaling│  │    │   & Recovery    │
                       │  └───────────┘  │    └─────────────────┘
                       └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │   KMS           │
                       │   Encryption    │
                       └─────────────────┘
```

## 📁 Estrutura do Projeto

```
aws-dynamodb-terraform/
├── .github/
│   └── workflows/
│       └── terraform.yml
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── providers.tf
│   └── environments/
│       ├── dev.tfvars
│       ├── staging.tfvars
│       └── prod.tfvars
└── README.md
```

## 🚀 Quick Start

### 1. Configure o Workflow

```yaml
# .github/workflows/terraform.yml
name: 'AWS DynamoDB Infrastructure'

on:
  push:
    branches: [ main, develop ]
    paths: [ 'terraform/**' ]
  pull_request:
    branches: [ main ]
    paths: [ 'terraform/**' ]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        default: 'development'
        type: choice
        options:
          - development
          - staging
          - production

jobs:
  terraform:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/terraform-cicd.yml@main
    with:
      terraform-directory: 'terraform'
      environment: ${{ github.event.inputs.environment || (github.ref == 'refs/heads/main' && 'production' || 'development') }}
      tf-vars-file: environments/${{ github.event.inputs.environment || (github.ref == 'refs/heads/main' && 'prod' || 'dev') }}.tfvars
      auto-approve: ${{ github.ref != 'refs/heads/main' }}
      cloud-provider: 'aws'
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_TERRAFORM_ROLE_ARN }}
```

### 2. Configure Terraform

**terraform/providers.tf:**
```hcl
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket = "your-terraform-state-bucket"
    key    = "dynamodb/terraform.tfstate"
    region = "us-east-1"
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment   = var.environment
      Project       = var.project_name
      ManagedBy     = "Terraform"
      Repository    = "aws-dynamodb-terraform"
    }
  }
}
```

**terraform/variables.tf:**
```hcl
variable "environment" {
  description = "Environment name"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "project_name" {
  description = "Project name"
  type        = string
  default     = "myapp"
}

variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "enable_encryption" {
  description = "Enable server-side encryption"
  type        = bool
  default     = true
}

variable "enable_point_in_time_recovery" {
  description = "Enable point-in-time recovery"
  type        = bool
  default     = false
}

variable "enable_auto_scaling" {
  description = "Enable auto scaling"
  type        = bool
  default     = true
}

variable "backup_retention_period" {
  description = "Backup retention period in days"
  type        = number
  default     = 7
}

variable "table_configs" {
  description = "DynamoDB table configurations"
  type = map(object({
    hash_key       = string
    range_key      = optional(string)
    billing_mode   = string
    read_capacity  = optional(number)
    write_capacity = optional(number)
    attributes = list(object({
      name = string
      type = string
    }))
    global_secondary_indexes = optional(list(object({
      name            = string
      hash_key        = string
      range_key       = optional(string)
      read_capacity   = optional(number)
      write_capacity  = optional(number)
      projection_type = string
      non_key_attributes = optional(list(string))
    })))
    local_secondary_indexes = optional(list(object({
      name               = string
      range_key          = string
      projection_type    = string
      non_key_attributes = optional(list(string))
    })))
    ttl = optional(object({
      attribute_name = string
      enabled        = bool
    }))
    stream_enabled   = optional(bool)
    stream_view_type = optional(string)
  }))
  
  default = {
    users = {
      hash_key     = "user_id"
      range_key    = "created_at"
      billing_mode = "PAY_PER_REQUEST"
      attributes = [
        { name = "user_id", type = "S" },
        { name = "created_at", type = "S" },
        { name = "email", type = "S" },
        { name = "status", type = "S" }
      ]
      global_secondary_indexes = [
        {
          name            = "email-index"
          hash_key        = "email"
          projection_type = "ALL"
        },
        {
          name            = "status-index"
          hash_key        = "status"
          range_key       = "created_at"
          projection_type = "INCLUDE"
          non_key_attributes = ["user_id", "email"]
        }
      ]
      ttl = {
        attribute_name = "expires_at"
        enabled        = false
      }
      stream_enabled   = true
      stream_view_type = "NEW_AND_OLD_IMAGES"
    }
    
    sessions = {
      hash_key     = "session_id"
      billing_mode = "PAY_PER_REQUEST"
      attributes = [
        { name = "session_id", type = "S" },
        { name = "user_id", type = "S" }
      ]
      global_secondary_indexes = [
        {
          name            = "user-sessions-index"
          hash_key        = "user_id"
          projection_type = "ALL"
        }
      ]
      ttl = {
        attribute_name = "expires_at"
        enabled        = true
      }
    }
    
    analytics = {
      hash_key       = "event_date"
      range_key      = "event_timestamp"
      billing_mode   = "PROVISIONED"
      read_capacity  = 5
      write_capacity = 5
      attributes = [
        { name = "event_date", type = "S" },
        { name = "event_timestamp", type = "S" },
        { name = "event_type", type = "S" },
        { name = "user_id", type = "S" }
      ]
      global_secondary_indexes = [
        {
          name            = "event-type-index"
          hash_key        = "event_type"
          range_key       = "event_timestamp"
          read_capacity   = 5
          write_capacity  = 5
          projection_type = "INCLUDE"
          non_key_attributes = ["user_id"]
        }
      ]
      local_secondary_indexes = [
        {
          name               = "user-events-index"
          range_key          = "user_id"
          projection_type    = "ALL"
        }
      ]
      stream_enabled   = false
    }
  }
}

variable "auto_scaling_config" {
  description = "Auto scaling configuration"
  type = object({
    target_tracking_scaling_policy = object({
      target_value = number
      scale_in_cooldown = number
      scale_out_cooldown = number
    })
    min_capacity = number
    max_capacity = number
  })
  default = {
    target_tracking_scaling_policy = {
      target_value       = 70
      scale_in_cooldown  = 60
      scale_out_cooldown = 60
    }
    min_capacity = 5
    max_capacity = 100
  }
}
```

**terraform/main.tf:**
```hcl
locals {
  table_name_prefix = "${var.project_name}-${var.environment}"
  
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}

# KMS Key for DynamoDB Encryption
resource "aws_kms_key" "dynamodb" {
  count = var.enable_encryption ? 1 : 0
  
  description             = "KMS key for DynamoDB encryption - ${local.table_name_prefix}"
  deletion_window_in_days = var.environment == "prod" ? 30 : 10

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action = "kms:*"
        Resource = "*"
      },
      {
        Effect = "Allow"
        Principal = {
          Service = "dynamodb.amazonaws.com"
        }
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey"
        ]
        Resource = "*"
      }
    ]
  })

  tags = merge(local.common_tags, {
    Name = "${local.table_name_prefix}-dynamodb-key"
  })
}

resource "aws_kms_alias" "dynamodb" {
  count = var.enable_encryption ? 1 : 0
  
  name          = "alias/${local.table_name_prefix}-dynamodb"
  target_key_id = aws_kms_key.dynamodb[0].key_id
}

# DynamoDB Tables
resource "aws_dynamodb_table" "tables" {
  for_each = var.table_configs
  
  name           = "${local.table_name_prefix}-${each.key}"
  billing_mode   = each.value.billing_mode
  hash_key       = each.value.hash_key
  range_key      = each.value.range_key
  read_capacity  = each.value.billing_mode == "PROVISIONED" ? each.value.read_capacity : null
  write_capacity = each.value.billing_mode == "PROVISIONED" ? each.value.write_capacity : null

  # Enable streams if configured
  stream_enabled   = each.value.stream_enabled
  stream_view_type = each.value.stream_enabled ? each.value.stream_view_type : null

  # Attributes
  dynamic "attribute" {
    for_each = each.value.attributes
    content {
      name = attribute.value.name
      type = attribute.value.type
    }
  }

  # Global Secondary Indexes
  dynamic "global_secondary_index" {
    for_each = each.value.global_secondary_indexes != null ? each.value.global_secondary_indexes : []
    content {
      name            = global_secondary_index.value.name
      hash_key        = global_secondary_index.value.hash_key
      range_key       = global_secondary_index.value.range_key
      read_capacity   = each.value.billing_mode == "PROVISIONED" ? global_secondary_index.value.read_capacity : null
      write_capacity  = each.value.billing_mode == "PROVISIONED" ? global_secondary_index.value.write_capacity : null
      projection_type = global_secondary_index.value.projection_type
      non_key_attributes = global_secondary_index.value.non_key_attributes
    }
  }

  # Local Secondary Indexes
  dynamic "local_secondary_index" {
    for_each = each.value.local_secondary_indexes != null ? each.value.local_secondary_indexes : []
    content {
      name               = local_secondary_index.value.name
      range_key          = local_secondary_index.value.range_key
      projection_type    = local_secondary_index.value.projection_type
      non_key_attributes = local_secondary_index.value.non_key_attributes
    }
  }

  # TTL Configuration
  dynamic "ttl" {
    for_each = each.value.ttl != null ? [each.value.ttl] : []
    content {
      attribute_name = ttl.value.attribute_name
      enabled        = ttl.value.enabled
    }
  }

  # Encryption
  dynamic "server_side_encryption" {
    for_each = var.enable_encryption ? [1] : []
    content {
      enabled     = true
      kms_key_id  = aws_kms_key.dynamodb[0].arn
    }
  }

  # Point-in-time recovery
  point_in_time_recovery {
    enabled = var.enable_point_in_time_recovery
  }

  tags = merge(local.common_tags, {
    Name      = "${local.table_name_prefix}-${each.key}"
    TableType = each.key
  })
}

# Auto Scaling for Read Capacity
resource "aws_appautoscaling_target" "read_target" {
  for_each = {
    for k, v in var.table_configs : k => v
    if var.enable_auto_scaling && v.billing_mode == "PROVISIONED"
  }
  
  max_capacity       = var.auto_scaling_config.max_capacity
  min_capacity       = var.auto_scaling_config.min_capacity
  resource_id        = "table/${aws_dynamodb_table.tables[each.key].name}"
  scalable_dimension = "dynamodb:table:ReadCapacityUnits"
  service_namespace  = "dynamodb"

  tags = local.common_tags
}

resource "aws_appautoscaling_policy" "read_policy" {
  for_each = {
    for k, v in var.table_configs : k => v
    if var.enable_auto_scaling && v.billing_mode == "PROVISIONED"
  }
  
  name               = "${local.table_name_prefix}-${each.key}-read-scaling-policy"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.read_target[each.key].resource_id
  scalable_dimension = aws_appautoscaling_target.read_target[each.key].scalable_dimension
  service_namespace  = aws_appautoscaling_target.read_target[each.key].service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "DynamoDBReadCapacityUtilization"
    }
    target_value       = var.auto_scaling_config.target_tracking_scaling_policy.target_value
    scale_in_cooldown  = var.auto_scaling_config.target_tracking_scaling_policy.scale_in_cooldown
    scale_out_cooldown = var.auto_scaling_config.target_tracking_scaling_policy.scale_out_cooldown
  }
}

# Auto Scaling for Write Capacity
resource "aws_appautoscaling_target" "write_target" {
  for_each = {
    for k, v in var.table_configs : k => v
    if var.enable_auto_scaling && v.billing_mode == "PROVISIONED"
  }
  
  max_capacity       = var.auto_scaling_config.max_capacity
  min_capacity       = var.auto_scaling_config.min_capacity
  resource_id        = "table/${aws_dynamodb_table.tables[each.key].name}"
  scalable_dimension = "dynamodb:table:WriteCapacityUnits"
  service_namespace  = "dynamodb"

  tags = local.common_tags
}

resource "aws_appautoscaling_policy" "write_policy" {
  for_each = {
    for k, v in var.table_configs : k => v
    if var.enable_auto_scaling && v.billing_mode == "PROVISIONED"
  }
  
  name               = "${local.table_name_prefix}-${each.key}-write-scaling-policy"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.write_target[each.key].resource_id
  scalable_dimension = aws_appautoscaling_target.write_target[each.key].scalable_dimension
  service_namespace  = aws_appautoscaling_target.write_target[each.key].service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "DynamoDBWriteCapacityUtilization"
    }
    target_value       = var.auto_scaling_config.target_tracking_scaling_policy.target_value
    scale_in_cooldown  = var.auto_scaling_config.target_tracking_scaling_policy.scale_in_cooldown
    scale_out_cooldown = var.auto_scaling_config.target_tracking_scaling_policy.scale_out_cooldown
  }
}

# Backup Configuration
resource "aws_dynamodb_backup" "backup" {
  for_each = var.table_configs
  
  name     = "${local.table_name_prefix}-${each.key}-backup-${formatdate("YYYY-MM-DD-hhmmss", timestamp())}"
  table_arn = aws_dynamodb_table.tables[each.key].arn

  tags = merge(local.common_tags, {
    BackupType = "manual"
    Table      = each.key
  })
}

# CloudWatch Alarms
resource "aws_cloudwatch_metric_alarm" "read_throttled_requests" {
  for_each = var.table_configs
  
  alarm_name          = "${local.table_name_prefix}-${each.key}-read-throttled-requests"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "ReadThrottledRequests"
  namespace           = "AWS/DynamoDB"
  period              = "300"
  statistic           = "Sum"
  threshold           = "0"
  alarm_description   = "This metric monitors read throttled requests for ${each.key} table"

  dimensions = {
    TableName = aws_dynamodb_table.tables[each.key].name
  }

  tags = local.common_tags
}

resource "aws_cloudwatch_metric_alarm" "write_throttled_requests" {
  for_each = var.table_configs
  
  alarm_name          = "${local.table_name_prefix}-${each.key}-write-throttled-requests"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "WriteThrottledRequests"
  namespace           = "AWS/DynamoDB"
  period              = "300"
  statistic           = "Sum"
  threshold           = "0"
  alarm_description   = "This metric monitors write throttled requests for ${each.key} table"

  dimensions = {
    TableName = aws_dynamodb_table.tables[each.key].name
  }

  tags = local.common_tags
}

resource "aws_cloudwatch_metric_alarm" "consumed_read_capacity" {
  for_each = {
    for k, v in var.table_configs : k => v
    if v.billing_mode == "PROVISIONED"
  }
  
  alarm_name          = "${local.table_name_prefix}-${each.key}-high-read-capacity"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "ConsumedReadCapacityUnits"
  namespace           = "AWS/DynamoDB"
  period              = "300"
  statistic           = "Average"
  threshold           = each.value.read_capacity * 0.8
  alarm_description   = "This metric monitors consumed read capacity for ${each.key} table"

  dimensions = {
    TableName = aws_dynamodb_table.tables[each.key].name
  }

  tags = local.common_tags
}

# Data source
data "aws_caller_identity" "current" {}
```

**terraform/outputs.tf:**
```hcl
output "table_names" {
  description = "Names of all DynamoDB tables"
  value = {
    for key, table in aws_dynamodb_table.tables : key => table.name
  }
}

output "table_arns" {
  description = "ARNs of all DynamoDB tables"
  value = {
    for key, table in aws_dynamodb_table.tables : key => table.arn
  }
}

output "table_stream_arns" {
  description = "Stream ARNs for DynamoDB tables"
  value = {
    for key, table in aws_dynamodb_table.tables : key => table.stream_arn
    if table.stream_enabled
  }
}

output "users_table_name" {
  description = "Name of the users table"
  value       = aws_dynamodb_table.tables["users"].name
}

output "sessions_table_name" {
  description = "Name of the sessions table"
  value       = aws_dynamodb_table.tables["sessions"].name
}

output "analytics_table_name" {
  description = "Name of the analytics table"
  value       = aws_dynamodb_table.tables["analytics"].name
}

output "kms_key_id" {
  description = "ID of the KMS key for DynamoDB encryption"
  value       = var.enable_encryption ? aws_kms_key.dynamodb[0].id : null
}

output "backup_arns" {
  description = "ARNs of DynamoDB backups"
  value = {
    for key, backup in aws_dynamodb_backup.backup : key => backup.arn
  }
}

output "cloudwatch_alarm_arns" {
  description = "ARNs of CloudWatch alarms"
  value = {
    read_throttled = {
      for key, alarm in aws_cloudwatch_metric_alarm.read_throttled_requests : key => alarm.arn
    }
    write_throttled = {
      for key, alarm in aws_cloudwatch_metric_alarm.write_throttled_requests : key => alarm.arn
    }
    high_read_capacity = {
      for key, alarm in aws_cloudwatch_metric_alarm.consumed_read_capacity : key => alarm.arn
    }
  }
}

output "auto_scaling_policies" {
  description = "Auto scaling policy ARNs"
  value = {
    read_policies = {
      for key, policy in aws_appautoscaling_policy.read_policy : key => policy.arn
    }
    write_policies = {
      for key, policy in aws_appautoscaling_policy.write_policy : key => policy.arn
    }
  }
}
```

## ⚙️ Arquivos de Configuração por Ambiente

**terraform/environments/dev.tfvars:**
```hcl
environment                    = "dev"
project_name                  = "myapp"
aws_region                    = "us-east-1"
enable_encryption             = false
enable_point_in_time_recovery = false
enable_auto_scaling           = false
backup_retention_period       = 3

# Simplified table configs for development
table_configs = {
  users = {
    hash_key     = "user_id"
    billing_mode = "PAY_PER_REQUEST"
    attributes = [
      { name = "user_id", type = "S" },
      { name = "email", type = "S" }
    ]
    global_secondary_indexes = [
      {
        name            = "email-index"
        hash_key        = "email"
        projection_type = "ALL"
      }
    ]
    stream_enabled = false
  }
  
  sessions = {
    hash_key     = "session_id"
    billing_mode = "PAY_PER_REQUEST"
    attributes = [
      { name = "session_id", type = "S" }
    ]
    ttl = {
      attribute_name = "expires_at"
      enabled        = true
    }
  }
}

auto_scaling_config = {
  target_tracking_scaling_policy = {
    target_value       = 70
    scale_in_cooldown  = 60
    scale_out_cooldown = 60
  }
  min_capacity = 1
  max_capacity = 10
}
```

**terraform/environments/staging.tfvars:**
```hcl
environment                    = "staging"
project_name                  = "myapp"
aws_region                    = "us-east-1"
enable_encryption             = true
enable_point_in_time_recovery = true
enable_auto_scaling           = true
backup_retention_period       = 7

# Use default table_configs with some modifications
auto_scaling_config = {
  target_tracking_scaling_policy = {
    target_value       = 70
    scale_in_cooldown  = 60
    scale_out_cooldown = 60
  }
  min_capacity = 5
  max_capacity = 50
}
```

**terraform/environments/prod.tfvars:**
```hcl
environment                    = "prod"
project_name                  = "myapp"
aws_region                    = "us-east-1"
enable_encryption             = true
enable_point_in_time_recovery = true
enable_auto_scaling           = true
backup_retention_period       = 30

# Use default table_configs (full configuration)

auto_scaling_config = {
  target_tracking_scaling_policy = {
    target_value       = 70
    scale_in_cooldown  = 300  # 5 minutes
    scale_out_cooldown = 60   # 1 minute
  }
  min_capacity = 10
  max_capacity = 200
}
```

## 🔧 Como Usar

### 1. Setup Inicial

```bash
# Clone e configure o projeto
git clone <your-repo>
cd aws-dynamodb-terraform

# Configure secrets no GitHub
# AWS_TERRAFORM_ROLE_ARN
```

### 2. Code Examples

**Python (boto3):**
```python
import boto3
from boto3.dynamodb.conditions import Key, Attr

# Initialize DynamoDB
dynamodb = boto3.resource('dynamodb')
users_table = dynamodb.Table('myapp-prod-users')
sessions_table = dynamodb.Table('myapp-prod-sessions')

# Create user
def create_user(user_id, email, name):
    response = users_table.put_item(
        Item={
            'user_id': user_id,
            'created_at': '2024-01-01T00:00:00Z',
            'email': email,
            'name': name,
            'status': 'active'
        }
    )
    return response

# Query by email using GSI
def get_user_by_email(email):
    response = users_table.query(
        IndexName='email-index',
        KeyConditionExpression=Key('email').eq(email)
    )
    return response['Items']

# Query users by status
def get_users_by_status(status, limit=10):
    response = users_table.query(
        IndexName='status-index',
        KeyConditionExpression=Key('status').eq(status),
        Limit=limit
    )
    return response['Items']

# Batch write example
def batch_create_sessions(sessions):
    with sessions_table.batch_writer() as batch:
        for session in sessions:
            batch.put_item(Item=session)

# Usage examples
create_user('123', 'user@example.com', 'John Doe')
users = get_user_by_email('user@example.com')
active_users = get_users_by_status('active')
```

**Node.js (AWS SDK v3):**
```javascript
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import {
  DynamoDBDocumentClient,
  PutCommand,
  QueryCommand,
  BatchWriteCommand
} from "@aws-sdk/lib-dynamodb";

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

// Create user
async function createUser(userId, email, name) {
  const command = new PutCommand({
    TableName: "myapp-prod-users",
    Item: {
      user_id: userId,
      created_at: new Date().toISOString(),
      email: email,
      name: name,
      status: "active"
    }
  });
  
  return await docClient.send(command);
}

// Query by email
async function getUserByEmail(email) {
  const command = new QueryCommand({
    TableName: "myapp-prod-users",
    IndexName: "email-index",
    KeyConditionExpression: "email = :email",
    ExpressionAttributeValues: {
      ":email": email
    }
  });
  
  const response = await docClient.send(command);
  return response.Items;
}

// Usage
await createUser("456", "jane@example.com", "Jane Doe");
const users = await getUserByEmail("jane@example.com");
```

## 📊 Monitoramento

### CloudWatch Metrics
- **ReadThrottledRequests**: Requests rejeitados por falta de read capacity
- **WriteThrottledRequests**: Requests rejeitados por falta de write capacity
- **ConsumedReadCapacityUnits**: Utilização de read capacity
- **ConsumedWriteCapacityUnits**: Utilização de write capacity

### Custom Dashboards
```json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/DynamoDB", "ConsumedReadCapacityUnits", "TableName", "myapp-prod-users"],
          [".", "ConsumedWriteCapacityUnits", ".", "."]
        ],
        "period": 300,
        "stat": "Sum",
        "region": "us-east-1",
        "title": "DynamoDB Capacity Utilization"
      }
    }
  ]
}
```

## 🔒 Segurança

### Encryption
- **At Rest**: KMS encryption para dados armazenados
- **In Transit**: SSL/TLS para comunicação

### IAM Policies
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:Query",
        "dynamodb:Scan",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem"
      ],
      "Resource": [
        "arn:aws:dynamodb:*:*:table/myapp-*",
        "arn:aws:dynamodb:*:*:table/myapp-*/index/*"
      ]
    }
  ]
}
```

## 🎯 Best Practices

### 1. Partition Key Design
- Use high cardinality attributes
- Distribute traffic evenly
- Avoid hot partitions

### 2. Query Patterns
- Design tables based on access patterns
- Use GSIs for alternative queries
- Minimize scan operations

### 3. Cost Optimization
- Use PAY_PER_REQUEST for unpredictable workloads
- Use PROVISIONED for consistent traffic
- Monitor capacity utilization

## 📚 Próximos Passos

1. Implemente Lambda triggers para DynamoDB Streams
2. Configure Global Tables para multi-region
3. Setup de DAX para caching
4. Implemente backup strategies avançadas
5. Configure VPC endpoints para acesso privado

---

💡 **Dica**: Design suas tabelas baseado nos padrões de acesso da sua aplicação, não na estrutura relacional tradicional!