# Quando Usar Cada Tipo

Este guia ajuda você a escolher entre Workflows Reutilizáveis, Ações Compostas e Templates.

## Comparação Rápida

| Característica | Workflows Reutilizáveis | Ações Compostas | Templates |
|----------------|------------------------|-----------------|-----------|
| **Complexidade** | Baixa | Média | Baixa |
| **Flexibilidade** | Média | Alta | Alta |
| **Manutenção** | Centralizada | Centralizada | Local |
| **Atualizações** | Automáticas | Automáticas | Manuais |
| **Customização** | Limitada | Média | Total |
| **Jobs completos** | ✅ Sim | ❌ Não | ✅ Sim |
| **Orquestração** | ✅ Sim | ❌ Não | ✅ Sim |

## Workflows Reutilizáveis

### ✅ Quando Usar

- Quer pipeline CI/CD completo "out of the box"
- Precisa de múltiplos jobs orquestrados
- Quer padronização entre projetos
- Prefere menos código no seu repositório
- Quer atualizações centralizadas automáticas

### 📝 Exemplo

```yaml
jobs:
  test:
    uses: adoliveira/gh-actions-workflows-hub/.github/workflows/node-test.yml@v1
    with:
      node-version: '20'
```

### 🎯 Casos de Uso Ideais

- **CI/CD Padrão**: Projetos que seguem padrões comuns
- **Múltiplos Projetos**: Equipes com vários repositórios similares
- **Onboarding Rápido**: Novos projetos que precisam CI/CD rapidamente
- **Manutenção Reduzida**: Menos código para manter

### ⚠️ Limitações

- Menos controle sobre detalhes
- Precisa estar em outro repositório
- Inputs limitados aos definidos
- Não pode usar secrets locais diretamente

## Ações Compostas

### ✅ Quando Usar

- Precisa de flexibilidade no workflow
- Quer combinar múltiplas ações
- Tem workflow complexo e customizado
- Quer reutilizar steps específicos
- Precisa de controle fino sobre execução

### 📝 Exemplo

```yaml
jobs:
  custom-job:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup
        uses: adoliveira/gh-actions-workflows-hub/actions/setup-node-env@v1
        with:
          node-version: '20'
      
      - name: Custom step
        run: |
          npm run custom-command
          npm run another-command
      
      - name: Docker
        uses: adoliveira/gh-actions-workflows-hub/actions/docker-build-push@v1
        with:
          image-name: my-app
```

### 🎯 Casos de Uso Ideais

- **Workflows Personalizados**: Necessidades específicas do projeto
- **Mix de Tecnologias**: Projetos com múltiplas linguagens
- **Steps Customizados**: Entre steps padrão e customizados
- **Granularidade**: Controle step-by-step

### ⚠️ Limitações

- Mais código no seu repositório
- Precisa gerenciar orquestração de jobs
- Precisa lidar com cache manualmente

## Templates

### ✅ Quando Usar

- Quer total controle sobre o workflow
- Precisa de customizações profundas
- Está começando um novo projeto
- Prefere ter tudo no seu repositório
- Não quer dependências externas

### 📝 Exemplo

1. Copiar template para `.github/workflows/`
2. Customizar conforme necessário
3. Manter e atualizar localmente

### 🎯 Casos de Uso Ideais

- **Projetos Únicos**: Requisitos muito específicos
- **Aprendizado**: Entender como workflows funcionam
- **Independência**: Não quer dependência externa
- **Controle Total**: Precisa modificar tudo

### ⚠️ Limitações

- Atualizações são manuais
- Mais código para manter
- Duplicação entre projetos
- Sem padronização automática

## Cenários Específicos

### Cenário 1: Startup com Múltiplos Microservices

**Recomendação:** Workflows Reutilizáveis

**Por quê:**
- Padronização entre todos os serviços
- Manutenção centralizada
- Onboarding rápido de novos serviços
- Menos código duplicado

```yaml
# Mesmo workflow em todos os microservices
uses: adoliveira/gh-actions-workflows-hub/.github/workflows/node-test.yml@v1
```

### Cenário 2: Monorepo com Múltiplas Aplicações

**Recomendação:** Ações Compostas

**Por quê:**
- Workflows customizados por aplicação
- Compartilhamento de steps comuns
- Flexibilidade para diferentes tecnologias

```yaml
jobs:
  frontend:
    steps:
      - uses: adoliveira/.../actions/setup-node-env@v1
      - run: npm run build:frontend
  
  backend:
    steps:
      - uses: adoliveira/.../actions/setup-python-env@v1
      - run: python build_backend.py
```

### Cenário 3: Projeto Open Source Único

**Recomendação:** Templates (customizados)

**Por quê:**
- Controle total sobre o workflow
- Sem dependências externas
- Transparência para contribuidores
- Customizações específicas do projeto

### Cenário 4: Aplicação Empresarial Complexa

**Recomendação:** Mix de Workflows Reutilizáveis + Ações Compostas

**Por quê:**
- Workflows reutilizáveis para CI/CD padrão
- Ações compostas para steps específicos
- Balance entre padronização e flexibilidade

```yaml
jobs:
  test:
    uses: adoliveira/.../node-test.yml@v1
  
  custom-deploy:
    steps:
      - uses: adoliveira/.../actions/setup-node-env@v1
      - run: ./custom-deploy-script.sh
```

## Matriz de Decisão

### Use Workflows Reutilizáveis se:

- ✅ Quer começar rápido
- ✅ Pipeline padrão atende necessidades
- ✅ Tem múltiplos projetos similares
- ✅ Prefere manutenção centralizada

### Use Ações Compostas se:

- ✅ Precisa de flexibilidade
- ✅ Workflow tem partes customizadas
- ✅ Quer reutilizar steps específicos
- ✅ Mix de tecnologias

### Use Templates se:

- ✅ Projeto único e específico
- ✅ Quer independência total
- ✅ Precisa customizar profundamente
- ✅ Aprendendo GitHub Actions

## Combinações Recomendadas

### Melhor: Mix Inteligente

```yaml
name: CI/CD

jobs:
  # Workflow reutilizável para CI padrão
  test:
    uses: adoliveira/.../node-test.yml@v1
  
  # Ações compostas para steps customizados
  custom-build:
    runs-on: ubuntu-latest
    steps:
      - uses: adoliveira/.../actions/setup-node-env@v1
      - name: Custom build
        run: npm run custom-build
  
  # Workflow reutilizável para release
  release:
    needs: [test, custom-build]
    uses: adoliveira/.../semantic-release.yml@v1
```

## Perguntas Frequentes

### Q: Posso misturar workflows reutilizáveis e ações compostas?
**A:** Sim! É até recomendado para balance entre padronização e flexibilidade.

### Q: Templates são atualizados automaticamente?
**A:** Não, você precisa atualizar manualmente copiando novas versões.

### Q: Workflows reutilizáveis funcionam em repositórios privados?
**A:** Sim, mas o repositório com os workflows precisa ser acessível.

### Q: Qual é mais performático?
**A:** Performance é similar. Workflows reutilizáveis podem ter overhead mínimo de chamada.

### Q: Posso converter templates em workflows reutilizáveis depois?
**A:** Sim, mas requer refatoração dos workflows existentes.

## Conclusão

**Regra de Ouro:**
- 🚀 Comece com **Workflows Reutilizáveis**
- 🔧 Customize com **Ações Compostas** quando necessário
- 📝 Use **Templates** apenas quando precisar de controle total

**Lembre-se:** Você pode sempre migrar entre as opções conforme seu projeto evolui!
