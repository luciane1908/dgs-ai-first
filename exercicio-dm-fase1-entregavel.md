# Exercícios Fase 1 — Papel: Delivery Manager
**Elaborado por:** Luciane Baldo — Delivery Manager Sênior, DB1 Global Software
**Data:** 27/05/2026
**Certificação:** DGS IA First

---

# EXERCÍCIO 1.1 — Avaliação de Viabilidade com Fundamentos de IA

## Documento de Análise de Riscos — Projeto Assistente IA NovaTech

**Versão:** 1.0
**Finalidade:** Kickoff interno — avaliação de viabilidade técnica e riscos do projeto

---

### 1. Contexto do Projeto

A NovaTech contratou a DB1 para construir um assistente de IA que permita ao time de atendimento (45 pessoas) consultar sua base documental (~1.250 fontes entre PDFs, wikis e planilhas) em linguagem natural, com respostas fundamentadas e com indicação de fonte. A expectativa da diretoria é reduzir o tempo de busca de 12 para menos de 2 minutos por chamado.

O assistente será construído sobre arquitetura **RAG (Retrieval-Augmented Generation)**: documentos são fragmentados em pedaços (*chunks*), transformados em representações numéricas (*embeddings*) e recuperados por similaridade quando o atendente faz uma pergunta. O modelo de linguagem (LLM) recebe esses fragmentos como contexto e gera a resposta.

Antes de confirmar o cronograma de 3 meses, esta análise mapeia os riscos técnicos derivados das características fundamentais da IA generativa.

---

### 2. Conceitos Técnicos Relevantes para o Projeto

| Conceito | O que é | Por que importa neste projeto |
|----------|---------|-------------------------------|
| **Token** | Unidade mínima de processamento do LLM (~0,75 palavras em português). Documentos, perguntas e respostas são todos medidos em tokens. | A base da NovaTech (~800 PDFs + 400 páginas wiki + 50 planilhas) pode ultrapassar **12 milhões de tokens**. Nenhum modelo lê tudo de uma vez — daí a necessidade do RAG. |
| **Janela de contexto** | Quantidade máxima de tokens que o modelo consegue processar em uma única interação. O GPT-4o suporta ~128K tokens. | A cada pergunta do atendente, o modelo recebe: system prompt + chunks recuperados + histórico de conversa. Se esse total ultrapassar a janela, o modelo trunca informação — silenciosamente. |
| **Temperatura** | Parâmetro (0 a 1) que controla a aleatoriedade das respostas. Temperatura 0 = determinístico; temperatura 1 = mais criativo. | Para um assistente que cita prazos e regras contratuais, temperatura baixa (0–0.2) é obrigatória. Temperatura alta aumenta o risco de o modelo "criar" procedimentos que não existem. |
| **Grounding** | Técnica de ancorar as respostas do modelo em fontes verificáveis, em vez de depender do conhecimento interno (pré-treinamento) do modelo. | O RAG é a implementação de grounding deste projeto. A qualidade do grounding depende diretamente da qualidade dos documentos indexados — documentos contraditórios ou mal formatados contaminam as respostas. |

---

### 3. Matriz de Riscos

#### Risco 1 — Alucinação em perguntas sem cobertura documental

**Descrição:** O LLM pode gerar respostas que parecem corretas e confiantes, mas são fabricadas a partir do seu conhecimento de pré-treinamento, quando nenhum chunk relevante foi recuperado. Isso é especialmente crítico no contexto da NovaTech, onde um prazo ou valor inventado pode gerar promessa indevida ao cliente.

**Exemplo concreto:** Um atendente pergunta "Qual o prazo de frete padrão para cargas abaixo de 500kg?" — essa informação não existe na documentação atual (só o frete especial acima de 500kg está documentado). O modelo pode inferir um valor plausível com alta aparência de certeza.

**Probabilidade:** Alta
**Impacto:** Qualidade (alto) / Prazo (médio) — exige retrabalho de prompt, guardrails e testes

