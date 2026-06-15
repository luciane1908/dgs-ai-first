---
exercise: 2.1
title: Recorte de Domínio e Spec SDD — NovaTech Assistant
role: Product Specialist
phase: Estruturação
status: done
author: Luciane Baldo
created_at: 2026-06-11
cowork_card: NOVA-30
---

## Objetivo

Delimitar os bounded contexts do assistente NovaTech, extrair linguagem
ubíqua do domínio logístico e formalizar o requirements.md do módulo
query-endpoint no formato SDD.

---

## 1. Mapa de Bounded Contexts

O assistente NovaTech opera sobre 4 bounded contexts distintos,
derivados das categorias de descoberta (prazos, frete, devolução, SLAs)
e dos documentos do Anexo A. A divisão é por domínio de negócio,
não por camada técnica.

---

### Context 1 — Atendimento ao Cliente

**Dentro do escopo:**
- Consulta de SLAs por tier (Gold, Silver, Standard)
- Classificação de incidentes como críticos ou gerais
- Regras de escalação: quando transferir para supervisor ou Gestão de Riscos
- Informação sobre gestor de conta exclusivo (somente Gold)

**Fora do escopo:**
- Concessão de descontos (exclusivo do Comercial)
- Reclassificação de tier de cliente
- Acesso a dados do sistema de chamados Azure DevOps

**Relacionamentos:**
- Depende do Context 2 (Frete) para calcular prazo em chamados de rastreamento
- Depende do Context 3 (Devoluções) para triagem de chamados de devolução
- Aciona Context 4 (Contratos) ao informar penalidades por descumprimento de SLA

---

### Context 2 — Frete Especial

**Dentro do escopo:**
- Cálculo de frete para cargas acima de 500kg (PROC-042 v2 vigente)
- Multiplicadores regionais por destino
- Fatores de peso por faixa (500–1.000kg, 1.001–3.000kg, acima de 3.000kg)
- Prazo de entrega especial (padrão da rota + 3 dias úteis — v2)
- Identificação de qual versão de PROC-042 se aplica ao pedido do cliente

**Fora do escopo:**
- Frete padrão para cargas abaixo de 500kg (não documentado formalmente)
- Negociação de descontos volumétricos (exclusivo do Comercial)
- Aprovação de cargas acima de 5.000kg (exclusivo do gerente regional)

**Relacionamentos:**
- Consome tabela mensal de tarifas (fonte externa — não indexada neste ciclo)
- Encaminha cargas perigosas para Context 3 (Devoluções/Riscos)

---

### Context 3 — Devoluções e Cargas de Risco

**Dentro do escopo:**
- Prazo de devolução padrão: 7 dias úteis a partir da confirmação de recebimento
- Processo de solicitação via Portal do Cliente (portal.novatech.com.br)
- Elegibilidade: motivos válidos, documentação necessária (CT-e, fotos, motivo)
- Cargas não elegíveis: perigosas (classes 1–6 ANTT), refrigeradas com cadeia
  de frio rompida, lacres violados sem documentação
- Encaminhamento para Gestão de Riscos (ramal 4500) para exceções
- Regras de reembolso por tipo de motivo

**Fora do escopo:**
- Seguro de carga e indenização por danos em trânsito
  (processo via sinistros@novatech.com.br — não documentado formalmente)
- Autorização de devolução de carga perigosa (exclusivo da Gestão de Riscos)

**Relacionamentos:**
- Recebe escalações do Context 1 (Atendimento)
- Consome restrições ANTT definidas externamente

---

### Context 4 — Contratos e Compliance

**Dentro do escopo:**
- Definição de tiers e critérios de elegibilidade (SLA-2024)
- Penalidades por descumprimento de SLA
- Informação sobre relatórios mensais de desempenho por tier

**Fora do escopo:**
- Renegociação contratual
- Emissão de notas fiscais ou documentos fiscais

**Relacionamentos:**
- Provê regras de tier para o Context 1 (Atendimento)
- Provê regras de desconto para o Context 2 (Frete)

---

## 2. Linguagem Ubíqua — Glossário do Domínio

Termos que um LLM confundiria sem definição explícita, extraídos
diretamente dos documentos NovaTech.

