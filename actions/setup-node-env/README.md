# Setup Node.js Environment

Ação composta que configura o ambiente Node.js com cache e instalação automática de dependências.

## Inputs

| Input | Descrição | Obrigatório | Padrão |
|-------|-----------|-------------|--------|
| `node-version` | Versão do Node.js | Não | `20` |
| `cache` | Gerenciador de pacotes para cache (npm, yarn, pnpm) | Não | `npm` |
| `install-dependencies` | Instalar dependências automaticamente | Não | `true` |

## Exemplo de Uso

```yaml
- name: Setup Node.js with npm
  uses: adoliveira/gh-actions-workflows-hub/actions/setup-node-env@v1
  with:
    node-version: '20'
    cache: 'npm'
```

```yaml
- name: Setup Node.js with pnpm
  uses: adoliveira/gh-actions-workflows-hub/actions/setup-node-env@v1
  with:
    node-version: '18'
    cache: 'pnpm'
    install-dependencies: 'true'
```
