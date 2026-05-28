---
tipo: evidencia-de-uso
ferramenta: Claude Code (CLI) + Agentes Paralelos
papel: Product Specialist
fase: 1
autora: Luciane Baldo
empresa: DB1 Global Software
data: 2026-05-28
entregaveis:
  - deliverables/ps/ex-1.1-context-engineering.md
  - deliverables/ps/ex-1.2-jornada-atendente.md
  - deliverables/ps/ex-1.3-requisitos-rag.md
---

# Evidência de Uso — Claude Code como Ferramenta de Iteração
## Papel: Product Specialist | Fase 1

> Este documento registra o histórico de decisões, iterações e refinamentos
> realizados com o **Claude Code** durante a produção dos entregáveis do papel
> Product Specialist, demonstrando o uso da ferramenta como par de trabalho
> — não como gerador acrítico de conteúdo.

---

## 1. Planejamento Estratégico Antes da Execução

**Ação:** Antes de produzir qualquer linha dos exercícios, foi solicitado ao Claude Code que analisasse os três exercícios do PS e propusesse a melhor forma de estruturá-los considerando os princípios de token economy, janela de contexto, temperatura e grounding.

**Prompt de entrada:**
> *"Tendo como premissa o adequado e bom uso dos conceitos e fundamentos de IA generativa de tokens, janelas de contexto, temperatura, grounding — me dê a melhor forma de criar e estruturar o exercício para o papel de Product Specialist contido no arquivo exercicio-fase-1-entendimento.md"*

**Diagnóstico gerado pelo Claude Code:**

| Problema identificado | Impacto se não tratado |
|-----------------------|------------------------|
| Arquivo monolítico com 3 exercícios misturados | Impossibilita reusar etapas isoladamente; chunk gigante no RAG |
| Ex 1.1 é sobre context engineering — o entregável precisa *espelhar* a técnica | Contradição entre conteúdo e forma |
| Temperatura não diferenciada por exercício | Ex 1.3 (spec formal) e Ex 1.1 (análise exploratória) têm naturezas opostas |
| Contexto do cenário repetido em cada arquivo | ~500 tokens desperdiçados por leitura |

**Decisão tomada:** estrutura de 3 arquivos separados, um por exercício, com frontmatter YAML, temperatura explícita por arquivo e referências ao invés de repetição de contexto.

---

## 2. Estrutura Adotada para os Entregáveis PS

```
deliverables/ps/
├── ex-1.1-context-engineering.md   ← log de iteração em 3 etapas
├── ex-1.2-jornada-atendente.md     ← jornada textual + diagrama ASCII
├── ex-1.3-requisitos-rag.md        ← spec v1 + feedback + spec v2
└── evidencia-uso-claude-code.md    ← este arquivo
```

**Decisão de temperatura por exercício:**

| Arquivo | Temperatura | Justificativa |
|---------|-------------|---------------|
| ex-1.1 | 0.3 | Análise exploratória admite variação criativa para identificar padrões não óbvios |
| ex-1.2 | 0.1 | Jornada com guardrails — imprecisão pode gerar comportamento incorreto do assistente |
| ex-1.3 | 0.1 | Especificação formal — ambiguidade invalida o requisito |

---

## 3. Exercício 1.1 — Engenharia de Contexto em 3 Etapas

### Decisão de contexto (planejamento prévio documentado)

O Ex 1.1 exige demonstrar **progressive disclosure** — a técnica de fornecer contexto em etapas crescentes em vez de tudo de uma vez. A decisão de quanto fornecer em cada etapa foi planejada e documentada antes da execução:

| Etapa | Tokens fornecidos | O que NÃO foi fornecido e por quê |
|-------|------------------|-----------------------------------|
| 1 | ~420 tokens (só metadados) | Conteúdo completo (~15K tokens) — saturaria o orçamento de atenção; modelo tenderia a sumarizar em vez de analisar |
| 2 | ~3.200 tokens (2 docs completos) | Os outros 3 docs — a etapa 1 já indicou onde estava o risco principal; foco cirúrgico gera análise mais profunda |
| 3 | ~5.400 tokens (outputs ant. + FAQ) | FAQ na etapa 1 — sem o mapa de contradições, o cruzamento seria superficial |

### Iteração crítica: documentos contraditórios identificados

A análise da Etapa 2 gerou a tabela comparativa entre PROC-042 v1 e v2, revelando divergências em todas as dimensões:

| Dimensão | v1 (mar/2023) | v2 (nov/2023) | Variação |
|----------|--------------|--------------|----------|
| Multiplicador Norte | 1.6 | 1.8 | +12,5% |
| Multiplicador Sul | 1.2 | 1.3 | +8,3% |
| Fator peso 1.001–3.000kg | 1.2 | 1.15 | -4,2% |
| Prazo adicional | +2 dias | +3 dias | +50% |
| Threshold de desconto | > 10 fretes/mês | ≥ 8 fretes/mês | diferente |

