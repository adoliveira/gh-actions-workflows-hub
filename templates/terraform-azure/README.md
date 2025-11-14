# 🔷 Template: Terraform Azure

Template completo para gerenciar infraestrutura Azure usando Terraform com pipeline CI/CD automatizado.

## 🎯 Características

- ✅ **Pipeline completo** para Terraform com Azure
- ✅ **Múltiplos ambientes** (dev, staging, prod)
- ✅ **Drift detection** automático
- ✅ **Aprovação manual** para produção
- ✅ **Comentários automáticos** em Pull Requests
- ✅ **Operações manuais** via workflow dispatch
- ✅ **Notificações** de deploy e drift

## 📁 Estrutura Recomendada

```
seu-projeto/
├── .github/
│   └── workflows/
│       └── terraform.yml          # Copie o arquivo deste template
├── terraform/
│   ├── main.tf                    # Configuração principal
│   ├── variables.tf               # Variáveis Terraform
│   ├── outputs.tf                 # Outputs Terraform
│   ├── providers.tf               # Providers Azure
│   └── environments/
│       ├── dev.tfvars            # Variáveis para desenvolvimento
│       ├── staging.tfvars        # Variáveis para staging
│       └── prod.tfvars           # Variáveis para produção
└── README.md
```

## 🔧 Setup Inicial

### 1. Configure Secrets no GitHub

Vá para **Settings > Secrets and variables > Actions** e adicione:

```bash
# Azure Service Principal
AZURE_CREDENTIALS='{
  "clientId": "your-client-id",
  "clientSecret": "your-client-secret", 
  "subscriptionId": "your-subscription-id",
  "tenantId": "your-tenant-id"
}'
```

### 2. Configure Variables no GitHub

Em **Variables** na mesma seção:

```bash
AZURE_SUBSCRIPTION_ID=your-subscription-id
AZURE_LOCATION="East US"
```

### 3. Configure Backend Terraform

Crie `terraform/providers.tf`:

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
    storage_account_name = "stterraformstate001"  # Deve ser único globalmente
    container_name       = "tfstate"
    key                  = "terraform.tfstate"
  }
}

provider "azurerm" {
  features {}
}
```

### 4. Configure Ambientes

**terraform/environments/dev.tfvars:**
```hcl
environment = "dev"
location    = "East US"
app_name    = "myapp"

# VM settings
vm_size = "Standard_B1s"
```

**terraform/environments/staging.tfvars:**
```hcl
environment = "staging"
location    = "East US"
app_name    = "myapp"

# VM settings
vm_size = "Standard_B2s"
```

**terraform/environments/prod.tfvars:**
```hcl
environment = "prod"
location    = "East US"
app_name    = "myapp"

# VM settings
vm_size = "Standard_D2s_v3"
```

## 🚀 Como Usar

### 1. Copie o Workflow

```bash
cp templates/terraform-azure/.github-workflows-terraform.yml .github/workflows/terraform.yml
```

### 2. Atualize Referências

No arquivo copiado, substitua:

```yaml
# De:
uses: ./.github/actions/setup-tools

# Para:
uses: YOUR-ORG/gh-actions-workflows-hub/.github/actions/setup-tools@main
```

### 3. Customize Configurações

Edite as variáveis no topo do workflow conforme necessário:

```yaml
env:
  TF_VERSION: 'latest'           # Versão do Terraform
  AZURE_LOCATION: 'East US'     # Localização padrão
  ARM_USE_OIDC: true            # Use OIDC para autenticação
