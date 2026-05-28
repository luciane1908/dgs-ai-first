---
papel: Product Specialist
fase: 1
exercicio: 1.1
titulo: Mapeamento de Intent com Engenharia de Contexto
tecnica: progressive-disclosure
temperatura-usada: 0.3
autora: Luciane Baldo
empresa: DB1 Global Software
data: 2026-05-28
versao: 1.0
enunciado: exercises/exercicio-fase-1-entendimento.md#product-specialist-exercicio-11
contexto_usado:
  - etapa1: metadados dos 5 docs (sem conteúdo)
  - etapa2: context/PROC-042-frete-especial-v1.md + context/PROC-042-v2-frete-especial-revisado.md
  - etapa3: outputs etapas 1+2 + context/FAQ-atendimento.md
---

# PS — Exercício 1.1: Mapeamento de Intent com Engenharia de Contexto

> Enunciado: `exercises/exercicio-fase-1-entendimento.md`
> Técnica aplicada: **progressive disclosure** — contexto fornecido em 3 etapas crescentes

---

## Decisão de contexto (planejamento prévio)

Antes de iniciar qualquer prompt, defini a estratégia de como alimentar o modelo:

| Etapa | O que fornecer | Por que NÃO fornecer tudo |
|-------|---------------|---------------------------|
| 1 | Apenas títulos + metadados dos 5 docs (~400 tokens) | Jogar 5 docs completos (~15K tokens) de uma vez satura o orçamento de atenção — o modelo tende a sumarizar superficialmente em vez de analisar |
| 2 | Conteúdo completo apenas dos 2 docs contraditórios (~3K tokens) | A etapa 1 já indicou onde está o maior risco; foco cirúrgico gera análise mais profunda |
| 3 | Outputs das etapas 1+2 + FAQ completo (~4K tokens) | O FAQ só faz sentido cruzar depois de ter o mapa de contradições — sem isso, o cruzamento seria superficial |

**Temperatura escolhida: 0.3**
Análise exploratória admite alguma variação criativa para identificar padrões não óbvios. Se fosse geração de requisitos formais (Ex 1.3), usaria 0.1.

---

## Etapa 1 — Visão Geral (somente metadados)

**Tokens fornecidos:** ~420 tokens
**Arquivos referenciados:** metadados dos 5 docs (sem conteúdo)

**Contexto fornecido ao modelo:**

```
Você está auxiliando um Product Specialist da DB1 na fase de Intent de um projeto
de assistente de IA com RAG para a NovaTech (logística). Abaixo estão os metadados
dos 5 documentos-chave disponíveis. NÃO forneço o conteúdo completo agora.

1. POL-001 — Política de Devolução de Mercadorias
   Versão: 3.1 | Atualização: 15/01/2024 | Responsável: Diretoria de Operações
   Classificação: Normativo — uso obrigatório pelo time de atendimento

2. PROC-042 — Procedimento de Cálculo de Frete Especial
   Versão: 1.0 | Emissão: 03/03/2023 | Responsável: Diretoria Comercial
   Status: Sem indicação formal de vigência. Coexiste com PROC-042-v2.

3. PROC-042-v2 — Procedimento de Cálculo de Frete Especial (Revisado)
   Versão: 2.0 | Emissão: 10/11/2023 | Responsável: Diretoria Comercial
   Status: Sem indicação formal de que substitui o v1. Ambos coexistem.

4. SLA-2024 — Tabela de SLA por Tipo de Cliente
   Versão: 2024.1 | Atualização: 02/01/2024 | Responsável: Comercial + Operações
   Classificação: Contratual — compromissos formais com clientes

5. FAQ-Atendimento — Perguntas Frequentes do Time de Suporte
   Versão: Não controlada | Responsável: Nenhum formal
   Classificação: Informal — NÃO validado por Compliance ou Operações

Com base apenas nesses metadados, responda:
a) Quais temas esses documentos cobrem? Há gaps visíveis?
b) Quais riscos você identifica para um pipeline de RAG?
c) Quais documentos merecem análise mais profunda na próxima etapa?
```

**Output obtido:**

