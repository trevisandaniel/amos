# AMOS

AMOS é um harness de agentes de programação escrito em Go.

Seu objetivo é coordenar, de forma controlada e auditável, a análise de repositórios, o planejamento de alterações, a execução iterativa, a validação, a revisão e o versionamento de código.

O AMOS separa:

- decisões semânticas realizadas por modelos de IA;
- verificações determinísticas executadas pelo runtime;
- ações que dependem de aprovação humana.

## Princípios

- Nenhuma implementação deve começar sem uma tarefa planejada.
- O planejamento não deve modificar o código-fonte.
- A execução deve trabalhar somente dentro do escopo autorizado.
- Nenhuma implementação deve ser aprovada apenas pelo agente executor.
- Testes, build, lint e typecheck devem ser executados pelo runtime.
- Toda ação relevante deve produzir registros e evidências.
- Operações de Git devem respeitar as aprovações configuradas.
- Push e merge permanecem sob responsabilidade do usuário no MVP.

## Estrutura interna

Ao ser inicializado, o AMOS cria o diretório `.amos` no repositório:

```text
.amos/
├── config.yaml
├── project.yaml
├── tasks/
├── runs/
├── evidence/
└── skills/
```

Esse diretório armazena a configuração, o mapa do projeto, as tarefas, as trajetórias de execução e as evidências coletadas.

# Comandos

## `amos init`

Inicializa o AMOS no repositório atual.

```bash
amos init
```

### Em um repositório existente

O comando deve:

- identificar linguagens e frameworks;
- encontrar comandos de build, testes, lint e typecheck;
- localizar arquivos de instrução e documentação;
- identificar a estrutura e os módulos principais;
- criar a configuração do AMOS;
- produzir um mapa inicial do projeto.

A análise inicial será registrada em:

```text
.amos/config.yaml
.amos/project.yaml
```

O mapa do projeto deve ser tratado como um artefato atualizável, não como uma descrição permanente e infalível do repositório.

### Em um repositório vazio

O comando deve:

- inicializar somente a estrutura interna do AMOS;
- perguntar ou receber qual tipo de projeto será criado;
- registrar as informações fornecidas pelo usuário;
- preparar o repositório para as etapas posteriores.

O `amos init` não deve gerar uma aplicação completa automaticamente. 

---

## `amos plan`

Transforma uma solicitação de feature, correção ou manutenção em uma tarefa estruturada.

```bash
amos plan "Adicionar filtro por empresa ao relatório de vendas"
```

Esse comando opera em modo somente leitura e não modifica o código-fonte.

### Fluxo

1. Receber a descrição do usuário.
2. Detectar informações ausentes ou ambíguas.
3. Fazer perguntas quando o requisito não estiver suficientemente claro.
4. Investigar o projeto em modo somente leitura.
5. Definir requisitos, premissas, restrições e critérios de aceite.
6. Decompor a implementação em subtarefas verificáveis.
7. Salvar a tarefa.

### Exemplo de saída

```text
.amos/tasks/TASK-0001.yaml
```

Exemplo simplificado:

```yaml
id: TASK-0001
title: Adicionar filtro por empresa
status: planned

requirement:
  objective: Adicionar filtro por empresa ao relatório de vendas
  context: ""
  acceptance_criteria: []
  assumptions: []
  constraints: []
  out_of_scope: []
  open_questions: []

implementation:
  subtasks:
    - id: TASK-0001-01
      objective: ""
      status: pending
      depends_on: []
      expected_files: []
      validation_commands: []

created_at: ""
updated_at: ""
```

Uma tarefa só pode ser considerada pronta para execução quando não possuir perguntas bloqueantes e tiver critérios de aceite verificáveis.

---

## `amos execute`

Executa uma tarefa criada pelo `amos plan`.

```bash
amos execute TASK-0001
```

O executor implementa as subtarefas planejadas, mas não aprova sua própria implementação.

