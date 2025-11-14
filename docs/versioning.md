# Guia de Versionamento Semântico

Este projeto segue o [Semantic Versioning 2.0.0](https://semver.org/).

## Formato de Versão

```
MAJOR.MINOR.PATCH
```

- **MAJOR**: Mudanças incompatíveis na API
- **MINOR**: Novas funcionalidades compatíveis
- **PATCH**: Correções de bugs compatíveis

## Conventional Commits

Usamos [Conventional Commits](https://www.conventionalcommits.org/) para automatizar o versionamento.

### Tipos de Commit

#### `feat:` - Nova Funcionalidade (MINOR)
```
feat: adicionar suporte para pnpm cache
feat(actions): adicionar nova action setup-go-env
```

#### `fix:` - Correção de Bug (PATCH)
```
fix: corrigir cache do pip não funcionando
fix(docker): corrigir build multi-plataforma
```

#### `BREAKING CHANGE:` - Mudança Incompatível (MAJOR)
```
feat!: remover suporte para Node.js 14

BREAKING CHANGE: Node.js 14 não é mais suportado
```

#### Outros Tipos
```
docs: atualizar documentação
chore: atualizar dependências
ci: melhorar workflow de CI
refactor: refatorar código
test: adicionar testes
perf: melhorar performance
```

## Tags de Versão

### Tags Principais
- `v1`, `v2`, `v3` - Tags major (movíveis)
- `v1.0`, `v1.1`, `v1.2` - Tags minor (movíveis)
- `v1.0.0`, `v1.0.1` - Tags patch (fixas)

### Recomendações de Uso

**Para Produção:**
```yaml
uses: adoliveira/gh-actions-workflows-hub/actions/setup-node-env@v1.2.3
```

**Para Desenvolvimento:**
```yaml
uses: adoliveira/gh-actions-workflows-hub/actions/setup-node-env@v1
```

**Para Testes:**
```yaml
uses: adoliveira/gh-actions-workflows-hub/actions/setup-node-env@main
```

## Processo de Release

1. **Desenvolvimento**
   - Commits seguindo Conventional Commits
   - Pull requests para branch `main`

2. **Release Automático**
   - Merge para `main` trigger semantic-release
   - Semantic-release analisa commits
   - Gera nova versão automaticamente
   - Cria tag e release no GitHub
   - Atualiza CHANGELOG.md

3. **Atualização de Tags**
   - Tags major e minor são atualizadas automaticamente
   - Tags patch são criadas e fixas

## Exemplos de Versionamento

### Cenário 1: Correção de Bug
```
Commits:
- fix: corrigir cache npm

Versão atual: v1.2.3
Nova versão: v1.2.4
```

### Cenário 2: Nova Funcionalidade
```
Commits:
- feat: adicionar suporte para Go

Versão atual: v1.2.4
Nova versão: v1.3.0
```

### Cenário 3: Breaking Change
```
Commits:
- feat!: remover Node.js 14
- BREAKING CHANGE: Node.js 14 não é mais suportado

Versão atual: v1.3.0
Nova versão: v2.0.0
```

## CHANGELOG

O CHANGELOG é gerado automaticamente pelo semantic-release e inclui:

- Lista de features
- Lista de fixes
- Breaking changes
- Links para commits e PRs

## Migration Guide

Ao fazer breaking changes, sempre incluir guia de migração:

```markdown
## Migração v1 → v2

### Breaking Changes

1. Node.js 14 não é mais suportado
   - **Antes:** `node-version: '14'`
   - **Depois:** `node-version: '16'` (mínimo)

2. Input `package-manager` renomeado para `cache`
   - **Antes:** `package-manager: 'npm'`
   - **Depois:** `cache: 'npm'`
```

## Referências

- [Semantic Versioning](https://semver.org/)
- [Conventional Commits](https://www.conventionalcommits.org/)
- [semantic-release](https://github.com/semantic-release/semantic-release)
