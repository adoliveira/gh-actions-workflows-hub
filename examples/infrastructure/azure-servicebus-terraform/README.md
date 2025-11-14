# Azure Service Bus with Terraform Example

Este exemplo demonstra como provisionar Service Bus no Azure usando Terraform com o workflow hub.

## 📋 Recursos Provisionados

- **Service Bus Namespace**: Namespace principal para organização
- **Queues**: Filas para processamento FIFO
- **Topics**: Tópicos para pub/sub messaging
- **Subscriptions**: Assinaturas com filtros e regras
- **Authorization Rules**: Regras de acesso e permissões
- **Monitoring**: Application Insights e alertas

## 🏗️ Arquitetura

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Application   │───▶│  Service Bus    │───▶│   Subscription  │
└─────────────────┘    │  Namespace      │    │   App A         │
                       │  ┌───────────┐  │    └─────────────────┘
                       │  │   Queue   │  │
                       │  └───────────┘  │    ┌─────────────────┐
                       │  ┌───────────┐  │───▶│   Subscription  │
                       │  │   Topic   │  │    │   App B         │
                       │  └───────────┘  │    └─────────────────┘
                       └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │ Application     │
                       │ Insights        │
                       └─────────────────┘
```

## 📁 Estrutura do Projeto

```
azure-servicebus-terraform/
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
name: 'Azure Service Bus Infrastructure'

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
      cloud-provider: 'azure'
    secrets:
      ARM_CLIENT_ID: ${{ secrets.ARM_CLIENT_ID }}
      ARM_CLIENT_SECRET: ${{ secrets.ARM_CLIENT_SECRET }}
      ARM_TENANT_ID: ${{ secrets.ARM_TENANT_ID }}
      ARM_SUBSCRIPTION_ID: ${{ secrets.ARM_SUBSCRIPTION_ID }}
```

### 2. Configure Terraform

**terraform/providers.tf:**
```hcl
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
  
  backend "azurerm" {
    storage_account_name = "your-terraform-storage"
    container_name       = "tfstate"
    key                  = "servicebus/terraform.tfstate"
  }
}

