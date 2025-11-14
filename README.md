# 🚀 GitHub Actions Workflows Hub

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GitHub release](https://img.shields.io/github/release/adoliveira/gh-actions-workflows-hub.svg)](https://github.com/adoliveira/gh-actions-workflows-hub/releases)
[![Maintenance](https://img.shields.io/badge/Maintained%3F-yes-green.svg)](https://github.com/adoliveira/gh-actions-workflows-hub/graphs/commit-activity)

> **Hub centralizado de workflows e actions reutilizáveis do GitHub Actions para CI/CD padronizado e escalável**

Acelere seus pipelines com workflows testados e otimizados que suportam múltiplas linguagens, clouds (AWS/Azure) e estratégias de deployment, incluindo Infrastructure as Code com Terraform, AWS CDK e Azure Bicep.

## 🌟 Características Principais

### 🔧 Composite Actions (5)
- **setup-node-app**: Setup Node.js com cache inteligente
- **build-test-artifact**: Build e teste com geração de artifacts
- **security-scan**: Análise completa de segurança (SAST/DAST)
- **docker-build-push**: Build Docker multi-arquitetura
- **terraform-setup**: Setup Terraform multi-cloud

### 🔄 Reusable Workflows (7)
- **ci-reusable**: Pipeline CI genérico e flexível
- **docker-build**: Build Docker avançado com cache
- **terraform-cicd**: Terraform CI/CD completo
- **multi-env-deploy**: Deploy para múltiplos ambientes
- **blue-green-deploy**: Deploy blue-green automático
- **destroy-infrastructure**: Destruição segura de recursos
- **security-audit**: Auditoria completa de segurança

### � Templates (3)
- **basic-ci-cd**: Template básico para projetos simples
- **terraform-infrastructure**: Template para projetos de IaC
- **microservices-deployment**: Template para microsserviços

### 🏗️ Exemplos Práticos (10)

#### Infrastructure Examples (5)
- **AWS SQS**: Message queuing com Terraform
- **AWS SNS**: Notification service com multi-env
- **AWS DynamoDB**: NoSQL database com backup
- **Azure Service Bus**: Messaging enterprise
- **Azure VNet**: Network infrastructure

#### Application Examples (5)  
- **Simple Node.js**: API Express com Lambda deploy
- **Microservices**: Arquitetura distribuída multi-cloud
- **Serverless AWS**: Lambda + API Gateway + DynamoDB
- **Container WebApp**: React + Express containerizado
- **Full-Stack MonoRepo**: Enterprise-grade com Next.js + Node.js + Python

## 🚀 Início Rápido

### 1. Use um workflow reutilizável

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main]

jobs:
  ci-cd:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/ci-reusable.yml@main
    with:
      node-version: '18'
      test-command: 'npm test'
      build-command: 'npm run build'
```

### 2. Deploy multi-ambiente

```yaml
jobs:
  deploy:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/multi-env-deploy.yml@main
    with:
      environment: 'production'
      cloud-provider: 'aws'
      deployment-type: 'lambda'
      app-name: 'my-app'
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### 3. Infrastructure as Code

```yaml
jobs:
  terraform:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/terraform-cicd.yml@main
    with:
      terraform-directory: './infrastructure'
      environment: 'production'
      auto-approve: false
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_TERRAFORM_ROLE_ARN }}
```

## 📁 Estrutura do Projeto

```
gh-actions-workflows-hub/
├── .github/
│   ├── actions/                 # 🔧 5 Composite Actions
│   │   ├── setup-node-app/      # Setup Node.js padronizado
│   │   ├── build-test-artifact/ # Build e testes com artifacts
│   │   ├── security-scan/       # Análise de segurança
│   │   ├── docker-build-push/   # Build Docker multi-arch
│   │   └── terraform-setup/     # Setup Terraform
│   └── workflows/               # 🔄 7 Reusable Workflows
│       ├── ci-reusable.yml      # CI genérico reutilizável
│       ├── docker-build.yml     # Build Docker avançado
│       ├── terraform-cicd.yml   # Terraform CI/CD completo
│       ├── multi-env-deploy.yml # Deploy multi-ambiente
│       ├── blue-green-deploy.yml # Deploy blue-green
│       ├── destroy-infrastructure.yml # Destruição segura
│       └── security-audit.yml   # Auditoria de segurança
├── templates/                   # 📋 3 Template Workflows
│   ├── basic-ci-cd.yml         # Template básico CI/CD
│   ├── terraform-infrastructure.yml # Template Terraform
│   └── microservices-deployment.yml # Template microsserviços
├── examples/                    # 🏗️ 10 Exemplos Práticos
│   ├── infrastructure/         # 5 Exemplos de Infraestrutura
│   │   ├── aws-sqs-terraform/   # AWS SQS com Terraform
│   │   ├── aws-sns-terraform/   # AWS SNS com Terraform
│   │   ├── aws-dynamodb-terraform/ # AWS DynamoDB
│   │   ├── azure-servicebus-terraform/ # Azure Service Bus
│   │   └── azure-vnet-terraform/ # Azure Virtual Network
│   └── applications/           # 5 Exemplos de Aplicação
│       ├── simple-node-app/     # App Node.js simples
│       ├── microservices/       # Arquitetura microsserviços
│       ├── serverless-aws/      # Aplicação serverless
│       ├── container-webapp/    # WebApp containerizada
│       └── full-stack-app/      # Monorepo full-stack
├── docs/                       # 📚 Documentação
│   ├── getting-started.md      # Guia inicial
│   ├── best-practices.md       # Melhores práticas
│   ├── troubleshooting.md      # Solução de problemas
│   └── architecture.md         # Arquitetura do hub
├── HUB-SUMMARY.md             # 📊 Resumo executivo completo
└── README.md                  # 📖 Documentação principal
```

## 🛠️ Workflows Disponíveis

| Workflow | Descrição | Use Cases |
|----------|-----------|-----------|
| **ci-reusable** | Pipeline CI genérico | Qualquer linguagem/framework |
| **docker-build** | Build Docker multi-arch | Apps containerizadas |
| **terraform-cicd** | Terraform completo | Infrastructure as Code |
| **aws-cdk-deploy** | Deploy AWS CDK | IaC para AWS |
| **azure-bicep-deploy** | Deploy Azure Bicep | IaC para Azure |
| **multi-env-deploy** | Deploy multi-ambiente | Apps web, APIs |
| **blue-green-deploy** | Deploy blue-green | Zero-downtime deploys |

## 🏗️ Tecnologias Suportadas

### ☁️ Cloud Providers
- **AWS**: Lambda, ECS, S3, CloudFormation, CDK
- **Azure**: Web Apps, Functions, Container Instances, Bicep
- **GCP**: Em desenvolvimento

### 🔤 Linguagens
- **Node.js** (npm, yarn)
- **Python** (pip, poetry)
- **Java** (Maven, Gradle)
- **Go**, **.NET** e mais

### 🏗️ Infrastructure as Code
- **Terraform** (validate, plan, apply)
- **AWS CDK** (TypeScript, Python)
- **Azure Bicep**
- **Pulumi** (roadmap)

## 📚 Documentação

- 📖 [**Visão Geral**](docs/overview.md) - Entenda o projeto
- ⚡ [**Início Rápido**](docs/quickstart.md) - Configure em 5 minutos
- 🔧 [**Configuração**](docs/setup.md) - Setup para organizações
- 📋 [**Referência de Workflows**](docs/workflows-reference.md) - Todos os workflows
- 🔧 [**Referência de Actions**](docs/actions-reference.md) - Todas as actions
- 🏗️ [**Guia Terraform**](docs/terraform-guide.md) - IaC com Terraform
- 🔒 [**Boas Práticas de Segurança**](docs/security-best-practices.md) - Segurança
- 🛠️ [**Troubleshooting**](docs/troubleshooting.md) - Solução de problemas

## 🎯 Exemplos

### Node.js com AWS Lambda
```yaml
uses: adoliveira/gh-actions-workflows-hub/.github/workflows/ci-reusable.yml@main
```

### Terraform Multi-Ambiente
```yaml
uses: adoliveira/gh-actions-workflows-hub/.github/workflows/terraform-cicd.yml@main
```

### Docker + Azure
```yaml
uses: adoliveira/gh-actions-workflows-hub/.github/workflows/docker-build.yml@main
```

Veja mais exemplos em [`/examples`](examples/).

## 🔧 Configuração

### Secrets Necessários

```bash
# AWS
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY  
AWS_ROLE_ARN

# Azure
AZURE_CREDENTIALS

# Docker (opcional)
DOCKERHUB_USERNAME
DOCKERHUB_TOKEN
```

### Variables Recomendadas

```bash
AWS_REGION=us-east-1
AZURE_SUBSCRIPTION_ID=your-subscription-id
AZURE_RESOURCE_GROUP=your-rg
```

## 🤝 Contribuindo

Contribuições são bem-vindas! Veja o [guia de contribuição](docs/contributing.md).

1. Fork o projeto
2. Crie sua feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit suas mudanças (`git commit -m 'Add some AmazingFeature'`)
4. Push para a branch (`git push origin feature/AmazingFeature`)
5. Abra um Pull Request

## 📊 Stats

- 🔧 **5+ Actions** compostas
- 🔄 **7+ Workflows** reutilizáveis  
- ☁️ **2 Cloud** providers suportados
- 🏗️ **3 IaC** ferramentas integradas
- 📚 **Documentação** completa
- ✅ **Templates** prontos para uso

## 📄 Licença

Distribuído sob a licença MIT. Veja [`LICENSE`](LICENSE) para mais informações.

## 📞 Suporte

- 📖 [Documentação](docs/)
- 🐛 [Reportar Bugs](issues/new?template=bug_report.md)
- 💡 [Sugerir Features](issues/new?template=feature_request.md)
- 💬 [Discussões](discussions)

## 🔗 Links Úteis

- [🎯 Roadmap](docs/roadmap.md)
- [📈 Changelog](CHANGELOG.md)
- [🏷️ Releases](releases)
- [📊 Insights](pulse)

---

⭐ **Se este projeto te ajudou, considere dar uma estrela!** ⭐
