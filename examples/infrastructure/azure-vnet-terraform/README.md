# Azure Virtual Network with Terraform Example

Este exemplo demonstra como provisionar Virtual Networks no Azure usando Terraform com o workflow hub.

## 📋 Recursos Provisionados

- **Virtual Network (VNet)**: Rede principal isolada
- **Subnets**: Sub-redes para diferentes tiers (web, app, data)
- **Network Security Groups (NSG)**: Regras de firewall
- **Route Tables**: Tabelas de roteamento customizadas
- **NAT Gateway**: Para acesso à internet das subnets privadas
- **Application Gateway**: Load balancer layer 7
- **Monitoring**: Network Watcher e Flow Logs

## 🏗️ Arquitetura

```
┌─────────────────────────────────────────────────────────────────┐
│                        Virtual Network                          │
│                         10.0.0.0/16                            │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │   Web Subnet    │  │   App Subnet    │  │  Data Subnet    │ │
│  │   10.0.1.0/24   │  │   10.0.2.0/24   │  │  10.0.3.0/24    │ │
│  │   (Public)      │  │   (Private)     │  │  (Private)      │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│           │                     │                     │        │
│           │                     │                     │        │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │      NSG        │  │      NSG        │  │      NSG        │ │
│  │   Web Rules     │  │   App Rules     │  │  Data Rules     │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
            │                                         │
            │                                         │
   ┌─────────────────┐                       ┌─────────────────┐
   │ Application     │                       │   NAT Gateway   │
   │ Gateway         │                       │                 │
   └─────────────────┘                       └─────────────────┘
```

## 📁 Estrutura do Projeto