**Mitigação:**
- Configurar temperatura = 0 no ambiente de produção
- Instruir o modelo no system prompt a responder "Não encontrei essa informação na documentação disponível — por favor, escale para o supervisor" quando os chunks recuperados tiverem score de similaridade abaixo de um threshold definido
- Implementar validação determinística pós-geração: verificar se a resposta contém ao menos uma citação de fonte antes de exibir ao atendente

---

#### Risco 2 — Respostas contaminadas por documentos contraditórios

**Descrição:** A NovaTech possui ao menos dois documentos ativos que descrevem o mesmo procedimento com valores diferentes: PROC-042 (v1, mar/2023) e PROC-042-v2 (nov/2023), com multiplicadores regionais e fatores de peso distintos. Nenhum dos dois está marcado como obsoleto no SharePoint. Se ambos forem indexados sem tratamento, o pipeline de RAG pode recuperar chunks de versões diferentes na mesma resposta, gerando um cálculo de frete incorreto — sem que o atendente perceba.

**Probabilidade:** Alta (a contradição já existe hoje)
**Impacto:** Qualidade (alto) / Custo (médio) — impacto direto no negócio se o frete for calculado errado

**Mitigação:**
- Implementar metadado obrigatório de vigência e versão em todos os documentos no pipeline de ingestão
- Criar regra no system prompt: "quando houver chunks de versões diferentes do mesmo procedimento, apresentar ambas ao atendente com indicação clara de data e alertar sobre a contradição"
- Incluir auditoria de documentos contraditórios como entregável da fase de discovery, antes do desenvolvimento

---

#### Risco 3 — Context Rot em sessões longas no Teams

**Descrição:** A janela de contexto do modelo (~128K tokens no GPT-4o) é compartilhada entre: system prompt, histórico da conversa, chunks recuperados e a pergunta atual. Em conversas longas no Teams, o histórico cresce progressivamente e consome o orçamento de contexto disponível para novos chunks. Quando o limite é atingido, o modelo começa a "esquecer" instruções fornecidas no início da sessão — incluindo os guardrails. Esse fenômeno é chamado de **context rot**.

**Estimativa de impacto:** System prompt (~2K tokens) + 5 turns de conversa (~3K tokens) + 10 chunks de 500 tokens (~5K tokens) = ~10K tokens por query. Uma sessão com 20 perguntas pode atingir 50-60K tokens antes de degradar. Para clientes Gold com chamados complexos, isso é facilmente atingível.

**Probabilidade:** Média
**Impacto:** Qualidade (alto) — guardrails silenciosamente ignorados são o pior tipo de falha

**Mitigação:**
- Definir política de compactação de histórico: após N turns, resumir as últimas interações em vez de manter o histórico completo
- Implementar monitoramento de tamanho do contexto por sessão — alertar quando atingir 70% da janela
- Testar o comportamento dos guardrails especificamente em sessões longas (10+ perguntas) durante a fase de QA

---

#### Risco 4 — Expectativa da diretoria vs. capacidade real da tecnologia

**Descrição:** A NovaTech espera um assistente que "responda tudo certo, como o ChatGPT mas com nossos dados". Essa expectativa ignora que: (a) respostas de LLMs são probabilísticas, não determinísticas; (b) a qualidade depende diretamente da qualidade dos documentos-fonte; (c) o assistente não "aprende" com o uso — sem manutenção ativa da base documental, a qualidade degrada conforme os documentos ficam desatualizados.

**Probabilidade:** Alta (a expectativa já foi explicitada pelo diretor de operações)
**Impacto:** Prazo (alto) / Custo (alto) — escopo inflado por expectativas não gerenciadas é a causa mais comum de atrasos

**Mitigação:**
- Apresentar ao cliente, no kickoff, critérios de sucesso mensuráveis: "% de respostas com citação de fonte verificável", "taxa de escalonamento para supervisor" e "tempo médio de consulta"
- Incluir cláusula de qualidade no contrato baseada em métricas objetivas
- Realizar demo com casos de falha conhecidos (ex: pergunta sem cobertura) para calibrar expectativas antes da aprovação do cronograma

