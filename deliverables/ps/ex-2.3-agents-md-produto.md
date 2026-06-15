---
exercise: 2.3
title: AGENTS.md — Product Rules & Guardrails — NovaTech Assistant
role: Product Specialist
phase: Estruturação
status: done
author: Luciane Baldo
created_at: 2026-06-11
cowork_card: NOVA-33
---

## Objetivo

Produzir a seção "Product Rules & Guardrails" do AGENTS.md do projeto
NovaTech Assistant — prescritiva, machine-readable, com glossário de
linguagem ubíqua e restrições que influenciam geração de código.
Consistente com os bounded contexts do Ex. 2.1 e guardrails do Ex. 2.2.

---

## Conteúdo para /docs/AGENTS.md (seção de produto)

---

## Product Rules & Guardrails

> Seção de responsabilidade do Product Specialist.
> Regras aqui definem comportamento do assistente e restrições de
> geração de código. Atualizada pelo PS a cada alteração de guardrail
> aprovada via GATE 1 do módulo query-endpoint.

### Comportamentos Obrigatórios (DEVE)

- Toda resposta DEVE citar o documento fonte no formato:
  `[NOME-DOC, seção N]`. Exemplos válidos: `[SLA-2024, seção 2]`,
  `[POL-001 v3.1, seção 3]`, `[PROC-042 v2, seção 2.1]`.
- Toda resposta da API DEVE retornar `source_document` no payload JSON.
  Campo obrigatório mesmo em recusas e respostas de baixa confiança.
- Quando múltiplas versões de documento estiverem disponíveis,
  a resposta DEVE declarar qual versão foi usada e qual é a vigente.
  Formato: `"Usando PROC-042 v2 (vigente desde 01/12/2023)."`
- Respostas DEVEM estar em português formal. Sem gírias, abreviações
  informais ou linguagem de chat.
- Quando confiança for baixa (score abaixo do threshold configurado
  em `/config/retrieval-thresholds.json`), a resposta DEVE iniciar
  com: `"Resposta com baixa confiança. Recomendo verificar com supervisor."`

### Comportamentos Proibidos (NÃO DEVE)

- O assistente NÃO DEVE gerar valores numéricos (prazos, multiplicadores,
  percentuais, valores monetários) que não estejam literalmente nos
  documentos indexados em `/data/documents/`.
- O assistente NÃO DEVE afirmar que carga perigosa (classes 1–6 ANTT)
  pode seguir o processo padrão de devolução (POL-001). DEVE sempre
  encaminhar para ramal 4500 (Gestão de Riscos).
- O assistente NÃO DEVE mencionar tiers além de `Gold`, `Silver` e
  `Standard`. Qualquer menção a `Platinum`, `Diamond` ou outro tier
  DEVE ser refutada com os tiers válidos listados.
- O assistente NÃO DEVE usar multiplicadores da PROC-042 v1 para
  pedidos com data de abertura posterior a 01/12/2023.
- O assistente NÃO DEVE responder sobre: seguro de carga, danos em
  trânsito, concessão de descontos, aprovação de carga > 5.000kg,
  ou reclassificação de tier. DEVE nomear o canal correto:
  - Seguro / danos: sinistros@novatech.com.br
  - Descontos: Comercial
  - Carga > 5.000kg: gerente regional de operações
  - Reclassificação de tier: Comercial

### Fallbacks (QUANDO EM DÚVIDA)

- Quando múltiplas versões de documento estiverem disponíveis: usar
  a mais recente e notificar o atendente para confirmar data do pedido.
- Quando score de retrieval estiver abaixo do threshold: prefixar
  resposta com aviso de baixa confiança (ver acima).
- Quando não encontrar resposta nos documentos indexados: NÃO dizer
  apenas "não encontrei". DEVE indicar: (1) que o assunto pertence
  ao domínio NovaTech, (2) o canal de escalação adequado,
  (3) resposta parcial com nível de confiança se disponível.

### Restrições que Impactam Geração de Código

Agentes que geram código para o módulo `query-endpoint` DEVEM seguir:

```typescript
// Schema obrigatório de resposta — toda implementação de output pipeline
interface QueryResponse {
  answer: string;
  source_document: string;      // obrigatório — nunca null
  confidence: "high" | "medium" | "low";
  escalation: string | null;    // canal de escalação quando aplicável
  version_used?: string;        // ex: "PROC-042 v2" quando aplicável
}

// Validação obrigatória antes de enviar resposta ao Teams
function validateResponse(r: QueryResponse): boolean {
  if (!r.source_document) throw new Error("source_document obrigatório");
  if (!["high","medium","low"].includes(r.confidence))
    throw new Error("confidence inválido");
  return true;
}
```

- O filtro de carga perigosa DEVE ser implementado no pre-processing,
  antes da consulta ao LLM. Detectar entidades: "carga perigosa",
  "ANTT", "classes 1", "classes 2", "classes 3", "classes 4",
  "classes 5", "classes 6", "material perigoso", "produto químico".
  Se detectado, encaminhar diretamente para ramal 4500 sem consultar o LLM.
- O seletor de versão PROC-042 DEVE usar metadado `order_date` do
  pedido: se `order_date < 2023-12-01`, recuperar PROC-042 v1;
  caso contrário, PROC-042 v2.
- Threshold de confiança configurado em `/config/retrieval-thresholds.json`.
  Valor padrão inicial: `0.75`. Ajustar após calibração com QA.

### Glossário de Linguagem Ubíqua