provider "azurerm" {
  features {}
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

variable "location" {
  description = "Azure region"
  type        = string
  default     = "East US"
}

variable "resource_group_name" {
  description = "Resource group name (if null, will be created)"
  type        = string
  default     = null
}

variable "servicebus_sku" {
  description = "Service Bus SKU tier"
  type        = string
  validation {
    condition     = contains(["Basic", "Standard", "Premium"], var.servicebus_sku)
    error_message = "SKU must be Basic, Standard, or Premium."
  }
  default = "Standard"
}

variable "servicebus_capacity" {
  description = "Service Bus messaging units (Premium only)"
  type        = number
  default     = 1
}

variable "enable_partitioning" {
  description = "Enable partitioning for queues and topics"
  type        = bool
  default     = true
}

variable "default_message_ttl" {
  description = "Default message TTL in ISO 8601 duration format"
  type        = string
  default     = "P14D" # 14 days
}

variable "max_delivery_count" {
  description = "Maximum delivery count before message goes to DLQ"
  type        = number
  default     = 10
}

variable "queues" {
  description = "Service Bus queues configuration"
  type = map(object({
    max_size_in_megabytes                = optional(number, 1024)
    lock_duration                        = optional(string, "PT1M")
    max_delivery_count                   = optional(number, 10)
    requires_duplicate_detection         = optional(bool, false)
    requires_session                     = optional(bool, false)
    default_message_ttl                  = optional(string, "P14D")
    dead_lettering_on_message_expiration = optional(bool, true)
    enable_partitioning                  = optional(bool, true)
    enable_express                       = optional(bool, false)
    forward_to                           = optional(string, null)
    forward_dead_lettered_messages_to    = optional(string, null)
  }))
  default = {
    orders = {
      max_size_in_megabytes = 1024
      requires_session      = true
    }
    notifications = {
      max_size_in_megabytes = 2048
      enable_express        = true
    }
    events = {
      max_size_in_megabytes = 5120
      requires_duplicate_detection = true
    }
  }
}

variable "topics" {
  description = "Service Bus topics configuration"
  type = map(object({
    max_size_in_megabytes        = optional(number, 1024)
    requires_duplicate_detection = optional(bool, false)
    default_message_ttl          = optional(string, "P14D")
    enable_partitioning          = optional(bool, true)
    enable_express               = optional(bool, false)
    support_ordering             = optional(bool, false)
    auto_delete_on_idle          = optional(string, null)
  }))
  default = {
    user_events = {
      max_size_in_megabytes = 2048
      support_ordering      = true
    }
    system_events = {
      max_size_in_megabytes = 5120
      requires_duplicate_detection = true
    }
    integration_events = {
      max_size_in_megabytes = 1024
      enable_express        = true
    }
  }
}

variable "topic_subscriptions" {
  description = "Service Bus topic subscriptions configuration"
  type = map(object({
    topic_name                           = string
    max_delivery_count                   = optional(number, 10)
    lock_duration                        = optional(string, "PT1M")
    requires_session                     = optional(bool, false)
    default_message_ttl                  = optional(string, "P14D")
    dead_lettering_on_message_expiration = optional(bool, true)
    dead_lettering_on_filter_evaluation_error = optional(bool, true)
    enable_batched_operations            = optional(bool, true)
    forward_to                           = optional(string, null)
    forward_dead_lettered_messages_to    = optional(string, null)
    sql_filter                           = optional(string, null)
    correlation_filter = optional(object({
      correlation_id      = optional(string)
      message_id          = optional(string)
      to                  = optional(string)
      reply_to            = optional(string)
      label               = optional(string)
      session_id          = optional(string)
      reply_to_session_id = optional(string)
      content_type        = optional(string)
      properties          = optional(map(string))
    }))
  }))
  default = {
    user_events_app_a = {
      topic_name = "user_events"
      sql_filter = "user_type = 'premium'"
    }
    user_events_app_b = {
      topic_name = "user_events"
      correlation_filter = {
        label = "user.created"
        properties = {
          "source" = "web"
        }
      }
    }
    system_events_monitoring = {
      topic_name = "system_events"
      sql_filter = "severity IN ('error', 'critical')"
    }
    integration_events_webhook = {
      topic_name = "integration_events"
      correlation_filter = {
        correlation_id = "webhook"
      }
    }
  }
}

variable "authorization_rules" {
  description = "Service Bus authorization rules"
  type = map(object({
    listen = bool
    send   = bool
    manage = bool
  }))
  default = {
    SendOnly = {
      listen = false
      send   = true
      manage = false
    }
    ListenOnly = {
      listen = true
      send   = false
      manage = false
    }
    SendListen = {
      listen = true
      send   = true
      manage = false
    }
  }
}

variable "enable_monitoring" {
  description = "Enable Application Insights monitoring"
  type        = bool
  default     = true
}
```

**terraform/main.tf:**
```hcl
locals {
  resource_name_prefix = "${var.project_name}-${var.environment}"
  
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "Terraform"
  }
}

# Resource Group
resource "azurerm_resource_group" "main" {
  count = var.resource_group_name == null ? 1 : 0
  
  name     = "${local.resource_name_prefix}-rg"
  location = var.location
  tags     = local.common_tags
}

# Use existing or created resource group
locals {
  resource_group_name = var.resource_group_name != null ? var.resource_group_name : azurerm_resource_group.main[0].name
}

# Service Bus Namespace
resource "azurerm_servicebus_namespace" "main" {
  name                = "${local.resource_name_prefix}-sb"
  location            = var.location
  resource_group_name = local.resource_group_name
  
  sku      = var.servicebus_sku
  capacity = var.servicebus_sku == "Premium" ? var.servicebus_capacity : null

  # Premium features
  zone_redundant = var.servicebus_sku == "Premium" && var.environment == "prod"
  
  tags = local.common_tags
}