---

#### Risco 5 — Degradação de qualidade por volume e diversidade da base documental

**Descrição:** A base da NovaTech soma ~1.250 fontes em três formatos distintos (PDF, HTML/wiki, XLSX), com qualidade heterogênea: há documentos escaneados (que exigem OCR), planilhas com fórmulas interdependentes, e wikis com macros customizadas. Documentos mal extraídos geram chunks corrompidos que "poluem" o índice vetorial. A temperatura baixa que reduz alucinação não elimina esse tipo de erro, pois o modelo está sendo fiel a um chunk que é genuinamente errado.

**Probabilidade:** Alta
**Impacto:** Qualidade (alto) / Prazo (médio) — extração e limpeza de dados raramente cabem no prazo inicial

**Mitigação:**
- Dedicar ao menos 2 semanas da fase de discovery para auditoria e classificação dos documentos-fonte antes de iniciar o pipeline de ingestão
- Criar critério de aceite para documentos: apenas documentos que passem na validação de extração são indexados na versão inicial
- Priorizar os documentos de maior impacto (POL, PROC, SLA) para a MVP e adicionar wikis e planilhas em fases posteriores

---

### 4. Resumo Executivo dos Riscos

| # | Risco | Probabilidade | Impacto | Mitigação Principal |
|---|-------|--------------|---------|---------------------|
| 1 | Alucinação em perguntas sem cobertura | Alta | Qualidade: Alto | Temperatura 0 + threshold de confiança + guardrail determinístico |
| 2 | Respostas contaminadas por docs contraditórios | Alta | Qualidade: Alto | Metadado de vigência + auditoria no discovery |
| 3 | Context rot em sessões longas | Média | Qualidade: Alto | Compactação de histórico + monitoramento de janela |
| 4 | Expectativa vs realidade | Alta | Prazo/Custo: Alto | Critérios de sucesso mensuráveis + demo calibrado |
| 5 | Degradação por qualidade da base | Alta | Qualidade/Prazo: Alto | Auditoria prévia + MVP com documentos priorizados |

---

### 5. Três Perguntas ao Tech Lead antes de Confirmar o Cronograma

**Pergunta 1 — Sobre o pipeline de dados:**
> "Qual é o plano de extração de texto para os ~15% de PDFs escaneados da base? O OCR já está previsto no escopo e no prazo, ou vamos descobrir isso depois que começarmos a ingestão? Qual a sua estimativa de documentos que terão qualidade de extração insuficiente para serem indexados na MVP?"

*Por que esta pergunta importa:* A qualidade do RAG não é determinada pelo modelo escolhido — é determinada pelo que entra no índice. Um pipeline de dados subestimado atrasa tudo que vem depois.

**Pergunta 2 — Sobre o tratamento de documentos contraditórios:**
> "Como o pipeline vai identificar e tratar documentos que descrevem o mesmo procedimento com regras diferentes? Vamos resolver isso no layer de dados (metadados de vigência, arquivamento dos obsoletos) ou no layer do prompt (instruir o modelo a sinalizar contradições)? Qual dessas abordagens está contemplada no prazo atual?"

*Por que esta pergunta importa:* Resolver contradições no prompt é possível, mas frágil. Resolver na camada de dados é mais robusto, mas exige processo com as áreas da NovaTech — o que tem impacto direto no cronograma.

**Pergunta 3 — Sobre estratégia de chunking e janela de contexto:**
> "Qual é a estratégia de chunking definida — tamanho fixo, por seção semântica, com ou sem overlap? E como vamos validar que o retriever está trazendo os chunks corretos para cada tipo de pergunta antes de subir para produção? Temos um conjunto de perguntas de referência com respostas esperadas para usar como gabarito nos testes de retrieval?"

*Por que esta pergunta importa:* Chunking inadequado é invisível até o QA — e o retrabalho de reindexar toda a base consome dias. Sem um gabarito de retrieval, não há como saber se o sistema está funcionando ou apenas parecendo funcionar.

