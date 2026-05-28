---
papel: Product Specialist
fase: 1
exercicio: "1.2"
titulo: "Jornada do Atendente com Assistente de IA — NovaTech"
temperatura-usada: 0.1
autora: Luciane Baldo
data: 2026-05-28
versao: "2.0"
sessao: isolada
enunciado: exercises/exercicio-fase-1-entendimento.md
---

# PS — Exercício 1.2: Jornada do Atendente com Assistente de IA

> Enunciado: `exercises/exercicio-fase-1-entendimento.md`
> Dados do discovery (simulados): 4 fontes por chamado; dúvidas: prazos (35%), frete (25%), devolução (20%), outros (20%); 15% escalados ao supervisor.

---

# Fluxo Principal — Consulta com resposta encontrada

**Cenário:** atendente recebe dúvida de cliente e utiliza o assistente integrado ao Microsoft Teams.

1. **Recebimento da dúvida**
   O atendente recebe a pergunta do cliente via telefone, chat ou e-mail.

2. **Abertura do assistente no Teams**
   O atendente aciona o assistente pelo painel lateral do Teams (bot integrado via Azure Bot Service). Não é necessário sair da plataforma.

3. **Formulação da consulta em linguagem natural**
   O atendente digita a pergunta sem necessidade de palavras-chave específicas.
   Exemplo: `"Prazo de entrega para Curitiba, cliente PJ, modalidade expressa"`

4. **Processamento via pipeline RAG**
   O sistema executa busca semântica nos documentos indexados. O pipeline recupera os trechos com maior score de similaridade e os passa como contexto para o modelo Azure OpenAI.
   - **Temperatura aplicada: 0.1** — configuração intencional para respostas com prazos e valores contratuais. Temperatura baixa evita que o modelo interpole valores entre contratos diferentes. Um prazo gerado criativamente poderia comprometer SLAs reais.

5. **Geração da resposta com citação obrigatória**
   O assistente retorna resposta em formato padronizado:
   ```
   Resposta: O prazo padrão para Curitiba na modalidade expressa é de 2 dias úteis.
   Fonte: Tabela de Frete Regional — v3.2, seção 4.1 (atualizada em 15/03/2026)
   Confiança: Alta
   ```
   O campo **Fonte** é obrigatório em toda resposta (Guardrail 1). O campo **Confiança** sinaliza o grau de certeza com base no score de recuperação.

6. **Uso pelo atendente no chamado**
   O atendente repassa a informação ao cliente. O chamado é encerrado com a resposta fornecida.

7. **Registro automático da interação**
   A sessão é registrada no log de uso (Azure Application Insights) para auditoria e monitoramento de qualidade.

---

# Fluxo de Fallback — Sem resposta ou baixa confiança

**Cenário:** o assistente não localiza informação suficiente, ou o score de recuperação está abaixo do threshold.

**Critério de threshold:** o sistema classifica a confiança como "Baixa" quando o score de similaridade semântica do melhor trecho é inferior a 0,72. Abaixo desse valor, o assistente não afirma — sinaliza a limitação.

1. **Detecção de baixa confiança**
   O pipeline retorna chunks com score inferior ao threshold, ou zero resultados relevantes.

2. **Resposta de fallback estruturada**
   ```
   Não encontrei uma resposta com confiança suficiente para esta pergunta.
   Fragmentos relacionados encontrados: [Política de Devolução — seção 2, trecho parcial]
   Recomendação: consulte o supervisor ou acione o canal #duvidas-compliance no Teams.
   Confiança: Baixa
   ```
   O assistente **nunca completa a lacuna com suposição própria** (Guardrail 1). Um prazo inventado poderia gerar promessa indevida com impacto contratual.

3. **Decisão do atendente**
   - **Opção A — Escalar para supervisor:** encaminha o chamado com o fragmento parcial como contexto.
   - **Opção B — Registrar para curadoria:** se o atendente souber a resposta por experiência, usa e sinaliza a lacuna via fluxo de feedback.

