# Exemplo: Matrix Strategy

Este exemplo demonstra o uso de matrix strategy com os workflows reutilizáveis.

## Teste em múltiplas versões do Node.js

```yaml
name: Test Multiple Versions

on: [push, pull_request]

jobs:
  test:
    strategy:
      matrix:
        node-version: [18, 20, 21]
        os: [ubuntu-latest, windows-latest, macos-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: adoliveira/gh-actions-workflows-hub/actions/setup-node-env@v1
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      
      - name: Run tests
        run: npm test
```

## Teste em múltiplas versões do Python

```yaml
name: Test Multiple Versions

on: [push, pull_request]

jobs:
  test:
    strategy:
      matrix:
        python-version: ['3.9', '3.10', '3.11', '3.12']
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Python
        uses: adoliveira/gh-actions-workflows-hub/actions/setup-python-env@v1
        with:
          python-version: ${{ matrix.python-version }}
          cache: 'pip'
      
      - name: Run tests
        run: pytest
```

## Build Docker para múltiplas plataformas

```yaml
name: Multi-Platform Docker Build

on:
  push:
    branches: [ main ]

jobs:
  docker:
    runs-on: ubuntu-latest
    permissions:
      packages: write
    strategy:
      matrix:
        platform:
          - linux/amd64
          - linux/arm64
          - linux/arm/v7
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Build and push
        uses: adoliveira/gh-actions-workflows-hub/actions/docker-build-push@v1
        with:
          registry: 'ghcr.io'
          image-name: ${{ github.repository }}
          tags: 'latest-${{ matrix.platform }}'
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          platforms: ${{ matrix.platform }}
```