| Termo | Definição precisa no contexto NovaTech | Confusão comum de LLM |
|---|---|---|
| **Tier Gold** | Cliente com contrato anual > R$500k OU > 200 ops/mês (SLA-2024) | Confundir com "premium genérico" ou inventar tier Platinum |
| **Tier Silver** | Contrato entre R$100k–R$500k OU 50–200 ops/mês | Confundir faixas de valor/volume |
| **Tier Standard** | Todos os demais clientes | Tratar como "padrão" sem especificidade |
| **Carga perigosa** | Materiais das classes 1–6 conforme normas ANTT | Generalizar como "carga sensível" ou aplicar processo padrão |
| **Frete especial** | Cargas acima de 500kg — calculado pela PROC-042 v2 vigente | Confundir com frete expresso ou frete prioritário |
| **Incidente crítico** | Carga > R$100k com status desconhecido > 6h; irregularidade em carga perigosa; múltiplos chamados recorrentes | Tratar qualquer reclamação como crítica |
| **Data de recebimento confirmada** | Timestamp de confirmação no sistema de tracking (portal.novatech.com.br) | Usar data de entrega estimada ou data de nota fiscal |
| **Ramal 4500** | Contato direto com Gestão de Riscos — obrigatório para cargas perigosas e exceções | Tentar resolver sem escalação |
| **CT-e** | Conhecimento de Transporte Eletrônico — documento obrigatório para devolução | Aceitar número de pedido ou NF como substituto |
| **PROC-042 v2** | Versão vigente desde 01/12/2023. Multiplicadores: Sul 1.3, Sudeste 1.1, C-O 1.4, NE 1.5, Norte 1.8. Prazo +3 dias | Usar PROC-042 v1 (desatualizada) com multiplicadores errados |
| **Cadeia de frio rompida** | Temperatura fora da faixa especificada na NF por mais de 30 minutos | Qualquer variação de temperatura |
| **Dias úteis** | Exclui fins de semana e feriados nacionais | Confundir com dias corridos |
| **Gestor de conta exclusivo** | Somente clientes Gold têm gestor dedicado | Oferecer para Silver ou Standard |
| **Multiplicador regional** | Coeficiente de custo por destino (PROC-042 v2) — não é desconto nem taxa adicional | Somar ao valor base em vez de multiplicar |

---

## 3. requirements.md — Módulo query-endpoint

```yaml
---
module: query-endpoint
type: requirements
status: approved
version: 1.0
author: Luciane Baldo
reviewer: tech-lead
approved_by: ~
created_at: 2026-06-11
updated_at: 2026-06-11
cowork_card: NOVA-31
---
```

### Outcomes (orientados ao resultado do usuário final)

1. **Atendente resolve consulta de SLA sem abrir outra ferramenta**
   Um atendente do time NovaTech (45 pessoas) recebe a resposta correta
   sobre prazos de SLA por tier em menos de 30 segundos, com a fonte
   citada, sem precisar consultar o SLA-2024 manualmente.

2. **Atendente calcula frete especial sem erro de versão**
   Para qualquer carga acima de 500kg, o assistente aplica os
   multiplicadores da PROC-042 v2 (vigente desde 01/12/2023) e indica
   explicitamente qual versão foi usada, prevenindo o erro de usar v1.

3. **Atendente identifica elegibilidade de devolução sem interpretação**
   Para qualquer solicitação de devolução, o assistente informa se é
   elegível, o prazo (7 dias úteis da confirmação de recebimento), os
   documentos necessários e, quando não elegível, indica o motivo e
   o encaminhamento correto (ramal 4500 ou Comercial).

4. **Atendente escalona para o canal correto sem ambiguidade**
   Para qualquer caso fora do escopo do assistente (seguro de carga,
   danos em trânsito, aprovação de carga > 5.000kg), o assistente
   nomeia o canal correto com contato específico, sem tentar responder.

### Scope Boundaries

**Dentro do escopo do query-endpoint:**
- Consultas sobre SLAs dos tiers Gold, Silver e Standard (fonte: SLA-2024)
- Cálculo de frete especial para cargas > 500kg (fonte: PROC-042 v2)
- Elegibilidade e processo de devolução (fonte: POL-001 v3.1)
- Encaminhamentos para Gestão de Riscos, Comercial ou Sinistros

**Fora do escopo — encaminhar sem responder:**
- Frete para cargas < 500kg (sem documentação formal indexada)
- Seguro de carga e danos em trânsito (processo via sinistros@novatech.com.br)
- Aprovação de cargas perigosas para devolução (exclusivo ramal 4500)
- Concessão ou negociação de descontos (exclusivo Comercial)
- Reclassificação de tier de cliente (exclusivo Comercial)

### Prior Decisions (ADRs do Cenário 1)

- **ADR-001:** Modelo Azure OpenAI GPT-4o, janela 128K tokens.
  O query-endpoint deve operar dentro do budget de ~12K tokens por
  resposta (4K system prompt + 5 chunks × ~1.500 tokens).
- **ADR-002:** 5 chunks por resposta, ~1.500 tokens cada.
  O requirements.md deve garantir que respostas com múltiplas fontes
  (ex: SLA + PROC-042) não excedam este limite.
- **ADR-003:** Base documental de 847 documentos válidos.
  Consultas sobre documentos não indexados DEVEM retornar recusa
  explícita, não resposta inventada.

### Constraints

- Toda resposta DEVE citar o documento fonte com nome e seção
  (ex: "SLA-2024, seção 2").
- O assistente NUNCA deve gerar valores numéricos (prazos, multiplicadores,
  percentuais) que não estejam literalmente nos documentos indexados.
- Para cargas perigosas, o assistente DEVE sempre incluir o encaminhamento
  para Gestão de Riscos (ramal 4500), independente do tipo de consulta.
