# GitHub Actions Workflows Hub 🚀

Repositório central de Actions e Workflows reutilizáveis para GitHub Actions. Inclui ações compostas e workflows modulares (CI/CD: build, test, lint, release, deploy), exemplos e documentação. Padroniza projetos, reduz duplicação e acelera pipelines via workflow_call e actions compartilhadas.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GitHub release](https://img.shields.io/github/release/adoliveira/gh-actions-workflows-hub.svg)](https://github.com/adoliveira/gh-actions-workflows-hub/releases)
[![Semantic Release](https://img.shields.io/badge/%20%20%F0%9F%93%A6%F0%9F%9A%80-semantic--release-e10079.svg)](https://github.com/semantic-release/semantic-release)

## 📋 Índice

- [Características](#-características)
- [Início Rápido](#-início-rápido)
- [Ações Compostas](#-ações-compostas)
- [Workflows Reutilizáveis](#-workflows-reutilizáveis)
- [Templates de Workflow](#-templates-de-workflow)
- [Exemplos](#-exemplos)
- [Versionamento](#-versionamento)
- [Documentação](#-documentação)
- [Contribuindo](#-contribuindo)

## ✨ Características

- 🎯 **Ações Compostas**: Componentes reutilizáveis para tarefas comuns
- 🔄 **Workflows Reutilizáveis**: Pipelines CI/CD completos via `workflow_call`
- 📦 **Templates**: Templates prontos para iniciar rapidamente
- 🏷️ **Versionamento Semântico**: Releases automatizados e versionamento consistente
- 📚 **Documentação Completa**: Guias e exemplos para todos os componentes
- 🛠️ **Multi-linguagem**: Suporte para Node.js, Python, Docker e mais
- ⚡ **Cache Inteligente**: Otimização automática de cache para builds rápidos
- 🌍 **Multi-plataforma**: Suporte para Linux, Windows e macOS

## 🚀 Início Rápido

### Usando Workflows Reutilizáveis

Adicione ao seu `.github/workflows/ci.yml`:

```yaml
name: CI

on: [push, pull_request]

jobs:
  test:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/node-test.yml@v1
    with:
      node-version: '20'
      cache: 'npm'
```

### Usando Ações Compostas

```yaml
steps:
  - name: Setup Node.js
    uses: adoliveira/gh-actions-workflows-hub/actions/setup-node-env@v1
    with:
      node-version: '20'
      cache: 'npm'
```

## 🎯 Ações Compostas

### Disponíveis

| Ação | Descrição | Documentação |
|------|-----------|--------------|
| `checkout-with-cache` | Checkout com cache de dependências | [📖 Docs](./actions/checkout-with-cache/README.md) |
| `setup-node-env` | Setup Node.js + cache + install | [📖 Docs](./actions/setup-node-env/README.md) |
| `setup-python-env` | Setup Python + cache + install | [📖 Docs](./actions/setup-python-env/README.md) |
| `docker-build-push` | Build e push Docker multi-plataforma | [📖 Docs](./actions/docker-build-push/README.md) |
| `semantic-release` | Releases automatizados | [📖 Docs](./actions/semantic-release/README.md) |

[📚 Ver todas as ações](./docs/actions.md)

## 🔄 Workflows Reutilizáveis

### Node.js

```yaml
jobs:
  lint:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/node-lint.yml@v1
  test:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/node-test.yml@v1
  build:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/node-build.yml@v1
```

### Python

```yaml
jobs:
  lint:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/python-lint.yml@v1
  test:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/python-test.yml@v1
  build:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/python-build.yml@v1
```

### Docker

```yaml
jobs:
  docker:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/docker-build-push.yml@v1
    secrets:
      registry-username: ${{ github.actor }}
      registry-password: ${{ secrets.GITHUB_TOKEN }}
    with:
      image-name: ${{ github.repository }}
```

### Release & Deploy

```yaml
jobs:
  release:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/semantic-release.yml@v1
    secrets:
      github-token: ${{ secrets.GITHUB_TOKEN }}
  
  deploy:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/deploy-github-pages.yml@v1
```

[📚 Ver todos os workflows](./docs/reusable-workflows.md)

## 📋 Templates de Workflow

Templates disponíveis em `.github/workflow-templates/`:

- `node-ci.yml` - CI completo para Node.js
- `python-ci.yml` - CI completo para Python
- `docker-build-push.yml` - Build e push Docker
- `release.yml` - Release automatizado

## 📚 Exemplos

Exemplos completos na pasta [`examples/`](./examples/):

- [Node.js CI/CD Completo](./examples/node-ci-cd-complete.md)
- [Python CI/CD Completo](./examples/python-ci-cd-complete.md)
- [Uso de Ações Compostas](./examples/composite-actions-usage.md)
- [Matrix Strategy](./examples/matrix-strategy.md)

## 🏷️ Versionamento

Este projeto usa [Semantic Versioning](https://semver.org/) e [Conventional Commits](https://www.conventionalcommits.org/).

### Uso Recomendado

```yaml
# Produção - versão exata
uses: adoliveira/gh-actions-workflows-hub/actions/setup-node-env@v1.2.3

# Desenvolvimento - versão major
uses: adoliveira/gh-actions-workflows-hub/actions/setup-node-env@v1

# Testes - branch main
uses: adoliveira/gh-actions-workflows-hub/actions/setup-node-env@main
```

[📚 Guia completo de versionamento](./docs/versioning.md)

## 📖 Documentação

- [Ações Compostas](./docs/actions.md)
- [Workflows Reutilizáveis](./docs/reusable-workflows.md)
- [Versionamento Semântico](./docs/versioning.md)
- [Guia de Contribuição](./docs/CONTRIBUTING.md)

## 🤝 Contribuindo

Contribuições são bem-vindas! Por favor:

1. Fork o repositório
2. Crie uma branch: `git checkout -b feat/minha-feature`
3. Commit suas mudanças: `git commit -m 'feat: adicionar nova feature'`
4. Push para a branch: `git push origin feat/minha-feature`
5. Abra um Pull Request

Veja o [Guia de Contribuição](./docs/CONTRIBUTING.md) para mais detalhes.

## 📄 Licença

MIT © [adoliveira](https://github.com/adoliveira)

## 🙏 Agradecimentos

Este projeto foi inspirado por:
- [GitHub Actions](https://github.com/features/actions)
- [actions/toolkit](https://github.com/actions/toolkit)
- [semantic-release](https://github.com/semantic-release/semantic-release)

---

**Feito com ❤️ para a comunidade GitHub Actions**
