---
papel: Product Specialist
fase: 1
exercicio: "1.3"
titulo: "Especificação de Requisitos do Produto — Assistente IA com RAG para Atendimento NovaTech"
temperatura-usada: 0.1
iteracoes: 2
autora: Luciane Baldo
data: 2026-05-28
versao: "2.0"
sessao: isolada
enunciado: exercises/exercicio-fase-1-entendimento.md
contexto_usado: context/ (todos os docs)
---

# PS — Exercício 1.3: Especificação de Requisitos de RAG

> Temperatura: **0.1** — spec formal não admite ambiguidade
> Processo: Spec v1 → Claude como revisor → Spec v2 (final)

---

# Spec v1 — Primeira Versão

## Contexto e Escopo

Especificação de requisitos de produto para o assistente de IA com RAG da NovaTech (45 atendentes). O sistema indexará ~800 PDFs, ~400 páginas wiki e ~50 planilhas, respondendo perguntas ancoradas em fontes verificáveis via Microsoft Teams.

---

## REQ-01 — Fontes a Indexar e Fontes Excluídas

**Fontes elegíveis:**
- Documentos com responsável formal e data de atualização explícita
- Documentos aprovados com número de versão controlado

**Não devem ser indexados:**
- Documentos sem versão controlada
- Documentos com status "obsoleto" ou sem data de vigência definida

## REQ-02 — Tratamento de Documentos Contraditórios

Quando dois documentos cobrirem o mesmo tema com regras diferentes:
- Indexar apenas a versão mais recente
- Registrar a versão anterior como obsoleta nos metadados

## REQ-03 — Comportamento quando não há resposta

Quando nenhum chunk relevante for encontrado:
- Informar ao atendente que não encontrou a informação
- Sugerir escalonamento para o supervisor

## REQ-04 — Prazo de Atualização

Novos documentos publicados devem estar disponíveis no assistente em até 48 horas.

## REQ-05 — Rastreabilidade

Toda resposta deve citar o documento de origem.

---

# Feedback do Claude (Claude como Revisor)

**Prompt enviado:**
```
Você é um engenheiro sênior de produto especializado em sistemas RAG corporativos.
Revise a especificação de requisitos v1 abaixo e identifique: ambiguidades, requisitos
não testáveis, gaps que um desenvolvedor encontraria ao implementar, e cenários não
cobertos. Mínimo 5 críticas com referência ao número do requisito.
```

**Críticas recebidas:**

**C1 — REQ-01: ausência de processo de curadoria antes da indexação**
O REQ-01 define critérios de elegibilidade, mas não especifica quem aprova a inclusão e quando. Um documento pode ter todos os metadados corretos e ainda conter erros de conteúdo. Sem gate de aprovação humana, o sistema absorve documentos incorretos que passam nos filtros automáticos.

**C2 — REQ-01: PDFs escaneados ficam em limbo operacional**
O requisito exclui PDFs escaneados sem OCR, mas não define o critério de qualidade do OCR para que o documento seja elegível após processamento. Uma extração com 70% de precisão é aceita? Cria um grupo de documentos em estado indefinido.

**C3 — REQ-02: metadado de vigência sem responsável pela atribuição**
O REQ-02 depende criticamente dos metadados de status, mas não define quem os atribui, quando e por qual processo. Se o processo documental já é descentralizado em 3 áreas sem processo unificado, esse metadado nunca será atualizado de forma confiável.

**C4 — REQ-02: ignora regras transitórias do PROC-042-v2**
"Indexar apenas a versão mais recente" ignora que a seção 5 do PROC-042-v2 define regras transitórias: chamados abertos antes de 01/12/2023 ainda usam a v1. Excluir a v1 completamente quebra esse requisito de negócio.

**C5 — REQ-03: threshold não definido**
O requisito não define o valor numérico do score de similaridade que determina "nenhum chunk relevante". Sem esse número, a implementação é arbitrária. Também não cobre a zona intermediária (chunks recuperados com baixa confiança).