# Service Bus Queues
resource "azurerm_servicebus_queue" "queues" {
  for_each = var.queues
  
  name         = each.key
  namespace_id = azurerm_servicebus_namespace.main.id
  
  max_size_in_megabytes                = each.value.max_size_in_megabytes
  lock_duration                        = each.value.lock_duration
  max_delivery_count                   = each.value.max_delivery_count
  requires_duplicate_detection         = each.value.requires_duplicate_detection
  requires_session                     = each.value.requires_session
  default_message_ttl                  = each.value.default_message_ttl
  dead_lettering_on_message_expiration = each.value.dead_lettering_on_message_expiration
  enable_partitioning                  = var.servicebus_sku != "Premium" ? each.value.enable_partitioning : false
  enable_express                       = var.servicebus_sku == "Premium" ? false : each.value.enable_express
  forward_to                           = each.value.forward_to
  forward_dead_lettered_messages_to    = each.value.forward_dead_lettered_messages_to
}

# Service Bus Topics
resource "azurerm_servicebus_topic" "topics" {
  for_each = var.topics
  
  name         = each.key
  namespace_id = azurerm_servicebus_namespace.main.id
  
  max_size_in_megabytes        = each.value.max_size_in_megabytes
  requires_duplicate_detection = each.value.requires_duplicate_detection
  default_message_ttl          = each.value.default_message_ttl
  enable_partitioning          = var.servicebus_sku != "Premium" ? each.value.enable_partitioning : false
  enable_express               = var.servicebus_sku == "Premium" ? false : each.value.enable_express
  support_ordering             = each.value.support_ordering
  auto_delete_on_idle          = each.value.auto_delete_on_idle
}

# Service Bus Topic Subscriptions
resource "azurerm_servicebus_subscription" "subscriptions" {
  for_each = var.topic_subscriptions
  
  name     = each.key
  topic_id = azurerm_servicebus_topic.topics[each.value.topic_name].id
  
  max_delivery_count                           = each.value.max_delivery_count
  lock_duration                                = each.value.lock_duration
  requires_session                             = each.value.requires_session
  default_message_ttl                          = each.value.default_message_ttl
  dead_lettering_on_message_expiration         = each.value.dead_lettering_on_message_expiration
  dead_lettering_on_filter_evaluation_error   = each.value.dead_lettering_on_filter_evaluation_error
  enable_batched_operations                    = each.value.enable_batched_operations
  forward_to                                   = each.value.forward_to
  forward_dead_lettered_messages_to            = each.value.forward_dead_lettered_messages_to
}

# Subscription Rules (SQL Filters)
resource "azurerm_servicebus_subscription_rule" "sql_filters" {
  for_each = {
    for k, v in var.topic_subscriptions : k => v
    if v.sql_filter != null
  }
  
  name            = "${each.key}-sql-filter"
  subscription_id = azurerm_servicebus_subscription.subscriptions[each.key].id
  filter_type     = "SqlFilter"
  sql_filter      = each.value.sql_filter
}

# Subscription Rules (Correlation Filters)
resource "azurerm_servicebus_subscription_rule" "correlation_filters" {
  for_each = {
    for k, v in var.topic_subscriptions : k => v
    if v.correlation_filter != null
  }
  
  name            = "${each.key}-correlation-filter"
  subscription_id = azurerm_servicebus_subscription.subscriptions[each.key].id
  filter_type     = "CorrelationFilter"
  
  correlation_filter {
    correlation_id      = each.value.correlation_filter.correlation_id
    message_id          = each.value.correlation_filter.message_id
    to                  = each.value.correlation_filter.to
    reply_to            = each.value.correlation_filter.reply_to
    label               = each.value.correlation_filter.label
    session_id          = each.value.correlation_filter.session_id
    reply_to_session_id = each.value.correlation_filter.reply_to_session_id
    content_type        = each.value.correlation_filter.content_type
    properties          = each.value.correlation_filter.properties
  }
}

