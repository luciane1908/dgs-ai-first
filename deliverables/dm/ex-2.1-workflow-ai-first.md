---
exercise: 2.1
title: Workflow AI First — NovaTech Assistant
role: Delivery Manager
phase: Estruturação
status: done
author: Luciane Baldo
created_at: 2026-06-11
cowork_card: NOVA-20
---

## Objetivo

Mapear como cada papel usa ferramentas de IA em cada fase do projeto
NovaTech Assistant, definindo validation gates proporcionais ao risco
e checklists executáveis por qualquer membro do time.

---

## Fluxo por Papel e Fase

| Fase | Delivery Manager (Cowork + Claude) | Product Specialist (Claude) | Dev (Copilot) | Tech Lead (Copilot) | QA (Copilot) |
|---|---|---|---|---|---|
| **Spec** | Usa Claude para rascunho de escopo, riscos e comunicação de mudanças; cria card no Cowork com `status: draft` | Usa Claude para expandir requisitos, identificar gaps e formalizar guardrails | — | Consulta spec para avaliar viabilidade técnica | Deriva cenários de aceite a partir da spec |
| **Plan** | Valida GATE 1; move card para `approved`; comunica dependências externas ao cliente | Confirma glossário de domínio e guardrails | Usa Copilot para quebrar spec em tasks com estimativa | Usa Copilot para rascunho de tech-plan; atualiza AGENTS.md | Escreve test plan com Copilot baseado nos requisitos |
| **Dev** | Monitora board; bloqueia tasks sem GATE 2 aprovado; escala impedimentos | Responde dúvidas de produto via Cowork | Implementa com Copilot passando trechos da spec como contexto | Revisa código gerado; atualiza AGENTS.md com decisões técnicas | Executa testes; reporta bugs com referência ao card Cowork |
| **Review** | Conduz GATE 3; assina checklist; registra decisão no Cowork | Valida comportamento do assistente contra guardrails definidos | Abre PR com link para spec e tasks relacionadas | Faz code review assistido por Copilot | Assina cobertura mínima de 80%; seta `qa_sign_off: true` |
| **Deploy** | Aprova go/no-go; documenta decisão em `/docs/comunicados/`; notifica cliente com 24h de antecedência | Monitora respostas do assistente em produção no primeiro dia | Executa deploy com Copilot gerando scripts Bicep | Valida infraestrutura e rollback plan | Executa smoke tests pós-deploy |

**Regra de ouro:** Copilot é a ferramenta de geração de código e artefatos técnicos.
Claude e Cowork são as ferramentas de gestão, contexto e comunicação.
Nenhum papel usa "tudo ao mesmo tempo" sem distinção de propósito.

---

## Validation Gates

### GATE 1 — Spec Aprovada (antes de Plan)

```
Responsável por preencher: Product Specialist
Aprovador: Delivery Manager
SLA de aprovação: 1 dia útil
SLA se DM ausente: PS pode escalar para TL após 4h sem resposta
```

**Checklist executável — `/docs/checklists/gate-1.md`**
```
[ ] Arquivo criado em /specs/<modulo>/requirements.md
[ ] Frontmatter YAML preenchido (autor, data, versão, status: in-review)
[ ] Seção "Objetivo" com métrica de sucesso mensurável
[ ] Seção "Fora de Escopo" com pelo menos 2 itens explícitos
[ ] Guardrails listados no formato: "O assistente NUNCA deve..."
[ ] Conflitos com documentos existentes identificados (ex: PROC-042 v1 vs v2)
[ ] TL consultado sobre viabilidade técnica (comentário no card)
[ ] PS e TL adicionaram comentário de aprovação no card Cowork
[ ] DM moveu card para status: approved no board
```

**Critério de rejeição:** qualquer item desmarcado → card volta para `draft`
com comentário obrigatório indicando o motivo.

---

### GATE 2 — Plan Aprovado (antes de Dev)

```
Responsável por preencher: Tech Lead
Aprovador: Delivery Manager
SLA de aprovação: 4 horas
```

**Checklist executável — `/docs/checklists/gate-2.md`**
```
[ ] /specs/<modulo>/tech-plan.md criado com frontmatter YAML
[ ] Tasks criadas no Cowork com campo spec_ref preenchido
[ ] Nenhuma task com estimativa > 2 dias (quebrar se necessário)
[ ] AGENTS.md atualizado em /docs/AGENTS.md (seção Tech Decisions)
[ ] MCP servers necessários listados no card
[ ] Impacto em budget de contexto avaliado (limite: ~12K tokens por resposta)
[ ] DM moveu card para status: in-progress no board
```

