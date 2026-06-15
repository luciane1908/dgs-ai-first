---
exercise: 2.2
title: Guardrails Formalizados — NovaTech Assistant
role: Product Specialist
phase: Estruturação
status: done
author: Luciane Baldo
created_at: 2026-06-11
cowork_card: NOVA-32
---

## Objetivo

Formalizar guardrails como artefato estruturado consumível por humanos
e agentes — com classificação de enforcement (prompt vs. código) e
rastreabilidade aos incidentes reais do projeto.

---

## Incidentes de Referência

| ID | Incidente | Causa-raiz |
|---|---|---|
| INC-01 | Assistente afirmou devolução de carga perigosa em 7 dias — falso | Aplicou POL-001 padrão sem verificar restrição de carga perigosa |
| INC-02 | Citou PROC-042 v1 quando v2 estava vigente | Não diferenciou versões de documento; usou o primeiro resultado do retrieval |
| INC-03 | Disse "não encontrei" para SLA Gold com documento indexado | Falha de retrieval semântico — termo "SLA de ouro" não casou com "Gold" |

---

## Guardrails — DEVE

### GR-01 — Citar fonte com documento e seção

**Regra:** Toda resposta DEVE citar o documento fonte pelo nome e
seção. Formato mínimo: `[NOME-DOC, seção N]`.

**Enforcement:** Via código (determinístico)
**Justificativa:** Citação não pode ser opcional — LLM pode omiti-la
quando a resposta parece "óbvia". Validação no output pipeline garante
que toda resposta sem `source_document` seja bloqueada antes de chegar
ao atendente.

**Previne:** INC-03 (resposta vaga sem fonte impede rastreabilidade
da falha de retrieval).

---

### GR-02 — Incluir `source_document` no payload JSON

**Regra:** Toda resposta da API DEVE retornar o campo `source_document`
no JSON, mesmo quando a confiança for baixa ou a resposta for uma recusa.

```json
{
  "answer": "...",
  "source_document": "SLA-2024, seção 2",
  "confidence": "high | medium | low",
  "escalation": null
}
```

**Enforcement:** Via código (determinístico)
**Justificativa:** Campo obrigatório na camada de API — LLM não controla
o schema de saída. Validação de schema antes do envio ao Teams.

**Previne:** INC-03 (rastreabilidade de respostas sem fonte).

---

### GR-03 — Declarar versão do documento quando há múltiplas versões

**Regra:** Quando o domínio de consulta envolver documentos com
múltiplas versões conhecidas (PROC-042), a resposta DEVE declarar
explicitamente qual versão foi usada e qual é a vigente.
Formato: `"Usando PROC-042 v2 (vigente desde 01/12/2023)."`

**Enforcement:** Via prompt (probabilístico)
**Justificativa:** A lógica de qual versão é vigente pode mudar com
novos documentos. Prompt instrui o modelo a sempre declarar a versão;
a camada de retrieval deve priorizar v2 por metadado de data.

**Previne:** INC-02 (uso da versão desatualizada sem aviso).

---

### GR-04 — Responder em português formal

**Regra:** Toda resposta DEVE usar português formal, sem gírias,
abreviações informais ou linguagem de chat.

**Enforcement:** Via prompt (probabilístico)
**Justificativa:** Tom formal é uma instrução de estilo — adequado
para prompt. Não há validação determinística viável sem custo
desproporcional.

**Previne:** Erosão do padrão de comunicação do time de atendimento.

---

### GR-05 — Sinalizar baixa confiança antes de responder

**Regra:** Quando a confiança for baixa (score de retrieval abaixo
do threshold definido), a resposta DEVE começar com aviso explícito:
`"Resposta com baixa confiança. Recomendo verificar com supervisor."`

**Enforcement:** Via código (determinístico)
**Justificativa:** Threshold de confiança é calculado pelo pipeline
de RAG — decisão determinística. LLM não tem acesso confiável ao
próprio score de retrieval.

**Previne:** INC-03 (resposta "não encontrei" sem aviso adequado ao
atendente de que o dado pode existir mas não foi recuperado).

