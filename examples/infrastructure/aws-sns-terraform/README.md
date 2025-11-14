# AWS SNS with Terraform Example

Este exemplo demonstra como provisionar tópicos SNS na AWS usando Terraform com o workflow hub.

## 📋 Recursos Provisionados

- **SNS Topics**: Tópicos para diferentes tipos de notificação
- **SNS Subscriptions**: Assinaturas Email, SMS, SQS, Lambda
- **Access Control**: Políticas de acesso e permissões
- **Cross-Account Access**: Políticas para acesso entre contas
- **CloudWatch Monitoring**: Métricas e alarmes

## 🏗️ Arquitetura

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Application   │───▶│   SNS Topic     │───▶│     Email       │
└─────────────────┘    │   (Alerts)      │    │  Subscription   │
                       └─────────────────┘    └─────────────────┘
                                │
                                ├──────────────▶ ┌─────────────────┐
                                │                │     SMS         │
                                │                │  Subscription   │
                                │                └─────────────────┘
                                │
                                ├──────────────▶ ┌─────────────────┐
                                │                │     SQS         │
                                │                │  Subscription   │
                                │                └─────────────────┘
                                │
                                └──────────────▶ ┌─────────────────┐
                                                 │    Lambda       │
                                                 │  Subscription   │
                                                 └─────────────────┘
```

## 📁 Estrutura do Projeto

```
aws-sns-terraform/
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
name: 'AWS SNS Infrastructure'

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
    key    = "sns/terraform.tfstate"
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
      Repository    = "aws-sns-terraform"
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

variable "email_endpoints" {
  description = "List of email endpoints for notifications"
  type        = list(string)
  default     = ["devops@yourcompany.com"]
}

variable "sms_endpoints" {
  description = "List of SMS endpoints for critical alerts"
  type        = list(string)
  default     = []
}

variable "cross_account_principals" {
  description = "List of AWS account IDs that can publish to topics"
  type        = list(string)
  default     = []
}

variable "lambda_function_arns" {
  description = "List of Lambda function ARNs for subscriptions"
  type        = list(string)
  default     = []
}

variable "sqs_queue_arns" {
  description = "List of SQS queue ARNs for subscriptions"
  type        = list(string)
  default     = []
}

variable "delivery_retry_policy" {
  description = "Delivery retry policy configuration"
  type = object({
    num_retries         = number
    num_max_delay_retries = number
    min_delay_target    = number
    max_delay_target    = number
    num_min_delay_retries = number
  })
  default = {
    num_retries         = 3
    num_max_delay_retries = 2
    min_delay_target    = 20
    max_delay_target    = 20
    num_min_delay_retries = 0
  }
}
```

**terraform/main.tf:**
```hcl
locals {
  topic_name_prefix = "${var.project_name}-${var.environment}"
  
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
  }
  
  # Define different topic types with their configurations
  topic_configs = {
    alerts = {
      name            = "${local.topic_name_prefix}-alerts"
      display_name    = "Application Alerts - ${title(var.environment)}"
      delivery_policy = jsonencode(var.delivery_retry_policy)
    }
    notifications = {
      name            = "${local.topic_name_prefix}-notifications"
      display_name    = "User Notifications - ${title(var.environment)}"
      delivery_policy = jsonencode(var.delivery_retry_policy)
    }
    system_events = {
      name            = "${local.topic_name_prefix}-system-events"
      display_name    = "System Events - ${title(var.environment)}"
      delivery_policy = jsonencode(var.delivery_retry_policy)
    }
  }
}

# KMS Key for SNS Encryption
resource "aws_kms_key" "sns" {
  count = var.enable_encryption ? 1 : 0
  
  description             = "KMS key for SNS encryption - ${local.topic_name_prefix}"
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
          Service = "sns.amazonaws.com"
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
    Name = "${local.topic_name_prefix}-sns-key"
  })
}

resource "aws_kms_alias" "sns" {
  count = var.enable_encryption ? 1 : 0
  
  name          = "alias/${local.topic_name_prefix}-sns"
  target_key_id = aws_kms_key.sns[0].key_id
}