---

### GATE 3 — Entrega Aprovada (antes de Deploy)

```
Responsável por preencher: QA
Aprovador: Delivery Manager
SLA de aprovação: 2 horas
```

**Checklist executável — `/docs/checklists/gate-3.md`**
```
[ ] PR aberto com link para spec e tasks relacionadas no Cowork
[ ] Cobertura de testes >= 80% (unit + integration)
[ ] Zero bugs críticos abertos
[ ] Smoke tests passando em staging
[ ] QA setou qa_sign_off: true no card Cowork
[ ] Changelog atualizado em /prompts/CHANGELOG.md (se prompt mudou)
[ ] Nenhum TODO/FIXME crítico no código
[ ] DM aprovou go/no-go e registrou decisão no card
```

---

## Protocolo de Change Management

Mudanças em specs com `status: approved` ou `in-progress` seguem este protocolo:

```
1. Autor abre comentário no card Cowork (campo: change-request) com:
   - O que muda e por quê
   - Módulos e tasks impactados
   - Estimativa de impacto em horas

2. DM avalia em até 4 horas:
   - Impacto < 2h  → aprova no comentário; spec volta para in-review; mantém versão
   - Impacto >= 2h → cria nova spec com version incrementado; arquiva versão anterior
                     Tasks afetadas bloqueadas no Cowork até novo GATE 1

3. Toda mudança aprovada gera commit com formato:
   chore(NOVA-<id>): change-request aprovado — <resumo-em-uma-linha>
```

---

## Limitações e Riscos

- **Gates aumentam lead time:** aprovações acumuladas pelo DM podem gerar
  gargalo. Mitigação: DM pode delegar GATE 1 ao PS após 4h de ausência,
  registrando delegação no Cowork.

- **Copilot sem contexto NovaTech por padrão:** até a configuração MCP ser
  concluída (Exercício 2 do Dev), devs precisam colar trechos da spec
  manualmente no prompt a cada sessão. O workflow atual não elimina esse
  atrito — apenas o documenta como risco conhecido.

- **Cowork como fonte única de verdade:** se qualquer papel usar Azure DevOps
  ou Jira em paralelo, a rastreabilidade quebra. Decisão registrada: Cowork
  é a única ferramenta de tracking válida para o projeto NovaTech. Outras
  ferramentas são somente leitura.

- **Conflito PROC-042 v1 vs v2 não resolvido:** o Anexo A documenta que
  ambas as versões coexistem sem hierarquia. O GATE 1 do módulo
  pipeline-ingestao está bloqueado até PS + Diretoria Comercial definirem
  qual versão é indexada. Isso é um impedimento real registrado no board.

---

## Evidência de Uso — Claude Code

### Iteração 1

**Prompt enviado:**
"Gere um workflow de desenvolvimento AI First para o projeto NovaTech
Assistant com 5 papéis: DM, PS, Dev, TL e QA. Mapeie as ferramentas
usadas por cada papel em cada fase."

**Resultado:** Tabela genérica sem distinção real de ferramentas por papel.
Todos os papéis apareciam usando "IA" sem especificidade.

### Iteração 2

**Refinamento:**
"Revise o workflow. Copilot é exclusivo para papéis técnicos (Dev, TL, QA).
Claude e Cowork são as ferramentas do DM e PS. Adicione a regra de que
nenhum papel usa tudo ao mesmo tempo sem distinção de propósito."

**Resultado:** Tabela com diferenciação clara, mas gates sem SLAs numéricos.

### Iteração 3

**Refinamento:**
"Adicione SLAs numéricos a cada gate. Inclua o cenário de ausência do DM
e o protocolo quando uma spec muda após aprovação. Referencie o conflito
PROC-042 como exemplo de impedimento real no GATE 1."

**Resultado final:** Versão atual com SLAs (1 dia, 4h, 2h), protocolo de
delegação e change management diferenciado por porte de impacto.

**O que mudou entre iterações:**
- SLAs numéricos adicionados após perceber que "prazo razoável" não é
  executável por agente
- Change management separado em pequeno/grande após simular o cenário
  real do conflito de versões da PROC-042
- Limitação do Copilot sem contexto MCP documentada explicitamente após
  reconhecer que o workflow descrevia um estado futuro, não o atual