```

## 🔄 Fluxos de Trabalho

### 📊 Pull Request
Quando você criar um PR:
1. ✅ Terraform validate
2. 📋 Terraform plan
3. 💬 Comentário automático no PR com detalhes do plan

### 🚀 Deploy Development
No push para `develop`:
1. ✅ Deploy automático para ambiente de desenvolvimento
2. 📊 Summary com outputs da infraestrutura

### 🎯 Deploy Staging/Production
No push para `main`:
1. 🔄 Deploy automático para staging
2. ⏸️ **Aguarda aprovação manual** para produção
3. 🚀 Deploy para produção após aprovação
4. 📢 Notificação de sucesso

### 🔍 Drift Detection
Diariamente às 3h da manhã:
1. 🔎 Verifica drift na infraestrutura de produção
2. 🚨 Cria issue automática se drift for detectado
3. 📋 Inclui detalhes das mudanças necessárias

### ⚙️ Operação Manual
Via workflow dispatch:
1. 🎯 Escolha o ambiente (dev/staging/prod)
2. 🔧 Deploy ou destroy da infraestrutura
3. ✅ Aprovação automática (exceto produção)

## 🛡️ Ambientes Protegidos

Configure ambientes protegidos no GitHub:

1. **Settings > Environments**
2. **Create environment** para `production`
3. **Required reviewers**: Adicione aprovadores
4. **Deployment branches**: Apenas `main`

## 📚 Exemplo de Infraestrutura

**terraform/main.tf:**
```hcl
variable "environment" {
  description = "Environment name"
  type        = string
}

variable "location" {
  description = "Azure location"
  type        = string
  default     = "East US"
}

variable "app_name" {
  description = "Application name"
  type        = string
}

# Resource Group
resource "azurerm_resource_group" "main" {
  name     = "rg-${var.app_name}-${var.environment}"
  location = var.location

  tags = {
    Environment = var.environment
    Project     = var.app_name
    ManagedBy   = "Terraform"
  }
}

# Storage Account
resource "azurerm_storage_account" "main" {
  name                     = "st${var.app_name}${var.environment}"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = var.environment == "prod" ? "GRS" : "LRS"

  tags = azurerm_resource_group.main.tags
}

# App Service Plan
resource "azurerm_service_plan" "main" {
  name                = "asp-${var.app_name}-${var.environment}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  os_type             = "Linux"
  sku_name            = var.environment == "prod" ? "P1v3" : "B1"

  tags = azurerm_resource_group.main.tags
}

# Web App
resource "azurerm_linux_web_app" "main" {
  name                = "app-${var.app_name}-${var.environment}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  service_plan_id     = azurerm_service_plan.main.id

  site_config {
    application_stack {
      node_version = "18-lts"
    }
  }

  app_settings = {
    "ENVIRONMENT" = var.environment
    "APP_NAME"    = var.app_name
  }

  tags = azurerm_resource_group.main.tags
}
```

**terraform/outputs.tf:**
```hcl
output "resource_group_name" {
  description = "Name of the resource group"
  value       = azurerm_resource_group.main.name
}

output "web_app_url" {
  description = "URL of the web app"
  value       = "https://${azurerm_linux_web_app.main.default_hostname}"
}

output "web_app_name" {
  description = "Name of the web app"
  value       = azurerm_linux_web_app.main.name
}

output "storage_account_name" {
  description = "Name of the storage account"
  value       = azurerm_storage_account.main.name
}
```

## 🔧 Customizações

### Alterar Horário do Drift Detection
```yaml
schedule:
  - cron: '0 8 * * *'  # 8h da manhã
```

### Adicionar Notificações Slack
```yaml
- name: Notify Slack
  if: failure()
  uses: 8398a7/action-slack@v3
  with:
    status: failure
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

### Múltiplos Backends
```yaml
- name: Terraform Init
  run: |
    terraform init \
      -backend-config="key=apps/${{ github.event.inputs.environment }}/terraform.tfstate"
```

## 🛠️ Troubleshooting

### ❌ "Authentication failed"
- Verifique se `AZURE_CREDENTIALS` está configurado corretamente
- Confirme que o Service Principal tem permissões adequadas

### ❌ "Backend configuration changed"
- Execute `terraform init -reconfigure` localmente
- Verifique configuração do backend no `providers.tf`

### ❌ "Resource already exists"
- Use `terraform import` para importar recursos existentes
- Ou ajuste nomes para evitar conflitos

## 📖 Próximos Passos

1. 📚 Leia sobre [Azure Provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
2. 🔒 Configure [Azure Key Vault](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/key_vault) para secrets
3. 🏗️ Explore [Azure Application Gateway](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/application_gateway)
4. 📊 Configure [Azure Monitor](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/monitor_workspace)

---

💡 **Dica**: Use este template como base e customize conforme suas necessidades específicas!