**C6 — REQ-04: SLA único inadequado por tipo de documento**
48 horas aplica a todos os documentos indiscriminadamente. Um procedimento normativo crítico e um FAQ informal têm urgências completamente diferentes. O requisito também não define o que acontece com chamados em andamento quando um documento é atualizado.

**C7 — REQ-05: citação de documento não é suficiente**
Citar o documento de origem é necessário, mas não suficiente. O atendente precisa saber qual trecho foi usado — não apenas qual documento. Sem o trecho, a verificação é impraticável para documentos longos. Além disso, o impacto em tokens das citações não está especificado como requisito testável.

---

# Decisões Tomadas com Base no Feedback

| Crítica | Decisão | Justificativa |
|---------|---------|---------------|
| C1 — sem gate humano | Criar REQ-07: curadoria com aprovação humana antes da indexação | Critérios formais são necessários mas não suficientes para garantir qualidade de conteúdo |
| C2 — PDFs em limbo | Incluir no REQ-01 critério mínimo OCR (≥ 95%) e SLA de processamento | Documentos indefinidos criam falsos negativos no retrieval |
| C3 — metadado sem dono | Incluir no REQ-07 que o responsável documental atribui metadado de vigência no ato da publicação | O metadado é um contrato entre o processo documental e o sistema RAG — precisa de dono |
| C4 — regras transitórias | Manter ambas as versões com metadado de vigência; sistema não decide automaticamente | A regra de transição de negócio tem precedência sobre a simplicidade técnica |
| C5 — threshold | Definir 0.75 como valor inicial + comportamento por faixa + revisão periódica (REQ-08) | Threshold configurável com valor padrão documentado e testável |
| C6 — SLA único | Separar SLA por tipo de documento em REQ dedicado | Documentos normativos e FAQ têm criticidades diferentes |
| C7 — trecho obrigatório | Exigir documento + versão + seção + trecho literal + orçamento de tokens definido | Verificação pelo atendente precisa ser prática, não apenas possível |

---

# Spec v2 — Versão Final

## Contexto e Escopo

Especificação de requisitos de produto para o assistente de IA com RAG da NovaTech (45 atendentes). O sistema indexará ~800 PDFs, ~400 páginas wiki e ~50 planilhas, respondendo perguntas ancoradas em fontes verificáveis via Microsoft Teams. **Temperatura de geração: 0.1.**

---

## REQ-01 — Fontes Elegíveis, Fontes Excluídas e Qualidade de Ingestão

**Fontes elegíveis:**

| Tipo | Condição de elegibilidade |
|------|--------------------------|
| Procedimentos (PROC-xxx) | Versão, data e responsável de área presentes nos metadados |
| Manuais normativos | Aprovação formal registrada com data e assinante |
| Wiki corporativa | Responsável de área identificado e data de última revisão |
| Planilhas de referência | Cabeçalho com data de atualização e responsável |
| PDFs digitais nativos | Texto extraível diretamente com metadados de versão e data |
| PDFs escaneados pós-OCR | Precisão OCR ≥ 95% validada por amostragem; processamento em até 10 dias úteis |

**Fontes excluídas:**
- PDFs escaneados com OCR < 95% ou ainda não processados
- Documentos sem data de publicação identificável
- E-mails, chats e comunicações informais
- Rascunhos com status "draft" sem aprovação formal
- FAQ-Atendimento — tratado em REQ-02

**Estado de documentos pendentes:** documentos com `indexacao: pendente` **não aparecem** em resultados até receberem `indexacao: aprovado`.

**Testável por:** auditoria de 20 docs indexados (100% com metadados); submeter 10 PDFs com OCR < 95% e confirmar ausência nos resultados; confirmar que docs `pendente` não retornam em nenhuma query.

---

## REQ-02 — Tratamento de Fontes Informais (FAQ-Atendimento)

O FAQ cobre temas sem documentação formal (carga danificada, frete expresso para carga perigosa). Exclusão total criaria lacunas. Inclusão sem tratamento criaria risco de informação não auditada sendo apresentada como normativa.

