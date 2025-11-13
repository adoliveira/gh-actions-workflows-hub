# Workflows Reutilizáveis

Este diretório contém workflows reutilizáveis que podem ser chamados via `workflow_call`.

## Lista de Workflows

### Node.js

#### node-build.yml
Build de projetos Node.js com suporte a cache e upload de artefatos.

**Inputs:**
- `node-version`: Versão do Node.js (padrão: '20')
- `cache`: Gerenciador de pacotes (npm, yarn, pnpm)
- `build-command`: Comando de build (padrão: 'npm run build')
- `artifact-name`: Nome do artefato
- `artifact-path`: Caminho do artefato

**Exemplo:**
```yaml
jobs:
  build:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/node-build.yml@v1
    with:
      node-version: '20'
      cache: 'npm'
```

#### node-test.yml
Execução de testes Node.js com suporte a coverage.

**Inputs:**
- `node-version`: Versão do Node.js (padrão: '20')
- `cache`: Gerenciador de pacotes
- `test-command`: Comando de teste
- `enable-coverage`: Habilitar coverage (padrão: false)

#### node-lint.yml
Linting de projetos Node.js.

**Inputs:**
- `node-version`: Versão do Node.js (padrão: '20')
- `lint-command`: Comando de lint
- `format-check-command`: Comando de verificação de formatação

### Python

#### python-build.yml
Build de pacotes Python.

**Inputs:**
- `python-version`: Versão do Python (padrão: '3.11')
- `cache`: Gerenciador de pacotes (pip, pipenv, poetry)
- `build-command`: Comando de build

#### python-test.yml
Execução de testes Python com suporte a coverage.

**Inputs:**
- `python-version`: Versão do Python (padrão: '3.11')
- `test-command`: Comando de teste
- `enable-coverage`: Habilitar coverage

#### python-lint.yml
Linting de projetos Python.

**Inputs:**
- `python-version`: Versão do Python (padrão: '3.11')
- `lint-command`: Comando de lint
- `format-check-command`: Comando de verificação de formatação

### Docker

#### docker-build-push.yml
Build e push de imagens Docker.

**Inputs:**
- `registry`: Registro de container (padrão: 'ghcr.io')
- `image-name`: Nome da imagem (obrigatório)
- `tags`: Tags da imagem
- `platforms`: Plataformas alvo

**Secrets:**
- `registry-username`: Nome de usuário do registro
- `registry-password`: Senha/token do registro

### Release & Deploy

#### semantic-release.yml
Release automatizado com versionamento semântico.

**Inputs:**
- `dry-run`: Modo dry-run (padrão: false)
- `branches`: Branches para release (padrão: 'main')

**Secrets:**
- `github-token`: Token GitHub (obrigatório)

#### deploy-github-pages.yml
Deploy para GitHub Pages.

**Inputs:**
- `node-version`: Versão do Node.js
- `build-command`: Comando de build
- `build-path`: Caminho do build

## Uso Básico

```yaml
jobs:
  job-name:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/<workflow-name>.yml@v1
    with:
      input1: 'value1'
    secrets:
      secret1: ${{ secrets.SECRET1 }}
```

## Versionamento

Use tags de versão semântica:

- `@v1` - Última versão major 1
- `@v1.2` - Última versão minor 1.2
- `@v1.2.3` - Versão exata 1.2.3

## Exemplos

Veja exemplos completos na pasta [examples](../examples/).
