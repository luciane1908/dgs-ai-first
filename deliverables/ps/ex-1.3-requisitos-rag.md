---
papel: Product Specialist
fase: 1
exercicio: 1.3
titulo: Especificação de Requisitos de RAG — Ponto de Vista do Produto
temperatura-usada: 0.1
iteracoes: 2
autora: Luciane Baldo
empresa: DB1 Global Software
data: 2026-05-28
versao: 1.0
enunciado: exercises/exercicio-fase-1-entendimento.md#product-specialist-exercicio-13
contexto_usado:
  - context/ (todos os docs)
  - outputs: ps/ex-1.1-context-engineering.md
---

# PS — Exercício 1.3: Especificação de Requisitos de RAG

> Temperatura usada: **0.1** — especificação formal, linguagem precisa e não ambígua
> Processo: spec v1 → Claude como revisor → spec v2 (final)

---

## Spec v1 — Primeira versão

> Produzida com base no cenário e nos documentos de `context/`.

### REQ-01 — Fontes de dados a indexar

Devem ser indexados:
- Documentos com status ativo e responsável formal identificado
- Documentos com data de atualização nos últimos 24 meses

Não devem ser indexados:
- Documentos sem versão controlada
- Documentos com status "obsoleto" ou sem data de vigência definida

### REQ-02 — Tratamento de documentos contraditórios

Quando dois documentos cobrirem o mesmo tema com regras diferentes, o pipeline deve:
- Indexar apenas a versão mais recente
- Registrar a versão anterior como obsoleta nos metadados

### REQ-03 — Comportamento quando não há resposta

Quando nenhum chunk relevante for encontrado, o assistente deve:
- Informar ao atendente que não encontrou a informação
- Sugerir escalonamento para o supervisor

### REQ-04 — Prazo de atualização

Novos documentos publicados devem estar disponíveis no assistente em até 48 horas.

### REQ-05 — Rastreabilidade

Toda resposta deve citar o documento de origem.

---

## Feedback do Claude (Claude como revisor)

**Prompt enviado:**
```
Você é um revisor técnico especialista em sistemas RAG. Avalie a especificação
abaixo e identifique: ambiguidades, requisitos não testáveis, gaps que um
desenvolvedor encontraria ao implementar, e cenários que a spec não cobre.
Seja direto e específico.

[spec v1 acima]
```

**Críticas recebidas:**

> **REQ-01 — Ambiguidade crítica:** O critério "documentos com status ativo" pressupõe que todos os documentos têm um campo de status explícito. Na NovaTech, os dois PROC-042 coexistem *sem* indicação de status — nenhum dos dois seria excluído por este critério, e o problema de contradição persistiria. O requisito precisa cobrir o caso de documentos sem metadado de status.
>
> **REQ-02 — Não testável e potencialmente incorreto:** "Indexar apenas a versão mais recente" parece simples, mas exige que o sistema consiga identificar automaticamente qual versão é mais recente. Se dois documentos têm nomes diferentes (PROC-042 e PROC-042-v2), como o sistema sabe que são versões do mesmo procedimento? Além disso, a seção 5 do PROC-042-v2 define regras transitórias: chamados abertos antes de 01/12/2023 ainda usam a v1. Excluir a v1 completamente quebra esse requisito de negócio.
>
> **REQ-03 — Incompleto:** O requisito não define o threshold de similaridade que determina "nenhum chunk relevante". Sem esse número, a implementação é arbitrária. Também não cobre o caso de chunks recuperados com baixa confiança (score médio) — o assistente deve responder com ressalva ou se recusar?
>
> **REQ-04 — Prazo sem contexto:** 48 horas é razoável para documentos normativos (POL, PROC, SLA). Mas o FAQ informal é atualizado de forma colaborativa e contínua — o mesmo prazo se aplica? Além disso, o requisito não define o que acontece com chamados em andamento quando um documento é atualizado no meio do dia.
>
> **REQ-05 — Parcial:** Citar o documento de origem é necessário, mas não suficiente. O atendente precisa saber *qual trecho* foi usado para gerar a resposta, não apenas qual documento. Sem o trecho, a verificação é impraticável para documentos longos.
>
> **Gap não coberto:** Nenhum requisito define o que fazer com o FAQ informal. Ele deve ser indexado? Com que nível de confiança? Deve ser sinalizado diferentemente na resposta?

---

## Decisões tomadas com base no feedback

| Crítica | Decisão | Justificativa |
|---------|---------|---------------|
| REQ-01: docs sem metadado de status | Incluir sub-requisito para docs sem status explícito | A NovaTech tem docs ativos sem campo de status — ignorar esse caso invalida o requisito |
| REQ-02: identificação automática de versões | Substituir por critério baseado em metadado explícito de vigência | Detecção automática de "mesmo procedimento" é frágil; melhor exigir metadado humano |
| REQ-02: regras transitórias do PROC-042-v2 | Manter ambas as versões com metadado de vigência, não excluir | A regra de transição de negócio tem precedência sobre a simplicidade técnica |
| REQ-03: threshold não definido | Definir 0.75 como threshold padrão + comportamento para zona intermediária | Threshold configurável, com valor padrão documentado e testável |
| REQ-04: FAQ e docs em andamento | Separar SLA de atualização por tipo de documento | Documentos normativos e FAQ têm naturezas diferentes |
| REQ-05: citar trecho | Exigir citação de documento + seção + trecho destacado | Verificação pelo atendente precisa ser prática, não só possível |
| Gap: FAQ informal | Adicionar REQ-06 específico para fontes informais | É o risco mais alto identificado no Ex 1.1 — não pode ficar sem requisito |