```
azure-vnet-terraform/
├── .github/
│   └── workflows/
│       └── terraform.yml
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── providers.tf
│   ├── modules/
│   │   ├── nsg-rules/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   └── outputs.tf
│   │   └── subnet/
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       └── outputs.tf
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
name: 'Azure Virtual Network Infrastructure'

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
    key                  = "vnet/terraform.tfstate"
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

variable "vnet_address_space" {
  description = "Virtual network address space"
  type        = list(string)
  default     = ["10.0.0.0/16"]
}

variable "subnets" {
  description = "Subnet configurations"
  type = map(object({
    address_prefixes                               = list(string)
    private_endpoint_network_policies_enabled     = optional(bool, false)
    private_link_service_network_policies_enabled = optional(bool, false)
    service_endpoints                              = optional(list(string), [])
    delegation = optional(object({
      name = string
      service_delegation = object({
        name    = string
        actions = list(string)
      })
    }))
  }))
  default = {
    web = {
      address_prefixes = ["10.0.1.0/24"]
      service_endpoints = [
        "Microsoft.Storage",
        "Microsoft.KeyVault"
      ]
    }
    app = {
      address_prefixes = ["10.0.2.0/24"]
      service_endpoints = [
        "Microsoft.Storage",
        "Microsoft.Sql",
        "Microsoft.KeyVault"
      ]
    }
    data = {
      address_prefixes                           = ["10.0.3.0/24"]
      private_endpoint_network_policies_enabled = true
      service_endpoints = [
        "Microsoft.Storage",
        "Microsoft.Sql"
      ]
    }
    gateway = {
      address_prefixes = ["10.0.4.0/24"]
    }
    bastion = {
      address_prefixes = ["10.0.5.0/24"]
    }
  }
}

variable "nsg_rules" {
  description = "Network Security Group rules configuration"
  type = map(list(object({
    name                       = string
    priority                   = number
    direction                  = string
    access                     = string
    protocol                   = string
    source_port_range          = string
    destination_port_range     = optional(string)
    destination_port_ranges    = optional(list(string))
    source_address_prefix      = optional(string)
    source_address_prefixes    = optional(list(string))
    destination_address_prefix = optional(string)
    destination_address_prefixes = optional(list(string))
  })))
  default = {
    web = [
      {
        name                       = "Allow-HTTP"
        priority                   = 100
        direction                  = "Inbound"
        access                     = "Allow"
        protocol                   = "Tcp"
        source_port_range          = "*"
        destination_port_range     = "80"
        source_address_prefix      = "*"
        destination_address_prefix = "*"
      },
      {
        name                       = "Allow-HTTPS"
        priority                   = 110
        direction                  = "Inbound"
        access                     = "Allow"
        protocol                   = "Tcp"
        source_port_range          = "*"
        destination_port_range     = "443"
        source_address_prefix      = "*"
        destination_address_prefix = "*"
      },
      {
        name                       = "Allow-SSH"
        priority                   = 120
        direction                  = "Inbound"
        access                     = "Allow"
        protocol                   = "Tcp"
        source_port_range          = "*"
        destination_port_range     = "22"
        source_address_prefix      = "10.0.5.0/24" # Bastion subnet
        destination_address_prefix = "*"
      }
    ]
    app = [
      {
        name                       = "Allow-Web-to-App"
        priority                   = 100
        direction                  = "Inbound"
        access                     = "Allow"
        protocol                   = "Tcp"
        source_port_range          = "*"
        destination_port_ranges    = ["8080", "8443"]
        source_address_prefix      = "10.0.1.0/24"
        destination_address_prefix = "*"
      },
      {
        name                       = "Allow-SSH"
        priority                   = 110
        direction                  = "Inbound"
        access                     = "Allow"
        protocol                   = "Tcp"
        source_port_range          = "*"
        destination_port_range     = "22"
        source_address_prefix      = "10.0.5.0/24" # Bastion subnet
        destination_address_prefix = "*"
      }
    ]
    data = [
      {
        name                       = "Allow-App-to-Data"
        priority                   = 100
        direction                  = "Inbound"
        access                     = "Allow"
        protocol                   = "Tcp"
        source_port_range          = "*"
        destination_port_ranges    = ["1433", "3306", "5432"]
        source_address_prefix      = "10.0.2.0/24"
        destination_address_prefix = "*"
      },
      {
        name                       = "Allow-SSH"
        priority                   = 110
        direction                  = "Inbound"
        access                     = "Allow"
        protocol                   = "Tcp"
        source_port_range          = "*"
        destination_port_range     = "22"
        source_address_prefix      = "10.0.5.0/24" # Bastion subnet
        destination_address_prefix = "*"
      }
    ]
    bastion = [
      {
        name                       = "Allow-RDP-SSH"
        priority                   = 100
        direction                  = "Inbound"
        access                     = "Allow"
        protocol                   = "Tcp"
        source_port_range          = "*"
        destination_port_ranges    = ["22", "3389"]
        source_address_prefix      = "*"
        destination_address_prefix = "*"
      }
    ]
  }
}

variable "enable_nat_gateway" {
  description = "Enable NAT Gateway for private subnets"
  type        = bool
  default     = true
}

variable "enable_application_gateway" {
  description = "Enable Application Gateway"
  type        = bool
  default     = false
}

variable "enable_ddos_protection" {
  description = "Enable DDoS protection (requires Standard plan)"
  type        = bool
  default     = false
}

variable "enable_flow_logs" {
  description = "Enable NSG Flow Logs"
  type        = bool
  default     = true
}

variable "dns_servers" {
  description = "Custom DNS servers for VNet"
  type        = list(string)
  default     = []
}

variable "route_tables" {
  description = "Custom route table configurations"
  type = map(object({
    disable_bgp_route_propagation = optional(bool, false)
    routes = list(object({
      name           = string
      address_prefix = string
      next_hop_type  = string
      next_hop_in_ip_address = optional(string)
    }))
  }))
  default = {}
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
  
  # Subnets that need NAT Gateway (private subnets)
  private_subnets = ["app", "data"]
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

# DDoS Protection Plan (optional)
resource "azurerm_network_ddos_protection_plan" "main" {
  count = var.enable_ddos_protection ? 1 : 0
  
  name                = "${local.resource_name_prefix}-ddos"
  location            = var.location
  resource_group_name = local.resource_group_name
  
  tags = local.common_tags
}

# Virtual Network
resource "azurerm_virtual_network" "main" {
  name                = "${local.resource_name_prefix}-vnet"
  address_space       = var.vnet_address_space
  location            = var.location
  resource_group_name = local.resource_group_name
  dns_servers         = var.dns_servers

  dynamic "ddos_protection_plan" {
    for_each = var.enable_ddos_protection ? [1] : []
    content {
      id     = azurerm_network_ddos_protection_plan.main[0].id
      enable = true
    }
  }

  tags = local.common_tags
}

# Public IP for NAT Gateway
resource "azurerm_public_ip" "nat_gateway" {
  count = var.enable_nat_gateway ? 1 : 0
  
  name                = "${local.resource_name_prefix}-nat-pip"
  location            = var.location
  resource_group_name = local.resource_group_name
  allocation_method   = "Static"
  sku                 = "Standard"
  zones               = ["1", "2", "3"]

  tags = local.common_tags
}

# NAT Gateway
resource "azurerm_nat_gateway" "main" {
  count = var.enable_nat_gateway ? 1 : 0
  
  name                    = "${local.resource_name_prefix}-nat"
  location                = var.location
  resource_group_name     = local.resource_group_name
  sku_name                = "Standard"
  idle_timeout_in_minutes = 10
  zones                   = ["1", "2", "3"]

  tags = local.common_tags
}

# Associate Public IP with NAT Gateway
resource "azurerm_nat_gateway_public_ip_association" "main" {
  count = var.enable_nat_gateway ? 1 : 0
  
  nat_gateway_id       = azurerm_nat_gateway.main[0].id
  public_ip_address_id = azurerm_public_ip.nat_gateway[0].id
}

# Network Security Groups
resource "azurerm_network_security_group" "nsgs" {
  for_each = var.nsg_rules
  
  name                = "${local.resource_name_prefix}-${each.key}-nsg"
  location            = var.location
  resource_group_name = local.resource_group_name

  tags = local.common_tags
}

# Network Security Group Rules
resource "azurerm_network_security_rule" "rules" {
  for_each = {
    for rule in flatten([
      for nsg_key, rules in var.nsg_rules : [
        for rule in rules : {
          nsg_key = nsg_key
          rule    = rule
          key     = "${nsg_key}-${rule.name}"
        }
      ]
    ]) : rule.key => rule
  }

  name                       = each.value.rule.name
  priority                   = each.value.rule.priority
  direction                  = each.value.rule.direction
  access                     = each.value.rule.access
  protocol                   = each.value.rule.protocol
  source_port_range          = each.value.rule.source_port_range
  destination_port_range     = each.value.rule.destination_port_range
  destination_port_ranges    = each.value.rule.destination_port_ranges
  source_address_prefix      = each.value.rule.source_address_prefix
  source_address_prefixes    = each.value.rule.source_address_prefixes
  destination_address_prefix = each.value.rule.destination_address_prefix
  destination_address_prefixes = each.value.rule.destination_address_prefixes
  resource_group_name        = local.resource_group_name
  network_security_group_name = azurerm_network_security_group.nsgs[each.value.nsg_key].name
}

# Route Tables
resource "azurerm_route_table" "route_tables" {
  for_each = var.route_tables
  
  name                          = "${local.resource_name_prefix}-${each.key}-rt"
  location                      = var.location
  resource_group_name           = local.resource_group_name
  disable_bgp_route_propagation = each.value.disable_bgp_route_propagation

  dynamic "route" {
    for_each = each.value.routes
    content {
      name           = route.value.name
      address_prefix = route.value.address_prefix
      next_hop_type  = route.value.next_hop_type
      next_hop_in_ip_address = route.value.next_hop_in_ip_address
    }
  }

  tags = local.common_tags
}

# Subnets
resource "azurerm_subnet" "subnets" {
  for_each = var.subnets
  
  name                 = "${local.resource_name_prefix}-${each.key}-subnet"
  resource_group_name  = local.resource_group_name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = each.value.address_prefixes
  
  private_endpoint_network_policies_enabled     = each.value.private_endpoint_network_policies_enabled
  private_link_service_network_policies_enabled = each.value.private_link_service_network_policies_enabled
  service_endpoints                              = each.value.service_endpoints

  dynamic "delegation" {
    for_each = each.value.delegation != null ? [each.value.delegation] : []
    content {
      name = delegation.value.name
      service_delegation {
        name    = delegation.value.service_delegation.name
        actions = delegation.value.service_delegation.actions
      }
    }
  }
}

# Associate NSG to Subnets
resource "azurerm_subnet_network_security_group_association" "nsg_associations" {
  for_each = {
    for subnet_key, subnet in var.subnets : subnet_key => subnet
    if contains(keys(var.nsg_rules), subnet_key)
  }
  
  subnet_id                 = azurerm_subnet.subnets[each.key].id
  network_security_group_id = azurerm_network_security_group.nsgs[each.key].id
}

# Associate Route Tables to Subnets
resource "azurerm_subnet_route_table_association" "route_associations" {
  for_each = var.route_tables
  
  subnet_id      = azurerm_subnet.subnets[each.key].id
  route_table_id = azurerm_route_table.route_tables[each.key].id
}

# Associate NAT Gateway to private subnets
resource "azurerm_subnet_nat_gateway_association" "nat_associations" {
  for_each = var.enable_nat_gateway ? toset(local.private_subnets) : []
  
  subnet_id      = azurerm_subnet.subnets[each.value].id
  nat_gateway_id = azurerm_nat_gateway.main[0].id
}

# Storage Account for Flow Logs
resource "azurerm_storage_account" "flow_logs" {
  count = var.enable_flow_logs ? 1 : 0
  
  name                     = "${replace(local.resource_name_prefix, "-", "")}flowlogs"
  resource_group_name      = local.resource_group_name
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  
  tags = local.common_tags
}

# Network Watcher (required for Flow Logs)
resource "azurerm_network_watcher" "main" {
  count = var.enable_flow_logs ? 1 : 0
  
  name                = "${local.resource_name_prefix}-nw"
  location            = var.location
  resource_group_name = local.resource_group_name
  
  tags = local.common_tags
}

# NSG Flow Logs
resource "azurerm_network_watcher_flow_log" "flow_logs" {
  for_each = var.enable_flow_logs ? var.nsg_rules : {}
  
  network_watcher_name = azurerm_network_watcher.main[0].name
  resource_group_name  = local.resource_group_name
  name                 = "${local.resource_name_prefix}-${each.key}-flowlog"

  network_security_group_id = azurerm_network_security_group.nsgs[each.key].id
  storage_account_id        = azurerm_storage_account.flow_logs[0].id
  enabled                   = true

  retention_policy {
    enabled = true
    days    = var.environment == "prod" ? 30 : 7
  }

  traffic_analytics {
    enabled               = true
    workspace_id          = azurerm_log_analytics_workspace.main[0].workspace_id
    workspace_region      = var.location
    workspace_resource_id = azurerm_log_analytics_workspace.main[0].id
    interval_in_minutes   = 10
  }

  tags = local.common_tags
}

# Log Analytics Workspace for Traffic Analytics
resource "azurerm_log_analytics_workspace" "main" {
  count = var.enable_flow_logs ? 1 : 0
  
  name                = "${local.resource_name_prefix}-law"
  location            = var.location
  resource_group_name = local.resource_group_name
  sku                 = "PerGB2018"
  retention_in_days   = var.environment == "prod" ? 30 : 7

  tags = local.common_tags
}

# Application Gateway (optional)
resource "azurerm_public_ip" "app_gateway" {
  count = var.enable_application_gateway ? 1 : 0
  
  name                = "${local.resource_name_prefix}-appgw-pip"
  resource_group_name = local.resource_group_name
  location            = var.location
  allocation_method   = "Static"
  sku                 = "Standard"
  zones               = ["1", "2", "3"]

  tags = local.common_tags
}

resource "azurerm_application_gateway" "main" {
  count = var.enable_application_gateway ? 1 : 0
  
  name                = "${local.resource_name_prefix}-appgw"
  resource_group_name = local.resource_group_name
  location            = var.location

  sku {
    name     = "Standard_v2"
    tier     = "Standard_v2"
    capacity = 2
  }

  gateway_ip_configuration {
    name      = "gateway-ip-configuration"
    subnet_id = azurerm_subnet.subnets["gateway"].id
  }

  frontend_port {
    name = "http-port"
    port = 80
  }

  frontend_port {
    name = "https-port"
    port = 443
  }

  frontend_ip_configuration {
    name                 = "frontend-ip"
    public_ip_address_id = azurerm_public_ip.app_gateway[0].id
  }

  backend_address_pool {
    name = "backend-pool"
  }

  backend_http_settings {
    name                  = "http-settings"
    cookie_based_affinity = "Disabled"
    path                  = "/"
    port                  = 80
    protocol              = "Http"
    request_timeout       = 60
  }

  http_listener {
    name                           = "http-listener"
    frontend_ip_configuration_name = "frontend-ip"
    frontend_port_name             = "http-port"
    protocol                       = "Http"
  }

  request_routing_rule {
    name                       = "routing-rule"
    rule_type                  = "Basic"
    http_listener_name         = "http-listener"
    backend_address_pool_name  = "backend-pool"
    backend_http_settings_name = "http-settings"
    priority                   = 100
  }

  tags = local.common_tags
}
```

