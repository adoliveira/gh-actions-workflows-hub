# Checkout with Cache

Ação composta que realiza checkout do repositório com suporte a cache de dependências para múltiplos gerenciadores de pacotes.

## Inputs

| Input | Descrição | Obrigatório | Padrão |
|-------|-----------|-------------|--------|
| `token` | GitHub token para checkout | Não | `${{ github.token }}` |
| `cache` | Habilitar cache (npm, yarn, pnpm, pip, maven, gradle) | Não | `none` |
| `cache-dependency-path` | Caminho para arquivo(s) de dependência para chave do cache | Não | `''` |

## Exemplo de Uso

```yaml
- name: Checkout with npm cache
  uses: adoliveira/gh-actions-workflows-hub/actions/checkout-with-cache@v1
  with:
    cache: 'npm'
    cache-dependency-path: 'package-lock.json'
```

```yaml
- name: Checkout with pip cache
  uses: adoliveira/gh-actions-workflows-hub/actions/checkout-with-cache@v1
  with:
    cache: 'pip'
    cache-dependency-path: 'requirements.txt'
```
