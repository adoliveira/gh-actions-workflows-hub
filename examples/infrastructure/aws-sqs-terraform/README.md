# AWS SQS with Terraform Example

Este exemplo demonstra como provisionar filas SQS na AWS usando Terraform com o workflow hub.

## 📋 Recursos Provisionados

- **Standard Queue**: Fila padrão para processamento normal
- **FIFO Queue**: Fila FIFO para garantir ordem de processamento
- **Dead Letter Queue**: Fila para mensagens com falha
- **IAM Policies**: Políticas de acesso para as filas
- **CloudWatch Alarms**: Alertas para monitoramento

## 🏗️ Arquitetura

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Application   │───▶│  Standard SQS   │───▶│ Dead Letter SQS │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │   FIFO SQS      │
                       └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │ CloudWatch      │
                       │ Monitoring      │
                       └─────────────────┘
```

## 📁 Estrutura do Projeto

```
aws-sqs-terraform/
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
name: 'AWS SQS Infrastructure'

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
      tf-vars-file: 'environments/${{ github.event.inputs.environment || (github.ref == 'refs/heads/main' && 'prod' || 'dev') }}.tfvars'
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
    key    = "sqs/terraform.tfstate"
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
      Repository    = "aws-sqs-terraform"
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

variable "message_retention_seconds" {
  description = "Message retention period in seconds"
  type        = number
  default     = 1209600 # 14 days
}

variable "visibility_timeout_seconds" {
  description = "Visibility timeout in seconds"
  type        = number
  default     = 30
}

variable "max_receive_count" {
  description = "Maximum receive count before moving to DLQ"
  type        = number
  default     = 3
}

variable "enable_encryption" {
  description = "Enable server-side encryption"
  type        = bool
  default     = true
}

variable "fifo_queue_enabled" {
  description = "Enable FIFO queue creation"
  type        = bool
  default     = true
}

variable "cloudwatch_alarms_enabled" {
  description = "Enable CloudWatch alarms"
  type        = bool
  default     = true
}
```

**terraform/main.tf:**
```hcl
locals {
  queue_name_prefix = "${var.project_name}-${var.environment}"
  
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}

# Dead Letter Queue
resource "aws_sqs_queue" "dead_letter" {
  name                      = "${local.queue_name_prefix}-dlq"
  message_retention_seconds = var.message_retention_seconds
  
  dynamic "server_side_encryption" {
    for_each = var.enable_encryption ? [1] : []
    content {
      kms_master_key_id = aws_kms_key.sqs[0].id
    }
  }

  tags = local.common_tags
}

# Standard Queue
resource "aws_sqs_queue" "standard" {
  name                      = "${local.queue_name_prefix}-standard"
  message_retention_seconds = var.message_retention_seconds
  visibility_timeout_seconds = var.visibility_timeout_seconds
  
  # Dead Letter Queue Configuration
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.dead_letter.arn
    maxReceiveCount     = var.max_receive_count
  })
  
  dynamic "server_side_encryption" {
    for_each = var.enable_encryption ? [1] : []
    content {
      kms_master_key_id = aws_kms_key.sqs[0].id
    }
  }

  tags = local.common_tags
}

# FIFO Queue
resource "aws_sqs_queue" "fifo" {
  count = var.fifo_queue_enabled ? 1 : 0
  
  name                        = "${local.queue_name_prefix}-fifo.fifo"
  fifo_queue                  = true
  content_based_deduplication = true
  message_retention_seconds   = var.message_retention_seconds
  visibility_timeout_seconds  = var.visibility_timeout_seconds
  
  # FIFO Dead Letter Queue
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.fifo_dlq[0].arn
    maxReceiveCount     = var.max_receive_count
  })
  
  dynamic "server_side_encryption" {
    for_each = var.enable_encryption ? [1] : []
    content {
      kms_master_key_id = aws_kms_key.sqs[0].id
    }
  }

  tags = local.common_tags
}

