# Exemplo: CI/CD completo para Node.js

Este exemplo demonstra um workflow CI/CD completo para um projeto Node.js.

## Arquivo: `.github/workflows/ci-cd.yml`

```yaml
name: CI/CD

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
      cache: 'npm'
      lint-command: 'npm run lint'
      format-check-command: 'npm run format:check'
  
  test:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/node-test.yml@v1
    with:
      node-version: '20'
      cache: 'npm'
      test-command: 'npm test'
      coverage-command: 'npm run coverage'
      enable-coverage: true
  
  build:
    needs: [lint, test]
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/node-build.yml@v1
    with:
      node-version: '20'
      cache: 'npm'
      build-command: 'npm run build'
      artifact-name: 'production-build'
      artifact-path: 'dist'
  
  release:
    needs: [build]
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/semantic-release.yml@v1
    secrets:
      github-token: ${{ secrets.GITHUB_TOKEN }}
    with:
      branches: 'main'
  
  deploy:
    needs: [release]
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/deploy-github-pages.yml@v1
    with:
      node-version: '20'
      build-command: 'npm run build'
      build-path: 'dist'
```

## Estrutura do projeto

```
my-project/
├── .github/
│   └── workflows/
│       └── ci-cd.yml
├── src/
│   └── index.js
├── dist/
├── package.json
└── README.md
```

## package.json scripts

```json
{
  "scripts": {
    "lint": "eslint src",
    "format:check": "prettier --check src",
    "test": "jest",
    "coverage": "jest --coverage",
    "build": "webpack --mode production"
  }
}
```

## Requisitos

- Node.js 20
- npm ou yarn
- Scripts de lint, test e build configurados no package.json
