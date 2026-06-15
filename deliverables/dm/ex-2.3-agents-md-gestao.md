---
exercise: 2.3
title: AGENTS.md — Project Management Rules — NovaTech Assistant
role: Delivery Manager
phase: Estruturação
status: done
author: Luciane Baldo
created_at: 2026-06-11
cowork_card: NOVA-22
---

## Objetivo

Produzir a seção "Project Management Rules" do AGENTS.md do projeto
NovaTech Assistant — prescritiva, parseável por agentes, com caminhos
de repositório concretos e coerente com os gates definidos no Ex. 2.1.

---

## Conteúdo para /docs/AGENTS.md (seção de gestão)

---

# AGENTS.md — NovaTech Assistant

> Este documento é a constituição do projeto. Contém decisões duráveis
> que agentes de IA DEVEM seguir ao gerar código, artefatos ou sugestões.
> Regras aqui superam qualquer instrução de prompt ad hoc.

---

## Project Management Rules

### Spec Lifecycle

- Toda spec vive em `/specs/<modulo>/<tipo>.md`.
  Valores válidos para `<modulo>`: `pipeline-ingestao`, `query-endpoint`,
  `feedback-api`, `teams-bot`, `painel-web`.
  Valores válidos para `<tipo>`: `requirements`, `tech-plan`, `tasks`.
- Nenhuma task pode ser iniciada sem `status: approved` no
  `requirements.md` do respectivo módulo.
- `tech-plan.md` NÃO DEVE ser iniciado sem `requirements.md` com
  `status: approved` no mesmo módulo.
- `tasks.md` NÃO DEVE ser iniciado sem `tech-plan.md` com
  `status: approved` no mesmo módulo.
- Agentes que recebem instrução de gerar código ou artefatos para um módulo
  DEVEM verificar o status dos 3 arquivos de spec antes de prosseguir.
  Se qualquer arquivo estiver ausente ou com status diferente de `approved`
  ou `in-progress`, o agente DEVE recusar e indicar qual arquivo está pendente.
- Specs com `status: done` por mais de 30 dias DEVEM ter `status` alterado
  para `archived` pelo Delivery Manager.

### Validation Gates

- GATE 1 (Spec Aprovada): checklist completo em `/docs/checklists/gate-1.md`.
  PS preenche e menciona @delivery-manager no card Cowork referenciado
  em `cowork_card` para aprovação. SLA: 1 dia útil.
- GATE 2 (Plan Aprovado): checklist completo em `/docs/checklists/gate-2.md`.
  TL preenche antes de criar tasks. SLA: 4 horas.
- GATE 3 (Entrega Aprovada): checklist completo em `/docs/checklists/gate-3.md`.
  QA preenche e seta `qa_sign_off: true` no card Cowork. SLA: 2 horas.
- Agentes NÃO DEVEM gerar código, tech-plan ou tasks para módulo cujo
  GATE correspondente não foi aprovado.

### Nomenclatura e Caminhos de Arquivo

```
/specs/<modulo>/requirements.md   ← PS
/specs/<modulo>/tech-plan.md      ← TL
/specs/<modulo>/tasks.md          ← Dev + Copilot
/docs/AGENTS.md                   ← constituição do projeto
/docs/checklists/gate-1.md        ← checklist GATE 1
/docs/checklists/gate-2.md        ← checklist GATE 2
/docs/checklists/gate-3.md        ← checklist GATE 3
/docs/adr/ADR-<NNN>-<titulo>.md   ← Architecture Decision Records
/docs/templates/comunicacao-cliente.md
/docs/comunicados/AAAA-MM-DD-<modulo>.md
/prompts/system-prompt.md         ← único arquivo de prompt do assistente
/prompts/CHANGELOG.md             ← histórico de alterações do prompt
```

Agentes NÃO DEVEM criar arquivos de prompt fora de `/prompts/`.
Agentes NÃO DEVEM criar specs fora de `/specs/<modulo>/`.

### Rastreabilidade de Commits

- Todo commit que altera `/specs/` DEVE referenciar o `cowork_card` no
  título. Formato obrigatório: `feat(NOVA-<id>): <descricao-em-uma-linha>`.
- Todo PR DEVE conter link para o card Cowork no campo "Description".
  Formato: `Cowork: NOVA-<id>`.
- Commits sem referência a `NOVA-<id>` em arquivos sob `/specs/` NÃO DEVEM
  ser aceitos via merge. Agentes que geram commits DEVEM incluir o campo.

### Decisões do Cenário 1 — Não Reabrir Sem ADR

As decisões abaixo foram tomadas na Fase de Discovery e são fechadas para
este ciclo. Qualquer spec que proponha alterar esses valores DEVE criar um
ADR em `/docs/adr/` aprovado por TL e DM antes do GATE 1.

| Decisão | Valor definido | Referência |
|---|---|---|
| Modelo LLM | Azure OpenAI GPT-4o | ADR-001 |
| Janela de contexto | 128K tokens | ADR-001 |
| Budget por resposta | ~12K tokens (4K system prompt + 8K chunks) | ADR-002 |
| Número de chunks por resposta | 5 × ~1.500 tokens | ADR-002 |
| Base documental | 847 documentos válidos indexados | ADR-003 |
| Stack backend | TypeScript + Azure Functions | ADR-004 |
| Stack frontend | React (painel web) | ADR-004 |
| Infraestrutura | Bicep + Azure AI Search + Cosmos DB | ADR-004 |

