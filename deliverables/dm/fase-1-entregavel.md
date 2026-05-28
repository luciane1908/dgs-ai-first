---
papel: Delivery Manager
fase: 1
titulo: Entendimento e Contexto — Fundamentos de IA Generativa
exercicios: [1.1, 1.2, 1.3]
autora: Luciane Baldo
empresa: DB1 Global Software
data: 2026-05-27
versao: 2.0
enunciado: exercises/exercicio-fase-1-entendimento.md
contexto_novatech: context/anexo-a-documentacao-simulada-novatech.md
---

# Entregável — Delivery Manager | Fase 1

> Enunciado completo: `exercises/exercicio-fase-1-entendimento.md`
> Documentação NovaTech: `context/`

---

## Exercício 1.1 — Análise de Riscos do Projeto

### Conceitos técnicos aplicados

| Conceito | Relevância para este projeto |
|----------|------------------------------|
| **Token** | A base (~800 PDFs + 400 wikis + 50 planilhas) pode ultrapassar 12M tokens. Nenhum modelo processa tudo de uma vez — daí a necessidade do RAG. |
| **Janela de contexto** | Cada query ao GPT-4o (~128K tokens) divide o orçamento entre: system prompt, histórico, chunks e pergunta. Ultrapassar esse limite causa truncamento silencioso. |
| **Temperatura** | Para respostas com prazos e valores contratuais, temperatura 0–0.2 é obrigatória. Temperatura alta aumenta alucinação em domínios factuais. |
| **Grounding** | RAG é a implementação de grounding: ancora respostas em fontes verificáveis. Sua qualidade depende da qualidade dos documentos indexados. |

---

### Matriz de Riscos

| # | Risco | Prob. | Impacto | Mitigação |
|---|-------|-------|---------|-----------|
| 1 | Alucinação em perguntas sem cobertura documental | Alta | Qualidade: Alto | Temperatura = 0 + threshold de confiança + guardrail determinístico (verificar citação antes de exibir) |
| 2 | Respostas contaminadas por docs contraditórios (ex: `context/PROC-042-v1` vs `context/PROC-042-v2`) | Alta | Qualidade: Alto | Metadado de vigência no pipeline de ingestão + auditoria no discovery |
| 3 | Context rot em sessões longas no Teams | Média | Qualidade: Alto | Compactação de histórico após N turns + monitoramento de uso da janela de contexto |
| 4 | Expectativa da diretoria vs. capacidade real da tecnologia | Alta | Prazo/Custo: Alto | Critérios de sucesso mensuráveis + demo com casos de falha no kickoff |
| 5 | Degradação por qualidade heterogênea da base (PDFs escaneados, macros, fórmulas) | Alta | Qualidade/Prazo: Alto | Auditoria prévia + MVP restrito a POL/PROC/SLA antes de wikis e planilhas |

**Detalhamento dos riscos críticos:**

**Risco 2 — Contradição documental (exemplo real na base):**
Os arquivos `context/PROC-042-frete-especial-v1.md` e `context/PROC-042-v2-frete-especial-revisado.md` coexistem sem indicação de qual é vigente. Se ambos forem indexados sem tratamento, o pipeline pode recuperar chunks de versões diferentes na mesma resposta — gerando multiplicadores de frete incorretos sem que o atendente perceba.

**Risco 3 — Context rot (estimativa):**
System prompt (~2K tokens) + 5 turns de histórico (~3K) + 10 chunks de 500 tokens (~5K) = ~10K tokens por query. Uma sessão com 20 perguntas pode atingir 50–60K tokens — ponto onde guardrails começam a ser ignorados silenciosamente.

---

### Perguntas ao Tech Lead antes de confirmar o cronograma

**1. Pipeline de dados:**
> "Qual é o plano para os ~15% de PDFs escaneados? OCR está no escopo e no prazo atual, ou vamos descobrir isso durante a ingestão?"

*Por que importa:* A qualidade do RAG é determinada pelo que entra no índice, não pelo modelo. Pipeline de dados subestimado é o principal causador de atrasos em projetos RAG.

**2. Tratamento de contradições:**
> "Vamos resolver docs contraditórios na camada de dados (metadados de vigência + arquivamento dos obsoletos) ou no layer do prompt? Qual das duas está contemplada no prazo?"

*Por que importa:* Resolver no prompt é frágil. Resolver nos dados exige processo com Operações, Compliance e Comercial da NovaTech — impacto direto no cronograma.

