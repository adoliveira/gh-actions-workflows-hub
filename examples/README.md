# Exemplos de Uso dos Workflows

Este diretório contém exemplos práticos de como usar os workflows e actions deste repositório.

## 📁 Estrutura dos Exemplos

```
examples/
├── infrastructure/           # Exemplos de Infrastructure as Code
│   ├── aws-sqs-terraform/        # AWS SQS com Terraform
│   ├── aws-sns-terraform/        # AWS SNS com Terraform
│   ├── aws-dynamodb-terraform/   # AWS DynamoDB com Terraform
│   ├── azure-servicebus-terraform/ # Azure Service Bus com Terraform
│   └── azure-vnet-terraform/     # Azure Virtual Network com Terraform
├── applications/             # Exemplos de Aplicações
│   ├── simple-node-app/          # Aplicação Node.js simples
│   ├── microservices/            # Arquitetura de microsserviços
│   ├── serverless-aws/           # Aplicação serverless na AWS
│   ├── container-webapp/         # Aplicação web containerizada
│   └── full-stack-app/           # Aplicação full-stack completa
└── README.md
```

## 🚀 Exemplos Disponíveis

### Infrastructure as Code (IaC)

#### AWS Infrastructure Examples
1. **📮 AWS SQS** - `infrastructure/aws-sqs-terraform/`
   - Filas Standard e FIFO com Dead Letter Queue
   - KMS encryption e CloudWatch monitoring
   - Auto-scaling e multiple environments

2. **📢 AWS SNS** - `infrastructure/aws-sns-terraform/`
   - Topics com múltiplas subscriptions (Email, SMS, SQS, Lambda)
   - Cross-account access policies e message filtering
   - CloudWatch alarms e delivery policies

3. **🗄️ AWS DynamoDB** - `infrastructure/aws-dynamodb-terraform/`
   - Tables com GSI/LSI e auto-scaling
   - Point-in-time recovery e backup automation
   - Encryption at rest e comprehensive monitoring

#### Azure Infrastructure Examples
4. **🚌 Azure Service Bus** - `infrastructure/azure-servicebus-terraform/`
   - Namespace com queues, topics e subscriptions
   - Message filtering e diferentes SKUs (Basic/Standard/Premium)
   - Application Insights monitoring

5. **🌐 Azure Virtual Network** - `infrastructure/azure-vnet-terraform/`
   - Multi-tier architecture (web/app/data subnets)
   - NAT Gateway, Application Gateway e NSG rules
   - DDoS protection e network flow logs

### Application Examples

#### 1. Aplicação Node.js Simples
- **Localização**: `applications/simple-node-app/`
- **Descrição**: Pipeline CI/CD básico para uma aplicação Node.js
- **Inclui**: Testes, linting, build, deploy para staging/prod
- **Cloud**: AWS (Lambda + API Gateway)

#### 2. Arquitetura de Microsserviços
- **Localização**: `applications/microservices/`
- **Descrição**: Deployment coordenado de múltiplos serviços
- **Inclui**: Matrix builds, service discovery, health checks
- **Cloud**: AWS (ECS) e Azure (Container Instances)

#### 3. Aplicação Serverless
- **Localização**: `applications/serverless-aws/`
- **Descrição**: Deploy de funções Lambda com infrastructure as code
- **Inclui**: Terraform + AWS CDK, multiple environments
- **Cloud**: AWS (Lambda, DynamoDB, API Gateway)

#### 4. Aplicação Web Containerizada
- **Localização**: `applications/container-webapp/`
- **Descrição**: Build e deploy de aplicação containerizada
- **Inclui**: Multi-arch build, security scanning, blue-green deployment
- **Cloud**: AWS (ECS) e Azure (Container Instances)

#### 5. Aplicação Full-Stack Completa
- **Localização**: `applications/full-stack-app/`
- **Descrição**: Pipeline completo com frontend, backend e banco de dados
- **Inclui**: Monorepo, dependency management, E2E testing
- **Cloud**: Multi-cloud (AWS + Azure)

## 📋 Pré-requisitos

Para usar estes exemplos, você precisará:

1. **Secrets configurados no GitHub:**
   ```
   # AWS Credentials
   AWS_ACCESS_KEY_ID
   AWS_SECRET_ACCESS_KEY
   AWS_ROLE_ARN
   
   # Azure Credentials
   ARM_CLIENT_ID
   ARM_CLIENT_SECRET
   ARM_TENANT_ID
   ARM_SUBSCRIPTION_ID
   AZURE_CREDENTIALS
   
   # Docker Registry (opcional)
   DOCKERHUB_USERNAME
   DOCKERHUB_TOKEN
   GHCR_TOKEN
   ```

2. **Variables configuradas no GitHub:**
   ```
   # AWS Configuration
   AWS_REGION
   AWS_ACCOUNT_ID
   
   # Azure Configuration
   AZURE_SUBSCRIPTION_ID
   AZURE_RESOURCE_GROUP
   AZURE_LOCATION
   
   # Application Configuration
   APP_NAME
   ENVIRONMENT
   ```

3. **Permissões necessárias:**
   - AWS: IAM roles com permissões adequadas
   - Azure: Service Principal com contribuidor
   - GitHub: Permissions para Actions e Packages

## 🔧 Como Usar

1. **Copie o exemplo desejado** para seu repositório
2. **Atualize as referências** para apontar para este hub:
   ```yaml
   uses: SEU-ORG/gh-actions-workflows-hub/.github/workflows/WORKFLOW.yml@main
   ```
3. **Configure os secrets e variables** necessários
4. **Adapte os parâmetros** para seu ambiente específico
5. **Teste em uma branch** antes de fazer merge para main

## 📚 Documentação Adicional

- [Configuração de Secrets](../docs/setup-secrets.md)
- [Guia de Boas Práticas](../docs/best-practices.md)
- [Troubleshooting](../docs/troubleshooting.md)
- [Customização](../docs/customization.md)

## 🤝 Contribuições

Quer adicionar um novo exemplo? Veja nosso [guia de contribuição](../docs/contributing.md).

## 📞 Suporte

- Abra uma [issue](../../issues) para reportar problemas
- Consulte a [documentação completa](../docs/)
- Entre em contato com a equipe de DevOps