# Authorization Rules
resource "azurerm_servicebus_namespace_authorization_rule" "auth_rules" {
  for_each = var.authorization_rules
  
  name         = each.key
  namespace_id = azurerm_servicebus_namespace.main.id
  
  listen = each.value.listen
  send   = each.value.send
  manage = each.value.manage
}

# Application Insights for monitoring
resource "azurerm_application_insights" "main" {
  count = var.enable_monitoring ? 1 : 0
  
  name                = "${local.resource_name_prefix}-ai"
  location            = var.location
  resource_group_name = local.resource_group_name
  application_type    = "other"
  
  tags = local.common_tags
}

# Monitor Action Group for alerts
resource "azurerm_monitor_action_group" "main" {
  count = var.enable_monitoring ? 1 : 0
  
  name                = "${local.resource_name_prefix}-ag"
  resource_group_name = local.resource_group_name
  short_name          = "sbAlerts"

  email_receiver {
    name          = "devops"
    email_address = "devops@yourcompany.com"
  }

  tags = local.common_tags
}

# Metric Alerts
resource "azurerm_monitor_metric_alert" "queue_length" {
  count = var.enable_monitoring ? 1 : 0
  
  name                = "${local.resource_name_prefix}-queue-length-alert"
  resource_group_name = local.resource_group_name
  scopes              = [azurerm_servicebus_namespace.main.id]
  description         = "Alert when queue length is high"

  criteria {
    metric_namespace = "Microsoft.ServiceBus/namespaces"
    metric_name      = "ActiveMessages"
    aggregation      = "Total"
    operator         = "GreaterThan"
    threshold        = var.environment == "prod" ? 1000 : 500

    dimension {
      name     = "EntityName"
      operator = "Include"
      values   = [for k, v in var.queues : k]
    }
  }

  action {
    action_group_id = azurerm_monitor_action_group.main[0].id
  }

  tags = local.common_tags
}

resource "azurerm_monitor_metric_alert" "dead_letter_messages" {
  count = var.enable_monitoring ? 1 : 0
  
  name                = "${local.resource_name_prefix}-dlq-alert"
  resource_group_name = local.resource_group_name
  scopes              = [azurerm_servicebus_namespace.main.id]
  description         = "Alert when messages are in dead letter queue"

  criteria {
    metric_namespace = "Microsoft.ServiceBus/namespaces"
    metric_name      = "DeadletteredMessages"
    aggregation      = "Total"
    operator         = "GreaterThan"
    threshold        = 0
  }

  action {
    action_group_id = azurerm_monitor_action_group.main[0].id
  }

  tags = local.common_tags
}

resource "azurerm_monitor_metric_alert" "failed_requests" {
  count = var.enable_monitoring ? 1 : 0
  
  name                = "${local.resource_name_prefix}-failed-requests-alert"
  resource_group_name = local.resource_group_name
  scopes              = [azurerm_servicebus_namespace.main.id]
  description         = "Alert when request failure rate is high"

  criteria {
    metric_namespace = "Microsoft.ServiceBus/namespaces"
    metric_name      = "ServerErrors"
    aggregation      = "Total"
    operator         = "GreaterThan"
    threshold        = var.environment == "prod" ? 10 : 20
  }

  action {
    action_group_id = azurerm_monitor_action_group.main[0].id
  }

  tags = local.common_tags
}
```

**terraform/outputs.tf:**
```hcl
output "servicebus_namespace_name" {
  description = "Name of the Service Bus namespace"
  value       = azurerm_servicebus_namespace.main.name
}

output "servicebus_namespace_id" {
  description = "ID of the Service Bus namespace"
  value       = azurerm_servicebus_namespace.main.id
}

output "servicebus_primary_connection_string" {
  description = "Primary connection string for the Service Bus namespace"
  value       = azurerm_servicebus_namespace.main.default_primary_connection_string
  sensitive   = true
}

output "servicebus_secondary_connection_string" {
  description = "Secondary connection string for the Service Bus namespace"
  value       = azurerm_servicebus_namespace.main.default_secondary_connection_string
  sensitive   = true
}

