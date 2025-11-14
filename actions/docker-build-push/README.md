# Docker Build and Push

Ação composta para build e push de imagens Docker para registros de container com suporte multi-plataforma.

## Inputs

| Input | Descrição | Obrigatório | Padrão |
|-------|-----------|-------------|--------|
| `registry` | Registro de container (docker.io, ghcr.io, etc.) | Não | `ghcr.io` |
| `image-name` | Nome da imagem (sem prefixo de registro) | Sim | - |
| `tags` | Tags da imagem (separadas por vírgula) | Não | `latest` |
| `username` | Nome de usuário do registro | Sim | - |
| `password` | Senha ou token do registro | Sim | - |
| `platforms` | Plataformas alvo (separadas por vírgula) | Não | `linux/amd64,linux/arm64` |
| `context` | Caminho do contexto de build | Não | `.` |
| `dockerfile` | Caminho para o Dockerfile | Não | `Dockerfile` |
| `push` | Push da imagem para o registro | Não | `true` |

## Exemplo de Uso

```yaml
- name: Build and push Docker image
  uses: adoliveira/gh-actions-workflows-hub/actions/docker-build-push@v1
  with:
    registry: 'ghcr.io'
    image-name: ${{ github.repository }}
    tags: 'latest,${{ github.sha }}'
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
```
