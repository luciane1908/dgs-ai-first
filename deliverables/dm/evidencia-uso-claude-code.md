---
tipo: evidencia-de-uso
ferramenta: Claude Code (CLI)
papel: Delivery Manager
fase: 1
autora: Luciane Baldo
empresa: DB1 Global Software
data: 2026-05-28
referencia_entregavel: deliverables/dm/fase-1-entregavel.md
---

# Evidência de Uso — Claude Code como Ferramenta de Iteração

> Este documento registra o histórico de iterações realizadas com o **Claude Code**
> durante a produção do entregável `fase-1-entregavel.md`, demonstrando o uso
> da ferramenta como par de trabalho — não como gerador acrítico de conteúdo.

---

## 1. Configuração do Ambiente (Git + GitHub)

**Ação:** Instalação e configuração do Git, criação do repositório público no GitHub e clone local.

**Interações relevantes:**
- Verificação da instalação: `git --version` → `git version 2.54.0.windows.1`
- Configuração de identidade:
  ```bash
  git config --global user.name "Luciane Baldo"
  git config --global user.email "luciane.baldo@db1.com.br"
  ```
- Repositório criado: https://github.com/luciane1908/dgs-ai-first
- Clone executado com sucesso; primeiro commit detectado: `ae591b7 Initial commit`

**Aprendizado registrado:** O Git foi configurado automaticamente com dados do hostname antes da configuração manual — o que demonstra na prática o conceito de comportamento padrão vs. comportamento configurado, análogo à temperatura padrão de um LLM.

---

## 2. Ingestão dos Arquivos de Contexto da NovaTech

**Ação:** Commit dos 8 arquivos de contexto da Prática 1 no repositório.

**Arquivos commitados:**
| Arquivo | Conteúdo |
|---------|----------|
| `FAQ-atendimento.md` | Perguntas frequentes do time de suporte (doc informal) |
| `POL-001-politica-devolucao.md` | Política oficial de devolução v3.1 |
| `PROC-042-frete-especial-v1.md` | Cálculo de frete especial — versão 1.0 (mar/2023) |
| `PROC-042-v2-frete-especial-revisado.md` | Cálculo de frete especial — versão 2.0 (nov/2023) |
| `SLA-2024-tabela-sla-clientes.md` | Tabela de SLA por tier de cliente |
| `anexo-a-documentacao-simulada-novatech.md` | Compilação completa dos 5 docs com meta-análise |
| `anexo-b-chunks-referencia-rag.md` | Chunks pré-processados + mapa de cobertura RAG |
| `exercicio-fase-1-entendimento.md` | Enunciado completo dos exercícios por papel |

**Comando executado:**
```bash
git add .
git commit -m "Adiciona arquivos de contexto da Prática 1"
git push
```

---

## 3. Leitura e Análise dos Arquivos de Contexto

**Ação:** Solicitação de resumo do conteúdo de todos os arquivos `.md` do repositório.

**Método:** Claude Code leu os 8 arquivos em paralelo e produziu resumo estruturado por arquivo, incluindo:
- Natureza do documento (formal/informal, versão, responsável)
- Principais regras e dados
- Alertas sobre contradições e gaps identificados

**Iteração crítica identificada nesta etapa:**
> A leitura revelou que `PROC-042-v1` e `PROC-042-v2` coexistem sem indicação de vigência — contradição que foi incorporada como **Risco 2** na matriz de riscos do Exercício 1.1, com referência explícita aos arquivos reais do repositório.

---

## 4. Produção do Entregável — Versão 1 (v1)

**Ação:** Geração do primeiro entregável completo dos 3 exercícios do papel Delivery Manager.

**Arquivo gerado:** `exercicio-dm-fase1-entregavel.md` (326 linhas)

**Conteúdo produzido:**
- Exercício 1.1: tabela de conceitos técnicos (tokens, janela de contexto, temperatura, grounding), matriz de 5 riscos com probabilidade/impacto/mitigação, 3 perguntas ao Tech Lead
- Exercício 1.2: e-mail ao diretor de operações + one-pager visual
- Exercício 1.3: plano de discovery com fase de Intent (IA) e discovery humano, cronograma de 2 semanas, tabela de insumos da NovaTech

**Commit:**
```bash
git commit -m "Adiciona entregavel exercicios DM Fase 1 - Fundamentos IA Generativa"
```

---

## 5. Iteração 1 — Revisão de Boas Práticas (v1 → v2)

**Prompt de refinamento:**
> *"Segundo as melhores práticas para economizar tokens e teoria de construção de agents, skills, harness etc... qual o melhor formato para conduzir e criar arquivos para resolver esse exercício?"*

**Diagnóstico gerado pelo Claude Code:**

| Problema identificado na v1 | Impacto |
|-----------------------------|---------|
| Cenário completo da NovaTech copiado no entregável | ~500 tokens desperdiçados por leitura |
| Tabela de conceitos repetida (já existe no enunciado) | ~200 tokens duplicados |
| Sem frontmatter de metadados | Nenhuma rastreabilidade para agentes |
| Sem referências cruzadas entre arquivos | Pipeline RAG não consegue fazer linkage |
| Estrutura flat (tudo na raiz) | Impossível aplicar chunking por responsabilidade |

**Decisão tomada:** Refatorar o repositório para separar responsabilidades.

---

## 6. Refatoração do Repositório

**Ação:** Reorganização completa da estrutura de arquivos.

