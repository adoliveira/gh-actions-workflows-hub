# Ações Compostas

Este diretório contém ações compostas reutilizáveis que podem ser usadas em seus workflows.

## Lista de Ações

### 1. Checkout with Cache
**Caminho:** `actions/checkout-with-cache`

Realiza checkout do repositório com suporte a cache de dependências.

**Principais casos de uso:**
- Checkout com cache npm/yarn/pnpm
- Checkout com cache pip/poetry
- Checkout com cache maven/gradle

[Documentação completa](./checkout-with-cache/README.md)

### 2. Setup Node.js Environment
**Caminho:** `actions/setup-node-env`

Configura ambiente Node.js com cache e instalação de dependências.

**Principais casos de uso:**
- Setup Node.js 18, 20 ou 21
- Suporte para npm, yarn e pnpm
- Instalação automática de dependências

[Documentação completa](./setup-node-env/README.md)

### 3. Setup Python Environment
**Caminho:** `actions/setup-python-env`

Configura ambiente Python com cache e instalação de dependências.

**Principais casos de uso:**
- Setup Python 3.9, 3.10, 3.11 ou 3.12
- Suporte para pip, pipenv e poetry
- Instalação automática de dependências

[Documentação completa](./setup-python-env/README.md)

### 4. Docker Build and Push
**Caminho:** `actions/docker-build-push`

Build e push de imagens Docker com suporte multi-plataforma.

**Principais casos de uso:**
- Build para múltiplas plataformas (amd64, arm64)
- Push para GitHub Container Registry
- Push para Docker Hub
- Cache otimizado

[Documentação completa](./docker-build-push/README.md)

### 5. Semantic Release
**Caminho:** `actions/semantic-release`

Versionamento semântico automatizado e gerenciamento de releases.

**Principais casos de uso:**
- Releases automatizados baseados em commits
- Geração de CHANGELOG
- Criação de tags e releases no GitHub
- Suporte a Conventional Commits

[Documentação completa](./semantic-release/README.md)

## Uso Básico

```yaml
steps:
  - name: Use uma ação composta
    uses: adoliveira/gh-actions-workflows-hub/actions/<action-name>@v1
    with:
      input1: 'value1'
      input2: 'value2'
```

## Versionamento

Use tags de versão semântica para referenciar ações:

- `@v1` - Última versão major 1
- `@v1.2` - Última versão minor 1.2
- `@v1.2.3` - Versão exata 1.2.3
- `@main` - Última versão (não recomendado para produção)

## Exemplos

Veja exemplos completos na pasta [examples](../examples/).
