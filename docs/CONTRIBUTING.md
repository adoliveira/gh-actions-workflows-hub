# Guia de Contribuição

Obrigado por considerar contribuir para o gh-actions-workflows-hub!

## Como Contribuir

### 1. Reportar Bugs

Antes de reportar um bug, verifique se ele já não foi reportado. Para reportar:

1. Abra uma issue
2. Use o template de bug report
3. Inclua:
   - Descrição clara do problema
   - Steps to reproduce
   - Comportamento esperado vs atual
   - Versão que está usando
   - Logs relevantes

### 2. Sugerir Funcionalidades

1. Abra uma issue
2. Use o template de feature request
3. Descreva:
   - Problema que resolve
   - Solução proposta
   - Alternativas consideradas
   - Exemplos de uso

### 3. Pull Requests

#### Processo

1. **Fork** o repositório
2. **Clone** seu fork
3. **Crie** uma branch: `git checkout -b feat/minha-feature`
4. **Commit** suas mudanças seguindo Conventional Commits
5. **Push** para seu fork: `git push origin feat/minha-feature`
6. **Abra** um Pull Request

#### Checklist

- [ ] Código segue os padrões do projeto
- [ ] Documentação atualizada
- [ ] Exemplos adicionados (se aplicável)
- [ ] Commits seguem Conventional Commits
- [ ] Testes adicionados (se aplicável)
- [ ] CHANGELOG será atualizado automaticamente

### 4. Conventional Commits

Use o formato:

```
<tipo>(<escopo>): <descrição>

[corpo opcional]

[rodapé opcional]
```

**Tipos:**
- `feat:` Nova funcionalidade
- `fix:` Correção de bug
- `docs:` Documentação
- `chore:` Manutenção
- `refactor:` Refatoração
- `test:` Testes
- `ci:` CI/CD

**Exemplos:**
```
feat(actions): adicionar setup-go-env action
fix(docker): corrigir build multi-plataforma
docs: atualizar README com novos exemplos
```

## Estrutura do Projeto

```
.
├── actions/                    # Ações compostas
│   ├── checkout-with-cache/
│   ├── setup-node-env/
│   └── ...
├── reusable-workflows/        # Workflows reutilizáveis
│   ├── node-build.yml
│   ├── python-test.yml
│   └── ...
├── .github/
│   └── workflow-templates/    # Templates de workflow
├── examples/                  # Exemplos de uso
├── docs/                      # Documentação
└── README.md
```

## Adicionando Nova Ação Composta

1. Crie diretório em `actions/<nome-da-action>/`
2. Crie `action.yml` com definição da ação
3. Crie `README.md` com documentação
4. Adicione à lista em `docs/actions.md`
5. Crie exemplo em `examples/`

**Template action.yml:**
```yaml
name: 'Nome da Ação'
description: 'Descrição breve'
author: 'adoliveira'

inputs:
  input-name:
    description: 'Descrição do input'
    required: false
    default: 'valor-padrão'

runs:
  using: 'composite'
  steps:
    - name: Step 1
      shell: bash
      run: |
        echo "Commands here"
```

## Adicionando Novo Workflow Reutilizável

1. Crie arquivo em `reusable-workflows/<nome>.yml`
2. Use `workflow_call` como trigger
3. Documente inputs e secrets
4. Adicione à lista em `docs/reusable-workflows.md`
5. Crie template em `.github/workflow-templates/`
6. Crie exemplo em `examples/`

**Template workflow reutilizável:**
```yaml
name: Nome do Workflow

on:
  workflow_call:
    inputs:
      input-name:
        description: 'Descrição'
        required: false
        type: string
        default: 'default-value'
    secrets:
      secret-name:
        description: 'Descrição'
        required: true

jobs:
  job-name:
    runs-on: ubuntu-latest
    steps:
      - name: Step
        run: echo "Steps here"
```

## Padrões de Código

### YAML
- Use 2 espaços para indentação
- Use aspas simples para strings
- Organize inputs alfabeticamente

### Markdown
- Use ATX headers (#)
- Adicione blank lines entre seções
- Use code blocks com syntax highlighting

### Commits
- Use Conventional Commits
- Escreva mensagens claras
- Referencie issues quando aplicável

## Testes

Antes de submeter PR:

1. **Valide YAML:**
   ```bash
   yamllint .
   ```

2. **Teste localmente:**
   - Use [act](https://github.com/nektos/act) para testar workflows
   - Ou crie branch de teste no seu fork

3. **Documente mudanças:**
   - Atualize README se necessário
   - Adicione exemplos
   - Atualize docs relevantes

## Code Review

Todos os PRs passam por code review:

- ✅ Seguir padrões do projeto
- ✅ Ter documentação adequada
- ✅ Ter exemplos (quando aplicável)
- ✅ Commits limpos e claros
- ✅ Não quebrar funcionalidades existentes

## Perguntas?

- Abra uma issue com label `question`
- Consulte a [documentação](./docs/)
- Veja os [exemplos](./examples/)

## Código de Conduta

Seja respeitoso e profissional em todas as interações.

## Licença

Ao contribuir, você concorda que suas contribuições serão licenciadas sob a mesma licença do projeto.