output "queue_names" {
  description = "Names of all Service Bus queues"
  value = {
    for key, queue in azurerm_servicebus_queue.queues : key => queue.name
  }
}

output "topic_names" {
  description = "Names of all Service Bus topics"
  value = {
    for key, topic in azurerm_servicebus_topic.topics : key => topic.name
  }
}

output "subscription_names" {
  description = "Names of all Service Bus topic subscriptions"
  value = {
    for key, subscription in azurerm_servicebus_subscription.subscriptions : key => subscription.name
  }
}

output "authorization_rule_connection_strings" {
  description = "Connection strings for authorization rules"
  value = {
    for key, rule in azurerm_servicebus_namespace_authorization_rule.auth_rules : 
    key => rule.primary_connection_string
  }
  sensitive = true
}

output "application_insights_instrumentation_key" {
  description = "Application Insights instrumentation key"
  value       = var.enable_monitoring ? azurerm_application_insights.main[0].instrumentation_key : null
  sensitive   = true
}

output "application_insights_connection_string" {
  description = "Application Insights connection string"
  value       = var.enable_monitoring ? azurerm_application_insights.main[0].connection_string : null
  sensitive   = true
}

output "resource_group_name" {
  description = "Name of the resource group"
  value       = local.resource_group_name
}
```

## ⚙️ Arquivos de Configuração por Ambiente

**terraform/environments/dev.tfvars:**
```hcl
environment    = "dev"
project_name   = "myapp"
location       = "East US"
servicebus_sku = "Basic"

enable_partitioning = false
enable_monitoring   = false

queues = {
  orders = {
    max_size_in_megabytes = 1024
  }
  notifications = {
    max_size_in_megabytes = 1024
    enable_express        = true
  }
}

topics = {
  user_events = {
    max_size_in_megabytes = 1024
  }
}

topic_subscriptions = {
  user_events_app_a = {
    topic_name = "user_events"
  }
}
```

**terraform/environments/staging.tfvars:**
```hcl
environment    = "staging"
project_name   = "myapp"
location       = "East US"
servicebus_sku = "Standard"

enable_partitioning = true
enable_monitoring   = true

# Use default queues, topics, and subscriptions
```

**terraform/environments/prod.tfvars:**
```hcl
environment         = "prod"
project_name        = "myapp"
location            = "East US"
servicebus_sku      = "Premium"
servicebus_capacity = 2

enable_partitioning = false  # Premium doesn't support partitioning
enable_monitoring   = true

# Use default queues, topics, and subscriptions with full configuration
```

## 🔧 Como Usar

### 1. Setup Inicial

```bash
# Clone e configure o projeto
git clone <your-repo>
cd azure-servicebus-terraform

# Configure secrets no GitHub
# ARM_CLIENT_ID, ARM_CLIENT_SECRET, ARM_TENANT_ID, ARM_SUBSCRIPTION_ID
```

### 2. Code Examples

**C# (.NET):**
```csharp
using Azure.Messaging.ServiceBus;

// Send message to queue
var client = new ServiceBusClient("connection-string");
var sender = client.CreateSender("orders");

var message = new ServiceBusMessage("Order created")
{
    SessionId = "order-session-1",
    MessageId = Guid.NewGuid().ToString(),
    Subject = "order.created"
};

await sender.SendMessageAsync(message);

// Receive messages from queue
var receiver = client.CreateReceiver("orders");
var messages = await receiver.ReceiveMessagesAsync(10);

foreach (var msg in messages)
{
    Console.WriteLine($"Received: {msg.Body}");
    await receiver.CompleteMessageAsync(msg);
}

// Publish to topic
var topicSender = client.CreateSender("user_events");
var topicMessage = new ServiceBusMessage("User registered")
{
    Subject = "user.created",
    ApplicationProperties = 
    {
        ["user_type"] = "premium",
        ["source"] = "web"
    }
};