**Insight surgido na Etapa 3 (impossível sem as etapas anteriores):**
O cruzamento com o FAQ revelou que **gaps são mais perigosos que contradições**. Uma contradição é visível — o modelo pode ser instruído a sinalizar quando dois chunks divergem. Uma informação que existe *apenas* no FAQ informal não tem "oponente" — o modelo a apresenta com confiança total, mesmo sendo prática não validada por Compliance.

### Reflexão documentada no entregável

O entregável inclui análise comparativa entre abordagem progressiva e "tudo de uma vez":

- Abordagem progressiva usou 3% dos tokens na primeira análise
- Separou limpo o referencial normativo do informal
- Produziu saídas rastreáveis por etapa
- O insight mais valioso (gaps > contradições) surgiu na Etapa 3 — seria improvável em prompt único com 15.000 tokens

---

## 4. Exercício 1.2 — Jornada do Atendente

### Iterações de refinamento

**Decisão de design — context rot no fluxo de feedback:**

O exercício exigia documentar o fluxo de feedback. Durante a produção, foi identificado que sessões longas no Teams criam um risco específico de context rot que precisa ser mitigado no *design* do fluxo — não apenas mencionado como risco.

**Decisão tomada e documentada no entregável:**
1. Feedback é uma ação atômica isolada — abre nova chamada independente ao backend, sem herdar o histórico da sessão
2. Limite de contexto configurado: o assistente descarta mensagens com mais de 10 turnos, mantendo system prompt + RAG atual + últimas 3 trocas

**Decisão de design — threshold de confiança no fallback:**

O fluxo de fallback poderia simplesmente dizer "não encontrei". A versão final documenta a lógica de threshold (0,72) e o custo operacional de não tê-lo:

> *"Em escala de 45 atendentes com centenas de chamados diários, loops de retry sem critério de parada representam custo operacional e latência relevantes. O threshold único (score < 0,72 → fallback imediato) é a proteção arquitetural contra esse desperdício."*

### Guardrails específicos ao domínio

Os dois guardrails foram desenvolvidos com motivação explícita ligada ao domínio NovaTech — não são genéricos:

| Guardrail | Motivação específica ao domínio |
|-----------|--------------------------------|
| Nunca afirmar prazo/valor sem citação de fonte | Prazos incorretos geram SLA breach e penalidade contratual com clientes |
| Sinalizar fonte FAQ informal | FAQ pode conter regras de frete revogadas por atualização contratual não documentada |

---

## 5. Exercício 1.3 — Especificação de Requisitos de RAG

### Ciclo de iteração documentado

**Processo:** Spec v1 produzida → Claude usado como revisor → 7 críticas recebidas → decisões documentadas → Spec v2 com 8 requisitos.

**Spec v1:** 5 requisitos, linguagem vaga, sem critérios de teste, sem processo de curadoria.

**Críticas recebidas do Claude (seleção das mais impactantes):**

> **C2 — REQ-01:** "O requisito exclui PDFs escaneados sem OCR, mas não define o critério de qualidade do OCR. Uma extração com 70% de precisão é aceita? Cria um grupo de documentos em estado indefinido."

> **C4 — REQ-02:** "'Indexar apenas a versão mais recente' ignora que a seção 5 do PROC-042-v2 define regras transitórias: chamados abertos antes de 01/12/2023 ainda usam a v1. Excluir a v1 completamente quebra esse requisito de negócio."

> **C7 — REQ-05:** "Citar o documento de origem é necessário, mas não suficiente. O atendente precisa saber qual trecho foi usado — não apenas qual documento. Sem o trecho, a verificação é impraticável para documentos longos."

**O que o Claude identificou que não estava na v1:**
- Ausência de processo de curadoria com gate humano (virou REQ-07)
- Necessidade de revisão periódica de thresholds (virou REQ-08)
- Impacto de tokens nas citações como requisito testável (incorporado ao REQ-06)

**Evolução entre versões:**

| Dimensão | Spec v1 | Spec v2 |
|----------|---------|---------|
| Número de requisitos | 5 | 8 |
| Requisitos com critério de teste | 0 | 8 (100%) |
| Threshold numérico definido | Não | Sim (0.75, com faixas) |
| SLA por tipo de documento | Único (48h) | 6 tipos distintos |
| Gate humano antes de indexar | Não mencionado | REQ-07 completo |
| Revisão periódica de parâmetros | Não mencionado | REQ-08 com ciclos definidos |

---

## 6. Sessões Isoladas por Exercício

**Ação:** Os três exercícios foram produzidos em sessões de agente separadas e paralelas, com contextos completamente isolados.

**Por que isso importa:**

Esta sessão principal acumula histórico extenso — leituras de arquivos, commits, análises, iterações do papel DM. Se os exercícios do PS fossem produzidos aqui, cada resposta processaria todo esse histórico, com dois riscos:

