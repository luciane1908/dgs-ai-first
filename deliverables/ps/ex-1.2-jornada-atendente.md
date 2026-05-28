---
papel: Product Specialist
fase: 1
exercicio: 1.2
titulo: Design de Jornada com Componente de IA
temperatura-usada: 0.2
autora: Luciane Baldo
empresa: DB1 Global Software
data: 2026-05-28
versao: 1.0
enunciado: exercises/exercicio-fase-1-entendimento.md#product-specialist-exercicio-12
contexto_usado:
  - context/anexo-a-documentacao-simulada-novatech.md
  - dados discovery (simulados): ver enunciado
---

# PS — Exercício 1.2: Design de Jornada com Componente de IA

> Enunciado: `exercises/exercicio-fase-1-entendimento.md`
> Base: dados do discovery simulados — 4 fontes por chamado, top dúvidas: prazos (35%), frete (25%), devolução (20%), outros (20%). 15% dos casos sem resposta → escalonamento.

---

## Jornada Textual Estruturada

### Fluxo Principal — Consulta com resposta encontrada

```
1. Atendente recebe dúvida do cliente durante o chamado

2. Atendente digita a pergunta em linguagem natural no assistente
   (ex: "qual o prazo de devolução para carga refrigerada?")

3. Pipeline RAG executa:
   a. Embedding da pergunta
   b. Busca vetorial nos chunks indexados
   c. Recuperação dos N chunks com maior score de similaridade
   d. Montagem do prompt: system prompt + chunks + pergunta

4. LLM gera resposta com:
   - Resposta direta em português formal
   - Citação obrigatória: [FONTE: POL-001, seção 3.2]
   - Trecho relevante destacado

5. Atendente lê a resposta e a usa no atendimento ao cliente

6. Atendente marca a resposta como útil (feedback positivo)
   → Dado registrado para monitoramento de qualidade
```

---

### Fluxo de Fallback — Assistente não encontra resposta

```
1. Score de similaridade dos chunks recuperados está abaixo do threshold
   (ex: < 0.75 — configurável)

2. Assistente exibe mensagem padronizada:
   "Não encontrei informação sobre este tema na documentação disponível.
    Recomendo escalar para o supervisor ou consultar diretamente
    [área responsável conforme o tema]."

3. Atendente decide:
   a. Escalar para supervisor → registra o tema no sistema de chamados
   b. Reformular a pergunta e tentar novamente
   c. Buscar manualmente nas fontes originais

4. Se o atendente escalou → supervisor resolve e registra a resposta
   → Gap documentado para alimentar processo de curadoria da base
```

---

### Fluxo de Feedback — Resposta incorreta ou desatualizada

```
1. Atendente recebe resposta do assistente

2. Atendente identifica que a resposta está incorreta ou desatualizada
   (ex: multiplicador de frete citado é da versão antiga do PROC-042)

3. Atendente clica em "Reportar problema" e seleciona o motivo:
   - Informação incorreta
   - Informação desatualizada
   - Fonte citada não existe
   - Resposta incompleta

4. Registro é enviado para fila de revisão da equipe de curadoria

5. Curador analisa:
   a. Confirma o problema → atualiza ou remove o documento da base
   b. Rejeita o reporte → registra justificativa

6. Base documental atualizada → pipeline de re-ingestão executado
   → Qualidade melhora progressivamente com o uso
```

---

## Guardrails de Comportamento (específicos ao domínio logística/atendimento)

**Guardrail 1 — Nunca afirmar prazo ou valor sem citação de fonte:**
> O assistente não pode gerar prazos (dias úteis, tempo de resposta SLA) ou valores (multiplicadores de frete, percentuais de seguro) sem citar explicitamente o documento e a seção de origem. Se os chunks recuperados não contiverem o valor exato, o assistente deve dizer que não encontrou, mesmo que possa inferir.

*Motivação:* Um prazo incorreto gera promessa indevida ao cliente. A citação da fonte permite que o atendente valide a informação antes de comunicá-la.

**Guardrail 2 — Sinalizar ativamente quando a fonte é o FAQ informal:**
> Quando a resposta for baseada em chunks do FAQ-Atendimento, o assistente deve exibir aviso explícito: *"Esta informação vem de um documento informal do time de atendimento e não foi validada por Compliance ou Operações. Confirme com seu supervisor antes de usar."*

*Motivação:* O FAQ cobre temas sem documentação formal (carga danificada, frete expresso para carga perigosa). Sem esse guardrail, o atendente não tem como distinguir se está lendo uma política oficial ou uma prática informal de um colega.

---

## Diagrama Visual de Fluxo

> Descrição para geração em Claude Design / ferramenta de diagramação.

```
DIAGRAMA: Jornada do Atendente — Assistente IA NovaTech
════════════════════════════════════════════════════════

[INÍCIO] Atendente recebe dúvida do cliente
              │
              ▼
     ┌─────────────────┐
     │ Digita pergunta │
     │ no assistente   │
     └────────┬────────┘
              │
              ▼
     ┌─────────────────────────────────────┐
     │ Pipeline RAG: busca + ranking chunks │
     └────────────────┬────────────────────┘
                      │
          ┌───────────┴───────────┐
          │                       │
    Score ≥ threshold        Score < threshold
          │                       │
          ▼                       ▼
  ┌───────────────┐      ┌─────────────────────┐
  │ Gera resposta │      │ "Não encontrei.      │
  │ com citação   │      │  Escale ou reformule"│
  └───────┬───────┘      └──────────┬──────────┘
          │                         │
          ▼                         ▼
  ┌───────────────┐         ┌───────────────┐
  │ Atendente usa │         │ Atendente     │
  │ no atendimento│         │ escala para   │
  └───────┬───────┘         │ supervisor    │
          │                 └───────┬───────┘
    ┌─────┴──────┐                  │
    │            │                  ▼
  Útil?      Problema?      ┌───────────────┐
    │            │           │ Gap registrado│
    ▼            ▼           │ para curadoria│
  [👍 OK]  [⚠ Reportar]     └───────────────┘
               │
               ▼
      ┌─────────────────┐
      │ Fila de revisão │
      │ de curadoria    │
      └────────┬────────┘
               │
               ▼
      ┌─────────────────┐
      │ Base atualizada │
      │ Re-ingestão     │
      └─────────────────┘

LEGENDA:
  Fluxo principal:  ─────►  (resposta encontrada)
  Fluxo fallback:   ─────►  (sem resposta)
  Fluxo feedback:   ─────►  (resposta incorreta)
```

---

## Nota sobre o feedback loop

Os três fluxos convergem para o mesmo princípio: **RAG precisa de manutenção contínua**. A qualidade do assistente no dia 1 é diferente da qualidade no mês 6 — e essa diferença depende de:

1. Atendentes reportando respostas incorretas (fluxo de feedback)
2. Gestores registrando temas sem cobertura (fluxo de fallback)
3. Curadores atualizando a base com base nos reportes

Sem esse ciclo, a base documental envelhece enquanto as perguntas evoluem — e o assistente passa a responder com confiança usando informações desatualizadas.
