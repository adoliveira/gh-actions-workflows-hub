# Semantic Release

Ação composta para versionamento semântico automatizado e gerenciamento de releases.

## Inputs

| Input | Descrição | Obrigatório | Padrão |
|-------|-----------|-------------|--------|
| `github-token` | Token GitHub para criar releases | Sim | - |
| `dry-run` | Executar em modo dry-run sem publicar | Não | `false` |
| `branches` | Configuração de branches para semantic-release | Não | `main` |
| `extra-plugins` | Plugins adicionais do semantic-release (separados por espaço) | Não | `''` |

## Exemplo de Uso

```yaml
- name: Semantic Release
  uses: adoliveira/gh-actions-workflows-hub/actions/semantic-release@v1
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    branches: 'main'
```

```yaml
- name: Semantic Release (dry-run)
  uses: adoliveira/gh-actions-workflows-hub/actions/semantic-release@v1
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    dry-run: 'true'
```

## Conventional Commits

Esta ação usa o padrão de Conventional Commits:
- `feat:` - Nova funcionalidade (incrementa MINOR)
- `fix:` - Correção de bug (incrementa PATCH)
- `BREAKING CHANGE:` - Mudança incompatível (incrementa MAJOR)