Agentes NÃO DEVEM sugerir troca de modelo, aumento de número de chunks,
expansão de base documental ou mudança de stack sem ADR aprovado.

### Escopo do Assistente — Regras Fixas

- O assistente NovaTech atende APENAS o time de atendimento (45 pessoas).
  Qualquer spec que proponha expandir o público-alvo REQUER novo GATE 1
  com `type: requirements` e `version` incrementado, além de aprovação
  da Diretoria de Operações.
- Respostas sem fonte em documento indexado DEVEM ser recusadas pelo
  assistente. Agentes que modificam `/prompts/system-prompt.md` DEVEM
  preservar esta regra.
- Cargas perigosas (classes 1–6 ANTT) DEVEM sempre direcionar para
  Gestão de Riscos (ramal 4500). Esta instrução DEVE permanecer no
  system prompt independente de outras alterações.

### Comunicação com Stakeholders

- Mudanças que alterem qualquer campo `due_date` ou `scope` em cards
  com label `cliente-visivel` DEVEM ser registradas em
  `/docs/comunicados/AAAA-MM-DD-<modulo>.md` com no mínimo 24h de
  antecedência usando o template em
  `/docs/templates/comunicacao-cliente.md`.
- Agentes NÃO DEVEM alterar campos `due_date` ou `scope` em cards
  `cliente-visivel` sem confirmar que o arquivo de comunicado foi criado.
- Registro da comunicação DEVE ser adicionado como comentário no card
  Cowork após envio.

### Atualização deste Documento

- AGENTS.md é atualizado pelo Tech Lead a cada GATE 2 aprovado.
- Toda alteração neste arquivo DEVE ter commit com formato:
  `docs(AGENTS.md): <descricao-da-mudança> — NOVA-<id>`.
- Seções podem ser adicionadas por papel (Tech, Produto, QA).
  Nenhuma seção pode ser removida sem aprovação do DM registrada
  no card Cowork.

---

## Limitações desta Seção

- As regras de rastreabilidade de commits dependem de disciplina humana
  até integração com GitHub Actions via MCP ser configurada (Exercício 2
  do Dev). Até lá, o DM é responsável por revisar commits manualmente.
- A regra de verificação de status de specs por agentes depende de que
  os arquivos estejam atualizados no repositório. Se o Cowork e o
  repositório divergirem, o repositório é a fonte de verdade.
- A seção "Comunicação com Stakeholders" é a menos executável por agente
  de forma autônoma — serve como referência para humanos até que um MCP
  server de comunicação seja configurado.

---

## Evidência de Uso — Claude Code

### Iteração 1

**Prompt enviado:**
"Gere a seção Project Management Rules de um AGENTS.md para um projeto
de software com modelo SDD e validation gates. Use tom prescritivo."

**Resultado:** Regras genéricas sem caminhos de arquivo, sem referência
a ferramentas específicas. Usava linguagem como "o time deve considerar"
— não executável por agente.

### Iteração 2

**Refinamento:**
"Reescreva usando apenas DEVE / NÃO DEVE. Adicione caminhos de arquivo
concretos do projeto NovaTech. Inclua os 5 módulos válidos por nome e
os 3 tipos de spec por nome. Agentes devem conseguir verificar specs
sem precisar de interpretação humana."

**Resultado:** Estrutura prescritiva, mas sem referência às decisões
do Cenário 1 (ADRs) e sem regras de escopo do assistente.

### Iteração 3

**Refinamento:**
"Adicione seção de decisões fechadas do Cenário 1 com os valores reais:
GPT-4o, 128K tokens, budget de 12K, 847 documentos, stack TypeScript +
Azure Functions. Inclua regra explícita de que agentes não reabrem essas
decisões sem ADR. Adicione regras de escopo do assistente referenciando
os 45 atendentes e a regra de cargas perigosas com ramal 4500."

**Resultado:** Versão com ADRs, escopo fixo e regras de cargas perigosas
derivadas diretamente do contexto NovaTech.

### Iteração 4

**Refinamento:**
"Adicione seção de limitações honestas: o que estas regras não conseguem
garantir sozinhas sem automação MCP. Inclua qual é a fonte de verdade
em caso de divergência entre Cowork e repositório."

**Resultado final:** Versão atual com limitações explícitas e hierarquia
de fontes de verdade definida.

**O que mudou entre iterações:**
- Tom prescritivo (DEVE / NÃO DEVE) adicionado após perceber que
  linguagem narrativa não é parseável por agente (D3 flag)
- ADRs do Cenário 1 incluídos após reconhecer que o AGENTS.md sem
  memória das decisões anteriores contradiz o propósito do artefato
- Limitações adicionadas após testar o prompt com cenário onde Cowork
  e repositório divergiam — sem regra de desempate, agente ficava ambíguo