# FIFO Dead Letter Queue
resource "aws_sqs_queue" "fifo_dlq" {
  count = var.fifo_queue_enabled ? 1 : 0
  
  name                      = "${local.queue_name_prefix}-fifo-dlq.fifo"
  fifo_queue               = true
  message_retention_seconds = var.message_retention_seconds
  
  dynamic "server_side_encryption" {
    for_each = var.enable_encryption ? [1] : []
    content {
      kms_master_key_id = aws_kms_key.sqs[0].id
    }
  }

  tags = local.common_tags
}

# KMS Key for SQS Encryption
resource "aws_kms_key" "sqs" {
  count = var.enable_encryption ? 1 : 0
  
  description             = "KMS key for SQS encryption - ${local.queue_name_prefix}"
  deletion_window_in_days = var.environment == "prod" ? 30 : 10

  tags = merge(local.common_tags, {
    Name = "${local.queue_name_prefix}-sqs-key"
  })
}

resource "aws_kms_alias" "sqs" {
  count = var.enable_encryption ? 1 : 0
  
  name          = "alias/${local.queue_name_prefix}-sqs"
  target_key_id = aws_kms_key.sqs[0].key_id
}

# IAM Policy for SQS Access
resource "aws_iam_policy" "sqs_access" {
  name        = "${local.queue_name_prefix}-sqs-access"
  description = "IAM policy for SQS access"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "sqs:SendMessage",
          "sqs:ReceiveMessage",
          "sqs:DeleteMessage",
          "sqs:GetQueueAttributes",
          "sqs:GetQueueUrl"
        ]
        Resource = [
          aws_sqs_queue.standard.arn,
          aws_sqs_queue.dead_letter.arn,
          var.fifo_queue_enabled ? aws_sqs_queue.fifo[0].arn : "",
          var.fifo_queue_enabled ? aws_sqs_queue.fifo_dlq[0].arn : ""
        ]
      },
      {
        Effect = "Allow"
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey"
        ]
        Resource = var.enable_encryption ? [aws_kms_key.sqs[0].arn] : []
        Condition = {
          StringEquals = {
            "kms:ViaService" = "sqs.${var.aws_region}.amazonaws.com"
          }
        }
      }
    ]
  })

  tags = local.common_tags
}

# CloudWatch Alarms
resource "aws_cloudwatch_metric_alarm" "queue_depth_alarm" {
  count = var.cloudwatch_alarms_enabled ? 1 : 0
  
  alarm_name          = "${local.queue_name_prefix}-sqs-depth"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "ApproximateNumberOfMessages"
  namespace           = "AWS/SQS"
  period              = "60"
  statistic           = "Sum"
  threshold           = var.environment == "prod" ? "100" : "50"
  alarm_description   = "This metric monitors SQS queue depth"
  alarm_actions       = [aws_sns_topic.alerts[0].arn]

  dimensions = {
    QueueName = aws_sqs_queue.standard.name
  }

  tags = local.common_tags
}

resource "aws_cloudwatch_metric_alarm" "dlq_alarm" {
  count = var.cloudwatch_alarms_enabled ? 1 : 0
  
  alarm_name          = "${local.queue_name_prefix}-dlq-messages"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "1"
  metric_name         = "ApproximateNumberOfMessages"
  namespace           = "AWS/SQS"
  period              = "60"
  statistic           = "Sum"
  threshold           = "0"
  alarm_description   = "This metric monitors dead letter queue messages"
  alarm_actions       = [aws_sns_topic.alerts[0].arn]

  dimensions = {
    QueueName = aws_sqs_queue.dead_letter.name
  }

  tags = local.common_tags
}

# SNS Topic for Alerts
resource "aws_sns_topic" "alerts" {
  count = var.cloudwatch_alarms_enabled ? 1 : 0
  
  name = "${local.queue_name_prefix}-sqs-alerts"

  tags = local.common_tags
}

resource "aws_sns_topic_subscription" "email_alerts" {
  count = var.cloudwatch_alarms_enabled ? 1 : 0
  
  topic_arn = aws_sns_topic.alerts[0].arn
  protocol  = "email"
  endpoint  = "devops@yourcompany.com" # Replace with actual email
}
```

**terraform/outputs.tf:**
```hcl
output "standard_queue_url" {
  description = "URL of the standard SQS queue"
  value       = aws_sqs_queue.standard.url
}