# SNS Topics
resource "aws_sns_topic" "topics" {
  for_each = local.topic_configs
  
  name         = each.value.name
  display_name = each.value.display_name
  
  # Enable encryption if specified
  kms_master_key_id = var.enable_encryption ? aws_kms_key.sns[0].id : null
  
  # Delivery policy for retry configuration
  delivery_policy = each.value.delivery_policy

  tags = merge(local.common_tags, {
    Name     = each.value.name
    TopicType = each.key
  })
}

# Topic Policies for Cross-Account Access
resource "aws_sns_topic_policy" "cross_account" {
  for_each = length(var.cross_account_principals) > 0 ? local.topic_configs : {}
  
  arn = aws_sns_topic.topics[each.key].arn

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          AWS = [for principal in var.cross_account_principals : "arn:aws:iam::${principal}:root"]
        }
        Action = [
          "sns:Publish",
          "sns:GetTopicAttributes"
        ]
        Resource = aws_sns_topic.topics[each.key].arn
        Condition = {
          StringEquals = {
            "sns:Protocol" = ["sqs", "lambda", "email", "sms"]
          }
        }
      }
    ]
  })
}

# Email Subscriptions
resource "aws_sns_topic_subscription" "email" {
  for_each = {
    for combo in setproduct(keys(local.topic_configs), var.email_endpoints) :
    "${combo[0]}-${replace(combo[1], "@", "-at-")}" => {
      topic_key = combo[0]
      endpoint  = combo[1]
    }
  }
  
  topic_arn = aws_sns_topic.topics[each.value.topic_key].arn
  protocol  = "email"
  endpoint  = each.value.endpoint
}

# SMS Subscriptions (for critical alerts only)
resource "aws_sns_topic_subscription" "sms" {
  for_each = length(var.sms_endpoints) > 0 ? {
    for endpoint in var.sms_endpoints :
    "alerts-${replace(endpoint, "+", "plus")}" => endpoint
  } : {}
  
  topic_arn = aws_sns_topic.topics["alerts"].arn
  protocol  = "sms"
  endpoint  = each.value
}

# SQS Subscriptions
resource "aws_sns_topic_subscription" "sqs" {
  for_each = {
    for combo in setproduct(keys(local.topic_configs), var.sqs_queue_arns) :
    "${combo[0]}-${basename(combo[1])}" => {
      topic_key = combo[0]
      queue_arn = combo[1]
    }
  }
  
  topic_arn = aws_sns_topic.topics[each.value.topic_key].arn
  protocol  = "sqs"
  endpoint  = each.value.queue_arn
  
  # Enable raw message delivery for SQS
  raw_message_delivery = true
  
  # Filter policy example (optional)
  filter_policy = jsonencode({
    source = ["application"]
  })
}

# Lambda Subscriptions
resource "aws_sns_topic_subscription" "lambda" {
  for_each = {
    for combo in setproduct(keys(local.topic_configs), var.lambda_function_arns) :
    "${combo[0]}-${basename(combo[1])}" => {
      topic_key     = combo[0]
      function_arn  = combo[1]
    }
  }
  
  topic_arn = aws_sns_topic.topics[each.value.topic_key].arn
  protocol  = "lambda"
  endpoint  = each.value.function_arn
}

# CloudWatch Alarms for SNS
resource "aws_cloudwatch_metric_alarm" "sns_failed_notifications" {
  for_each = local.topic_configs
  
  alarm_name          = "${each.value.name}-failed-notifications"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "NumberOfNotificationsFailed"
  namespace           = "AWS/SNS"
  period              = "300"
  statistic           = "Sum"
  threshold           = var.environment == "prod" ? "1" : "5"
  alarm_description   = "This metric monitors SNS notification failures for ${each.value.display_name}"
  alarm_actions       = [aws_sns_topic.topics["alerts"].arn]

  dimensions = {
    TopicName = each.value.name
  }

  tags = local.common_tags
}

