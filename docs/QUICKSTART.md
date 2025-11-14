# Guia de Início Rápido

Este guia irá ajudá-lo a começar a usar o gh-actions-workflows-hub em seus projetos.

## 1. Escolha seu Método

Você pode usar este repositório de três formas:

### A. Workflows Reutilizáveis (Recomendado)
Use workflows completos via `workflow_call` - mais simples e padronizado.

### B. Ações Compostas
Use ações específicas em seus próprios workflows - mais flexível.

### C. Templates
Copie templates prontos para seu repositório - mais rápido para começar.

## 2. Workflows Reutilizáveis

### Para Projetos Node.js

Crie `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  lint:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/node-lint.yml@v1
    with:
      node-version: '20'
      lint-command: 'npm run lint'
  
  test:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/node-test.yml@v1
    with:
      node-version: '20'
      test-command: 'npm test'
  
  build:
    needs: [lint, test]
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/node-build.yml@v1
    with:
      node-version: '20'
      build-command: 'npm run build'
```

**Requisitos:**
- `package.json` com scripts `lint`, `test`, `build`
- Node.js configurado no projeto

### Para Projetos Python

Crie `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  lint:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/python-lint.yml@v1
    with:
      python-version: '3.11'
      lint-command: 'flake8 .'
  
  test:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/python-test.yml@v1
    with:
      python-version: '3.11'
      test-command: 'pytest'
  
  build:
    needs: [lint, test]
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/python-build.yml@v1
    with:
      python-version: '3.11'
```

**Requisitos:**
- `requirements.txt` com dependências
- `setup.py` ou `pyproject.toml` configurado

## 3. Ações Compostas

### Setup Node.js Completo

```yaml
name: Build

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: adoliveira/gh-actions-workflows-hub/actions/setup-node-env@v1
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Build
        run: npm run build
```

### Docker Build e Push

```yaml
name: Docker

on:
  push:
    branches: [ main ]

jobs:
  docker:
    runs-on: ubuntu-latest
    permissions:
      packages: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Build and Push
        uses: adoliveira/gh-actions-workflows-hub/actions/docker-build-push@v1
        with:
          registry: 'ghcr.io'
          image-name: ${{ github.repository }}
          tags: 'latest'
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
```

### Release Automatizado

```yaml
name: Release

on:
  push:
    branches: [ main ]

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      issues: write
      pull-requests: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Semantic Release
        uses: adoliveira/gh-actions-workflows-hub/actions/semantic-release@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

## 4. Templates

Para usar templates:

1. Vá para seu repositório no GitHub
2. Clique em "Actions" > "New workflow"
3. Procure por "adoliveira" ou o nome do template
4. Clique em "Configure" no template desejado
5. Customize conforme necessário
6. Commit o arquivo

Templates disponíveis:
- **Node.js CI** - CI completo para Node.js
- **Python CI** - CI completo para Python
- **Docker Build and Push** - Build Docker
- **Release** - Release automatizado

## 5. Configuração de Secrets

Alguns workflows precisam de secrets:

### GitHub Token (já disponível)
```yaml
secrets:
  github-token: ${{ secrets.GITHUB_TOKEN }}
```

### Container Registry
```yaml
secrets:
  registry-username: ${{ github.actor }}
  registry-password: ${{ secrets.GITHUB_TOKEN }}
```

### NPM Token (para publicar)
1. Crie token em npmjs.com
2. Adicione em Settings > Secrets > New repository secret
3. Nome: `NPM_TOKEN`

## 6. Versionamento

Use tags de versão para estabilidade:

```yaml
# Produção - versão major (recebe updates compatíveis)
@v1

# Específico - versão exata (não muda)
@v1.2.3

# Desenvolvimento - branch main (pode quebrar)
@main
```

## 7. Próximos Passos

- 📚 [Ver exemplos completos](../examples/)
- 📖 [Ler documentação](../docs/)
- 🔧 [Customizar workflows](../docs/reusable-workflows.md)
- 🤝 [Contribuir](../docs/CONTRIBUTING.md)

## 8. Troubleshooting

### Erro: "workflow is not accessible"
- Certifique-se de usar `@v1` ou uma tag válida
- Verifique se o repositório é público

### Erro: "invalid workflow"
- Valide sintaxe YAML
- Verifique indentação (use 2 espaços)

### Erro: "required input not provided"
- Revise inputs obrigatórios na documentação
- Adicione inputs faltantes

### Permissões insuficientes
Adicione permissões necessárias:
```yaml
permissions:
  contents: write
  packages: write
  issues: write
```

## Suporte

- 🐛 [Reportar bug](https://github.com/adoliveira/gh-actions-workflows-hub/issues/new)
- 💡 [Sugerir feature](https://github.com/adoliveira/gh-actions-workflows-hub/issues/new)
- 📖 [Documentação completa](../docs/)
