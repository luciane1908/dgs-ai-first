---
exercise: 2.2
title: Governança de Specs — SDD NovaTech Assistant
role: Delivery Manager
phase: Estruturação
status: done
author: Luciane Baldo
created_at: 2026-06-11
cowork_card: NOVA-21
---

## Objetivo

Estabelecer specs como artefatos vivos com ciclo de vida definido,
atribuições claras por papel, gestão explícita de mudanças e board
de tracking visual e filtrável no Cowork — alinhado com a estrutura
de diretórios do Anexo C.

---

## Modelo SDD — 3 Documentos por Módulo

Cada um dos 5 módulos do projeto NovaTech tem exatamente 3 specs
em sequência obrigatória:

```
┌──────────────────────────────────────────────────────────────────┐
│  /specs/<modulo>/                                                │
│                                                                  │
│  requirements.md  →  tech-plan.md  →  tasks.md                  │
│  PS escreve          TL escreve       Dev + Copilot              │
│  GATE 1 aqui ↓       GATE 2 aqui ↓   tasks criadas no Cowork    │
└──────────────────────────────────────────────────────────────────┘
```

**Regra crítica:** `tech-plan.md` nunca é iniciado sem `requirements.md`
com `status: approved`. `tasks.md` nunca é iniciado sem `tech-plan.md`
com `status: approved`. Agentes que recebem task de um módulo devem
verificar o status dos 3 arquivos antes de gerar código.

---

## Módulos do Projeto

| Módulo | Diretório | Responsável PS | Responsável TL |
|---|---|---|---|
| Pipeline de Ingestão | `/specs/pipeline-ingestao/` | a definir | a definir |
| Query Endpoint | `/specs/query-endpoint/` | a definir | a definir |
| Feedback API | `/specs/feedback-api/` | a definir | a definir |
| Teams Bot | `/specs/teams-bot/` | a definir | a definir |
| Painel Web | `/specs/painel-web/` | a definir | a definir |

---

## Ciclo de Vida de uma Spec

```
[draft] ──→ [in-review] ──→ [approved] ──→ [in-progress] ──→ [done] ──→ [archived]
                 ↑                 ↓
            [rejected] ←── mudança de escopo detectada no GATE
```

### Transições e Responsáveis

| Transição | Quem move | Condição obrigatória |
|---|---|---|
| `draft → in-review` | Product Specialist | Checklist GATE 1 preenchido |
| `in-review → approved` | Delivery Manager | Todos os critérios GATE 1 satisfeitos |
| `in-review → rejected` | Delivery Manager | Comentário obrigatório com motivo específico |
| `approved → in-progress` | Tech Lead | Tasks criadas no Cowork com `spec_ref` |
| `in-progress → done` | QA | Checklist GATE 3 assinado + `qa_sign_off: true` |
| `done → archived` | Delivery Manager | 30 dias após deploy em produção confirmado |

---

## Frontmatter YAML Obrigatório

Todo arquivo de spec começa com este frontmatter:

```yaml
---
module: pipeline-ingestao        # um dos 5 módulos válidos
type: requirements               # requirements | tech-plan | tasks
status: draft                    # segue o ciclo de vida acima
version: 1.0                     # incrementa a cada change-request aprovado
author: nome-sobrenome
reviewer: nome-sobrenome
approved_by: ~                   # preenchido pelo DM ao mover para approved
created_at: 2026-06-11
updated_at: 2026-06-11
cowork_card: NOVA-42             # ID do card no Cowork — obrigatório
---
```

---

## Board de Tracking no Cowork

### Colunas do Board

```
| Backlog | Draft | In Review | Approved | In Progress | Done | Archived |
```

### Campos Obrigatórios em Cada Card

| Campo | Tipo | Obrigatório em |
|---|---|---|
| `titulo` | texto livre | sempre |
| `modulo` | select — 5 opções | sempre |
| `tipo` | select — requirements / tech-plan / tasks | sempre |
| `status` | select — segue ciclo de vida | sempre |
| `responsavel` | pessoa | sempre |
| `revisor` | pessoa | sempre |
| `spec_ref` | link para arquivo no repositório | sempre |
| `cowork_card` | ID gerado automaticamente | sempre |
| `gate_checklist` | checklist inline (items do gate correspondente) | sempre |
| `qa_sign_off` | boolean | obrigatório para mover para `done` |
| `change_request` | campo de texto para mudanças em specs aprovadas | quando aplicável |

### Views Salvas Recomendadas