---

# EXERCÍCIO 1.2 — Comunicação de Expectativas com o Cliente

## E-mail para o Diretor de Operações da NovaTech

**Para:** Diretor de Operações — NovaTech
**De:** Luciane Baldo — Delivery Manager, DB1 Global Software
**Assunto:** Projeto Assistente IA — Alinhamento de expectativas e critérios de sucesso

---

Prezado,

Obrigada pelo entusiasmo compartilhado sobre o projeto — é exatamente o tipo de engajamento da liderança que faz a diferença em iniciativas de IA. A demo do Copilot que vocês viram é um ótimo ponto de partida para nossas conversas.

Quero ser direta sobre o que vamos construir — não para reduzir a ambição do projeto, mas para garantir que chegamos ao go-live com um produto que o time realmente vai confiar e usar no dia a dia.

**O que o assistente fará muito bem:**
Ele vai permitir que um atendente escreva uma pergunta em português, como faz com um colega experiente, e receba em segundos uma resposta baseada nos documentos oficiais da NovaTech — com indicação da fonte e do trecho exato. Para as consultas mais frequentes (prazos de entrega, regras de frete, políticas de devolução), a redução de 12 para menos de 2 minutos é plenamente alcançável.

**O que ele não fará — e por que é importante você saber isso:**
O assistente não "sabe tudo" no sentido humano. Ele funciona assim: quando o atendente faz uma pergunta, o sistema busca nos documentos indexados os trechos mais relevantes e os entrega ao modelo de IA para que ele formule a resposta. Se a informação não estiver nos documentos — ou se estiver desatualizada — o sistema não tem como inventar a resposta correta. E quando isso acontece, um assistente bem configurado diz explicitamente "não encontrei essa informação" em vez de adivinhar.

Uma analogia útil: imagine um funcionário novo muito inteligente que leu todos os manuais da empresa. Ele responde muito bem tudo o que está nos manuais. Mas se você perguntar algo que nenhum manual cobre, ele não vai inventar — vai pedir para confirmar com o supervisor. Isso não é uma limitação a ser corrigida; é um comportamento correto que protege a NovaTech.

**O que isso significa na prática:**
A qualidade do assistente depende diretamente da qualidade da documentação que vocês têm hoje. Uma das primeiras atividades do projeto será mapear quais documentos estão atualizados, quais têm versões conflitantes e quais temas ainda não estão documentados. Esse trabalho de curadoria é tão importante quanto o desenvolvimento técnico.

**Como mediremos o sucesso:**
Proponho três indicadores objetivos que podemos acompanhar juntos desde o primeiro mês em produção:

1. **Tempo médio de consulta:** reduzir de 12 para menos de 2 minutos por chamado (meta da diretoria)
2. **Taxa de respostas com fonte verificável:** ≥ 85% das respostas do assistente devem citar um documento com número de versão e seção
3. **Taxa de escalonamento:** acompanhar se o % de chamados escalados ao supervisor diminui ao longo do tempo — isso indica que o assistente está cobrindo os gaps de documentação

Esses três números nos dirão se o sistema está entregando valor real, não apenas se ele parece funcional.

Preparei um one-pager que resume o funcionamento do assistente e os critérios de sucesso — está anexo a este e-mail e pode ser usado nas comunicações internas com o time de atendimento.

Estou à disposição para uma call antes do kickoff caso queira aprofundar algum ponto.

Atenciosamente,
**Luciane Baldo**
Delivery Manager Sênior | DB1 Global Software

---

## One-Pager: Como o Assistente de IA da NovaTech Funciona