resource "aws_cloudwatch_metric_alarm" "sns_message_size" {
  for_each = local.topic_configs
  
  alarm_name          = "${each.value.name}-message-size"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "PublishSize"
  namespace           = "AWS/SNS"
  period              = "300"
  statistic           = "Average"
  threshold           = "200000" # 200KB
  alarm_description   = "This metric monitors SNS message size for ${each.value.display_name}"
  alarm_actions       = [aws_sns_topic.topics["alerts"].arn]

  dimensions = {
    TopicName = each.value.name
  }

  tags = local.common_tags
}

# Data sources
data "aws_caller_identity" "current" {}
```

**terraform/outputs.tf:**
```hcl
output "topic_arns" {
  description = "ARNs of all SNS topics"
  value = {
    for key, topic in aws_sns_topic.topics : key => topic.arn
  }
}

output "topic_names" {
  description = "Names of all SNS topics"
  value = {
    for key, topic in aws_sns_topic.topics : key => topic.name
  }
}

output "alerts_topic_arn" {
  description = "ARN of the alerts topic"
  value       = aws_sns_topic.topics["alerts"].arn
}

output "notifications_topic_arn" {
  description = "ARN of the notifications topic"
  value       = aws_sns_topic.topics["notifications"].arn
}

output "system_events_topic_arn" {
  description = "ARN of the system events topic"
  value       = aws_sns_topic.topics["system_events"].arn
}

output "email_subscriptions" {
  description = "Email subscription ARNs"
  value = {
    for key, sub in aws_sns_topic_subscription.email : key => sub.arn
  }
}

output "sms_subscriptions" {
  description = "SMS subscription ARNs"
  value = {
    for key, sub in aws_sns_topic_subscription.sms : key => sub.arn
  }
}

output "sqs_subscriptions" {
  description = "SQS subscription ARNs"
  value = {
    for key, sub in aws_sns_topic_subscription.sqs : key => sub.arn
  }
}

output "lambda_subscriptions" {
  description = "Lambda subscription ARNs"
  value = {
    for key, sub in aws_sns_topic_subscription.lambda : key => sub.arn
  }
}

output "kms_key_id" {
  description = "ID of the KMS key for SNS encryption"
  value       = var.enable_encryption ? aws_kms_key.sns[0].id : null
}

output "cloudwatch_alarm_arns" {
  description = "ARNs of CloudWatch alarms"
  value = {
    failed_notifications = {
      for key, alarm in aws_cloudwatch_metric_alarm.sns_failed_notifications : key => alarm.arn
    }
    message_size = {
      for key, alarm in aws_cloudwatch_metric_alarm.sns_message_size : key => alarm.arn
    }
  }
}
```

## ⚙️ Arquivos de Configuração por Ambiente

**terraform/environments/dev.tfvars:**
```hcl
environment         = "dev"
project_name       = "myapp"
aws_region         = "us-east-1"
enable_encryption  = false

email_endpoints = [
  "dev-team@yourcompany.com"
]

sms_endpoints = []

cross_account_principals = []

lambda_function_arns = []

sqs_queue_arns = []

delivery_retry_policy = {
  num_retries           = 2
  num_max_delay_retries = 1
  min_delay_target      = 10
  max_delay_target      = 10
  num_min_delay_retries = 0
}
```

**terraform/environments/staging.tfvars:**
```hcl
environment         = "staging"
project_name       = "myapp"
aws_region         = "us-east-1"
enable_encryption  = true

email_endpoints = [
  "staging-alerts@yourcompany.com",
  "devops@yourcompany.com"
]

sms_endpoints = [
  "+5511999999999"
]

cross_account_principals = [
  "123456789012"  # Development account
]

lambda_function_arns = [
  # "arn:aws:lambda:us-east-1:account:function:staging-processor"
]

sqs_queue_arns = [
  # "arn:aws:sqs:us-east-1:account:staging-notification-queue"
]

delivery_retry_policy = {
  num_retries           = 3
  num_max_delay_retries = 2
  min_delay_target      = 20
  max_delay_target      = 30
  num_min_delay_retries = 0
}
```

**terraform/environments/prod.tfvars:**
```hcl
environment         = "prod"
project_name       = "myapp"
aws_region         = "us-east-1"
enable_encryption  = true