**terraform/outputs.tf:**
```hcl
output "vnet_id" {
  description = "ID of the Virtual Network"
  value       = azurerm_virtual_network.main.id
}

output "vnet_name" {
  description = "Name of the Virtual Network"
  value       = azurerm_virtual_network.main.name
}

output "vnet_address_space" {
  description = "Address space of the Virtual Network"
  value       = azurerm_virtual_network.main.address_space
}

output "subnet_ids" {
  description = "IDs of all subnets"
  value = {
    for key, subnet in azurerm_subnet.subnets : key => subnet.id
  }
}

output "subnet_names" {
  description = "Names of all subnets"
  value = {
    for key, subnet in azurerm_subnet.subnets : key => subnet.name
  }
}

output "nsg_ids" {
  description = "IDs of Network Security Groups"
  value = {
    for key, nsg in azurerm_network_security_group.nsgs : key => nsg.id
  }
}

output "nat_gateway_id" {
  description = "ID of the NAT Gateway"
  value       = var.enable_nat_gateway ? azurerm_nat_gateway.main[0].id : null
}

output "nat_gateway_public_ip" {
  description = "Public IP of the NAT Gateway"
  value       = var.enable_nat_gateway ? azurerm_public_ip.nat_gateway[0].ip_address : null
}

output "application_gateway_id" {
  description = "ID of the Application Gateway"
  value       = var.enable_application_gateway ? azurerm_application_gateway.main[0].id : null
}

output "application_gateway_public_ip" {
  description = "Public IP of the Application Gateway"
  value       = var.enable_application_gateway ? azurerm_public_ip.app_gateway[0].ip_address : null
}

output "log_analytics_workspace_id" {
  description = "ID of the Log Analytics Workspace"
  value       = var.enable_flow_logs ? azurerm_log_analytics_workspace.main[0].id : null
}

output "resource_group_name" {
  description = "Name of the resource group"
  value       = local.resource_group_name
}

output "ddos_protection_plan_id" {
  description = "ID of the DDoS Protection Plan"
  value       = var.enable_ddos_protection ? azurerm_network_ddos_protection_plan.main[0].id : null
}
```

