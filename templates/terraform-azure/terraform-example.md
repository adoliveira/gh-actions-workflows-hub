# Terraform Azure Example

Este exemplo mostra uma configuração Terraform para Azure com Web App, Storage Account e recursos relacionados.

## 📁 Arquivos de Configuração

### providers.tf
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
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "stterraformstate001"
    container_name       = "tfstate"
    key                  = "myapp/terraform.tfstate"
  }
}

provider "azurerm" {
  features {
    key_vault {
      purge_soft_delete_on_destroy    = true
      recover_soft_deleted_key_vaults = true
    }
  }
}
```

### variables.tf
```hcl
variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "location" {
  description = "Azure location for resources"
  type        = string
  default     = "East US"
}

variable "app_name" {
  description = "Application name"
  type        = string
}

variable "vm_size" {
  description = "Size of the App Service Plan"
  type        = string
  default     = "B1"
}

variable "node_version" {
  description = "Node.js version for the web app"
  type        = string
  default     = "18-lts"
}

variable "enable_monitoring" {
  description = "Enable Application Insights monitoring"
  type        = bool
  default     = false
}

variable "custom_domain" {
  description = "Custom domain for the web app"
  type        = string
  default     = null
}

variable "tags" {
  description = "Resource tags"
  type        = map(string)
  default     = {}
}
```

### main.tf
```hcl
locals {
  # Common tags applied to all resources
  common_tags = merge(var.tags, {
    Environment = var.environment
    Project     = var.app_name
    ManagedBy   = "Terraform"
    CreatedAt   = timestamp()
  })
  
  # Environment-specific settings
  environment_config = {
    dev = {
      sku_name                = "B1"
      storage_replication     = "LRS"
      enable_backup          = false
      min_instances         = 1
      max_instances         = 2
    }
    staging = {
      sku_name                = "S1"
      storage_replication     = "LRS"
      enable_backup          = true
      min_instances         = 1
      max_instances         = 3
    }
    prod = {
      sku_name                = "P1v3"
      storage_replication     = "GRS"
      enable_backup          = true
      min_instances         = 2
      max_instances         = 10
    }
  }
}

# Resource Group
resource "azurerm_resource_group" "main" {
  name     = "rg-${var.app_name}-${var.environment}"
  location = var.location
  tags     = local.common_tags
}

# Storage Account
resource "azurerm_storage_account" "main" {
  name                     = "st${replace(var.app_name, "-", "")}${var.environment}"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = local.environment_config[var.environment].storage_replication
  
  blob_properties {
    versioning_enabled = var.environment == "prod"
    
    delete_retention_policy {
      days = var.environment == "prod" ? 30 : 7
    }
  }

  tags = local.common_tags
}

# Storage Container for application data
resource "azurerm_storage_container" "app_data" {
  name                  = "app-data"
  storage_account_name  = azurerm_storage_account.main.name
  container_access_type = "private"
}

# Application Insights (if monitoring enabled)
resource "azurerm_application_insights" "main" {
  count               = var.enable_monitoring ? 1 : 0
  name                = "ai-${var.app_name}-${var.environment}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  application_type    = "web"
  retention_in_days   = var.environment == "prod" ? 90 : 30

  tags = local.common_tags
}

# Log Analytics Workspace
resource "azurerm_log_analytics_workspace" "main" {
  name                = "law-${var.app_name}-${var.environment}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "PerGB2018"
  retention_in_days   = var.environment == "prod" ? 90 : 30

  tags = local.common_tags
}

# App Service Plan
resource "azurerm_service_plan" "main" {
  name                = "asp-${var.app_name}-${var.environment}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  os_type             = "Linux"
  sku_name            = local.environment_config[var.environment].sku_name

  tags = local.common_tags
}