4. **Contenção de tokens desnecessários**
   O design do fallback é intencional quanto ao custo: sinalizar baixa confiança sem reprocessar evita chamadas adicionais ao modelo. Em escala de 45 atendentes com centenas de chamados diários, loops de retry sem critério de parada representam custo operacional e latência relevantes. O threshold único (score < 0,72 → fallback imediato) é a proteção arquitetural contra esse desperdício.

5. **Registro do fallback**
   O evento é registrado com tag `fallback=true` no log, alimentando o painel de lacunas para priorização de curadoria.

---

# Fluxo de Feedback — Resposta incorreta ou desatualizada

**Cenário:** o atendente identifica que a resposta está incorreta, desatualizada ou incompleta.

1. **Identificação do problema**
   Exemplos: o prazo indicado foi alterado por aditivo contratual não indexado; a política exibida é versão anterior; a resposta omite uma exceção importante.

2. **Acionamento do feedback inline**
   O atendente clica em **"Resposta incorreta / desatualizada"** e preenche:
   - Tipo: `[Incorreto] [Desatualizado] [Incompleto]`
   - Descrição breve (máx. 280 caracteres)
   - Documento correto, se conhecido (campo opcional)

3. **Mitigação de context rot em sessões longas**
   Sessões extensas no Teams acumulam histórico, expandindo a janela de contexto. Em janelas longas, instruções do system prompt podem ser "diluídas" — **context rot**. O fluxo de feedback mitiga isso de duas formas:
   - **Feedback é ação atômica isolada:** abre nova chamada independente ao backend, sem herdar o histórico da sessão. O registro chega limpo ao sistema de curadoria.
   - **Limite de contexto configurado:** o assistente descarta automaticamente mensagens com mais de 10 turnos anteriores, mantendo system prompt + contexto RAG atual + últimas 3 trocas. Isso preserva precisão ao longo de sessões longas.

4. **Roteamento do feedback**
   Enviado para fila de curadoria no SharePoint (gerenciada por Compliance + Operações). Cada item inclui: ID do chamado, trecho sinalizado, tipo de problema, atendente (anonimizado), documento de referência.

5. **Ciclo de curadoria**
   O time revisa semanalmente:
   - Documento desatualizado → atualização no SharePoint + re-indexação incremental via Azure AI Search
   - Resposta estruturalmente errada → ajuste no prompt de sistema ou metadados
   - Lacuna identificada → encaminhamento para criação de novo documento formal validado por Compliance

6. **Fechamento do loop**
   O sistema notifica o atendente via Teams: *"A lacuna que você reportou foi corrigida. Obrigado pela contribuição."* Esse fechamento é importante para manter adesão — atendentes sem retorno tendem a abandonar o hábito de sinalizar problemas.

---

# Guardrails de Comportamento

## Guardrail 1 — Nunca afirmar prazo ou valor sem citação explícita de fonte e seção

**Descrição:**
O assistente não retorna nenhuma resposta com prazo (dias, horas, datas) ou valor (frete, multa contratual, custo de devolução) sem incluir obrigatoriamente nome do documento, versão e seção específica.

**Implementação:**
- Instrução no system prompt: *"Never state delivery deadlines, monetary values, or contractual terms without citing the exact document name, version number, and section. If the retrieved context does not contain this information explicitly, respond with the fallback message."*
- Validação pós-geração: camada de parsing verifica padrões de prazo/valor (regex) e presença do campo Fonte. Se ausente, a resposta é bloqueada e o fallback é exibido.

**Motivação no domínio NovaTech:**
Prazos de entrega têm impacto direto em SLAs contratuais. Uma afirmação incorreta de "3 dias úteis" quando o contrato prevê "2 dias" pode gerar expectativa errada, reclamação formal e penalidade contratual. Sem esse **grounding** obrigatório, o modelo poderia interpolar valores de contratos diferentes e gerar respostas plausíveis, mas factualmente erradas.