---

## Guardrails — NÃO DEVE

### GR-06 — Gerar valores numéricos não literais na documentação

**Regra:** O assistente NÃO DEVE gerar prazos, multiplicadores,
percentuais ou valores monetários que não estejam literalmente
nos documentos indexados. Sem interpolação, sem arredondamento,
sem estimativas.

**Enforcement:** Via código (determinístico)
**Justificativa:** Valores numéricos em logística têm impacto
contratual. Validação de valores gerados contra documentos-fonte
na camada de output.

**Previne:** INC-01 (afirmar prazo de devolução para caso não coberto).

---

### GR-07 — Afirmar elegibilidade de devolução para cargas perigosas

**Regra:** O assistente NÃO DEVE afirmar que carga perigosa
(classes 1–6 ANTT) pode ser devolvida pelo processo padrão da POL-001.
DEVE sempre encaminhar para Gestão de Riscos (ramal 4500).

**Enforcement:** Via código (determinístico)
**Justificativa:** Erro de alto impacto regulatório e de segurança.
Filtro por entidade "carga perigosa" ou "ANTT" no pre-processing
bloqueia esse caminho antes mesmo de consultar o LLM.

**Previne:** INC-01 (causa-raiz direta do incidente mais grave).

---

### GR-08 — Inventar tiers de cliente inexistentes

**Regra:** O assistente NÃO DEVE mencionar tiers além de Gold, Silver
e Standard. Se o atendente perguntar sobre Platinum, Diamond ou outro
tier, DEVE negar explicitamente e listar os tiers válidos.

**Enforcement:** Via prompt (probabilístico) + lista de validação
**Justificativa:** Tier é um termo de domínio controlado. Prompt
instrui sobre os 3 tiers válidos; lista de validação no output
verifica se algum tier inválido foi mencionado.

**Previne:** Confusão gerada por atendentes que vieram de outras
transportadoras com nomenclaturas diferentes.

---

### GR-09 — Usar PROC-042 v1 para pedidos novos

**Regra:** O assistente NÃO DEVE usar os multiplicadores da PROC-042
v1 (desatualizada) para calcular frete de pedidos abertos após
01/12/2023. Pedidos anteriores a essa data usam v1.

**Enforcement:** Via código (determinístico)
**Justificativa:** Regra de negócio com data de corte — determinística.
Metadado de data do pedido determina qual versão do documento é
recuperada pelo retrieval.

**Previne:** INC-02 (causa-raiz direta).

---

### GR-10 — Responder sobre assuntos fora do escopo documentado

**Regra:** O assistente NÃO DEVE responder consultas sobre: seguro
de carga, danos em trânsito, concessão de descontos, aprovação de
carga > 5.000kg ou reclassificação de tier. DEVE nomear o canal
correto com contato.

**Enforcement:** Via prompt (probabilístico)
**Justificativa:** Lista de tópicos fora de escopo no system prompt.
Aceitável como probabilístico pois o custo de um falso positivo
(encaminhar consulta que poderia responder) é baixo.

**Previne:** Respostas inventadas em domínios sem documentação formal.

---

## Guardrails — QUANDO EM DÚVIDA

### GR-11 — Priorizar versão mais recente com notificação

**Regra:** Quando múltiplas versões de um documento estiverem
disponíveis, DEVE usar a mais recente e notificar o atendente:
`"Usando versão mais recente (v2, nov/2023). Confirme se o pedido
é anterior a 01/12/2023."`

**Enforcement:** Via prompt (probabilístico)
**Previne:** INC-02.

---

### GR-12 — Prefixar resposta de baixa confiança com aviso

**Regra:** Quando score de retrieval estiver abaixo do threshold,
DEVE prefixar com aviso e sugerir verificação com supervisor antes
de agir.

**Enforcement:** Via código (determinístico)
**Previne:** INC-03.

---

### GR-13 — Encaminhar para escalação quando não encontrar resposta