## ⚙️ Arquivos de Configuração por Ambiente

**terraform/environments/dev.tfvars:**
```hcl
environment    = "dev"
project_name   = "myapp"
location       = "East US"

vnet_address_space = ["10.0.0.0/16"]

# Simplified configuration for development
subnets = {
  web = {
    address_prefixes = ["10.0.1.0/24"]
  }
  app = {
    address_prefixes = ["10.0.2.0/24"]
  }
}

# Basic NSG rules
nsg_rules = {
  web = [
    {
      name                       = "Allow-HTTP"
      priority                   = 100
      direction                  = "Inbound"
      access                     = "Allow"
      protocol                   = "Tcp"
      source_port_range          = "*"
      destination_port_range     = "80"
      source_address_prefix      = "*"
      destination_address_prefix = "*"
    }
  ]
  app = [
    {
      name                       = "Allow-Web-to-App"
      priority                   = 100
      direction                  = "Inbound"
      access                     = "Allow"
      protocol                   = "Tcp"
      source_port_range          = "*"
      destination_port_range     = "8080"
      source_address_prefix      = "10.0.1.0/24"
      destination_address_prefix = "*"
    }
  ]
}

enable_nat_gateway         = false
enable_application_gateway = false
enable_ddos_protection     = false
enable_flow_logs          = false
```