### Responsabilidades

- carregar a tarefa selecionada;
- verificar se ela está pronta para execução;
- selecionar a próxima subtarefa desbloqueada;
- carregar a Skill de execução;
- montar o contexto necessário;
- iniciar o agent loop;
- aplicar limites de iterações, tokens, tempo e custo;
- autorizar, bloquear ou solicitar aprovação para ferramentas;
- persistir ações, observações e resultados;
- interromper a execução em caso de bloqueio;
- produzir evidências;
- atualizar o estado da tarefa.

### Resultados possíveis

```text
implementation_completed
blocked
needs_replan
iteration_limit_reached
budget_exceeded
execution_failed
```

Ao concluir a implementação, a tarefa deve seguir para validação. O status `implementation_completed` não significa que a solução foi aprovada.

---

## `amos validate`

Executa verificações determinísticas sobre a implementação.

```bash
amos validate TASK-0001
```

### Verificações possíveis

- testes definidos na tarefa;
- lint;
- build;
- typecheck;
- arquivos alterados;
- escopo permitido;
- critérios de aceite verificáveis;
- existência de diff;
- comandos obrigatórios;
- arquivos inesperados;
- alterações fora do workspace autorizado.

### Resultados possíveis

```text
validated
changes_requested
validation_failed
human_validation_required
```

Quando a validação falhar, o AMOS deve registrar as evidências e mudar o estado da tarefa para `changes_requested`.

No MVP, o `validate` não reinicia automaticamente o executor. O usuário poderá solicitar uma nova execução:

```bash
amos execute TASK-0001
```

Isso evita loops e custos inesperados.

---

## `amos review`

Realiza uma revisão semântica independente da implementação.

```bash
amos review TASK-0001
```

Diferentemente do `validate`, que executa verificações determinísticas, o `review` utiliza um modelo de IA em um contexto limpo.

### Entradas da revisão

- requisito;
- critérios de aceite;
- plano;
- diff;
- resultados da validação;
- evidências coletadas;
- mapa relevante do projeto.

O revisor não deve receber toda a conversa do executor, salvo quando uma informação específica da trajetória for necessária.

### Questões avaliadas

1. A solução atende à solicitação?
2. Existem mudanças fora do escopo?
3. Os testes são suficientes?
4. Há riscos arquiteturais?
5. Faltam casos extremos ou cenários relevantes?
6. O código está consistente com o restante do projeto?
7. As evidências comprovam os critérios de aceite?
8. A implementação introduz regressões ou dependências desnecessárias?

### Resultados possíveis

```text
approved
changes_requested
needs_replan
human_review_required
```

O revisor não deve modificar diretamente o código. Quando encontrar problemas, deve produzir observações verificáveis para uma nova execução ou para o replanejamento.

---

## `amos commit`

Realiza o versionamento controlado de uma implementação aprovada.

```bash
amos commit TASK-0001
```

### Pré-condições

O comando só pode ser executado quando:

- a execução estiver concluída;
- a validação estiver aprovada;
- a revisão estiver aprovada, quando habilitada;
- o usuário tiver autorizado a operação;
- nenhum arquivo inesperado estiver incluído;
- o repositório estiver em um estado compatível com o commit.

### Responsabilidades

- sugerir um nome de branch;
- gerar uma mensagem de commit;
- utilizar um prefixo permitido, como `feat`, `fix`, `chore` ou `hotfix`;
- apresentar os arquivos que serão adicionados;
- executar `git add` somente sobre os arquivos autorizados;
- executar `git commit`;
- registrar o hash do commit;
- associar o commit à tarefa correspondente.

Exemplo:

```text
feat(analytics): adiciona filtro por empresa ao relatório de vendas
```

O AMOS não deve executar `git push` ou merge automaticamente no primeiro MVP. Essas operações permanecem sob responsabilidade do usuário.

---

## `amos audit`

Apresenta os registros e as métricas coletados durante o fluxo.