**Regra:**
1. FAQ indexado com metadados `categoria: fonte_informal` e `validacao_compliance: pendente`
2. Toda resposta baseada em chunks do FAQ exibe **antes do conteúdo**, em bloco destacado:
   > **⚠️ Aviso: fonte informal.** Esta informação provém do FAQ-Atendimento, não validado por Compliance. Confirme com o responsável antes de aplicar em casos reais.
3. O sistema não combina chunks do FAQ com chunks normativos sem separação visual explícita
4. Quando FAQ for única fonte com score ≥ 0.75: responde com aviso obrigatório. Abaixo de 0.75: aplica comportamento de "sem resposta" do REQ-04

**Testável por:** perguntar sobre carga danificada → verificar aviso antes do conteúdo; verificar metadado `categoria: fonte_informal` no chunk; confirmar ausência de mistura com normativos sem separação.

---

## REQ-03 — Tratamento de Documentos Contraditórios e Controle de Vigência

A base contém versões concorrentes de procedimentos (PROC-042 v1 e v2) com conteúdo conflitante.

**Regra:**
1. Versões anteriores **não são excluídas** — permanecem indexadas com metadados `status: vigente` ou `status: substituido`
2. O metadado de vigência é atribuído pelo responsável documental no ato da publicação — não pelo sistema automaticamente (ver REQ-07)
3. Quando chunks de versões diferentes do mesmo procedimento forem recuperados:
   - Responder com base **exclusivamente** na versão `status: vigente`
   - Exibir nota: *"Existe versão anterior deste procedimento ([id]) com informação diferente. Resposta baseada na versão vigente ([id])."*
4. **Sem metadado de status:** sistema **não infere** qual é a vigente. Retorna: *"Versões conflitantes encontradas sem indicação de vigência. Consulte o responsável antes de prosseguir."*

**Testável por:** consultar multiplicadores regionais com v1 e v2 indexadas → verificar resposta baseada em v2 com nota sobre v1; repetir sem metadado → verificar mensagem de conflito sem conteúdo; confirmar v1 na base com `status: substituido`.

---

## REQ-04 — Comportamento por Faixa de Similaridade

Chunks com baixa similaridade consomem tokens da janela de contexto sem contribuir para precisão e aumentam risco de conexões não sustentadas.

| Score de similaridade | Comportamento |
|----------------------|---------------|
| ≥ 0.75 | Responder normalmente com citação completa (REQ-06) |
| 0.50 – 0.74 | Responder com aviso antes do conteúdo: *"Base documental não contém informação direta. Trechos a seguir têm relação parcial — verifique se atendem ao seu caso."* + citação |
| < 0.50 | Não gerar resposta. Retornar: *"Não encontrei informação suficientemente confiável. Consulte [canal/responsável configurado]."* |

**Nota de calibração:** threshold 0.75 é o valor inicial de go-live, recalibrado conforme REQ-08.

**Testável por:** perguntas calibradas com score esperado em cada faixa; verificar comportamento correspondente em 100% dos casos; confirmar que respostas com score < 0.50 não contêm conteúdo documental.

---

## REQ-05 — SLA de Atualização por Tipo de Documento

| Tipo de documento | SLA de indexação após publicação |
|-------------------|----------------------------------|
| Procedimento normativo (PROC-xxx) | 2 dias úteis |
| Manuais regulatórios | 2 dias úteis |
| Wiki corporativa | 5 dias úteis |
| Planilhas de referência | 3 dias úteis |
| PDFs digitais não normativos | 5 dias úteis |
| PDFs escaneados (após OCR aprovado) | 2 dias úteis após aprovação |
| FAQ-Atendimento | 10 dias úteis (sujeito à curadoria do REQ-07) |

**SLA conta a partir da data de publicação oficial**, não da data de recebimento pelo time de IA.

**Monitoramento:** o sistema registra data de publicação e data de indexação de cada documento. Relatório semanal de conformidade disponível para o gestor.

**Testável por:** publicar documento normativo de teste → verificar indexação em ≤ 2 dias úteis; consultar relatório de SLA e confirmar registros de datas; verificar alertas para documentos fora do SLA.

---

## REQ-06 — Rastreabilidade com Trecho Destacado e Orçamento de Tokens