**terraform/environments/staging.tfvars:**
```hcl
environment    = "staging"
project_name   = "myapp"
location       = "East US"

vnet_address_space = ["10.0.0.0/16"]

# Use default subnets and NSG rules
enable_nat_gateway         = true
enable_application_gateway = false
enable_ddos_protection     = false
enable_flow_logs          = true
```

**terraform/environments/prod.tfvars:**
```hcl
environment    = "prod"
project_name   = "myapp"
location       = "East US"

vnet_address_space = ["10.0.0.0/16"]

# Use default subnets and NSG rules (full configuration)
enable_nat_gateway         = true
enable_application_gateway = true
enable_ddos_protection     = true
enable_flow_logs          = true

# Custom route tables for production
route_tables = {
  app = {
    disable_bgp_route_propagation = false
    routes = [
      {
        name           = "internet-route"
        address_prefix = "0.0.0.0/0"
        next_hop_type  = "Internet"
      }
    ]
  }
}
```

## 🔧 Como Usar

### 1. Setup Inicial

```bash
# Clone e configure o projeto
git clone <your-repo>
cd azure-vnet-terraform

# Configure secrets no GitHub
# ARM_CLIENT_ID, ARM_CLIENT_SECRET, ARM_TENANT_ID, ARM_SUBSCRIPTION_ID
```