# Linux Web App
resource "azurerm_linux_web_app" "main" {
  name                = "app-${var.app_name}-${var.environment}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  service_plan_id     = azurerm_service_plan.main.id
  https_only          = true

  site_config {
    always_on                         = var.environment != "dev"
    ftps_state                       = "Disabled"
    http2_enabled                    = true
    minimum_tls_version              = "1.2"
    use_32_bit_worker                = false
    websockets_enabled               = true
    
    application_stack {
      node_version = var.node_version
    }

    # Health check path
    health_check_path = "/health"

    # Auto-scaling settings for production
    dynamic "auto_heal_setting" {
      for_each = var.environment == "prod" ? [1] : []
      content {
        action {
          action_type = "Recycle"
        }
        trigger {
          requests {
            count    = 100
            interval = "00:01:00"
          }
          status_code {
            count             = 10
            interval          = "00:01:00"
            status_code_range = "500-599"
          }
        }
      }
    }
  }

  app_settings = {
    "ENVIRONMENT"                     = var.environment
    "APP_NAME"                       = var.app_name
    "WEBSITE_NODE_DEFAULT_VERSION"   = var.node_version
    "WEBSITE_RUN_FROM_PACKAGE"      = "1"
    "AZURE_STORAGE_ACCOUNT"         = azurerm_storage_account.main.name
    "AZURE_STORAGE_CONNECTION_STRING" = azurerm_storage_account.main.primary_connection_string
    "APPINSIGHTS_INSTRUMENTATIONKEY" = var.enable_monitoring ? azurerm_application_insights.main[0].instrumentation_key : ""
    "APPLICATIONINSIGHTS_CONNECTION_STRING" = var.enable_monitoring ? azurerm_application_insights.main[0].connection_string : ""
    
    # Performance optimization
    "WEBSITE_ENABLE_SYNC_UPDATE_SITE" = "true"
    "WEBSITE_DYNAMIC_CACHE"           = "0"
  }

  connection_string {
    name  = "DefaultConnection"
    type  = "Custom"
    value = azurerm_storage_account.main.primary_connection_string
  }

  logs {
    detailed_error_messages = true
    failed_request_tracing  = true
    
    application_logs {
      file_system_level = "Information"
    }
    
    http_logs {
      file_system {
        retention_in_days = 7
        retention_in_mb   = 35
      }
    }
  }

  tags = local.common_tags

  lifecycle {
    ignore_changes = [
      app_settings["WEBSITE_ENABLE_SYNC_UPDATE_SITE"],
    ]
  }
}

# Auto-scaling settings for production
resource "azurerm_monitor_autoscale_setting" "main" {
  count               = var.environment == "prod" ? 1 : 0
  name                = "autoscale-${var.app_name}-${var.environment}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  target_resource_id  = azurerm_service_plan.main.id

  profile {
    name = "default"

    capacity {
      default = local.environment_config[var.environment].min_instances
      minimum = local.environment_config[var.environment].min_instances
      maximum = local.environment_config[var.environment].max_instances
    }

    rule {
      metric_trigger {
        metric_name        = "CpuPercentage"
        metric_resource_id = azurerm_service_plan.main.id
        time_grain         = "PT1M"
        statistic          = "Average"
        time_window        = "PT5M"
        time_aggregation   = "Average"
        operator           = "GreaterThan"
        threshold          = 75
      }

      scale_action {
        direction = "Increase"
        type      = "ChangeCount"
        value     = "1"
        cooldown  = "PT5M"
      }
    }

    rule {
      metric_trigger {
        metric_name        = "CpuPercentage"
        metric_resource_id = azurerm_service_plan.main.id
        time_grain         = "PT1M"
        statistic          = "Average"
        time_window        = "PT5M"
        time_aggregation   = "Average"
        operator           = "LessThan"
        threshold          = 25
      }

      scale_action {
        direction = "Decrease"
        type      = "ChangeCount"
        value     = "1"
        cooldown  = "PT5M"
      }
    }
  }

  tags = local.common_tags
}

# Custom Domain (if specified)
resource "azurerm_app_service_custom_hostname_binding" "main" {
  count               = var.custom_domain != null ? 1 : 0
  hostname            = var.custom_domain
  app_service_name    = azurerm_linux_web_app.main.name
  resource_group_name = azurerm_resource_group.main.name

  lifecycle {
    ignore_changes = [ssl_state, thumbprint]
  }
}