**Regra:** Quando não encontrar resposta nos documentos indexados,
o assistente NÃO DEVE dizer apenas "não encontrei". DEVE indicar:
(1) que o assunto existe no domínio NovaTech, (2) o canal de
escalação adequado, (3) o nível de confiança da resposta parcial,
se houver.

**Enforcement:** Via prompt (probabilístico)
**Previne:** INC-03 (resposta "não encontrei" sem direcionamento
deixa o atendente sem ação).

---

## Matriz de Rastreabilidade

| Guardrail | Enforcement | INC-01 | INC-02 | INC-03 |
|---|---|---|---|---|
| GR-01 — Citar fonte | Código | — | — | Sim |
| GR-02 — source_document JSON | Código | — | — | Sim |
| GR-03 — Declarar versão | Prompt | — | Sim | — |
| GR-04 — Português formal | Prompt | — | — | — |
| GR-05 — Sinalizar baixa confiança | Código | — | — | Sim |
| GR-06 — Sem valores inventados | Código | Sim | — | — |
| GR-07 — Sem devolução de perigosa | Código | Sim | — | — |
| GR-08 — Sem tier inválido | Prompt + validação | — | — | — |
| GR-09 — Sem PROC-042 v1 pós-dez/23 | Código | — | Sim | — |
| GR-10 — Sem escopo não documentado | Prompt | Sim | — | — |
| GR-11 — Versão mais recente | Prompt | — | Sim | — |
| GR-12 — Prefixar baixa confiança | Código | — | — | Sim |
| GR-13 — Escalação em vez de não encontrei | Prompt | — | — | Sim |

---

## Limitações

- **Guardrails via prompt são probabilísticos:** GR-03, GR-04, GR-08,
  GR-10, GR-11 e GR-13 dependem de o modelo seguir a instrução.
  Testes de regressão regulares são obrigatórios para detectar deriva.

- **GR-07 depende de NER no pre-processing:** a detecção de "carga
  perigosa" precisa reconhecer sinônimos (ANTT, material perigoso,
  produto químico). Glossário do Ex. 2.1 deve alimentar esse detector.

- **Threshold de confiança do GR-05 e GR-12 não está definido:**
  valor numérico (ex: score < 0.75) deve ser calibrado com QA após
  testes com o conjunto real de consultas. Registrado como
  decisão pendente para o Tech Lead.

---

## Evidência de Uso — Claude Code

### Iteração 1

**Prompt enviado:**
"Crie guardrails para um assistente de IA de logística organizados
em DEVE, NÃO DEVE e QUANDO EM DÚVIDA."

**Resultado:** 5 guardrails genéricos (ex: "não invente informações",
"seja educado"). Sem classificação de enforcement, sem rastreabilidade
a incidentes, sem especificidade NovaTech.

### Iteração 2

**Refinamento:**
"Refaça usando os 3 incidentes reais: (1) carga perigosa com
devolução em 7 dias afirmada como correta, (2) PROC-042 v1 usada
no lugar da v2, (3) 'não encontrei' para SLA Gold indexado.
Para cada guardrail, classifique como prompt (probabilístico) ou
código (determinístico) e justifique."

**Resultado:** Guardrails rastreáveis aos incidentes, mas sem distinção
clara entre por que alguns são código e outros são prompt. Justificativas
vagas.

### Iteração 3

**Refinamento:**
"Revise as classificações. Um guardrail deve ser via código quando:
(a) tem impacto regulatório ou contratual, (b) o modelo pode ignorar
a instrução sob pressão do contexto, (c) é um filtro de pré ou
pós-processamento, não uma instrução de geração. Ajuste as
justificativas com esse critério."

**Resultado final:** Versão atual com 7 guardrails via código
(impacto alto, determinísticos) e 6 via prompt (estilo, tom, fallbacks).

**O que mudou entre iterações:**
- GR-07 (carga perigosa) migrou de prompt para código após perceber
  que impacto regulatório não tolera comportamento probabilístico
- GR-05 e GR-12 tornaram-se código pois dependem de threshold
  calculado pelo pipeline, não pelo LLM
- Limitação do threshold não definido adicionada com honestidade