```
┌─────────────────────────────────────────────────────────────────────┐
│         ASSISTENTE DE IA NOVATECH — COMO ELE FUNCIONA               │
└─────────────────────────────────────────────────────────────────────┘

FLUXO DE UMA CONSULTA
─────────────────────
  Atendente digita           Sistema busca nos          IA formula a resposta
  a pergunta no Teams   →    documentos oficiais    →   com citação da fonte
                             (POL, PROC, SLA)
                                    ↓
                         Se não encontrar: "Não
                         localizei essa informação.
                         Escale para o supervisor."

┌──────────────────────────┐    ┌──────────────────────────────────────┐
│    O QUE ELE FAZ BEM     │    │       O QUE ELE NÃO FAZ              │
├──────────────────────────┤    ├──────────────────────────────────────┤
│ ✅ Responde em segundos  │    │ ❌ Não inventa respostas              │
│ ✅ Cita sempre a fonte   │    │ ❌ Não aprende sozinho com o uso      │
│ ✅ Busca em ~1.250 docs  │    │ ❌ Não cobre o que não está nos docs  │
│ ✅ Funciona em português │    │ ❌ Não substitui o julgamento humano  │
│ ✅ Disponível 24/7       │    │    em situações complexas            │
└──────────────────────────┘    └──────────────────────────────────────┘

COMO MEDIREMOS O SUCESSO
─────────────────────────
  📊 Tempo de consulta:     12 min → < 2 min por chamado
  📊 Respostas com fonte:   ≥ 85% com documento + seção citados
  📊 Escalonamentos:        Redução progressiva mês a mês

IMPORTANTE: A qualidade do assistente depende da qualidade
dos documentos. Curadoria da base = parte do projeto.
```

---

# EXERCÍCIO 1.3 — Planejamento de Discovery com IA

## Plano de Discovery — Projeto Assistente IA NovaTech

**Versão:** 1.0 | **DM:** Luciane Baldo | **Duração prevista:** 2 semanas

---

### Princípio orientador

No modelo AI First da DB1, a fase de **Intent** acontece *antes* das entrevistas com stakeholders. Agentes especializados de IA pré-analisam a documentação disponível, identificam padrões, contradições e gaps — e entregam ao time humano um mapa priorizado. Isso torna o discovery humano mais cirúrgico: em vez de descobrir durante as entrevistas que existem duas versões do mesmo procedimento, chegamos já sabendo e perguntamos *por que* isso acontece e *quem decide* qual é a oficial.

---

### Atividades da Fase de Intent (executadas por agentes de IA)

Estas atividades acontecem em paralelo nos primeiros 3 dias, com supervisão do Tech Lead e do Product Specialist da DB1:

| Atividade | O que o agente faz | Output esperado |
|-----------|-------------------|-----------------|
| **Catalogação da base** | Lê os metadados de todos os documentos do SharePoint e Confluence (título, data, responsável, versão) | Lista classificada por área, tipo e data de atualização |
| **Detecção de contradições** | Compara chunks de documentos com mesmo tema e identifica divergências de valores, prazos ou regras | Lista de pares/grupos de documentos contraditórios com trechos específicos |
| **Mapeamento de gaps** | Cruza os temas cobertos na documentação com as perguntas mais frequentes no sistema de chamados | Lista de temas sem cobertura documental |
| **Análise de qualidade de extração** | Processa amostra dos PDFs e wikis e avalia legibilidade do texto extraído (detecta scans, tabelas corrompidas, macros) | % de documentos com extração adequada por tipo de fonte |
| **Classificação por frequência de uso** | Analisa os ~3.000 chamados dos últimos 6 meses e mapeia quais temas aparecem com mais frequência | Top 10 temas por volume de chamado — base para priorização da MVP |

> **Por que IA faz bem essas atividades:** catalogar, comparar, classificar e quantificar em volume são tarefas onde LLMs têm alta performance e consistência. O agente processa 800 documentos em horas — o que levaria dias de trabalho humano.

---

### Atividades do Discovery Humano (semanas 1 e 2)

Estas atividades só fazem sentido *depois* que os outputs da fase de Intent estão disponíveis:

| Atividade | Responsável | Dependência |
|-----------|------------|-------------|
| **Entrevista: time de atendimento** | Product Specialist + DM | Mapa de gaps da IA |
| **Entrevista: supervisores de atendimento** | Product Specialist | Lista de temas sem cobertura |
| **Entrevista: donos dos documentos** (Operações, Compliance, Comercial) | DM | Lista de contradições identificadas pela IA |
| **Validação das contradições** | DM + representantes NovaTech | Pairs de documentos contraditórios |
| **Priorização da MVP** | Product Specialist + NovaTech | Top 10 temas por volume + gaps |
| **Definição de critérios de aceite** | QA + Product Specialist | Requisitos validados nas entrevistas |
| **Apresentação do mapa de discovery** | DM | Todos os outputs anteriores |

> **Por que humanos fazem essas atividades:** validar, priorizar, decidir e negociar requerem julgamento, contexto político e autoridade que nenhum agente de IA tem.

---

### Cronograma Visual — 2 Semanas de Discovery

```
SEMANA 1
────────────────────────────────────────────────────────────────
Seg  │ [INTENT - IA] Catalogação + detecção de contradições
Ter  │ [INTENT - IA] Mapeamento de gaps + análise de qualidade
Qua  │ [INTENT - IA] Classificação por frequência de uso
     │ [HUMANO] Revisão dos outputs da fase Intent (TL + PS + DM)
Qui  │ [HUMANO] Entrevistas: time de atendimento (2 grupos)
Sex  │ [HUMANO] Entrevistas: supervisores de atendimento

SEMANA 2
────────────────────────────────────────────────────────────────
Seg  │ [HUMANO] Entrevistas: donos dos documentos (Ops + Compliance)
Ter  │ [HUMANO] Entrevistas: donos dos documentos (Comercial + TI)
     │ [HUMANO] Validação das contradições identificadas pela IA
Qua  │ [HUMANO] Workshop de priorização da MVP (DB1 + NovaTech)
Qui  │ [HUMANO] Definição de critérios de aceite + acordos de SLA interno
Sex  │ [HUMANO] Apresentação do mapa de discovery + aprovação do backlog
────────────────────────────────────────────────────────────────
```

---

### O que a NovaTech precisa fornecer — e quando

| O que | Quem fornece | Até quando | Por que é crítico |
|-------|-------------|-----------|-------------------|
| Acesso de leitura ao SharePoint (todos os documentos) | TI NovaTech | Dia 1 (Seg semana 1) | Sem acesso, a fase de Intent não começa |
| Acesso de leitura ao Confluence | TI NovaTech | Dia 1 | Idem |
| Export dos chamados dos últimos 6 meses (anonimizado) | Operações | Dia 1 | Sem isso, a classificação por frequência não é possível |
| Lista de responsáveis por cada área documental | DM NovaTech | Dia 2 (Ter semana 1) | Necessário para agendar entrevistas com quem realmente decide |
| Disponibilidade de 4 atendentes para entrevistas (1h cada) | Gestão de Atendimento | Semana 1, Qui/Sex | Representatividade do uso real |
| Disponibilidade de 2 supervisores para entrevistas | Gestão de Atendimento | Semana 1, Sex | Visão de exceções e escalonamentos |
| Disponibilidade dos donos de documento (Ops, Compliance, Comercial) | Diretoria | Semana 2, Seg/Ter | Decisão sobre versão oficial dos documentos contraditórios |
| Aprovação para workshop de priorização (3h) | Diretor de Operações | Semana 2, Qua | Sem decisão de escopo, o desenvolvimento não começa com clareza |

---

### Critérios de conclusão do Discovery

O discovery está concluído quando:

- [ ] 100% dos documentos catalogados com metadado de vigência, responsável e status (ativo/obsoleto)
- [ ] Todas as contradições identificadas pela IA foram validadas por um humano com autoridade para decidir a versão oficial
- [ ] Top 10 temas da MVP definidos e aprovados pelo cliente
- [ ] Critérios de aceite escritos para ao menos os 5 principais fluxos
- [ ] Backlog inicial priorizado e estimado aprovado em sessão com a NovaTech