| View | Filtro | Para quem |
|---|---|---|
| Minha fila | `responsavel = eu AND status != done` | Todos |
| Aguardando DM | `status = in-review` | DM |
| Sem QA | `status = in-progress AND qa_sign_off = false` | DM + QA |
| Por módulo | `modulo = <modulo-selecionado>` | TL por módulo |
| Bloqueados | `status = rejected OR change_request != vazio` | DM |

---

## Gestão de Mudanças em Specs Aprovadas

```
CHANGE REQUEST — Fluxo

1. Autor detecta necessidade de mudança
   → Abre comentário no campo change_request do card Cowork
   → Inclui: o que muda, por quê, módulos afetados, estimativa em horas

2. DM avalia em até 4 horas

   Impacto < 2h (mudança pequena)
   → Aprova no comentário
   → Spec volta para in-review (mantém versão atual)
   → Nenhuma task é bloqueada

   Impacto >= 2h (mudança grande)
   → Cria novo arquivo com version incrementado (ex: 1.0 → 2.0)
   → Arquiva versão anterior (status: archived)
   → Tasks afetadas recebem label blocked no Cowork
   → Novo GATE 1 obrigatório antes de retomar desenvolvimento

3. Commit gerado após mudança aprovada:
   chore(NOVA-<id>): change-request aprovado — <resumo>
```

---

## Decisão Pendente — Bloqueio Registrado

**Módulo pipeline-ingestao — GATE 1 bloqueado**

O Anexo A documenta que PROC-042 v1 e v2 coexistem no SharePoint sem
hierarquia definida (multiplicadores regionais divergentes entre versões).

O requirements.md de pipeline-ingestao não pode ser aprovado sem que
PS + Diretoria Comercial definam qual versão é a fonte de verdade para
indexação.

Ação requerida: reunião de alinhamento com Diretoria Comercial antes
do início da spec deste módulo. Registrado no Cowork como impedimento
no card NOVA-21.

---

## Limitações e Decisões Abertas

- **847 documentos sem controle de versão formal:** o ciclo de vida de
  specs não resolve documentos-fonte desatualizados no SharePoint. Uma
  spec com `status: done` pode ficar inconsistente se o documento NovaTech
  for alterado. Mitigação necessária: política de revisão trimestral das
  specs aprovadas (fora do escopo deste exercício, documentado como risco).

- **Rastreabilidade manual até MCP estar configurado:** o campo `cowork_card`
  nos commits depende de disciplina do time, não de automação. Após o
  Exercício 2 do Dev (configuração MCP), este campo pode ser validado
  automaticamente via GitHub Actions.

- **Board no Cowork não substitui a spec no repositório:** o Cowork é o
  painel de gestão; o arquivo `.md` no repositório é a fonte de verdade.
  Em caso de divergência, o arquivo no repositório prevalece.

---

## Evidência de Uso — Claude Code

### Iteração 1

**Prompt enviado:**
"Crie um ciclo de vida para specs de um projeto de software com modelo SDD.
Inclua estados, transições e responsáveis por cada transição."

**Resultado:** Ciclo genérico com estados draft/review/done. Sem
frontmatter, sem referência a módulos específicos, sem change management.

### Iteração 2

**Refinamento:**
"Adapte para o projeto NovaTech Assistant. Os módulos são: pipeline-ingestao,
query-endpoint, feedback-api, teams-bot e painel-web. O modelo SDD tem
3 documentos por módulo: requirements (PS), tech-plan (TL) e tasks (Dev).
Inclua frontmatter YAML obrigatório com campo cowork_card."

**Resultado:** Estrutura com os 5 módulos e 3 documentos, mas sem
protocolo de change management e sem gestão de bloqueios.

### Iteração 3

**Refinamento:**
"Adicione protocolo de change management diferenciado por porte de mudança
(menos de 2h vs 2h ou mais). Inclua o caso real do conflito PROC-042 v1/v2
como exemplo de bloqueio de GATE 1. Adicione limitações honestas sobre o que
este ciclo não resolve (documentos no SharePoint sem versão)."

**Resultado final:** Versão atual com change management, decisão pendente
documentada e limitações explícitas.

**O que mudou entre iterações:**
- Change management adicionado após simular o cenário real onde uma spec
  aprovada precisaria ser alterada por conflito de versão de documento
- Limitação do SharePoint incluída após reconhecer que o ciclo de vida
  de specs não resolve o problema upstream dos documentos-fonte
- Bloqueio do módulo pipeline-ingestao documentado como decisão pendente
  real, não como caso hipotético