# Key Vault for secrets
resource "azurerm_key_vault" "main" {
  name                = "kv-${var.app_name}-${var.environment}"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  tenant_id           = data.azurerm_client_config.current.tenant_id
  sku_name            = "standard"

  purge_protection_enabled   = var.environment == "prod"
  soft_delete_retention_days = 7

  access_policy {
    tenant_id = data.azurerm_client_config.current.tenant_id
    object_id = data.azurerm_client_config.current.object_id

    secret_permissions = [
      "Get", "List", "Set", "Delete", "Recover", "Backup", "Restore", "Purge"
    ]
  }

  # Allow Web App to access Key Vault
  access_policy {
    tenant_id = data.azurerm_client_config.current.tenant_id
    object_id = azurerm_linux_web_app.main.identity[0].principal_id

    secret_permissions = [
      "Get", "List"
    ]
  }

  tags = local.common_tags
}

# Get current Azure client configuration
data "azurerm_client_config" "current" {}

# Add managed identity to Web App
resource "azurerm_linux_web_app" "main" {
  # ... other configuration ...

  identity {
    type = "SystemAssigned"
  }
}
```

### outputs.tf
```hcl
output "resource_group_name" {
  description = "Name of the resource group"
  value       = azurerm_resource_group.main.name
}

output "web_app_name" {
  description = "Name of the web app"
  value       = azurerm_linux_web_app.main.name
}

output "web_app_url" {
  description = "URL of the web app"
  value       = "https://${azurerm_linux_web_app.main.default_hostname}"
}

output "web_app_identity" {
  description = "Managed identity of the web app"
  value = {
    principal_id = azurerm_linux_web_app.main.identity[0].principal_id
    tenant_id    = azurerm_linux_web_app.main.identity[0].tenant_id
  }
}

output "storage_account_name" {
  description = "Name of the storage account"
  value       = azurerm_storage_account.main.name
}

output "storage_connection_string" {
  description = "Connection string for the storage account"
  value       = azurerm_storage_account.main.primary_connection_string
  sensitive   = true
}

output "key_vault_uri" {
  description = "URI of the Key Vault"
  value       = azurerm_key_vault.main.vault_uri
}

output "application_insights_key" {
  description = "Application Insights instrumentation key"
  value       = var.enable_monitoring ? azurerm_application_insights.main[0].instrumentation_key : null
  sensitive   = true
}

output "application_insights_connection_string" {
  description = "Application Insights connection string"
  value       = var.enable_monitoring ? azurerm_application_insights.main[0].connection_string : null
  sensitive   = true
}

output "custom_domain_verification" {
  description = "Custom domain verification details"
  value = var.custom_domain != null ? {
    hostname = azurerm_app_service_custom_hostname_binding.main[0].hostname
    # Add TXT record to your DNS: asuid.yourdomain.com -> web_app_identity.principal_id
    txt_record = "asuid.${var.custom_domain}"
    value      = azurerm_linux_web_app.main.identity[0].principal_id
  } : null
}
```

## 🎯 Arquivos de Variáveis por Ambiente

### environments/dev.tfvars
```hcl
environment      = "dev"
location        = "East US"
app_name        = "myawesomeapp"
vm_size         = "B1"
node_version    = "18-lts"
enable_monitoring = false

tags = {
  Team        = "Development"
  CostCenter  = "Engineering"
  Owner       = "dev-team@company.com"
}
```

### environments/staging.tfvars
```hcl
environment      = "staging"
location        = "East US"
app_name        = "myawesomeapp"
vm_size         = "S1"
node_version    = "18-lts"
enable_monitoring = true

tags = {
  Team        = "QA"
  CostCenter  = "Engineering"
  Owner       = "qa-team@company.com"
}
```

### environments/prod.tfvars
```hcl
environment      = "prod"
location        = "East US"
app_name        = "myawesomeapp"
vm_size         = "P1v3"
node_version    = "18-lts"
enable_monitoring = true
custom_domain   = "myapp.company.com"

tags = {
  Team        = "Platform"
  CostCenter  = "Operations"
  Owner       = "platform-team@company.com"
  Backup      = "Required"
  Monitoring  = "Critical"
}
```