1. **Context rot:** instruções e decisões tomadas no início da sessão perdem peso relativo
2. **Contaminação de contexto:** raciocínio sobre os exercícios do DM poderia influenciar os do PS

**Contexto enviado para cada agente (isolado e auto-suficiente):**

| Agente | Conteúdo do prompt | Tokens estimados |
|--------|-------------------|-----------------|
| Ex 1.1 | Metadados dos 5 docs + enunciado + conceitos a demonstrar | ~1.200 tokens |
| Ex 1.2 | Cenário + dados do discovery + enunciado + guardrails obrigatórios | ~900 tokens |
| Ex 1.3 | Contexto do projeto + problemas conhecidos + enunciado + requisitos obrigatórios | ~1.100 tokens |

**Resultado:** cada agente produziu conteúdo focado, sem herdar viés de outros exercícios ou da sessão principal.

---

## 7. Histórico de Commits — Rastreabilidade por Exercício

| Hash | Mensagem | Conteúdo |
|------|----------|---------|
| `9fdc0ba` | PS Ex 1.1: mapeamento de intent com progressive disclosure (3 etapas) | Criação inicial Ex 1.1 |
| `ecf6327` | PS Ex 1.2: jornada do atendente - fluxo principal, fallback e feedback loop | Criação inicial Ex 1.2 |
| `b5628e4` | PS Ex 1.3: especificacao de requisitos RAG - spec v1, feedback e spec v2 final | Criação inicial Ex 1.3 |
| `e4330cb` | PS Ex 1.2 v2: jornada atendente - sessao isolada | Versão refinada Ex 1.2 |
| `d58b68a` | PS Ex 1.3 v2: spec RAG 8 requisitos - sessao isolada | Versão refinada Ex 1.3 |

Commits separados por exercício permitem rastrear a evolução individual de cada entregável sem misturar mudanças.

---

## 8. Resumo das Iterações

| # | Ação | Ferramenta | Resultado |
|---|------|-----------|-----------|
| 1 | Planejamento estratégico da estrutura PS | Claude Code (análise) | Diagnóstico de problemas + estrutura de 3 arquivos separados |
| 2 | Definição de temperatura por exercício | Claude Code (raciocínio) | 0.3 para análise exploratória; 0.1 para spec formal e jornada |
| 3 | Produção Ex 1.1 com log das 3 etapas | Claude Code | Progressive disclosure documentado com tokens por etapa |
| 4 | Análise comparativa PROC-042 v1 vs v2 | Claude Code (Etapa 2) | Tabela com 5 divergências e variações percentuais |
| 5 | Cruzamento FAQ × normativos | Claude Code (Etapa 3) | Insight: gaps são mais perigosos que contradições |
| 6 | Produção Ex 1.2 com 3 fluxos | Claude Code | Fluxo com mitigação de context rot e threshold documentado |
| 7 | Produção Ex 1.3 spec v1 | Claude Code | 5 requisitos iniciais |
| 8 | Claude como revisor da spec v1 | Claude Code (crítico) | 7 críticas específicas; 2 novos requisitos identificados |
| 9 | Refinamento para spec v2 | Claude Code | 8 requisitos, todos testáveis, com conceito IA First explícito |
| 10 | Sessões isoladas por exercício | 3 Agentes paralelos | Contextos independentes; outputs sem contaminação cruzada |
| 11 | Commits separados por exercício | Git | Rastreabilidade individual de cada entregável |

---

## 9. Reflexão sobre o Uso da Ferramenta

Os exercícios do PS foram produzidos com três modos distintos de uso do Claude Code:

**Modo 1 — Claude como analisador (Ex 1.1):**
Fornecimento progressivo de contexto para mapear contradições e gaps. O modelo não recebeu tudo de uma vez — recebeu o que precisava em cada etapa, gerando análise mais profunda do que seria possível com um único prompt saturado.

**Modo 2 — Claude como designer (Ex 1.2):**
Produção de jornada com decisões técnicas explícitas: temperatura escolhida, lógica de threshold documentada, context rot mitigado no design do fluxo. O entregável não apenas descreve o que o assistente faz — justifica as escolhas com base nos fundamentos de IA generativa.

**Modo 3 — Claude como revisor (Ex 1.3):**
A spec v1 foi intencionalmente produzida com limitações para que o Claude pudesse atuar como revisor crítico. As críticas recebidas geraram dois requisitos inteiramente novos (REQ-07 e REQ-08) que não existiam na versão inicial. A iteração é verificável: a spec v2 tem 60% mais requisitos e 100% deles são testáveis.

**Princípio demonstrado em todos os exercícios:**
O Claude Code foi usado como par de trabalho iterativo — não como oráculo. Em nenhum momento o output inicial foi aceito sem análise crítica. A qualidade dos entregáveis é resultado das iterações, não do primeiro prompt.