### 2. Conectar VMs à VNet

```bash
# Criar VM na subnet web
az vm create \
  --resource-group myapp-prod-rg \
  --name web-vm-01 \
  --image UbuntuLTS \
  --subnet /subscriptions/.../subnets/myapp-prod-web-subnet \
  --admin-username azureuser \
  --generate-ssh-keys

# Criar VM na subnet app
az vm create \
  --resource-group myapp-prod-rg \
  --name app-vm-01 \
  --image UbuntuLTS \
  --subnet /subscriptions/.../subnets/myapp-prod-app-subnet \
  --admin-username azureuser \
  --generate-ssh-keys
```

### 3. Configurar Application Gateway Backend

```hcl
# Adicionar ao terraform/main.tf depois do Application Gateway
resource "azurerm_application_gateway_backend_address_pool" "web_servers" {
  count = var.enable_application_gateway ? 1 : 0
  
  name                = "web-servers"
  resource_group_name = local.resource_group_name
  application_gateway_name = azurerm_application_gateway.main[0].name
  
  backend_addresses {
    ip_address = "10.0.1.10"  # IP da VM web
  }
  
  backend_addresses {
    ip_address = "10.0.1.11"  # IP da VM web
  }
}
```

## 📊 Monitoramento

### Network Watcher Insights
```kusto
// Traffic Analytics Query
AzureNetworkAnalytics_CL
| where TimeGenerated > ago(1h)
| where SubType_s == "FlowLog"
| summarize TotalFlows = count() by SrcIP_s, DestIP_s, DestPort_d
| order by TotalFlows desc
| take 10
```

### NSG Flow Logs Analysis
```kusto
// Top rejected connections
AzureNetworkAnalytics_CL
| where TimeGenerated > ago(24h)
| where FlowStatus_s == "D"  // Denied
| summarize DeniedConnections = count() by SrcIP_s, DestPort_d, NSGRule_s
| order by DeniedConnections desc
| take 10
```

## 🔒 Segurança

### NSG Best Practices
1. **Least Privilege**: Abra apenas portas necessárias
2. **Source Restriction**: Use address ranges específicos
3. **Regular Review**: Revise regras periodicamente
4. **Monitoring**: Configure alertas para conexões negadas

### Network Security
```hcl
# Adicionar regras de segurança extras
variable "security_rules" {
  default = [
    {
      name                       = "Deny-Internet-Outbound-App"
      priority                   = 4000
      direction                  = "Outbound"
      access                     = "Deny"
      protocol                   = "*"
      source_port_range          = "*"
      destination_port_range     = "*"
      source_address_prefix      = "10.0.2.0/24"
      destination_address_prefix = "Internet"
    }
  ]
}
```

## 🎯 Best Practices

### 1. Subnet Design
- Separar tiers (web, app, data)
- Reserve address space para crescimento
- Use convenção de nomenclatura consistente

### 2. Routing
- Configure custom routes para controle
- Use NAT Gateway para egress privado
- Implemente hub-spoke para conectividade

### 3. Monitoring
- Enable Flow Logs para troubleshooting
- Configure Traffic Analytics
- Setup alertas para anomalias

## 🔄 Peering e Conectividade

### VNet Peering
```hcl
resource "azurerm_virtual_network_peering" "hub_to_spoke" {
  name                      = "hub-to-spoke"
  resource_group_name       = azurerm_resource_group.hub.name
  virtual_network_name      = azurerm_virtual_network.hub.name
  remote_virtual_network_id = azurerm_virtual_network.spoke.id
  
  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
  allow_gateway_transit        = false
  use_remote_gateways          = false
}
```

## 📚 Próximos Passos

1. Implemente Azure Firewall para controle centralizado
2. Configure VPN Gateway para conectividade híbrida
3. Setup Azure Bastion para acesso seguro
4. Implemente Private Endpoints para serviços PaaS
5. Configure hub-spoke topology para múltiplas VNets

---

💡 **Dica**: Use address spaces não-overlapping e planeje para crescimento futuro ao design sua VNet!