> **Temas cobertos:** devolução de mercadorias, cálculo de frete especial (cargas > 500kg), níveis de serviço por tier de cliente, conhecimento tácito do time de atendimento.
>
> **Gaps visíveis nos metadados:** (1) Não há documento sobre frete padrão (cargas < 500kg). (2) Não há procedimento formal sobre carga danificada em trânsito. (3) O FAQ é informal e não controlado — risco de desatualização.
>
> **Riscos para RAG:** O maior risco visível nos metadados é a coexistência dos dois PROC-042 sem hierarquia clara. Um pipeline que indexar ambos sem metadado de vigência pode recuperar chunks contraditórios na mesma resposta. O FAQ sem versão controlada é outro risco — pode conter práticas informais que divergem dos documentos normativos.
>
> **Documentos para análise profunda:** PROC-042 v1 e v2 — a contradição potencial aqui tem impacto direto em cálculo de frete, que é financeiramente sensível.

**Análise crítica da Etapa 1:**
O modelo identificou exatamente os dois documentos que precisam de atenção profunda. Fornecer apenas metadados forçou o modelo a raciocinar sobre estrutura e relacionamentos — não sobre conteúdo. Isso é mais valioso nesta etapa. Se eu tivesse fornecido os docs completos, o modelo provavelmente teria listado os conteúdos em vez de analisar os riscos sistêmicos.

---

## Etapa 2 — Análise Profunda dos Documentos Contraditórios

**Tokens fornecidos:** ~3.200 tokens (conteúdo completo dos 2 PROC-042)
**Arquivos:** `context/PROC-042-frete-especial-v1.md` + `context/PROC-042-v2-frete-especial-revisado.md`

**Contexto fornecido ao modelo:**

```
Na etapa anterior, identificamos que os dois PROC-042 são o maior risco para o pipeline
de RAG. Forneço agora o conteúdo completo de ambos. Analise:

a) Liste todas as divergências entre as duas versões (valores, prazos, regras)
b) O que aconteceria se o pipeline recuperasse chunks de versões diferentes?
c) Como um Product Specialist deveria endereçar isso nos requisitos do sistema?

[conteúdo completo de PROC-042-v1 e PROC-042-v2]
```

**Output obtido:**

> **Divergências identificadas:**
>
> | Dimensão | PROC-042 v1 (mar/2023) | PROC-042 v2 (nov/2023) |
> |----------|----------------------|----------------------|
> | Multiplicador Sul | 1.2 | 1.3 |
> | Multiplicador Sudeste | 1.0 | 1.1 |
> | Multiplicador Centro-Oeste | 1.3 | 1.4 |
> | Multiplicador Nordeste | 1.4 | 1.5 |
> | Multiplicador Norte | 1.6 | 1.8 |
> | Fator peso 1.001–3.000kg | 1.2 | 1.15 |
> | Fator peso > 3.000kg | 1.5 | 1.4 |
> | Prazo adicional | +2 dias úteis | +3 dias úteis |
> | Desconto volume | > 10 fretes/mês (negociado) | ≥ 8 fretes (5%) / ≥ 15 (10%) |
>
> **Cenário de falha no RAG:** Se o pipeline recuperar o Chunk PROC-042-B (multiplicador Norte = 1.6) e o Chunk PROC-042v2-C (prazo = +3 dias), o modelo geraria uma resposta com frete calculado pela tabela antiga mas prazo da nova — combinação que nunca existiu em nenhuma versão real.
>
> **Requisito para o Product Specialist:** O pipeline deve obrigatoriamente escolher uma versão como canônica antes de indexar. A seção 5 do PROC-042-v2 define uma data de transição (01/12/2023) — isso deveria ser usado como metadado de vigência. Chamados abertos antes dessa data: v1. Chamados novos: v2.

**Análise crítica da Etapa 2:**
A tabela comparativa gerada pelo modelo seria impossível sem o conteúdo completo — por isso essa análise profunda só fez sentido aqui, não na Etapa 1. O modelo identificou um cenário de falha específico (mistura de tabelas entre versões) que é mais preciso do que qualquer risco que eu teria descrito sem olhar os números reais.

---

## Etapa 3 — Cruzamento com Práticas Informais do FAQ

**Tokens fornecidos:** ~5.400 tokens (outputs etapas 1+2 + FAQ completo)
**Arquivo:** `context/FAQ-atendimento.md`

**Contexto fornecido ao modelo:**

```
Temos agora o mapa de contradições entre PROC-042 v1 e v2 (Etapa 2) e os gaps
identificados nos metadados (Etapa 1). Forneço agora o FAQ informal do time de
atendimento. Cruze as informações e responda:

a) O FAQ confirma, contradiz ou complementa as contradições já identificadas?
b) O FAQ cobre algum gap identificado na Etapa 1?
c) Quais informações do FAQ representam risco maior: as que contradizem os docs
   formais, ou as que cobrem temas sem documentação formal?

[output da Etapa 1]
[tabela comparativa da Etapa 2]
[conteúdo completo do FAQ-Atendimento]
```