**3. Validação de retrieval:**
> "Qual a estratégia de chunking definida? Temos um gabarito de perguntas com chunks esperados para validar se o retriever está funcionando antes de ir para produção?"

*Por que importa:* Chunking inadequado é invisível até o QA. Reindexar toda a base consome dias. Sem gabarito de retrieval, o sistema pode parecer funcionar sem funcionar de fato.

---

## Exercício 1.2 — Comunicação de Expectativas com o Cliente

### E-mail ao Diretor de Operações

**Para:** Diretor de Operações — NovaTech
**De:** Luciane Baldo — Delivery Manager, DB1 Global Software
**Assunto:** Projeto Assistente IA — Alinhamento de expectativas e critérios de sucesso

Prezado,

Obrigada pelo entusiasmo com o projeto — é o tipo de engajamento da liderança que faz a diferença em iniciativas de IA.

Quero ser direta sobre o que vamos construir — não para reduzir a ambição, mas para garantir que chegamos ao go-live com um produto que o time vai confiar e usar no dia a dia.

**O que o assistente fará muito bem:**
Responder perguntas em português com base nos documentos oficiais da NovaTech — citando a fonte e o trecho exato. Para as consultas mais frequentes (prazos, frete, devolução), a redução de 12 para menos de 2 minutos por chamado é plenamente alcançável.

**O que ele não fará — e por que é importante saber:**
O assistente não "sabe tudo". Ele busca nos documentos os trechos mais relevantes e os usa para formular a resposta. Se a informação não estiver nos documentos, ele não inventa — diz explicitamente "não encontrei, escale para o supervisor". Isso não é uma limitação a corrigir: é o comportamento correto que protege a NovaTech de promessas indevidas.

Uma analogia: um funcionário novo muito inteligente que leu todos os manuais da empresa. Responde muito bem tudo que está nos manuais. O que não está, ele não inventa.

**O que isso significa na prática:**
A qualidade do assistente depende diretamente da qualidade da documentação de vocês. Curadoria da base documental não é atividade secundária — é parte central do projeto.

**Critérios de sucesso mensuráveis:**

| Indicador | Meta |
|-----------|------|
| Tempo médio de consulta | 12 min → < 2 min por chamado |
| Respostas com fonte verificável | ≥ 85% citando documento + seção |
| Taxa de escalonamento ao supervisor | Redução progressiva mês a mês |

Esses três números nos dirão se o sistema entrega valor real.

Atenciosamente,
**Luciane Baldo** | Delivery Manager Sênior | DB1 Global Software

---

### One-Pager — Como o Assistente Funciona

```
ASSISTENTE DE IA NOVATECH — COMO ELE FUNCIONA
══════════════════════════════════════════════

FLUXO DE UMA CONSULTA
  Atendente digita         Sistema busca nos         IA formula a resposta
  a pergunta no Teams  →   documentos oficiais   →   com citação da fonte
                           (POL, PROC, SLA)
                                  ↓
                       Se não encontrar:
                       "Não localizei. Escale
                       para o supervisor."

┌─────────────────────────┐  ┌──────────────────────────────────┐
│   O QUE ELE FAZ BEM     │  │      O QUE ELE NÃO FAZ           │
├─────────────────────────┤  ├──────────────────────────────────┤
│ Responde em segundos    │  │ Não inventa respostas            │
│ Cita sempre a fonte     │  │ Não aprende sozinho com o uso    │
│ Busca em ~1.250 docs    │  │ Não cobre gaps de documentação   │
│ Funciona em português   │  │ Não substitui julgamento humano  │
│ Disponível 24/7         │  │ em situações complexas           │
└─────────────────────────┘  └──────────────────────────────────┘

COMO MEDIREMOS O SUCESSO
  Tempo de consulta:    12 min → < 2 min
  Respostas com fonte:  ≥ 85% com doc + seção citados
  Escalonamentos:       Redução progressiva mês a mês

IMPORTANTE: qualidade do assistente = qualidade dos documentos.
Curadoria da base é parte do projeto, não atividade opcional.
```

---

## Exercício 1.3 — Planejamento de Discovery com IA

### Princípio orientador

No modelo AI First da DB1, a fase de **Intent** precede o discovery humano. Agentes de IA pré-analisam a documentação e entregam ao time um mapa priorizado de contradições, gaps e temas mais frequentes. O discovery humano passa a ser cirúrgico: em vez de *descobrir* nas entrevistas que existem dois PROC-042, chegamos já sabendo e perguntamos *por que* e *quem decide* qual é o oficial.

