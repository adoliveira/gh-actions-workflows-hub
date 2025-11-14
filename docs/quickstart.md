# ⚡ Guia de Início Rápido

Configure e use workflows reutilizáveis em **menos de 5 minutos**!

## 🎯 Cenário: Deploy de uma App Node.js

Vamos configurar um pipeline completo para uma aplicação Node.js com deploy na AWS.

### 📋 Pré-requisitos (2 minutos)

1. **Repositório com aplicação Node.js**
2. **Secrets configurados no GitHub** (vai em Settings > Secrets and variables > Actions):
   ```
   AWS_ACCESS_KEY_ID=your-access-key
   AWS_SECRET_ACCESS_KEY=your-secret-key
   ```
3. **Variables configuradas** (Settings > Secrets and variables > Actions > Variables):
   ```
   AWS_REGION=us-east-1
   ```

### 🔧 Passo 1: Criar o Workflow (1 minuto)

Crie `.github/workflows/ci-cd.yml` no seu repositório:

```yaml
name: 'CI/CD Pipeline'

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  APP_NAME: 'my-awesome-app'

jobs:
  # Pipeline de CI
  ci:
    name: 'CI Pipeline'
    uses: YOUR-ORG/gh-actions-workflows-hub/.github/workflows/ci-reusable.yml@main
    with:
      node-version: '18'
      test-command: 'npm test'
      lint-command: 'npm run lint'
      build-command: 'npm run build'

  # Build Docker
  docker:
    name: 'Docker Build'
    needs: ci
    if: github.ref == 'refs/heads/main'
    uses: YOUR-ORG/gh-actions-workflows-hub/.github/workflows/docker-build.yml@main
    with:
      image-name: ${{ env.APP_NAME }}
      registry: 'ghcr'

  # Deploy
  deploy:
    name: 'Deploy'
    needs: docker
    if: github.ref == 'refs/heads/main'
    uses: YOUR-ORG/gh-actions-workflows-hub/.github/workflows/multi-env-deploy.yml@main
    with:
      environment: 'production'
      cloud-provider: 'aws'
      deployment-type: 'lambda'
      app-name: ${{ env.APP_NAME }}
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### 🎯 Passo 2: Ajustar Referência (30 segundos)

Substitua `YOUR-ORG` pela organização onde você fez fork ou cloneu este hub:

```yaml
# Se for sua organização pessoal:
uses: usuario/gh-actions-workflows-hub/.github/workflows/ci-reusable.yml@main

# Se for da empresa:
uses: empresa/gh-actions-workflows-hub/.github/workflows/ci-reusable.yml@main
```

### 🚀 Passo 3: Push e Observe (30 segundos)

```bash
git add .github/workflows/ci-cd.yml
git commit -m "Add CI/CD pipeline"
git push origin main
```

Vá para a aba **Actions** no GitHub e observe seu pipeline executando! 🎉

## 🏗️ Resultado

Seu pipeline agora:
- ✅ Executa testes automaticamente
- ✅ Faz lint do código
- ✅ Gera build da aplicação
- ✅ Constrói imagem Docker
- ✅ Faz deploy automático para AWS
- ✅ Notifica sobre status

## 🎨 Customizações Rápidas

### 🔧 Mudar Comandos
```yaml
with:
  test-command: 'npm run test:coverage'
  lint-command: 'npm run lint:fix'
  build-command: 'npm run build:prod'
```

### ☁️ Mudar Cloud Provider
```yaml
with:
  cloud-provider: 'azure'  # ao invés de 'aws'
  deployment-type: 'webapp'
```

### 🐳 Deploy para Container
```yaml
with:
  deployment-type: 'container'
  # Em vez de 'lambda'
```

## 📚 Mais Exemplos Rápidos

### 🎯 Terraform + AWS
```yaml
jobs:
  terraform:
    uses: YOUR-ORG/gh-actions-workflows-hub/.github/workflows/terraform-cicd.yml@main
    with:
      terraform-directory: './infrastructure'
      environment: 'production'
      auto-approve: false  # Requer aprovação manual
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_TERRAFORM_ROLE_ARN }}
```

### ☁️ Azure Bicep
```yaml
jobs:
  bicep:
    uses: YOUR-ORG/gh-actions-workflows-hub/.github/workflows/azure-bicep-deploy.yml@main
    with:
      bicep-directory: './bicep'
      resource-group: 'rg-myapp-prod'
      environment: 'production'
    secrets:
      AZURE_CREDENTIALS: ${{ secrets.AZURE_CREDENTIALS }}
```

### 🔄 Blue-Green Deployment
```yaml
jobs:
  blue-green:
    uses: YOUR-ORG/gh-actions-workflows-hub/.github/workflows/blue-green-deploy.yml@main
    with:
      environment: 'production'
      cloud-provider: 'aws'
      app-name: 'my-app'
      health-check-url: 'https://my-app.com/health'
      load-balancer: ${{ vars.PROD_ALB_ARN }}
```

## 🚨 Troubleshooting Rápido

### ❌ "Workflow not found"
- Verifique se você substituiu `YOUR-ORG` corretamente
- Confirme que o repositório do hub está acessível
- Use `@main` ou uma tag específica como `@v1.0.0`

### ❌ "Permission denied"
- Configure os secrets necessários
- Verifique permissões do GitHub Actions
- Para organizações: verifique policies de workflows externos

### ❌ "Build failed"
- Verifique se os comandos existem no seu `package.json`:
  ```json
  {
    "scripts": {
      "test": "jest",
      "lint": "eslint .",
      "build": "webpack --mode=production"
    }
  }
  ```

## 📖 Próximos Passos

Agora que você tem um pipeline básico funcionando:

1. 📖 **Explore**: Veja outros [workflows disponíveis](workflows-reference.md)
2. 🎨 **Customize**: Adapte os [parâmetros](parameters.md) às suas necessidades
3. 🔒 **Proteja**: Configure [environments protegidos](environments.md)
4. 📊 **Monitore**: Setup de [monitoramento](monitoring.md)
5. 🏗️ **Infrastructure**: Adicione [Terraform workflows](terraform-guide.md)

## 💡 Dicas Pro

- 🏷️ **Use tags**: `@v1.0.0` em vez de `@main` para produção
- 🔄 **Cache dependencies**: Os workflows já incluem cache otimizado
- 📝 **Matrix builds**: Para testar múltiplas versões de Node.js
- 🎯 **Path filtering**: Execute workflows apenas quando arquivos relevantes mudarem

```yaml
on:
  push:
    paths: ['src/**', 'package.json']  # Só executa se estes arquivos mudarem
```

---

🎉 **Parabéns!** Você agora tem um pipeline profissional configurado em minutos. Explore as [documentações avançadas](README.md) para descobrir todo o potencial!