---

## Spec v2 — Versão Final (após iteração)

### REQ-01 — Fontes de dados a indexar

**Devem ser indexados:**
- Documentos com responsável formal identificado E data de última atualização explícita
- Documentos aprovados pelo processo de curadoria antes da indexação (ver REQ-07)

**Não devem ser indexados sem tratamento:**
- Documentos sem responsável formal (ex: FAQ colaborativo sem dono)
- Documentos com data de vigência expirada
- Documentos sem número de versão controlado

**Documentos sem metadado de status explícito** (caso PROC-042 v1 e v2):
- Não devem ser excluídos automaticamente
- Devem ser sinalizados para revisão humana antes da indexação
- Só entram na base após decisão explícita do responsável pela área

**Testável por:** auditoria da lista de documentos indexados vs. lista total de fontes disponíveis.

---

### REQ-02 — Tratamento de documentos contraditórios

Quando dois ou mais documentos cobrirem o mesmo tema com regras divergentes:

1. O pipeline **não decide automaticamente** qual versão é a correta
2. Ambos os documentos recebem metadado explícito de vigência, definido pelo responsável humano da área:
   - `vigencia_inicio`: data a partir da qual o documento é aplicável
   - `vigencia_fim`: data até a qual o documento é aplicável (nulo = sem prazo definido)
   - `substituido_por`: referência ao documento mais recente (quando aplicável)
3. O pipeline indexa ambas as versões com esses metadados
4. Quando chunks de versões diferentes forem recuperados na mesma query, o assistente deve:
   - Apresentar as duas informações com indicação clara de qual versão cada uma pertence
   - Exibir alerta: *"Existem duas versões deste procedimento. Verifique com seu supervisor qual se aplica ao chamado em questão."*

**Caso específico PROC-042:** manter v1 e v2 indexadas com metadado de vigência baseado na seção 5 do PROC-042-v2 (transição em 01/12/2023).

**Testável por:** cenário de teste com pergunta sobre multiplicador de frete — verificar se a resposta exibe ambas as versões com alerta.

---

### REQ-03 — Comportamento quando não há resposta ou baixa confiança

**Threshold de confiança:** score mínimo de similaridade = **0.75** (configurável por ambiente).

| Score dos chunks recuperados | Comportamento do assistente |
|-----------------------------|-----------------------------|
| ≥ 0.75 | Responde normalmente com citação de fonte |
| 0.50 – 0.74 | Responde com ressalva: *"Encontrei informação parcialmente relacionada. Valide com o documento original antes de usar."* + citação |
| < 0.50 | *"Não encontrei informação sobre este tema na documentação disponível. Recomendo escalar para o supervisor."* |

**Testável por:** suite de perguntas com respostas conhecidas, verificando se o assistente responde, ressalva ou recusa conforme o score.

---

### REQ-04 — Prazo de atualização da base

| Tipo de documento | Prazo máximo para disponibilidade após publicação |
|-------------------|--------------------------------------------------|
| Documentos normativos (POL, PROC, SLA) | 24 horas úteis |
| Documentos informativos (wikis, comunicados) | 48 horas úteis |
| FAQ e documentos informais | Somente após aprovação de curadoria (sem SLA automático) |

**Documentos atualizados durante o dia:** chamados em andamento que já receberam uma resposta baseada na versão anterior não são retroativamente afetados. A nova versão passa a valer para novas consultas a partir da indexação.

**Testável por:** publicar um documento de teste, medir tempo até aparecer em uma query de validação.

---

### REQ-05 — Rastreabilidade de respostas

Toda resposta do assistente deve exibir obrigatoriamente:
- **Nome do documento:** ex: `POL-001 — Política de Devolução de Mercadorias`
- **Versão e data:** ex: `v3.1, 15/01/2024`
- **Seção:** ex: `Seção 3.2 — Exceções ao prazo geral`
- **Trecho destacado:** o parágrafo ou frase exata que fundamenta a resposta

Respostas sem citação completa não devem ser exibidas ao atendente — devem ser retidas pelo sistema e registradas como falha de geração.

**Testável por:** verificar estrutura da resposta em 100% das interações de teste; taxa de respostas sem citação completa deve ser 0%.

---

### REQ-06 — Tratamento de fontes informais (FAQ)

O FAQ-Atendimento pode ser indexado, com as seguintes restrições:

1. Chunks do FAQ recebem metadado `fonte_tipo: informal`
2. Quando a resposta for baseada em chunks com `fonte_tipo: informal`, o assistente exibe obrigatoriamente:
   *"⚠️ Esta informação vem de um documento informal do time de atendimento e não foi validada por Compliance ou Operações. Confirme com seu supervisor antes de usar."*
3. O FAQ não pode ser a única fonte de uma resposta sobre temas que possuam documento formal correspondente

**Testável por:** cenário de teste com pergunta sobre carga danificada (coberta apenas no FAQ) — verificar se o aviso aparece na resposta.

---

### REQ-07 — Processo de curadoria

Todo documento deve passar por aprovação de curadoria antes de ser indexado:

| Etapa | Responsável | Critério |
|-------|-------------|---------|
| Verificação de metadados | Equipe de dados (DB1) | Todos os campos obrigatórios preenchidos |
| Validação de conteúdo | Responsável da área (NovaTech) | Conteúdo atual e aprovado para uso |
| Classificação de vigência | Responsável da área | Datas de vigência definidas |
| Aprovação para indexação | DM do projeto | Documento liberado para entrar na base |

**Frequência:** processo executado em cada ciclo de atualização mensal e sempre que um documento novo for publicado fora do ciclo.