---

## Guardrail 2 — Sinalizar ativamente quando a fonte é o FAQ informal

**Descrição:**
O knowledge base inclui documentos oficiais validados por Compliance e FAQ informal não revisado. Toda resposta baseada no FAQ deve exibir aviso explícito.

**Implementação:**
- Metadado `compliance_validated: true/false` em cada arquivo indexado no SharePoint.
- Se qualquer chunk usado tiver `compliance_validated: false`, exibe banner **antes** do conteúdo:
  ```
  ⚠️ ATENÇÃO: Esta resposta foi baseada (total ou parcialmente) no FAQ informal,
  que ainda não passou por validação do Compliance. Use com cautela e confirme
  com seu supervisor antes de comprometer com o cliente.
  ```

**Motivação no domínio NovaTech:**
FAQs informais acumulam conhecimento tácito valioso, mas também regras desatualizadas e interpretações pessoais. Em logística, uma regra de frete anotada informalmente pode ter sido revogada por atualização contratual. Sem esse guardrail, o assistente trataria documentos não validados com o mesmo peso que contratos oficiais. O aviso cria fricção intencional e saudável.

---

# Diagrama Visual

```
ATENDENTE RECEBE DÚVIDA DO CLIENTE
              |
              v
    [Digita consulta no assistente Teams]
              |
              v
    +---------+---------+
    |  Pipeline RAG      |
    |  (Azure AI Search) |
    |  Busca semântica   |
    +---------+---------+
              |
    +---------v-----------+
    | Score >= 0,72?      |
    +----+----------+-----+
         |          |
        SIM        NÃO
         |          |
         v          v
  [FLUXO PRINCIPAL] [FLUXO FALLBACK]
         |          |
         |    +-----+---------------------------+
         |    | Exibe fallback sem suposição    |
         |    | Escalar supervisor ou registrar |
         |    +-----+---------------------------+
         |          |
         v          |
  [Resposta + Fonte + Seção]
  [Temperatura 0.1]
         |
  +------v------+
  | compliance_ |
  | validated?  |
  +--+------+---+
     |      |
    SIM    NÃO
     |      |
     |   [Banner ⚠️ FAQ informal]
     |      |
     +--+---+
        |
        v
  [Atendente usa no chamado]
        |
  +-----v------+
  | Correta?   |
  +--+-----+---+
     |     |
    SIM   NÃO
     |     |
     |  [FLUXO FEEDBACK]
     |     |
     |  [Formulário isolado — sem herdar
     |   contexto da sessão → mitiga context rot]
     |     |
     |  [Fila curadoria SharePoint]
     |     |
     |  [Re-indexação ou novo doc]
     |     |
     |  [Notificação ao atendente]
     |     |
     +--+--+
        |
        v
   [Log registrado + chamado encerrado]
```

---

# Nota sobre o Feedback Loop

## Por que RAG precisa de manutenção contínua

Um assistente RAG não é produto estático. Ele é tão bom quanto os documentos que indexa — e documentos de logística mudam constantemente: tabelas de frete são reajustadas, SLAs são renegociados, políticas evoluem com regulamentações. Sem curadoria ativa, o knowledge base envelhece silenciosamente.

**O problema do "drift" de conhecimento:**
O modelo de linguagem não aprende com o uso (não há fine-tuning contínuo). O que evolui é o knowledge base recuperado. Documentos defasados fazem o assistente responder com convicção sobre uma realidade que não existe mais — potencialmente pior do que não ter o assistente, pois gera falsa segurança.

**Cadência recomendada para a NovaTech:**
- **Semanal:** revisão da fila de feedbacks + re-indexação de documentos atualizados
- **Mensal:** revisão dos logs de fallback para identificar padrões de lacuna sistemática
- **Trimestral:** auditoria completa do knowledge base — remoção de obsoletos, verificação de versões