**Antes:**
```
dgs-ai-first/
├── README.md
├── FAQ-atendimento.md
├── POL-001-politica-devolucao.md
├── PROC-042-frete-especial-v1.md
├── PROC-042-v2-frete-especial-revisado.md
├── SLA-2024-tabela-sla-clientes.md
├── anexo-a-documentacao-simulada-novatech.md
├── anexo-b-chunks-referencia-rag.md
├── exercicio-fase-1-entendimento.md
└── exercicio-dm-fase1-entregavel.md   ← v1 (326 linhas, contexto repetido)
```

**Depois:**
```
dgs-ai-first/
├── README.md
├── CLAUDE.md                          ← novo: contexto persistente do projeto
├── context/                           ← novo: documentos NovaTech
│   ├── FAQ-atendimento.md
│   ├── POL-001-politica-devolucao.md
│   ├── PROC-042-frete-especial-v1.md
│   ├── PROC-042-v2-frete-especial-revisado.md
│   ├── SLA-2024-tabela-sla-clientes.md
│   ├── anexo-a-documentacao-simulada-novatech.md
│   └── anexo-b-chunks-referencia-rag.md
├── exercises/                         ← novo: enunciados
│   └── exercicio-fase-1-entendimento.md
└── deliverables/dm/                   ← novo: entregáveis DM
    └── fase-1-entregavel.md           ← v2 (183 linhas, sem repetição)
```

**Commit:**
```bash
git commit -m "Refatora estrutura do repositorio: context/, exercises/, deliverables/ + CLAUDE.md + entregavel DM v2 otimizado"
```

**Resultado:** 11 arquivos alterados, 257 inserções, 326 deleções.

---

## 7. Produção do CLAUDE.md

**Ação:** Criação do arquivo de contexto persistente do projeto.

**Conteúdo definido:**
- Identificação da autora e papel na certificação
- Cenário resumido da NovaTech (sem repetir os docs completos)
- Mapa da estrutura do repositório
- Regras para produção de entregáveis (nunca repetir contexto, usar frontmatter, referenciar por caminho)

**Princípio aplicado:** O `CLAUDE.md` é carregado automaticamente em toda sessão do Claude Code (~50 tokens), evitando que o agente precise "redescobrir" o contexto do projeto a cada interação.

---

## 8. Entregável v2 — Resultado da Iteração

**Arquivo:** `deliverables/dm/fase-1-entregavel.md`

**Melhorias em relação à v1:**

| Dimensão | v1 | v2 |
|----------|----|----|
| Linhas | 326 | 183 |
| Tokens estimados | ~4.200 | ~2.300 |
| Contexto repetido | Cenário completo copiado | Referência de 1 linha |
| Metadados | Nenhum | Frontmatter YAML completo |
| Referências cruzadas | Nenhuma | Cita `context/PROC-042-v1` por nome |
| Rastreabilidade | Nenhuma | Links para enunciado e contexto |
| Reusabilidade | Arquivo isolado | Integrado à estrutura do repositório |

**Estrutura do frontmatter:**
```yaml
---
papel: Delivery Manager
fase: 1
exercicios: [1.1, 1.2, 1.3]
autora: Luciane Baldo
data: 2026-05-27
versao: 2.0
enunciado: exercises/exercicio-fase-1-entendimento.md
contexto_novatech: context/anexo-a-documentacao-simulada-novatech.md
---
```

---

## 9. Validação do Conteúdo

**Ação:** Solicitação de exibição do conteúdo completo do entregável para validação humana.

**Método:** Claude Code leu o arquivo e exibiu as 233 linhas com numeração, permitindo revisão linha a linha.

**Foco da validação:** Matriz de 5 riscos (linhas 36–42), confirmada como correta e completa.

---

## Resumo das Iterações

| # | Ação | Ferramenta | Resultado |
|---|------|-----------|-----------|
| 1 | Configuração Git + GitHub | Claude Code + PowerShell | Repositório criado e configurado |
| 2 | Commit dos arquivos de contexto | Claude Code + Git | 8 arquivos versionados |
| 3 | Análise dos arquivos de contexto | Claude Code (leitura paralela) | Mapa de conteúdo + contradições identificadas |
| 4 | Geração do entregável v1 | Claude Code | 326 linhas, 3 exercícios completos |
| 5 | Revisão de boas práticas | Claude Code (agente especializado) | Diagnóstico de ineficiências |
| 6 | Refatoração do repositório | Claude Code + PowerShell + Git | Estrutura reorganizada em 4 camadas |
| 7 | Criação do CLAUDE.md | Claude Code | Contexto persistente do projeto |
| 8 | Geração do entregável v2 | Claude Code | 183 linhas, otimizado, com frontmatter |
| 9 | Validação do conteúdo | Claude Code (leitura) | Conteúdo confirmado |

---

## Reflexão sobre o Uso da Ferramenta

O Claude Code foi usado neste exercício como **par de trabalho iterativo**, não como gerador único de conteúdo. As iterações demonstram:

1. **Refinamento progressivo:** o entregável v1 foi produzido, avaliado criticamente e refatorado com base em princípios técnicos reais (token economy, separação de responsabilidades)

2. **Claude como revisor:** a pergunta sobre boas práticas de agents/harness usou o Claude para auditar o próprio output anterior — identificando redundâncias que um humano poderia não perceber

3. **Decisões justificadas:** cada mudança estrutural (criar `context/`, usar frontmatter, criar `CLAUDE.md`) foi motivada por um princípio técnico explicado, não por preferência estética

4. **Rastreabilidade:** este documento de evidência é ele próprio um entregável gerado via Claude Code, commitado no repositório — fechando o ciclo de uso da ferramenta como parte do processo de trabalho