email_endpoints = [
  "production-alerts@yourcompany.com",
  "devops@yourcompany.com",
  "oncall@yourcompany.com"
]

sms_endpoints = [
  "+5511999999999",  # On-call primary
  "+5511888888888"   # On-call secondary
]

cross_account_principals = [
  "123456789012",  # Development account
  "234567890123"   # Staging account
]

lambda_function_arns = [
  # "arn:aws:lambda:us-east-1:account:function:prod-notification-processor",
  # "arn:aws:lambda:us-east-1:account:function:prod-alert-enricher"
]

sqs_queue_arns = [
  # "arn:aws:sqs:us-east-1:account:prod-notification-queue",
  # "arn:aws:sqs:us-east-1:account:prod-dlq"
]

delivery_retry_policy = {
  num_retries           = 5
  num_max_delay_retries = 3
  min_delay_target      = 20
  max_delay_target      = 300
  num_min_delay_retries = 2
}
```

## 🔧 Como Usar

### 1. Setup Inicial

```bash
# Clone e configure o projeto
git clone <your-repo>
cd aws-sns-terraform

# Configure secrets no GitHub
# AWS_TERRAFORM_ROLE_ARN
```

### 2. Publish Message via AWS CLI

```bash
# Publish to alerts topic
aws sns publish \
  --topic-arn "arn:aws:sns:us-east-1:account:myapp-prod-alerts" \
  --subject "Critical Alert" \
  --message '{"source":"application","severity":"critical","message":"Database connection failed"}' \
  --message-attributes '{"source":{"DataType":"String","StringValue":"application"}}'

# Publish to notifications topic
aws sns publish \
  --topic-arn "arn:aws:sns:us-east-1:account:myapp-prod-notifications" \
  --subject "User Notification" \
  --message '{"user_id":"12345","type":"welcome","content":"Welcome to our platform!"}'
```

### 3. Code Example (Python)

```python
import boto3
import json

def send_alert(message, severity="info"):
    sns = boto3.client('sns')
    
    topic_arn = "arn:aws:sns:us-east-1:account:myapp-prod-alerts"
    
    sns.publish(
        TopicArn=topic_arn,
        Subject=f"{severity.upper()} Alert",
        Message=json.dumps({
            "source": "application",
            "severity": severity,
            "message": message,
            "timestamp": int(time.time())
        }),
        MessageAttributes={
            'source': {
                'DataType': 'String',
                'StringValue': 'application'
            },
            'severity': {
                'DataType': 'String',
                'StringValue': severity
            }
        }
    )

# Usage
send_alert("High CPU usage detected", "warning")
send_alert("Database connection failed", "critical")
```

## 📊 Monitoramento

O exemplo inclui CloudWatch alarms para:
- **Failed Notifications**: Monitora falhas na entrega
- **Message Size**: Alerta sobre mensagens muito grandes
- **Subscription Health**: Monitora health das subscriptions

## 🔒 Segurança

- **Encryption**: KMS encryption para mensagens
- **Cross-Account Access**: Políticas controladas para acesso entre contas
- **IAM Integration**: Políticas de acesso granulares
- **Filter Policies**: Filtragem de mensagens por atributos

## 📱 Tipos de Subscription

### Email
- Ideal para: Notificações para humanos, relatórios
- Configuração: Confirmação manual necessária

### SMS
- Ideal para: Alertas críticos, on-call notifications
- Limitação: Custos por mensagem

### SQS
- Ideal para: Processing assíncrono, sistemas desacoplados
- Benefício: Raw message delivery, DLQ support

### Lambda
- Ideal para: Processing em tempo real, enrichment
- Benefício: Processamento automático, transformações

## 📚 Próximos Passos

1. Implemente Lambda functions para processar mensagens
2. Configure SQS queues para subscribers
3. Setup de monitoring avançado com X-Ray
4. Implemente message filtering avançado
5. Configure cross-region replication

---

💡 **Dica**: Use message attributes para implementar filtering policies e rotear mensagens eficientemente!