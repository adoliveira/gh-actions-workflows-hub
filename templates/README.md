# 📄 Templates de Workflows

Esta pasta contém templates prontos para diferentes tipos de projetos e cenários, facilitando a configuração rápida de pipelines CI/CD.

## 🎯 Templates Disponíveis

### 📱 Aplicações

| Template | Descrição | Tecnologias | Cloud |
|----------|-----------|-------------|--------|
| [**node-app**](node-app/) | App Node.js completa | Node.js, Docker, Tests | AWS, Azure |

### 🏗️ Infrastructure as Code

| Template | Descrição | Tecnologias | Cloud |
|----------|-----------|-------------|--------|
| [**terraform-aws**](terraform-aws/) | Infraestrutura AWS | Terraform, CloudFormation | AWS |
| [**terraform-azure**](terraform-azure/) | Infraestrutura Azure | Terraform, ARM | Azure |
| [**azure-bicep**](azure-bicep/) | Infraestrutura Azure nativa | Bicep, ARM | Azure |

## 🚀 Como Usar um Template

### 1. Escolha o Template
Navegue para o diretório do template desejado e leia o README específico.

### 2. Copie os Arquivos
```bash
# Exemplo para Node.js app
cp -r templates/node-app/.github-workflows-ci-cd.yml .github/workflows/ci-cd.yml
```

### 3. Atualize Referências
Substitua `YOUR-ORG` pela sua organização:
```yaml
# De:
uses: YOUR-ORG/gh-actions-workflows-hub/.github/workflows/ci-reusable.yml@main

# Para:
uses: adoliveira/gh-actions-workflows-hub/.github/workflows/ci-reusable.yml@main
```

### 4. Configure Secrets
Cada template lista os secrets necessários no seu README.

### 5. Customize Parâmetros
Ajuste os inputs dos workflows conforme suas necessidades.

## 📋 Detalhes dos Templates

### 📱 Node.js Application (`node-app/`)

**Ideal para**: APIs REST, aplicações web, microserviços Node.js

**Pipeline inclui**:
- ✅ CI com testes e linting
- 🐳 Build Docker multi-arch
- 🚀 Deploy automático para dev/staging/prod
- 💙 Blue-green deployment para produção

**Secrets necessários**:
```bash
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

### 🔷 Terraform AWS (`terraform-aws/`)

**Ideal para**: Infraestrutura AWS com Terraform

**Pipeline inclui**:
- ✅ Validation e planning automático
- 📋 Comentários em PRs com planos
- 🔍 Drift detection diário
- 🎯 Deploy multi-ambiente
- ⚠️ Aprovação manual para produção

**Secrets necessários**:
```bash
AWS_TERRAFORM_ROLE_ARN
```

### 🔷 Terraform Azure (`terraform-azure/`) ⭐ **NOVO**

**Ideal para**: Infraestrutura Azure com Terraform

**Pipeline inclui**:
- ✅ Validation e planning automático
- 📋 Comentários em PRs com planos
- 🔍 Drift detection com criação automática de issues
- 🎯 Deploy multi-ambiente (dev/staging/prod)
- ⚙️ Operações manuais via workflow dispatch
- ⚠️ Aprovação manual para produção

**Características especiais**:
- 🚨 Drift detection avançado com issues automáticas
- 📊 Summaries detalhados de infraestrutura
- 🔧 Suporte a operações de destroy via interface
- 💬 Notificações automáticas de deploy

**Secrets necessários**:
```bash
AZURE_CREDENTIALS='{
  "clientId": "your-client-id",
  "clientSecret": "your-client-secret",
  "subscriptionId": "your-subscription-id", 
  "tenantId": "your-tenant-id"
}'
```

**Variables recomendadas**:
```bash
AZURE_SUBSCRIPTION_ID=your-subscription-id
AZURE_LOCATION="East US"
```

### ☁️ Azure Bicep (`azure-bicep/`)

**Ideal para**: Infraestrutura Azure nativa com Bicep

**Pipeline inclui**:
- ✅ Validation de templates Bicep
- 📊 What-if analysis em PRs
- 🚀 Deploy multi-ambiente
- ⚠️ Aprovação para produção

**Secrets necessários**:
```bash
AZURE_CREDENTIALS
```

## 🎨 Customizações Comuns

### 🔧 Mudar Versões de Ferramentas
```yaml
with:
  node-version: '20'          # Node.js 20
  terraform-version: '1.6.0'  # Terraform específico
```

### ☁️ Alterar Cloud Provider
```yaml
with:
  cloud-provider: 'azure'     # Em vez de 'aws'
  deployment-type: 'webapp'   # Em vez de 'lambda'
```

### 🎯 Configurar Ambientes
```yaml
environment: 'staging'        # development, staging, production
health-check-url: 'https://staging-app.com/health'
```

### 🔄 Estratégias de Deploy
```yaml
# Deploy simples
uses: .../multi-env-deploy.yml@main

# Blue-green deployment
uses: .../blue-green-deploy.yml@main
```

## 📚 Próximos Passos

1. 📖 **Escolha um template** baseado em sua tech stack
2. 📋 **Leia o README** específico do template
3. 🔧 **Configure secrets** necessários
4. 🎨 **Customize** conforme necessário
5. 🚀 **Deploy** e monitore o pipeline

## 💡 Dicas Pro

- 🏷️ **Use tags específicas** em produção: `@v1.0.0` em vez de `@main`
- 📁 **Organize por ambiente**: Separe configurações por pasta
- 🔒 **Proteja branches**: Configure branch protection rules
- 📊 **Monitor pipelines**: Use GitHub Insights para métricas
- 🚨 **Configure alertas**: Notificações no Slack/Teams para falhas

## 🤝 Contribuindo

Quer adicionar um novo template? 

1. 📁 Crie uma nova pasta com nome descritivo
2. 📄 Adicione o arquivo de workflow
3. 📚 Crie um README explicativo
4. 🧪 Teste em um projeto real
5. 📝 Envie um Pull Request

Templates úteis que gostaríamos de ver:
- 🐍 Python/Django application
- ☕ Java/Spring Boot application  
- 🔷 .NET Core application
- 🚀 Serverless framework
- 🎛️ Kubernetes deployments
- 📱 React/Next.js application

---

💬 **Precisa de ajuda?** Abra uma [issue](../../issues) ou consulte a [documentação completa](../docs/)!