- O campo `source_document` DEVE estar presente em toda resposta JSON,
  mesmo quando a confiança for baixa.
- Respostas em português formal — sem gírias, sem informalidade.

### Verification Criteria (testáveis pelo QA)

| ID | Critério | Como testar | Resultado esperado |
|---|---|---|---|
| VC-01 | Tempo de resposta < 30s | Medir latência end-to-end em 50 consultas consecutivas | P95 < 30s |
| VC-02 | Fonte citada em 100% das respostas | Inspecionar campo `source_document` em 100 respostas | Nenhuma resposta sem `source_document` |
| VC-03 | Aplicação correta de PROC-042 v2 | Enviar consultas de frete especial para as 5 regiões | Multiplicadores Sul 1.3, Sudeste 1.1, C-O 1.4, NE 1.5, Norte 1.8 |
| VC-04 | Recusa para carga perigosa sem escalação | Perguntar sobre devolução de carga ANTT classe 3 | Resposta inclui ramal 4500, sem afirmar elegibilidade |
| VC-05 | Recusa para tier Platinum | Perguntar sobre SLA Platinum | Resposta nega existência do tier e informa tiers válidos |
| VC-06 | Campo source_document no JSON | Inspecionar payload de 100 respostas | 100% com `source_document` preenchido |
| VC-07 | Fora do escopo: encaminhamento correto | Perguntar sobre seguro de carga | Resposta menciona sinistros@novatech.com.br, sem inventar valor |
| VC-08 | 15% consultas cross-context | Perguntar sobre SLA + frete para Gold > 500kg | Resposta integra ambos os contextos sem contradição |

---

## 4. Mockup — Interface de Resposta no Teams

```
┌─────────────────────────────────────────────────────────────┐
│  NovaTech Assistant                              🟢 Online  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Atendente: Qual o SLA para cliente Gold em incidente       │
│  crítico?                                                   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ NovaTech Assistant                                  │   │
│  │                                                     │   │
│  │ Para clientes **Gold**, o SLA em incidente crítico  │   │
│  │ é:                                                  │   │
│  │                                                     │   │
│  │ • Primeira resposta: até **30 minutos**             │   │
│  │ • Resolução: até **4 horas**                        │   │
│  │                                                     │   │
│  │ ⚠️ Incidente crítico inclui: carga > R$100k com    │   │
│  │ status desconhecido > 6h, irregularidade em carga   │   │
│  │ perigosa, ou múltiplos chamados recorrentes.        │   │
│  │                                                     │   │
│  │ Fonte: SLA-2024, seção 2 e seção 3                 │   │
│  │ Confiança: Alta                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  [ Digite sua pergunta... ]                    [ Enviar ]   │
└─────────────────────────────────────────────────────────────┘
```

**Componentes obrigatórios em toda resposta:**
- Resposta direta em destaque (negrito nos valores)
- Aviso contextual quando relevante (icone de aviso)
- Fonte com nome do documento e seção
- Indicador de confiança (Alta / Média / Baixa)
- Quando confiança Baixa: sugestão de escalação

---

## Evidência de Uso — Claude Code

### Iteração 1

**Prompt enviado:**
"Liste os bounded contexts do assistente NovaTech com base nos
documentos: POL-001 (devoluções), PROC-042 v1 e v2 (frete especial),
SLA-2024 (tiers) e FAQ-Atendimento."

**Resultado:** 3 contextos divididos por camada técnica: RAG, Interface,
Regras de Negócio. Divisão incorreta — não reflete o domínio.

### Iteração 2

**Refinamento:**
"Refaça dividindo por domínio de negócio, não por camada técnica.
Use as 4 categorias de discovery: prazos, frete, devolução, SLAs.
Para cada context: o que está dentro, fora e os relacionamentos."

**Resultado:** 4 bounded contexts corretos, mas sem linguagem ubíqua
e com termos genéricos (ex: 'carga especial' em vez de 'frete especial').

### Iteração 3

**Refinamento (como Tech Lead):**
"Atue como Tech Lead revisando este mapa. Identifique: (1) termos
que o LLM confundiria sem definição, (2) ambiguidades nos limites
de escopo, (3) o que está fora do escopo mas não está explicitado."

**Resultado do Tech Lead simulado:**
- 'Carga perigosa' precisa da referência ANTT classes 1-6
- 'Frete especial' sem referência à PROC-042 v2 é ambíguo
- Seguro de carga e danos em trânsito não estão no escopo mas
  serão perguntados — precisam de out-of-scope explícito

**Resultado final:** Versão atual com linguagem ubíqua detalhada,
boundaries explícitos e out-of-scope documentado.

**O que mudou entre iterações:**
- Divisão por domínio (não por técnica) corrigida na iteração 2
- Glossário surgiu da iteração com o Tech Lead simulado
- Out-of-scope para seguro e danos adicionado após perceber gap
  real na documentação (FAQ não é fonte formal)