**Campos obrigatórios em toda citação:**

| Campo | Exemplo |
|-------|---------|
| Nome do documento | PROC-042 — Procedimento de Frete Regional |
| Versão | v2 |
| Data de publicação | nov/2023 |
| Seção | 3.2 — Multiplicadores por Região Sul |
| Trecho destacado (citação literal) | *"O multiplicador para a região Sul é de 1,35 sobre a tarifa base..."* |

**Regras:**
1. Trecho destacado é o fragmento **exato** do documento, entre aspas, sem paráfrase
2. Máximo de **3 chunks** por resposta (maior score). Chunks adicionais omitidos com nota: *"Outros [N] documentos relacionados encontrados. Solicite detalhamento se necessário."*
3. **Orçamento de tokens:** cada citação completa ocupa ~80–120 tokens. Se soma de citações + resposta ultrapassar o limite configurado, truncar o trecho com indicação: *"[trecho parcial — consultar documento original]"*

**Testável por:** verificar 5 campos obrigatórios em 100% das respostas; verificar trecho no documento-fonte sem paráfrase; simular 4+ chunks → confirmar apenas 3 na resposta; simular limite de tokens → confirmar truncamento com indicação.

---

## REQ-07 — Processo de Curadoria com Aprovação Humana

Critérios formais de elegibilidade (REQ-01) são necessários, mas não suficientes. Um documento pode ter metadados corretos e conteúdo incorreto.

**Etapas obrigatórias antes da indexação:**

| Etapa | Responsável | Critério |
|-------|-------------|---------|
| Verificação de metadados | Equipe de dados (DB1) | Todos os campos obrigatórios preenchidos |
| Validação de conteúdo | Responsável da área (NovaTech) | Conteúdo atual e aprovado para uso |
| Atribuição de vigência | Responsável da área | `status: vigente` ou `status: substituido` definidos |
| Aprovação para indexação | DM do projeto | Documento liberado com `indexacao: aprovado` |

Documentos reprovados recebem `indexacao: rejeitado` com registro do motivo. Responsável notificado para correção.

**Testável por:** submeter documento com metadados corretos sem aprovação humana → confirmar ausência nos resultados; completar fluxo de aprovação → confirmar indexação; reprovar documento → confirmar não indexação + registro de motivo.

---

## REQ-08 — Revisão Periódica de Thresholds e Qualidade

**Revisão aos 90 dias de operação**, com base em:
- Taxa de perguntas respondidas por faixa de score
- Feedback do time de atendimento sobre qualidade
- Taxa de escalonamentos por ausência de resposta
- Precisão das citações (amostra de 50 respostas por tipo de fonte)

Thresholds podem ser ajustados por tipo de fonte com base nos dados. Toda alteração documentada com: data, valor anterior, novo valor, justificativa baseada em dados e aprovação do gestor.

**Revisões subsequentes:** ciclos de 180 dias ou após evento que altere significativamente a base (migração, reestruturação de áreas).

**Testável por:** verificar relatório de revisão aos 90 dias com os 4 indicadores; confirmar que alterações de threshold têm registro completo no log de configuração.

---

## Resumo — Spec v2

| Requisito | Tema | Conceito IA First demonstrado |
|-----------|------|-------------------------------|
| REQ-01 | Fontes elegíveis e qualidade de ingestão | Grounding — base de fontes verificáveis |
| REQ-02 | Fontes informais com aviso obrigatório | Grounding — transparência de origem |
| REQ-03 | Documentos contraditórios e vigência | Grounding — determinismo diante de conflito |
| REQ-04 | Comportamento por faixa de similaridade | Janela de contexto + threshold numérico |
| REQ-05 | SLA de atualização por tipo de documento | Grounding — atualidade da base |
| REQ-06 | Rastreabilidade com trecho e orçamento de tokens | Rastreabilidade + impacto de tokens |
| REQ-07 | Curadoria com aprovação humana | Grounding — gate de qualidade editorial |
| REQ-08 | Revisão periódica de thresholds | Temperatura / calibração contínua |