**Output obtido:**

> **FAQ confirma a contradição do PROC-042:**
> O Item 8 do FAQ diz: *"Cuidado: existem duas versões da PROC-042. A mais recente tem multiplicadores mais altos. Na dúvida, use a mais recente (v2), mas se o cliente reclamar do valor, pode ser que o contrato dele ainda esteja na tabela antiga."* — isso confirma que o time de atendimento já conhece a contradição e a resolve com critério informal (reclamação do cliente).
>
> **FAQ cobre gaps com risco alto:**
> - **Carga danificada (FAQ Item 38):** descreve um processo completo (registrar em 48h, enviar para sinistros@novatech.com.br) sem nenhum documento formal de respaldo. Se o pipeline recuperar esse chunk como fonte confiável, o atendente seguirá um processo informal como se fosse política oficial.
> - **Frete expresso para carga perigosa (FAQ Item 32):** diz que é possível com autorização do Compliance, mas não há PROC ou POL formal sobre isso. O modelo pode citar o FAQ com o mesmo nível de confiança que cita o POL-001.
>
> **Risco maior: informações que cobrem gaps sem documentação formal.**
> Uma contradição é visível — o modelo pode ser instruído a sinalizar quando dois chunks divergem. Mas uma informação que existe *apenas* no FAQ informal não tem "oponente" — o modelo a apresenta com confiança total, sem sinal de alerta, mesmo sendo uma prática não validada por Compliance.

**Análise crítica da Etapa 3:**
A descoberta mais importante surgiu aqui: **os gaps são mais perigosos que as contradições**. Isso só foi possível porque as Etapas 1 e 2 construíram o contexto necessário para que o cruzamento com o FAQ fosse significativo. Se eu tivesse jogado os 5 docs completos de uma vez na Etapa 1, o modelo provavelmente teria listado conteúdos em vez de identificar esse padrão sistêmico.

---

## Riscos identificados e encaminhamento para o Discovery Humano

| Risco | Origem | Como levar ao discovery humano |
|-------|--------|-------------------------------|
| Dois PROC-042 sem hierarquia clara | Etapas 1+2 | Levar tabela comparativa para entrevista com Diretoria Comercial — pedir decisão formal sobre qual é o vigente |
| Processo de carga danificada documentado apenas no FAQ | Etapa 3 | Levar para entrevista com Operações + Jurídico — verificar se existe PROC não indexado ou se precisa ser criado |
| Frete expresso para carga perigosa sem documento formal | Etapa 3 | Levar para Compliance — verificar se é prática autorizada e se deve ser documentada antes de entrar na base do RAG |
| FAQ sem controle de versão | Etapa 1 | Propor ao cliente: ou o FAQ passa a ter versão controlada, ou é excluído do índice e substituído por documentos formais |

---

## Reflexão: progressive disclosure vs. tudo de uma vez

**O que teria acontecido se eu tivesse colado os 5 documentos completos de uma vez:**

Os 5 documentos completos somam aproximadamente **15.000 tokens**. Junto com o system prompt e a pergunta, isso representaria ~16K tokens de contexto — dentro da janela do GPT-4o (128K), mas com um problema qualitativo: o modelo receberia todo o conteúdo ao mesmo tempo, sem hierarquia de importância.

Consequências prováveis:
1. **Sumarização superficial:** com tanto conteúdo, o modelo tende a produzir um resumo de cada documento em vez de identificar relacionamentos entre eles
2. **Perda do efeito "lost in the middle":** documentos posicionados no meio do contexto (PROC-042-v2, SLA-2024) receberiam menos atenção que os primeiros e últimos
3. **Ausência de análise comparativa:** a tabela de divergências entre os dois PROC-042 provavelmente não teria sido gerada — o modelo precisava de foco para construí-la
4. **FAQ tratado como equivalente aos normativos:** sem o contexto prévio de que o FAQ é informal, o modelo o trataria com o mesmo peso que o POL-001

**O que a abordagem progressiva permitiu:**
- Etapa 1 forçou o modelo a raciocinar sobre estrutura — não conteúdo
- Etapa 2 gerou análise comparativa precisa com foco total nos docs relevantes
- Etapa 3 usou o conhecimento acumulado das etapas anteriores como filtro para interpretar o FAQ
- O insight mais valioso (gaps > contradições) surgiu na Etapa 3 — seria improvável em abordagem de prompt único
