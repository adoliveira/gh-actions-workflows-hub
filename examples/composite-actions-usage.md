# Exemplo: Uso de Ações Compostas

Este exemplo mostra como usar as ações compostas diretamente em seus workflows.

## Exemplo 1: Checkout com Cache

```yaml
name: Build

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout with npm cache
        uses: adoliveira/gh-actions-workflows-hub/actions/checkout-with-cache@v1
        with:
          cache: 'npm'
          cache-dependency-path: 'package-lock.json'
      
      - name: Setup Node.js
        uses: adoliveira/gh-actions-workflows-hub/actions/setup-node-env@v1
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Build
        run: npm run build
```

## Exemplo 2: Build e Push Docker

```yaml
name: Docker

on:
  push:
    branches: [ main ]

jobs:
  docker:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Build and push
        uses: adoliveira/gh-actions-workflows-hub/actions/docker-build-push@v1
        with:
          registry: 'ghcr.io'
          image-name: ${{ github.repository }}
          tags: 'latest,${{ github.sha }}'
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          platforms: 'linux/amd64,linux/arm64'
```

## Exemplo 3: Setup Python com Poetry

```yaml
name: Test

on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Python with Poetry
        uses: adoliveira/gh-actions-workflows-hub/actions/setup-python-env@v1
        with:
          python-version: '3.11'
          cache: 'poetry'
      
      - name: Run tests
        run: poetry run pytest
```

## Exemplo 4: Semantic Release

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
          branches: 'main'
```