Termos que um LLM confundiria sem definição explícita.
Agentes DEVEM usar estas definições ao interpretar consultas.

| Termo | Definição no contexto NovaTech |
|---|---|
| `Gold` | Tier de cliente: contrato anual > R$500k OU > 200 ops/mês. SLA: resposta 2h, resolução 24h, incidente crítico 30min/4h. Gestor exclusivo: sim. |
| `Silver` | Tier: contrato R$100k–R$500k OU 50–200 ops/mês. SLA: 4h/48h, crítico 1h/8h. |
| `Standard` | Tier: todos os demais. SLA: 8h/72h, crítico 2h/24h. |
| `Carga perigosa` | Materiais das classes 1–6 ANTT. Processo especial obrigatório. Nunca elegível para devolução padrão. |
| `Frete especial` | Carga > 500kg. Calculado por: base x multiplicador regional (PROC-042 v2) x fator de peso. |
| `Incidente crítico` | Carga > R$100k com status desconhecido > 6h; irregularidade em carga perigosa; múltiplos chamados recorrentes. |
| `Data de recebimento confirmada` | Timestamp de confirmação no sistema de tracking (portal.novatech.com.br). Base para prazo de devolução. |
| `CT-e` | Conhecimento de Transporte Eletrônico. Documento obrigatório para abertura de devolução. |
| `PROC-042 v2` | Versão vigente desde 01/12/2023. Multiplicadores: Sul 1.3, Sudeste 1.1, C-O 1.4, NE 1.5, Norte 1.8. Prazo: rota + 3 dias úteis. |
| `PROC-042 v1` | Versão anterior. Usar apenas para pedidos abertos antes de 01/12/2023. Multiplicadores: Sul 1.2, Sudeste 1.0, C-O 1.3, NE 1.4, Norte 1.6. |
| `Dias úteis` | Dias úteis excluem fins de semana e feriados nacionais. |
| `Ramal 4500` | Contato direto com Gestão de Riscos NovaTech. Obrigatório para: cargas perigosas, exceções de devolução, situações de segurança. |
| `Platinum` | Tier INEXISTENTE na NovaTech. Responder com os tiers válidos: Gold, Silver, Standard. |

### Referências a Specs no Repositório

- Requisitos do query-endpoint: `/specs/query-endpoint/requirements.md`
- Guardrails formalizados completos: `deliverables/ps/ex-2.2-guardrails.md`
- Bounded contexts e linguagem ubíqua: `deliverables/ps/ex-2.1-recorte-dominio.md`
- Política de devolução: `/data/documents/POL-001-v3.1.md`
- Frete especial vigente: `/data/documents/PROC-042-v2.md`
- SLA contratual: `/data/documents/SLA-2024.md`
- Thresholds de retrieval: `/config/retrieval-thresholds.json`

### Atualização desta Seção

- Toda alteração em guardrail DEVE passar por GATE 1 do módulo
  `query-endpoint` antes de ser incorporada a este documento.
- Commit de alteração nesta seção DEVE ter formato:
  `docs(AGENTS.md): guardrail <ID> atualizado — NOVA-<id>`.
- Nenhum guardrail pode ser removido sem aprovação do DM registrada
  no card Cowork.

---

## Limitações desta Seção

- O threshold de confiança (GR-05/GR-12) está definido como 0.75
  como valor inicial. Valor definitivo depende de calibração com QA
  após testes com consultas reais. Registrado como decisão pendente
  em NOVA-33.
- O filtro de NER para carga perigosa (pre-processing) depende de
  implementação pelo Dev com o glossário acima. Até estar configurado,
  GR-07 depende apenas do prompt (probabilístico).
- Esta seção cobre apenas o contexto do Product Specialist. As seções
  de Tech Stack, Coding Standards, Testing Standards e Project
  Management Rules são responsabilidade de outros papéis e estão
  documentadas em seus respectivos entregáveis.

---

## Evidência de Uso — Claude Code

### Iteração 1

**Prompt enviado:**
"Escreva a seção Product Rules & Guardrails de um AGENTS.md para
um assistente de IA usando os guardrails que definimos."

**Resultado:** Texto narrativo descrevendo os guardrails. Frases como
"o assistente deve tentar citar fontes quando possível" — não
executável por agente.

### Iteração 2

**Refinamento:**
"Reescreva em formato prescritivo puro: DEVE / NÃO DEVE / QUANDO EM
DÚVIDA. Cada regra em bullet com sujeito explícito ('O assistente NÃO
DEVE...'). Sem linguagem descritiva ou narrativa."

**Resultado:** Tom prescritivo correto, mas sem glossário estruturado
e sem restrições de código concretas para o Copilot.

### Iteração 3

**Refinamento:**
"Adicione: (1) seção de glossário com tabela de termos e definições
precisas extraídas dos documentos NovaTech, (2) seção de restrições
de código com interface TypeScript obrigatória do QueryResponse,
(3) referências a caminhos concretos do repositório."

**Resultado final:** Versão atual com glossário, interface TypeScript
e referências de repositório concretas.

**O que mudou entre iterações:**
- Tom narrativo corrigido para prescritivo na iteração 2 após
  perceber que "deve tentar" não é executável por agente (D3 flag)
- Interface TypeScript adicionada após reconhecer que guardrails
  sem restrição de schema não influenciam o Copilot na geração
  de código (D1 gap)
- Limitação do threshold e NER documentada com honestidade na
  seção de limitações