await topicSender.SendMessageAsync(topicMessage);
```

**Python:**
```python
from azure.servicebus import ServiceBusClient, ServiceBusMessage

# Send message to queue
with ServiceBusClient.from_connection_string("connection-string") as client:
    with client.get_queue_sender("orders") as sender:
        message = ServiceBusMessage(
            "Order created",
            session_id="order-session-1",
            message_id=str(uuid.uuid4()),
            subject="order.created"
        )
        sender.send_messages(message)

    # Receive messages
    with client.get_queue_receiver("orders") as receiver:
        messages = receiver.receive_messages(max_message_count=10)
        for msg in messages:
            print(f"Received: {str(msg)}")
            receiver.complete_message(msg)

    # Publish to topic
    with client.get_topic_sender("user_events") as sender:
        message = ServiceBusMessage(
            "User registered",
            subject="user.created"
        )
        message.application_properties = {
            "user_type": "premium",
            "source": "web"
        }
        sender.send_messages(message)
```

**Node.js:**
```javascript
const { ServiceBusClient } = require("@azure/service-bus");

const client = new ServiceBusClient("connection-string");

// Send message to queue
const sender = client.createSender("orders");
const message = {
    body: "Order created",
    sessionId: "order-session-1",
    messageId: require('crypto').randomUUID(),
    subject: "order.created"
};

await sender.sendMessages(message);

// Receive messages
const receiver = client.createReceiver("orders");
const messages = await receiver.receiveMessages(10);

for (const message of messages) {
    console.log(`Received: ${message.body}`);
    await receiver.completeMessage(message);
}

// Publish to topic
const topicSender = client.createSender("user_events");
const topicMessage = {
    body: "User registered",
    subject: "user.created",
    applicationProperties: {
        user_type: "premium",
        source: "web"
    }
};

await topicSender.sendMessages(topicMessage);
```

## 📊 Monitoramento

### Métricas Importantes
- **ActiveMessages**: Mensagens ativas nas filas
- **DeadletteredMessages**: Mensagens na dead letter queue
- **IncomingMessages**: Taxa de mensagens recebidas
- **OutgoingMessages**: Taxa de mensagens processadas
- **ServerErrors**: Erros do servidor

### Application Insights Queries
```kusto
// Message processing rate
traces
| where customDimensions.EntityName != ""
| summarize Count=count() by bin(timestamp, 5m), tostring(customDimensions.EntityName)
| render timechart

// Dead letter queue monitoring
traces
| where customDimensions.OperationName == "DeadLetter"
| summarize Count=count() by bin(timestamp, 1h), tostring(customDimensions.EntityName)
| render barchart
```

## 🔒 Segurança

### Managed Identity
```hcl
# Add to your application
resource "azurerm_user_assigned_identity" "app" {
  name                = "${local.resource_name_prefix}-app-identity"
  resource_group_name = local.resource_group_name
  location            = var.location
}

# Role assignment
resource "azurerm_role_assignment" "servicebus_data_owner" {
  scope                = azurerm_servicebus_namespace.main.id
  role_definition_name = "Azure Service Bus Data Owner"
  principal_id         = azurerm_user_assigned_identity.app.principal_id
}
```

### Connection Strings
- Use Azure Key Vault para armazenar connection strings
- Configure Managed Identity quando possível
- Rotacione keys regularmente

## 🎯 Best Practices

### 1. Message Design
- Use correlation ID para tracking
- Implemente idempotência
- Configure TTL apropriado

### 2. Error Handling
- Configure dead letter queues
- Implemente retry policies
- Monitor failed messages

### 3. Performance
- Use sessões para ordering
- Configure partitioning (Standard tier)
- Use batching para throughput

## 📚 Próximos Passos

1. Implemente Azure Functions para processar mensagens
2. Configure auto-scaling baseado em queue length
3. Setup de disaster recovery
4. Implemente message encryption
5. Configure networking restrictions

---

💡 **Dica**: Use topics com subscriptions para implementar padrões pub/sub e desacoplar seus serviços!