```bash
amos audit TASK-0001
```

A auditoria é uma capacidade transversal. Todos os comandos devem registrar eventos enquanto são executados; o `amos audit` apenas consulta e apresenta esses dados.

### Exemplo

```text
Task: TASK-0001
Status: changes_requested

Model calls: 8
Tool calls: 21
Iterations: 5

Input tokens: 38.420
Output tokens: 5.320
Cached tokens: 17.100
Estimated cost: US$ 0,18
Elapsed time: 4m32s

Files read: 12
Files modified: 3
Commands executed: 6
Policy blocks: 1
Validation attempts: 2
```

### Informações auditáveis

- chamadas ao modelo;
- chamadas de ferramentas;
- comandos executados;
- arquivos lidos e modificados;
- decisões de política;
- bloqueios e aprovações;
- número de iterações;
- tokens de entrada, saída e cache;
- custo estimado;
- tempo total;
- resultados de validação;
- decisões do revisor;
- alterações de estado;
- commit relacionado.

Quando o provedor não informar determinada métrica, o AMOS deve apresentá-la como indisponível, e não como zero.

---

## `amos status`

Apresenta o estado atual do fluxo.

```bash
amos status
amos status TASK-0001
```

### Informações apresentadas

- tarefas existentes;
- estado de cada tarefa;
- próxima ação permitida;
- bloqueios;
- última execução;
- validações pendentes;
- revisão pendente;
- orçamento consumido;
- limite restante;
- subtarefa atual.

### Exemplo

```text
TASK-0001 — Adicionar filtro por empresa

Estado: changes_requested
Próxima ação: amos execute TASK-0001
Motivo: build falhou
Última execução: 2026-11-17 14:32
Iterações utilizadas: 3/6
```

Sem um identificador de tarefa, o comando apresenta um resumo de todas as tarefas conhecidas:

```bash
amos status
```

---

## `amos run`

Executará automaticamente o fluxo completo de uma tarefa.

```bash
amos run TASK-0001
```

Fluxo planejado:

```text
plan
→ execute
→ validate
→ execute novamente, quando necessário
→ review
→ solicitar aprovação
→ commit
```

Esse comando será implementado futuramente e permanece inativo no primeiro MVP.

A execução automática deverá respeitar:

- limite total de iterações;
- limite de custo;
- stopping conditions;
- operações que exigem aprovação;
- número máximo de retornos ao executor;
- bloqueios de política;
- solicitações de replanejamento.

# Estados das tarefas

Uma tarefa pode assumir os seguintes estados:

```text
draft
planned
ready
executing
implementation_completed
validating
changes_requested
validated
reviewing
approved
blocked
needs_replan
committed
failed
```

Transições inválidas devem ser rejeitadas. Por exemplo:

- não executar uma tarefa com perguntas bloqueantes;
- não revisar uma tarefa que ainda não foi validada;
- não criar commit antes da aprovação;
- não marcar uma tarefa como aprovada somente porque os testes passaram.

# Escopo do primeiro MVP

A primeira versão deve implementar:

- `amos init`;
- `amos plan`;
- `amos execute`;
- `amos validate`;
- `amos status`;
- `amos audit`;
- um único provedor de modelo;
- execução local;
- uma tarefa por vez;
- limite de iterações;
- ferramentas básicas;
- confirmação antes de ações sensíveis;
- persistência das tarefas;
- trajetória em JSONL;
- registro de tokens, custo e tempo quando disponíveis.

Não fazem parte do primeiro MVP:

- `amos run`;
- execução paralela;
- múltiplos executores;
- push automático;
- merge automático;
- criação automática de pull request;
- seleção automática de vários modelos;
- memória vetorial;
- interface gráfica;
- execução distribuída.

Os comandos `amos review` e `amos commit` podem ser adicionados após a validação do fluxo principal:

```text
init → plan → execute → validate → audit
```