---

### Fase de Intent — Atividades dos Agentes de IA (dias 1–3)

| Atividade | O que o agente faz | Output |
|-----------|-------------------|--------|
| Catalogação da base | Lê metadados de todos os docs do SharePoint e Confluence | Lista por área, tipo e data |
| Detecção de contradições | Compara chunks de docs com mesmo tema, identifica divergências | Pares contraditórios com trechos específicos |
| Mapeamento de gaps | Cruza temas documentados com perguntas frequentes dos chamados | Lista de temas sem cobertura |
| Análise de qualidade | Processa amostra de PDFs e wikis, avalia legibilidade da extração | % de docs com extração adequada por tipo |
| Frequência de uso | Analisa ~3.000 chamados dos últimos 6 meses | Top 10 temas por volume — base para MVP |

> IA faz bem: catalogar, comparar, classificar e quantificar em volume. 800 docs em horas vs. dias de trabalho humano.

---

### Discovery Humano — Atividades (semanas 1–2)

| Atividade | Responsável | Depende de |
|-----------|-------------|------------|
| Entrevistas: time de atendimento | PS + DM | Mapa de gaps da IA |
| Entrevistas: supervisores | PS | Lista de temas sem cobertura |
| Entrevistas: donos dos documentos (Ops, Compliance, Comercial) | DM | Lista de contradições da IA |
| Validação das contradições | DM + NovaTech | Pares contraditórios identificados |
| Workshop de priorização da MVP | PS + DM + NovaTech | Top 10 temas + gaps validados |
| Definição de critérios de aceite | QA + PS | Requisitos das entrevistas |
| Apresentação do mapa de discovery | DM | Todos os outputs anteriores |

> Humanos fazem: validar, priorizar, decidir e negociar — o que nenhum agente tem autoridade para fazer.

---

### Cronograma — 2 Semanas

```
SEMANA 1
─────────────────────────────────────────────────────────
Seg-Qua │ [IA - Intent] Catalogação, contradições, gaps,
        │               qualidade, frequência de uso
Qua     │ [Humano] Revisão dos outputs da fase Intent
Qui     │ [Humano] Entrevistas: time de atendimento (2 grupos)
Sex     │ [Humano] Entrevistas: supervisores

SEMANA 2
─────────────────────────────────────────────────────────
Seg     │ [Humano] Entrevistas: donos dos docs (Ops + Compliance)
Ter     │ [Humano] Entrevistas: donos dos docs (Comercial + TI)
        │          Validação das contradições
Qua     │ [Humano] Workshop de priorização da MVP (3h)
Qui     │ [Humano] Critérios de aceite + acordos de SLA interno
Sex     │ [Humano] Apresentação do mapa + aprovação do backlog
─────────────────────────────────────────────────────────
```

---

### O que a NovaTech precisa fornecer — e quando

| O que | Quem | Até quando | Por que é crítico |
|-------|------|-----------|-------------------|
| Acesso leitura SharePoint | TI NovaTech | Seg semana 1 | Sem acesso, Intent não começa |
| Acesso leitura Confluence | TI NovaTech | Seg semana 1 | Idem |
| Export chamados 6 meses (anonimizado) | Operações | Seg semana 1 | Base para classificação por frequência |
| Lista de responsáveis por área documental | DM NovaTech | Ter semana 1 | Necessário para agendar entrevistas |
| 4 atendentes disponíveis (1h cada) | Gestão Atendimento | Qui/Sex semana 1 | Representatividade do uso real |
| 2 supervisores disponíveis (1h cada) | Gestão Atendimento | Sex semana 1 | Visão de exceções e escalonamentos |
| Donos de documento (Ops, Compliance, Comercial) | Diretoria | Seg/Ter semana 2 | Decisão sobre versão oficial dos docs contraditórios |
| Aprovação para workshop (3h) | Diretor de Operações | Qua semana 2 | Sem decisão de escopo, desenvolvimento não começa |

---

### Critérios de conclusão do Discovery

- [ ] 100% dos documentos catalogados com metadado de vigência, responsável e status
- [ ] Todas as contradições validadas por humano com autoridade para decidir versão oficial
- [ ] Top 10 temas da MVP definidos e aprovados pelo cliente
- [ ] Critérios de aceite para os 5 principais fluxos escritos e validados
- [ ] Backlog inicial priorizado e estimado aprovado em sessão com a NovaTech