output "standard_queue_arn" {
  description = "ARN of the standard SQS queue"
  value       = aws_sqs_queue.standard.arn
}

output "dead_letter_queue_url" {
  description = "URL of the dead letter queue"
  value       = aws_sqs_queue.dead_letter.url
}

output "dead_letter_queue_arn" {
  description = "ARN of the dead letter queue"
  value       = aws_sqs_queue.dead_letter.arn
}

output "fifo_queue_url" {
  description = "URL of the FIFO SQS queue"
  value       = var.fifo_queue_enabled ? aws_sqs_queue.fifo[0].url : null
}

output "fifo_queue_arn" {
  description = "ARN of the FIFO SQS queue"
  value       = var.fifo_queue_enabled ? aws_sqs_queue.fifo[0].arn : null
}

output "sqs_access_policy_arn" {
  description = "ARN of the SQS access IAM policy"
  value       = aws_iam_policy.sqs_access.arn
}

output "kms_key_id" {
  description = "ID of the KMS key for SQS encryption"
  value       = var.enable_encryption ? aws_kms_key.sqs[0].id : null
}

output "cloudwatch_alarm_arns" {
  description = "ARNs of CloudWatch alarms"
  value = var.cloudwatch_alarms_enabled ? {
    queue_depth = aws_cloudwatch_metric_alarm.queue_depth_alarm[0].arn
    dead_letter = aws_cloudwatch_metric_alarm.dlq_alarm[0].arn
  } : {}
}
```

## ⚙️ Arquivos de Configuração por Ambiente

**terraform/environments/dev.tfvars:**
```hcl
environment                  = "dev"
project_name                = "myapp"
aws_region                  = "us-east-1"
message_retention_seconds   = 345600  # 4 days
visibility_timeout_seconds  = 30
max_receive_count           = 3
enable_encryption           = false
fifo_queue_enabled          = true
cloudwatch_alarms_enabled   = false
```

**terraform/environments/staging.tfvars:**
```hcl
environment                  = "staging"
project_name                = "myapp"
aws_region                  = "us-east-1"
message_retention_seconds   = 864000   # 10 days
visibility_timeout_seconds  = 60
max_receive_count           = 5
enable_encryption           = true
fifo_queue_enabled          = true
cloudwatch_alarms_enabled   = true
```

**terraform/environments/prod.tfvars:**
```hcl
environment                  = "prod"
project_name                = "myapp"
aws_region                  = "us-east-1"
message_retention_seconds   = 1209600  # 14 days
visibility_timeout_seconds  = 300      # 5 minutes
max_receive_count           = 3
enable_encryption           = true
fifo_queue_enabled          = true
cloudwatch_alarms_enabled   = true
```

## 🔧 Como Usar

### 1. Setup Inicial

```bash
# Clone e configure o projeto
git clone <your-repo>
cd aws-sqs-terraform

# Configure secrets no GitHub
# AWS_TERRAFORM_ROLE_ARN
```

### 2. Deploy para Development

```bash
# Push para develop branch
git checkout develop
git add .
git commit -m "Add SQS infrastructure"
git push origin develop
```

### 3. Deploy para Production

```bash
# Push para main branch
git checkout main
git merge develop
git push origin main
# Aprovação manual necessária
```

## 📊 Monitoramento

O exemplo inclui CloudWatch alarms para:
- **Queue Depth**: Monitora número de mensagens na fila
- **Dead Letter Queue**: Alerta quando mensagens chegam na DLQ
- **Email Notifications**: Via SNS topic

## 🔒 Segurança

- **Encryption**: KMS encryption para mensagens
- **IAM Policies**: Acesso mínimo necessário
- **VPC Endpoints**: Para comunicação privada (opcional)

## 📚 Próximos Passos

1. Configure VPC endpoints para acesso privado
2. Implemente Lambda functions para processar mensagens
3. Configure cross-region replication se necessário
4. Setup de monitoring avançado com X-Ray

---

💡 **Dica**: Ajuste os valores de `message_retention_seconds` e `visibility_timeout_seconds` baseado no seu caso de uso específico!