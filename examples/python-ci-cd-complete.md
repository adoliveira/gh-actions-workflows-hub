# Exemplo: CI/CD completo para Python

Este exemplo demonstra um workflow CI/CD completo para um projeto Python.

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
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/python-lint.yml@v1
    with:
      python-version: '3.11'
      cache: 'pip'
      lint-command: 'flake8 src tests'
      format-check-command: 'black --check src tests'
  
  test:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/python-test.yml@v1
    with:
      python-version: '3.11'
      cache: 'pip'
      requirements-file: 'requirements.txt'
      test-command: 'pytest tests/ -v'
      coverage-command: 'pytest tests/ --cov=src --cov-report=xml'
      enable-coverage: true
  
  build:
    needs: [lint, test]
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/python-build.yml@v1
    with:
      python-version: '3.11'
      cache: 'pip'
      requirements-file: 'requirements.txt'
      build-command: 'python -m build'
      artifact-name: 'python-package'
      artifact-path: 'dist'
  
  release:
    needs: [build]
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/semantic-release.yml@v1
    secrets:
      github-token: ${{ secrets.GITHUB_TOKEN }}
    with:
      branches: 'main'
```

## Estrutura do projeto

```
my-project/
├── .github/
│   └── workflows/
│       └── ci-cd.yml
├── src/
│   └── __init__.py
├── tests/
│   └── test_main.py
├── dist/
├── requirements.txt
├── setup.py
└── README.md
```

## requirements.txt

```
pytest>=7.0.0
pytest-cov>=4.0.0
flake8>=6.0.0
black>=23.0.0
build>=0.10.0
```

## Requisitos

- Python 3.11
- pip
- requirements.txt com dependências
- setup.py ou pyproject.toml configurado
