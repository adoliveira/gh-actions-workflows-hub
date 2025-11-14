# Setup Python Environment

Ação composta que configura o ambiente Python com cache e instalação automática de dependências.

## Inputs

| Input | Descrição | Obrigatório | Padrão |
|-------|-----------|-------------|--------|
| `python-version` | Versão do Python | Não | `3.11` |
| `cache` | Gerenciador de pacotes para cache (pip, pipenv, poetry) | Não | `pip` |
| `requirements-file` | Caminho para arquivo de requisitos | Não | `requirements.txt` |
| `install-dependencies` | Instalar dependências automaticamente | Não | `true` |

## Exemplo de Uso

```yaml
- name: Setup Python with pip
  uses: adoliveira/gh-actions-workflows-hub/actions/setup-python-env@v1
  with:
    python-version: '3.11'
    cache: 'pip'
```

```yaml
- name: Setup Python with poetry
  uses: adoliveira/gh-actions-workflows-hub/actions/setup-python-env@v1
  with:
    python-version: '3.10'
    cache: 'poetry'
```
