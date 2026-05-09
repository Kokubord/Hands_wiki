<!-- wiki-ingest: synced from HANDOVER-METHODOLOGY.md at 2026-05-09T18:04:15.344Z — edit upstream in monorepo, not here -->

# HANDOVER Methodology — documento-semente auto-contido

> **Este ficheiro é a "bíblia de recuperação" da metodologia HANDOVER.**
>
> Se perderes tudo — o computador, o repositório, o Cursor, as regras, as skills — e ficares só com este ficheiro (num email a ti próprio, num gist, num drive), qualquer agente LLM ou colaborador humano consegue **reinstalar o sistema inteiro num projecto novo** apenas lendo-o. Todos os templates estão aqui em cópia integral. Não há dependências externas.
>
> **Guarda este ficheiro fora do teu disco local.** Recomendações no final.

---

## Índice

1. [O problema que esta metodologia resolve](#1-o-problema-que-esta-metodologia-resolve)
2. [Os três pilares](#2-os-três-pilares)
3. [Fluxo diário — rituais de início e fim de sessão](#3-fluxo-diário--rituais-de-início-e-fim-de-sessão) — *subsecção «Precedência entre decisões» no mesmo capítulo*
4. [Glossário — o que conta como "decisão material"](#4-glossário--o-que-conta-como-decisão-material)
5. [Instalação manual num projecto novo (passo-a-passo, sem ferramentas)](#5-instalação-manual-num-projecto-novo)
6. [Template 1 — `HANDOVER.md`](#template-1--handovermd)
7. [Template 2 — `.cursor/rules/session-handover.mdc`](#template-2--cursorrulessession-handovermdc)
8. [Template 3 — `AGENTS.md`](#template-3--agentsmd)
9. [Template 4 — ADR inicial `docs/adr/0001-adopcao-metodologia-handover.md`](#template-4--adr-inicial)
10. [Como construir o agente Handover para speckit (extensão completa)](#10-como-construir-o-agente-handover-para-speckit)
11. [Como construir o agente Handover como Cursor skill user-global](#11-como-construir-o-agente-handover-como-cursor-skill-user-global)
12. [Como construir agentes Handover fora do Cursor (ChatGPT / Claude / aider / outros)](#12-como-construir-agentes-handover-fora-do-cursor)
13. [Cenários de recuperação](#13-cenários-de-recuperação)
14. [FAQ — perguntas frequentes](#14-faq)
15. [Onde guardar este documento (backup off-site)](#15-onde-guardar-este-documento)
16. [Licença e uso](#16-licença-e-uso)

---

## 1. O problema que esta metodologia resolve

Quando trabalhas num projecto com **múltiplos agentes LLM (Cursor, Claude, etc.) em paralelo** — várias janelas de chat abertas ao mesmo tempo, frequentemente por vários dias — acontece inevitavelmente o seguinte:

- **Chat A** decide que vai pivotar da tecnologia X para a tecnologia Y. Escreve código nesse sentido.
- **Chat B**, ao mesmo tempo, continua a assumir que a tecnologia é X. Escreve código incompatível.
- **Chat C**, dois dias depois, tenta editar um ficheiro e não tem contexto nenhum do que A ou B fizeram.

**Os chats não partilham contexto entre si.** O único canal de comunicação real entre eles é o **repositório git**. Tudo o que for decidido em conversa mas não aterrar num ficheiro versionado **deixa de existir** para os próximos agentes — mesmo que pareça óbvio no momento.

Esta metodologia resolve isto com **três ficheiros disciplinados + uma regra carregada automaticamente**. Em menos de 5 minutos de instalação, um projecto passa a ter memória partilhada entre chats paralelos.

---

## 2. Os três pilares

| Pilar | Ficheiro | Papel |
|---|---|---|
| **1. Memória viva** | `HANDOVER.md` na raiz do projecto (ou sub-pasta principal) | Fotografia operacional do projecto: trilhos em curso, decisões frescas dos últimos 7 dias, ficheiros em voo activo por cada chat, open threads pendentes. |
| **2. Reforço automático** | `.cursor/rules/session-handover.mdc` com `alwaysApply: true` | Carregado no system prompt de *toda* sessão de agente. Obriga a ler o HANDOVER no início e a escrever no fim (se houve decisão material). |
| **3. Trilha auditável** | `AGENTS.md` na raiz + ADRs em `docs/adr/NNNN-*.md` + convenção de commits com tags `DECISION(...)` e `HANDOVER(...)` | Para quem quiser reconstituir o histórico num ano, um `git log --grep='DECISION\|HANDOVER'` dá o esqueleto imediato. ADRs são a fonte de verdade arquitectural. |

Os três trabalham juntos: a **regra (pilar 2)** garante que os agentes **lêem e escrevem no HANDOVER (pilar 1)**, e os **commits tagueados + ADRs (pilar 3)** deixam trilha auditável para o futuro.

---

## 3. Fluxo diário — rituais de início e fim de sessão

### Início de cada sessão (agente ou humano)

A regra `alwaysApply: true` já faz os agentes LLM lerem isto sozinhos. Para humanos:

1. Abrir `HANDOVER.md` → ver «Meta», «Trilhos em curso», «Decisões frescas», «Ficheiros em voo activo».
2. Ler o ADR mais recente em `docs/adr/`.
3. `git status --short` + `git log --since="3 days ago" --oneline`.
4. Se o trabalho vai tocar num ficheiro listado em «Ficheiros em voo activo» por outro chat — **parar e perguntar** antes de editar.

### Fim de cada sessão que tomou decisão material

Três escritas obrigatórias, em ordem:

1. **`HANDOVER.md`** → adicionar uma linha em «Decisões frescas»: `YYYY-MM-DD — <decisão curta> — <chat-id ou ADR>`. Actualizar «Meta → Última actualização».
2. **ADR** → se for decisão arquitectural (pivot, novo contrato de API, modelo de dados, trade-off significativo), criar `docs/adr/NNNN-<slug>.md` seguindo o template.
3. **Commit** → `DECISION(scope): <resumo> [ADR-NNNN]`.

Se a sessão apenas consolidou notas no HANDOVER (sem código novo), o commit é `HANDOVER(scope): <o que consolidei>`.

### Regra de ouro

> **Se uma decisão foi tomada em conversa e não foi materializada num ficheiro deste repositório, ela não existe para os próximos agentes — mesmo se parece óbvia neste momento. Escreva.**

### Precedência entre decisões (o mesmo tema, datas diferentes)

Distinto de «pedido do utilizador vs o que está escrito no HANDOVER»: aqui **duas sessões** deixaram regras **incompatíveis** sobre o mesmo assunto (datas diferentes em «Decisões frescas»).

- **Regra:** **a definição mais recente prevalece** — desde que esteja consubstanciada no repositório (entrada datada, código, ADR novo ou aditamento).
- **Não** deixar dois bullets a contradizerem-se sem ponte. Ao actualizar: (1) nova entrada no topo com a regra **vigente**; (2) na entrada antiga, *superseded* ou fusão *Antes: … / Depois (YYYY-MM-DD): …*.
- **ADRs:** para o mesmo tema, prevalece o **ADR mais recente** (número mais alto ou aditamento). Se o `HANDOVER` citar comportamento obsoleto, **actualiza o HANDOVER**; se o ADR estiver errado, **corrige o ADR** e documenta.

O mesmo texto está replicado no [Template 2](#template-2--cursorrulessession-handovermdc) (ficheiro `.cursor/rules/session-handover.mdc`) e na regra viva do repositório.

---

## 4. Glossário — o que conta como "decisão material"

Considera-se **decisão material** (e portanto obrigatório HANDOVER + ADR + commit `DECISION`):

- Escolher entre dois caminhos técnicos com trade-offs relevantes.
- Abandonar, pivotar ou reintroduzir integração externa.
- Mudar contrato de API (request/response) ou de tabela (schema de BD).
- Mudar fluxo UX em ecrã partilhado por vários trilhos.
- Definir/alterar feature flag, regra de permissão (RBAC), convenção global.
- Editar uma migration já publicada (requer justificação no ADR).

**Não** é decisão material (commits normais, sem entrada no HANDOVER):

- Typo em copy.
- Renomeação local de variável.
- Fix de lint ou formatação.
- Pequeno ajuste CSS cosmético.
- Actualização de dependência patch/minor sem mudança de comportamento.

---

## 5. Instalação manual num projecto novo

Este processo funciona em **qualquer projecto**, use ou não use speckit, Cursor, ou qualquer outra ferramenta. Só precisa de git.

### Passo 1 — Decidir a localização dos ficheiros

Dois layouts comuns:

**Layout A — monorepo com app principal dentro** (caso ClinicaGestor):

```text
/
├── <app-principal>/
│   ├── HANDOVER.md
│   └── docs/adr/
├── .cursor/rules/session-handover.mdc
└── AGENTS.md
```

**Layout B — projecto simples, app na raiz**:

```text
/
├── HANDOVER.md
├── docs/adr/
├── .cursor/rules/session-handover.mdc
└── AGENTS.md
```

Escolhe um e mantém ao longo do projecto. No que se segue assumo Layout B (ajusta os caminhos se usares Layout A).

### Passo 2 — Criar os 3 ficheiros a partir dos templates

Copia o conteúdo das secções [Template 1](#template-1--handovermd), [Template 2](#template-2--cursorrulessession-handovermdc) e [Template 3](#template-3--agentsmd) para os caminhos correctos, substituindo os placeholders `{{PROJECT_NAME}}`, `{{PROJECT_DESCRIPTION}}`, `{{PRIMARY_LANGUAGE}}`, `{{TECH_STACK}}` pelos valores reais.

### Passo 3 — Criar pasta de ADRs e primeiro ADR

```bash
mkdir -p docs/adr
```

Copiar [Template 4](#template-4--adr-inicial) para `docs/adr/0001-adopcao-metodologia-handover.md`. Este primeiro ADR é auto-referencial: regista que *o projecto adoptou este protocolo*, com data e autor.

### Passo 4 — Primeiro commit

```bash
git add HANDOVER.md AGENTS.md .cursor/rules/session-handover.mdc docs/adr/0001-adopcao-metodologia-handover.md
git commit -m "DECISION(meta): adoptar metodologia HANDOVER para coordenação entre chats paralelos [ADR-0001]"
```

### Passo 5 — Verificação

Abrir um chat novo com um agente LLM que suporte Cursor rules (ou colar manualmente o conteúdo de `session-handover.mdc` no system prompt). Fazer uma pergunta técnica sobre o projecto. **Comportamento esperado:** o agente menciona que leu o `HANDOVER.md` antes de responder. Se sim, metodologia activa. ✅

---

## Template 1 — `HANDOVER.md`

> Copiar inteiro para `HANDOVER.md` (ou `<app>/HANDOVER.md` em Layout A). Substituir placeholders `{{...}}`. As secções exemplo de «Decisões frescas», «Trilhos» e «Ficheiros em voo» devem ficar vazias inicialmente com apenas um placeholder explicativo — vão-se enchendo naturalmente com o uso.

```markdown
# HANDOVER — estado vivo do projecto {{PROJECT_NAME}}

> **Se você é um agente LLM a começar uma sessão neste repositório, LEIA ISTO ANTES DE AGIR.**
> **Se você é humano, este ficheiro é a fotografia operacional mais recente, mantida por convenção (não por automação).**

---

## Meta

| Campo | Valor |
|---|---|
| **Última actualização** | {{YYYY-MM-DD}} (inicialização) |
| **Actualizado por** | chat inaugural (adopção da metodologia HANDOVER) |
| **Chats activos identificados** | (preencher quando houver mais de um em voo) |
| **Disciplina** | Toda sessão com decisão material: actualizar esta secção + «Decisões frescas» + ADR quando aplicável. **Precedência:** decisão mais recente prevalece; ver secção «Precedência entre decisões» mais abaixo no mesmo documento. |

---

## Trilhos em curso (agrupados por tema)

> Cada "trilho" é uma linha de trabalho semi-independente, potencialmente com um chat dedicado. Adicionar subsecções `### N. <Nome do trilho>` à medida que emergem.

### 1. Plataforma base — setup inicial
- **Estado:** em arranque.
- **Superfícies em voo:** (listar ficheiros a tocar quando começar)
- **Dependências:** nenhuma ainda.

---

## Decisões frescas (últimos 7 dias)

> Formato: `YYYY-MM-DD — <decisão resumida> — <chat/PR/ADR>`. Entradas mais recentes primeiro. Entradas com mais de 7 dias podem ser arquivadas em `docs/handover-archive/YYYY-MM.md` ou apagadas se já estiverem cobertas por um ADR.

- {{YYYY-MM-DD}} — Adopção da metodologia HANDOVER (este ficheiro + rule + AGENTS.md). — [ADR-0001]

---

## Ficheiros em voo activo — evitar colisões

> Antes de editar um ficheiro desta tabela, **confirme com o owner-chat** (ou com o utilizador se não tiver a certeza). Remova da tabela quando terminar.

| Ficheiro | Owner (chat) | Razão |
|---|---|---|
| _(vazio)_ | — | — |

---

## Convenções em vigor

- **Commit convention:** Conventional Commits + dois tags especiais — `DECISION(scope)` para decisão material (deve referenciar ADR), `HANDOVER(scope)` para actualização deste ficheiro. Ver `AGENTS.md` na raiz.
- **Idioma da documentação:** {{LANGUAGE}}.
- **Feature flags, env vars, etc.:** (acrescentar à medida que forem criados)

---

## Open threads entre chats — TODO consolidado

> Coisas deixadas pendentes por um chat e que outro chat pode pegar. Remover quando executadas.

- _(vazio)_

---

## Histórico da arqueologia

> Quando um chat destila decisões dos últimos N dias a partir de transcripts, git log ou outras fontes, regista aqui data + âmbito + autor para auditoria futura.

- _(vazio)_
```

---

## Template 2 — `.cursor/rules/session-handover.mdc`

> Copiar inteiro para `.cursor/rules/session-handover.mdc`. Este ficheiro é **crucial** — é o que faz o agente ler o HANDOVER automaticamente em todas as sessões. Se o teu editor não for Cursor, vê a secção «FAQ — o que faço se não uso Cursor?» no final.

```markdown
---
description: Protocolo obrigatório de handover entre chats paralelos do mesmo projecto
alwaysApply: true
---

# Handover obrigatório entre sessões de agente

Este projecto tem **múltiplos agentes a trabalhar em paralelo** (vários chats abertos simultaneamente, muitas vezes por vários dias). Os chats **não partilham contexto entre si** — só o repositório e git são canais de comunicação. Para evitar decisões perdidas, colisões de ficheiros e retrabalho, siga o protocolo abaixo com disciplina.

## Início de sessão — leitura obrigatória

Antes de tocar em ficheiros ou de prometer ao utilizador um caminho técnico, leia por esta ordem:

1. **`HANDOVER.md`** — fotografia operacional mais recente: trilhos em curso, decisões frescas, ficheiros em voo activo por outros chats.
2. **`docs/adr/`** — ADRs são a fonte de verdade para decisões arquitecturais. Privilegie o(s) numericamente mais alto(s) e qualquer um referenciado pelo HANDOVER.
3. **`git log --since="3 days ago" --oneline`** — só é útil quando há commits frescos; este projecto pode acumular trabalho sem commitar entre *milestones*, por isso a ausência de commits **não** significa ausência de trabalho. Complemente sempre com `git status --short`.

Se encontrar **conflito** entre o pedido do utilizador e algo registado no HANDOVER/ADR (ex.: vai editar um ficheiro marcado como «owner = outro chat», ou reverter uma decisão registada), **pare e pergunte** em vez de improvisar.

### Precedência entre decisões (o mesmo tema, datas diferentes)

Isto é distinto do conflito «utilizador vs HANDOVER» acima: aqui **duas sessões** registaram regras **incompatíveis** sobre o mesmo assunto.

- **Regra:** **a definição mais recente prevalece** — desde que esteja consubstanciada no repositório (entrada datada em «Decisões frescas», alteração de código, ou ADR novo / aditamento).
- **Não** deixar dois bullets a contradizerem-se sem ponte. Ao actualizar: (1) nova entrada no topo com data de hoje descrevendo a regra **vigente**; (2) na entrada antiga, nota curta *«superseded YYYY-MM-DD — ver entrada do mesmo dia»* **ou** fusão num único bullet *«Antes: … / Depois (YYYY-MM-DD): …»*.
- **ADRs:** para o mesmo tema, prevalece o **texto do ADR mais recente** (número mais alto ou aditamento explícito); se o `HANDOVER.md` ainda citar comportamento antigo, **corrige-se o HANDOVER**, não o contrário (excepto se o ADR estiver errado — aí corrige-se o ADR e documenta-se).

## Durante a sessão — disciplina mínima

- Não apague artefactos de outro trilho («limpeza» drive-by) sem alinhamento — mesmo que pareçam órfãos. Comunique e espere.
- Ao criar ficheiros novos que claramente pertencem a um trilho existente, alinhe-se com a convenção do trilho (nome de pastas, estilo de imports, localização).
- Respeite UI já aprovada pelo utilizador — não mude ecrãs validados sem consentimento explícito.

## Fim de sessão — escrita obrigatória

Se a sessão tomou alguma **decisão material**, altere os três níveis certos antes de terminar:

1. **`HANDOVER.md`** → acrescentar **uma entrada** em «Decisões frescas (últimos 7 dias)» com `YYYY-MM-DD — <decisão> — <chat/ADR>`. Actualizar «Meta → Última actualização» e «Trilhos em curso» se o estado do trilho mudou.
2. **ADR novo** em `docs/adr/NNNN-<slug>.md` se for decisão arquitectural (pivot tecnológico, contrato de API, modelo de dados, trade-off significativo).
3. **Commit** com tag `DECISION(scope):` referenciando o ADR. Ver `AGENTS.md`.

Considera-se «decisão material» (lista não exaustiva):
- Escolher entre dois caminhos técnicos com trade-offs relevantes.
- Abandonar, pivotar ou reintroduzir integração externa.
- Mudar contrato de API (request/response) ou de tabela (schema de BD).
- Mudar fluxo UX em ecrã partilhado por vários trilhos.
- Definir/alterar feature flag, regra de permissão (RBAC), convenção global.

Arranjos triviais (typo, renomeação local, fix de lint) **não** precisam de entrada no HANDOVER — só commit.

## Ficheiros em voo activo

O HANDOVER tem uma tabela «Ficheiros em voo activo — evitar colisões». **Antes de editar qualquer ficheiro listado aí**, confirme com o utilizador se o chat owner ainda está activo ou se foi libertado. Quando terminar o seu trabalho num ficheiro, remova-o dessa tabela (ou mantenha-o se continuar em curso noutros chats).

## Princípio-raiz

Se uma decisão foi tomada em conversa e **não** foi materializada num ficheiro deste repositório, ela **não existe** para os próximos agentes — mesmo se parece óbvia neste momento. Escreva.
```

---

## Template 3 — `AGENTS.md`

> Copiar inteiro para `AGENTS.md` na raiz. Substituir placeholders.

```markdown
# AGENTS.md — guia para agentes LLM e contribuidores deste repositório

Este ficheiro é o **primeiro ponto de contacto** para qualquer contribuidor — humano ou LLM — que abre este repositório. Mantenha-o curto e prático.

## Estrutura do repositório

```text
/
├── HANDOVER.md                 # ⭐ fotografia operacional viva — LEIA PRIMEIRO
├── docs/
│   ├── adr/                    # Architectural Decision Records (fonte de verdade das decisões)
│   └── manuais/                # (opcional) manuais de utilizador final
├── .cursor/
│   └── rules/                  # Regras carregadas automaticamente em cada sessão de agente
└── AGENTS.md                   # (este ficheiro)
```

## Leitura mínima antes de agir

Para qualquer agente que inicie uma sessão neste repositório, por esta ordem:

1. `HANDOVER.md` — estado dos trilhos em curso e decisões frescas.
2. ADR numericamente mais alto em `docs/adr/` — último contexto arquitectural.
3. `git status --short` + `git log --since="3 days ago" --oneline` — trabalho recente em voo.
4. Regras em `.cursor/rules/*.mdc` — são carregadas automaticamente, mas vale a pena lê-las uma vez para saber o que o agente já «sabe».

## Convenção de commit-messages

Baseada em **Conventional Commits**, com dois tags adicionais próprios deste projecto:

| Prefixo | Quando usar | Exemplo |
|---|---|---|
| `feat(scope):` | Nova funcionalidade utilizador | `feat(auth): adicionar login por OTP` |
| `fix(scope):` | Correcção de bug | `fix(login): preservar callbackUrl após OTP` |
| `docs(scope):` | Só documentação | `docs(adr): adicionar ADR-0003` |
| `test(scope):` | Testes novos ou refactor de testes | `test(auth): cobertura do OTP` |
| `refactor(scope):` | Refactor sem mudar comportamento | `refactor(db): extrair connection helper` |
| `chore(scope):` | Infra, deps, build | `chore(deps): bumpar versão Node` |
| **`DECISION(scope):`** | Decisão material (tecnologia, pivot, contrato de API, RBAC) — **obrigatório referenciar ADR** | `DECISION(auth): adoptar Auth.js v5 [ADR-0002]` |
| **`HANDOVER(scope):`** | Actualização exclusivamente do `HANDOVER.md` e afins (regras, este ficheiro) | `HANDOVER(docs): destilar decisões da semana` |

Um `git log --grep='DECISION\|HANDOVER'` deve dar, em qualquer momento, o histórico condensado de decisões materiais do projecto — é esse o motivo dos dois tags próprios.

**Regra de ouro:** commits `DECISION(...)` **obrigam** a um ADR (novo ou aditamento ao existente). Commits sem ADR que tomem decisão material devem ser evitados.

## Para agentes LLM em chats paralelos

Este projecto pode ter múltiplas sessões de agente a correr em paralelo, frequentemente ao longo de vários dias. Para não pisar o trabalho dos outros:

1. Respeite o protocolo em `.cursor/rules/session-handover.mdc` (é `alwaysApply: true`, portanto já está no seu system prompt).
2. Antes de editar qualquer ficheiro listado na tabela «Ficheiros em voo activo» do HANDOVER, **confirme** com o utilizador.
3. No fim da sua sessão, escreva a entrada de «Decisões frescas» no HANDOVER **antes** de encerrar. Se não escrever, a decisão perde-se.
4. Se encontrar **duas decisões antigas no HANDOVER** sobre o mesmo tema que se contradizem, **prevalece a mais recente**; actualize a entrada antiga com *superseded* ou *Antes/Depois* (ver `HANDOVER-METHODOLOGY.md`).

## Para contribuidores humanos

- Documentação em {{LANGUAGE}} (coerência com estilo existente nos ADRs).
- Nada de segredos em commits (`.env` deve permanecer git-ignored; só `.env.example` é versionado).
- Antes de abrir PR, correr os testes do projecto.
```

---

## Template 4 — ADR inicial

> Copiar inteiro para `docs/adr/0001-adopcao-metodologia-handover.md`. Substituir placeholders. Este ADR auto-referencial marca a data de adopção do protocolo.

```markdown
# ADR-0001 — Adopção da metodologia HANDOVER para coordenação entre chats paralelos

- **Status:** aceite
- **Data:** {{YYYY-MM-DD}}
- **Autor:** {{NOME_OU_CHAT_ID}}

## Contexto

Este projecto utiliza agentes LLM (Cursor, Claude, etc.) como principal ferramenta de desenvolvimento, frequentemente com várias sessões de chat abertas em paralelo ao longo de dias. Os chats **não partilham contexto entre si** — cada um começa com memória em branco. O único canal real de comunicação entre agentes é o próprio repositório git.

Sem disciplina, isto produz três falhas recorrentes:

1. **Decisões perdidas** — um chat decide pivotar de X para Y, mas outro chat continua a escrever código assumindo X.
2. **Colisões de ficheiros** — dois chats editam o mesmo ficheiro em paralelo, produzindo merges problemáticos.
3. **Retrabalho** — um chat reinventa algo que outro já resolveu ontem.

## Decisão

Adopta-se a metodologia **HANDOVER** — três artefactos simples, versionados no repositório, que dão memória partilhada entre chats paralelos:

1. **`HANDOVER.md`** — fotografia viva do projecto: trilhos em curso, decisões frescas dos últimos 7 dias, ficheiros em voo activo, open threads.
2. **`.cursor/rules/session-handover.mdc`** com `alwaysApply: true` — regra carregada automaticamente no system prompt de toda sessão, que obriga a ler o HANDOVER no início e a escrever no fim.
3. **`AGENTS.md`** + convenção de commits com tags `DECISION(scope)` e `HANDOVER(scope)` + `docs/adr/` — trilha auditável para futuro.

Considera-se obrigatório actualizar estes três níveis sempre que uma sessão tomar uma "decisão material" — a definição completa está no próprio `session-handover.mdc`.

## Alternativas consideradas

- **Não fazer nada** (status quo) — rejeitado por já ter produzido colisões observáveis.
- **Sistema externo de gestão de decisões** (Notion, Linear, Confluence) — rejeitado porque os agentes LLM não lêem automaticamente esses sistemas e exige passo manual.
- **Apenas commits com mensagens descritivas** — rejeitado porque os agentes frequentemente fazem trabalho sem commitar entre *milestones*, deixando decisões apenas na conversa.

## Consequências

- ✅ Memória partilhada entre chats sem custo de infraestrutura.
- ✅ Rastreabilidade das decisões arquitecturais via ADRs + commits tagueados.
- ✅ `git log --grep='DECISION'` dá histórico condensado a qualquer momento.
- ⚠️ Exige disciplina — se os chats não escreverem, o sistema degrada. Mitigado pela rule `alwaysApply: true`.
- ⚠️ Entrada manual em `HANDOVER.md`. Mitigação futura: comando `/speckit.handover.update` assistido (ver Roadmap no `HANDOVER-METHODOLOGY.md`).

## Referências

- Documento-semente completo da metodologia: `HANDOVER-METHODOLOGY.md` (raiz).
- Templates: secções 6-9 do documento-semente.
```

---

## 10. Como construir o agente Handover para speckit

Se o projecto usar [spec-kit](https://github.com/github/spec-kit) (pasta `.specify/` + comandos `/speckit.*`), o caminho mais integrado é criar uma **extensão speckit** que providencia três comandos slash:

- `/speckit.handover.init` — inaugura o protocolo (semeia 4 ficheiros + ADR inicial)
- `/speckit.handover.update` — assistente para registar decisões materiais
- `/speckit.handover.archaeology` — destila decisões retrospectivas de `git log`, `git diff` e transcripts

Cada comando é um ficheiro Markdown com frontmatter YAML: o speckit lê o comando e envia o corpo ao agente LLM como instrução. Esta secção dá o **conteúdo integral** de cada ficheiro.

### 10.1 Estrutura de pastas a criar

```text
.specify/extensions/handover/
├── extension.yml
├── README.md
├── config-template.yml
├── templates/
│   ├── HANDOVER.md.template                                # cópia do Template 1
│   ├── AGENTS.md.template                                  # cópia do Template 3
│   ├── session-handover.mdc.template                       # cópia do Template 2
│   └── adr-0001-adopcao-metodologia-handover.md.template   # cópia do Template 4
├── commands/
│   ├── speckit.handover.init.md
│   ├── speckit.handover.update.md
│   └── speckit.handover.archaeology.md
└── scripts/bash/
    ├── init-handover.sh
    ├── update-handover.sh
    └── archaeology-handover.sh
```

Os 4 ficheiros em `templates/` são **cópias integrais** dos Templates 1-4 desta mesma metodologia (secções 6-9). Crie-os copiando o conteúdo dessas secções, mantendo os placeholders `{{...}}` por preencher.

### 10.2 Ficheiro completo `extension.yml`

```yaml
schema_version: "1.0"

extension:
  id: handover
  name: "Handover Coordination Protocol"
  version: "1.0.0"
  description: "Memória partilhada entre chats paralelos via HANDOVER.md, rule Cursor e commits tagueados"
  author: "<teu-nome-ou-org>"
  repository: "https://github.com/<user>/handover-speckit-extension"
  license: CC0

requires:
  speckit_version: ">=0.2.0"

provides:
  commands:
    - name: speckit.handover.init
      file: commands/speckit.handover.init.md
      description: "Inaugura protocolo HANDOVER (semeia 4 ficheiros + ADR-0001)"
    - name: speckit.handover.update
      file: commands/speckit.handover.update.md
      description: "Assistente para registar decisão material (HANDOVER + ADR + commit)"
    - name: speckit.handover.archaeology
      file: commands/speckit.handover.archaeology.md
      description: "Destila decisões dos últimos N dias a partir de git log e transcripts"

  config:
    - name: "handover-config.yml"
      template: "config-template.yml"
      description: "Configuração de janela de retenção e caminhos"
      required: false

hooks:
  after_constitution:
    command: speckit.handover.init
    optional: true
    prompt: "Inaugurar protocolo HANDOVER neste projecto?"
    description: "Cria HANDOVER.md, AGENTS.md, rule e ADR-0001 a partir dos templates"
  after_plan:
    command: speckit.handover.update
    optional: true
    prompt: "Registar decisões do planeamento no HANDOVER?"
  after_tasks:
    command: speckit.handover.update
    optional: true
    prompt: "Registar decisões da geração de tarefas no HANDOVER?"
  after_implement:
    command: speckit.handover.update
    optional: true
    prompt: "Registar decisões materiais da implementação no HANDOVER?"
  after_analyze:
    command: speckit.handover.update
    optional: true
    prompt: "Registar conclusões da análise no HANDOVER?"

tags:
  - "handover"
  - "coordination"
  - "multi-agent"
  - "git-workflow"

config:
  defaults:
    fresh_decisions_window_days: 7
    archive_path: "docs/handover-archive"
```

### 10.3 Ficheiro completo `config-template.yml`

```yaml
# Janela temporal (em dias) que a secção «Decisões frescas» cobre.
# Entradas mais antigas devem ser arquivadas ou apagadas (se cobertas por ADR).
fresh_decisions_window_days: 7

# Onde arquivar entradas antigas. Será criado no primeiro uso do comando de rotação.
archive_path: "docs/handover-archive"

# Layout do projecto:
#   A = monorepo com app principal em subpasta (ex.: clinicagestor/)
#   B = app na raiz do repo
layout: "B"

# Caminho do app principal quando layout = A.
app_dir: ""

# Idioma das docs.
language: "pt-PT"
```

### 10.4 Ficheiro completo `commands/speckit.handover.init.md`

```markdown
---
description: Inaugura o protocolo HANDOVER no projecto actual. Cria HANDOVER.md, .cursor/rules/session-handover.mdc, AGENTS.md e docs/adr/0001-adopcao-metodologia-handover.md a partir dos templates canónicos.
handoffs:
  - label: Registar primeira decisão
    agent: speckit.handover.update
    prompt: Regista a adopção da metodologia como primeira entrada em «Decisões frescas».
    send: true
---

## User Input

`​`​`text
$ARGUMENTS
`​`​`

## Pre-Execution Checks

1. Verificar se o repositório já tem `HANDOVER.md` (na raiz ou em subpasta de app). Se sim, ERROR "HANDOVER já existe em <path>; use /speckit.handover.update em vez de init".
2. Verificar se existe `AGENTS.md` na raiz. Se sim, comparar com o template — se for diferente, pedir confirmação antes de sobrescrever.
3. Detectar layout: A (monorepo, app em subpasta) ou B (app na raiz). Heurísticas:
   - `package.json` na raiz → Layout B
   - `package.json` em subpasta única (ex.: `clinicagestor/package.json`) → Layout A
   - Em dúvida, perguntar ao user.

## Outline

1. **Recolher metadados** (de `$ARGUMENTS` ou interactivamente):
   - `PROJECT_NAME` — nome descritivo
   - `PROJECT_DESCRIPTION` — uma frase
   - `PRIMARY_LANGUAGE` — `pt-PT`, `pt-BR` ou `en-US`
   - `TECH_STACK` — uma linha (ex.: "Next.js 15 + Prisma + Postgres")
   - `LAYOUT` — A ou B
   - Se A: `APP_DIR` (nome da subpasta)

2. **Criar os 4 ficheiros a partir de `.specify/extensions/handover/templates/`**, substituindo `{{PROJECT_NAME}}`, `{{PROJECT_DESCRIPTION}}`, `{{PRIMARY_LANGUAGE}}`, `{{TECH_STACK}}`, `{{LANGUAGE}}`, `{{YYYY-MM-DD}}` (data de hoje) e `{{NOME_OU_CHAT_ID}}`:

   - `HANDOVER.md` → raiz (Layout B) ou `<APP_DIR>/HANDOVER.md` (Layout A)
   - `.cursor/rules/session-handover.mdc` → sempre na raiz
   - `AGENTS.md` → sempre na raiz
   - `docs/adr/0001-adopcao-metodologia-handover.md` → raiz (B) ou `<APP_DIR>/docs/adr/...` (A)

   Preferir executar via `.specify/extensions/handover/scripts/bash/init-handover.sh` quando disponível.

3. **Commit inaugural**:

   `​`​`bash
   git add HANDOVER.md AGENTS.md .cursor/rules/session-handover.mdc docs/adr/0001-adopcao-metodologia-handover.md
   git commit -m "DECISION(meta): adoptar metodologia HANDOVER para coordenação entre chats paralelos [ADR-0001]"
   `​`​`

4. **Reportar**:
   - Lista dos 4 ficheiros criados com caminhos absolutos
   - Confirmação de commit (SHA curto)
   - Instrução: na próxima sessão de agente, verificar que a rule é carregada automaticamente (o agente deve mencionar "li o HANDOVER" no início)
   - Apontar para as secções 3 (rituais) e 4 (glossário) de `HANDOVER-METHODOLOGY.md`
```

### 10.5 Ficheiro completo `commands/speckit.handover.update.md`

```markdown
---
description: Assistente interactivo para registar uma decisão material no HANDOVER. Adiciona entrada em «Decisões frescas», oferece criar ADR se for arquitectural, e propõe um commit tagueado.
---

## User Input

`​`​`text
$ARGUMENTS
`​`​`

## Pre-Execution Checks

1. Verificar que `HANDOVER.md` existe. Se não, ERROR "HANDOVER não existe; corra /speckit.handover.init primeiro".
2. Ler `HANDOVER.md` e extrair: trilhos existentes, chats registados na secção «Decisões frescas».

## Outline

1. **Recolher informação da decisão** (de `$ARGUMENTS` ou interactivamente):
   - O quê? (1 frase)
   - Porquê? (1 frase — o trade-off)
   - Trilho (escolher da lista ou "novo trilho" → pedir nome)
   - É decisão arquitectural? (sim/não — aplicar critério da secção 4 do METHODOLOGY)
   - Chat ID (últimos 8 chars do UUID da sessão, ou nome do user)
   - Scope do commit (ex.: `auth`, `db`, `agenda`, `meta`)

2. **Adicionar entrada em «Decisões frescas»**:

   `​`​`
   - YYYY-MM-DD — <decisão resumida> — chat `<chat-id>`[, ADR-NNNN]
   `​`​`

   Inserir no topo do bloco `### Chat <chat-id>` (criar bloco se não existir). Actualizar «Meta → Última actualização» para a data de hoje e «Actualizado por» para o chat-id actual.

3. **Se arquitectural**:
   - Determinar próximo NNNN: `ls docs/adr/ | grep -E '^[0-9]{4}-' | sort | tail -1`, +1.
   - Perguntar título do ADR e preencher as 4 secções: Contexto, Decisão, Alternativas consideradas, Consequências.
   - Criar `docs/adr/NNNN-<slug>.md` seguindo o formato dos ADRs existentes no projecto.
   - Anexar ` [ADR-NNNN]` à entrada de «Decisões frescas».

4. **Se mudou estado de um trilho** (ex.: "concluído", "bloqueado", "em revisão"):
   - Actualizar «Trilhos em curso» com o novo estado.

5. **Propor commit**:

   `​`​`bash
   git add HANDOVER.md[ docs/adr/NNNN-<slug>.md]
   git commit -m "DECISION(<scope>): <decisão> [ADR-NNNN]"
   # ou HANDOVER(<scope>): ... se não houver ADR
   `​`​`

   Pedir confirmação antes de executar.

6. **Reportar**:
   - Diff aplicado ao HANDOVER
   - ADR criado (caminho)
   - SHA do commit
```

### 10.6 Ficheiro completo `commands/speckit.handover.archaeology.md`

```markdown
---
description: Destila decisões retrospectivas a partir de git log, git diff e transcripts de chats Cursor (se acessíveis) dos últimos N dias. Propõe entradas para «Decisões frescas» do HANDOVER. Não escreve sem aprovação.
---

## User Input

`​`​`text
$ARGUMENTS
`​`​`

Argumentos aceites:
- `--days N` — janela temporal (default: 7)
- `--transcripts <path>` — pasta de transcripts Cursor (default: `~/.cursor/projects/<slug>/agent-transcripts`)
- `--dry-run` — só mostrar proposta, não escrever

## Outline

1. **Determinar janela temporal** (`--days N`, default 7).

2. **Recolher sinais**:
   - `git log --since="N days ago" --stat --format='commit %H%n%s%n%b%n---'`
   - `git diff --stat HEAD~<N>..HEAD` (se há commits na janela)
   - `git status --short` (trabalho não commitado)
   - Se `--transcripts <path>` fornecido e a pasta existir, listar `<uuid>.jsonl` com mtime nos últimos N dias.

3. **Processar transcripts** (se houver):
   - Para cada `.jsonl`, correr heurística de decisão material (regex sobre o conteúdo das mensagens):
     - Palavras-chave: `decidi(mos)?`, `pivot`, `escolh(i|emos)`, `abandon(ar|ámos)`, `vamos trocar`, `muda(r|mos) para`, `renomei(ar|ámos)`, `contrato de API`, `schema muda`, `feature flag`.
   - Se o transcript for grande (>200 KB), pode ser necessário delegar a sub-agente com prompt específico de extracção.

4. **Gerar propostas de entrada** no formato:

   `​`​`
   - YYYY-MM-DD — <decisão inferida> — <fonte: commit hash ou chat-id>*(inferido — validar)*
   `​`​`

   Marcar `*(inferido)*` sempre que a confiança não é alta. Agrupar por chat (se transcripts) ou por dia (se só commits).

5. **Apresentar diff proposto** ao user para `HANDOVER.md`. NÃO aplicar automaticamente.

6. **Se aprovado**:
   - Aplicar diff + commit `HANDOVER(docs): destilar decisões dos últimos N dias`.
   - Adicionar entrada em «Histórico da arqueologia» do HANDOVER com: data, janela, número de entradas, transcripts processados.

7. **Se recusado**: gravar a proposta como `.handover-archaeology-proposal-YYYY-MM-DD.md` no root (gitignored) para o user rever mais tarde.
```

### 10.7 Ficheiro completo `scripts/bash/init-handover.sh`

```bash
#!/usr/bin/env bash
# init-handover.sh — semear ficheiros HANDOVER num projecto novo
# Usage: init-handover.sh <project-name> <layout:A|B> <language> <tech-stack> [app-dir]

set -euo pipefail

PROJECT_NAME="${1:?project name required}"
LAYOUT="${2:-B}"
LANGUAGE="${3:-pt-PT}"
TECH_STACK="${4:-unspecified}"
APP_DIR="${5:-.}"
TODAY=$(date +%Y-%m-%d)
CHAT_ID="${HANDOVER_CHAT_ID:-$(whoami)@$(hostname -s)}"

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
TEMPLATES_DIR="$SCRIPT_DIR/templates"

if [ "$LAYOUT" = "B" ]; then
  APP_DIR="."
fi

render() {
  local src=$1
  local dst=$2
  mkdir -p "$(dirname "$dst")"
  sed -e "s|{{PROJECT_NAME}}|$PROJECT_NAME|g" \
      -e "s|{{PRIMARY_LANGUAGE}}|$LANGUAGE|g" \
      -e "s|{{LANGUAGE}}|$LANGUAGE|g" \
      -e "s|{{TECH_STACK}}|$TECH_STACK|g" \
      -e "s|{{YYYY-MM-DD}}|$TODAY|g" \
      -e "s|{{NOME_OU_CHAT_ID}}|$CHAT_ID|g" \
      "$src" > "$dst"
  echo "  created: $dst"
}

mkdir -p "$APP_DIR/docs/adr" ".cursor/rules"

render "$TEMPLATES_DIR/HANDOVER.md.template"                               "$APP_DIR/HANDOVER.md"
render "$TEMPLATES_DIR/AGENTS.md.template"                                  "AGENTS.md"
render "$TEMPLATES_DIR/session-handover.mdc.template"                       ".cursor/rules/session-handover.mdc"
render "$TEMPLATES_DIR/adr-0001-adopcao-metodologia-handover.md.template"   "$APP_DIR/docs/adr/0001-adopcao-metodologia-handover.md"

echo ""
echo "HANDOVER methodology initialised for '$PROJECT_NAME' (layout=$LAYOUT, lang=$LANGUAGE)."
echo ""
echo "Next step:"
echo "  git add $APP_DIR/HANDOVER.md AGENTS.md .cursor/rules/session-handover.mdc $APP_DIR/docs/adr/0001-adopcao-metodologia-handover.md"
echo "  git commit -m 'DECISION(meta): adoptar metodologia HANDOVER [ADR-0001]'"
```

Tornar executável: `chmod +x init-handover.sh`.

### 10.8 Registar a extensão em `.specify/extensions.yml`

Após criar a pasta da extensão, editar `.specify/extensions.yml` (na raiz) para registar:

```yaml
installed:
  - handover      # adicionar à lista existente
hooks:
  after_constitution:
  - extension: handover
    command: speckit.handover.init
    enabled: true
    optional: true
    prompt: "Inaugurar protocolo HANDOVER neste projecto?"
    description: "Cria HANDOVER.md, AGENTS.md, rule e ADR-0001"
    condition: null
  after_plan:
  - extension: handover
    command: speckit.handover.update
    enabled: true
    optional: true
    prompt: "Registar decisões do planeamento no HANDOVER?"
    condition: null
  # ... (repetir padrão para after_tasks, after_implement, after_analyze)
```

### 10.9 Espelhar comandos em `.cursor/commands/`

O Cursor procura comandos slash em `.cursor/commands/*.md`. Espelhar os 3 comandos da extensão:

```bash
cp .specify/extensions/handover/commands/speckit.handover.*.md .cursor/commands/
```

(Passo idempotente — repetir sempre que actualizar os comandos na extensão.)

### 10.10 Publicação como repo autónomo (opcional)

A extensão é auto-contida. Para reutilizar em vários projectos, extrair para repo público:

```bash
# No computador:
mkdir ~/projetos/handover-speckit-extension
cp -r .specify/extensions/handover/* ~/projetos/handover-speckit-extension/
cd ~/projetos/handover-speckit-extension
git init && git add . && git commit -m "Initial commit"
gh repo create handover-speckit-extension --public --source=. --push
```

Em projectos novos, instalar com:

```bash
cd .specify/extensions
git clone https://github.com/<teu-user>/handover-speckit-extension handover
# e registar em .specify/extensions.yml (secção 10.8)
```

---

## 11. Como construir o agente Handover como Cursor skill user-global

Skill = ficheiro `SKILL.md` em `~/.cursor/skills-cursor/<nome>/`. O Cursor carrega todas as skills **automaticamente em todos os projectos** abertos pelo teu user. Vantagem: zero configuração por-projecto. Desvantagem: é pessoal (não versionável em git com a equipa).

### 11.1 Localização

```text
~/.cursor/skills-cursor/handover/
├── SKILL.md                   # manifesto + instruções para o agente
└── templates/                 # cópia dos 4 templates (para independência)
    ├── HANDOVER.md.template
    ├── AGENTS.md.template
    ├── session-handover.mdc.template
    └── adr-0001-adopcao-metodologia-handover.md.template
```

Os templates embutidos na skill garantem que ela **funciona mesmo sem** `HANDOVER-METHODOLOGY.md` acessível — útil quando estás num computador novo e ainda não restauraste os backups.

### 11.2 Ficheiro completo `SKILL.md`

```markdown
---
name: handover
description: Protocolo de handover entre chats paralelos. Activar quando (1) o user abre um projecto sem HANDOVER.md, (2) pergunta sobre coordenação entre chats, ou (3) pede "instala handover" / "inaugura handover" / "regista decisão no handover". Esta skill inaugura o protocolo num projecto novo ou regista uma decisão material num projecto que já o tem.
---

# Handover — coordenação entre chats paralelos

## Quando activar

1. **Detecção automática no arranque da sessão:** verificar se o projecto tem `HANDOVER.md` na raiz OU em subpasta de app principal (`<app>/HANDOVER.md`). Se não tiver, perguntar ao user:
   > "Este projecto não tem HANDOVER.md. Queres inaugurar o protocolo de memória partilhada entre chats paralelos? (≈ 2 minutos, 4 ficheiros, 1 commit)"

2. **Invocação explícita pelo user:**
   - "inaugura handover" / "instala handover" / "configura handover" → fluxo «Inaugurar».
   - "regista esta decisão" / "faz update do handover" / "adiciona ao handover" → fluxo «Registar decisão».
   - "destila decisões" / "arqueologia" → fluxo «Arqueologia».

## Fluxo A — Inaugurar

1. Confirmar com o user:
   - Nome do projecto
   - Layout A (monorepo com app em subpasta) ou B (app na raiz)
   - Idioma das docs (pt-PT / pt-BR / en-US)
   - Stack tecnológico (1 linha)

2. Obter templates:
   - Preferir `~/.cursor/skills-cursor/handover/templates/` (cópia local da skill).
   - Fallback 1: `HANDOVER-METHODOLOGY.md` na raiz do projecto actual (secções 6-9).
   - Fallback 2: pedir URL público do METHODOLOGY ao user.
   - Último recurso: reconstruir a partir de memória e avisar o user de que os templates podem diferir da versão canónica.

3. Criar os 4 ficheiros substituindo placeholders `{{PROJECT_NAME}}`, `{{PRIMARY_LANGUAGE}}`, `{{TECH_STACK}}`, `{{YYYY-MM-DD}}`, `{{NOME_OU_CHAT_ID}}`:
   - `HANDOVER.md`
   - `.cursor/rules/session-handover.mdc`
   - `AGENTS.md`
   - `docs/adr/0001-adopcao-metodologia-handover.md`

4. Mostrar diff. Pedir confirmação.

5. Commit inaugural:
   `​`​`bash
   git add HANDOVER.md AGENTS.md .cursor/rules/session-handover.mdc docs/adr/0001-adopcao-metodologia-handover.md
   git commit -m "DECISION(meta): adoptar metodologia HANDOVER [ADR-0001]"
   `​`​`

6. Reportar: lista de ficheiros + SHA do commit + instrução para verificar na próxima sessão que a rule carrega sozinha.

## Fluxo B — Registar decisão material

1. Ler `HANDOVER.md` (extrair trilhos, chats registados).

2. Perguntar ao user:
   - O quê (1 frase), porquê (1 frase)
   - Trilho (lista ou novo)
   - Arquitectural? (sim → ADR; não → só HANDOVER)
   - Chat ID
   - Scope do commit

3. Escrever entrada em «Decisões frescas» no topo do bloco do chat correspondente, formato:
   `- YYYY-MM-DD — <decisão> — chat <chat-id>[, ADR-NNNN]`

4. Se arquitectural: criar `docs/adr/NNNN-<slug>.md` seguindo formato dos ADRs existentes no projecto (Contexto → Decisão → Alternativas → Consequências).

5. Propor commit `DECISION(<scope>): ... [ADR-NNNN]` ou `HANDOVER(<scope>): ...` (sem ADR).

6. Mostrar diff antes de aplicar. Aplicar só com confirmação.

## Fluxo C — Arqueologia retrospectiva

1. Perguntar janela (dias, default 7).

2. Recolher sinais:
   - `git log --since="N days ago" --stat`
   - `git diff --stat HEAD~N..HEAD`
   - `git status --short`
   - Transcripts Cursor: `~/.cursor/projects/<slug>/agent-transcripts/*.jsonl` com mtime recente.

3. Destilar decisões materiais (heurística: regex de "decidi(mos)", "pivot", "escolh(i|emos)", "abandon(ámos|ar)", "vamos trocar", "schema muda", "contrato de API", "feature flag").

4. Propor diff para HANDOVER com entradas marcadas `*(inferido — validar)*` quando a confiança for média.

5. Aplicar com commit `HANDOVER(docs): destilar decisões dos últimos N dias` + entrada em «Histórico da arqueologia».

## Princípios obrigatórios

- **Nunca escrever sem mostrar diff e pedir confirmação.**
- **Linguagem = linguagem do projecto.** Se os ADRs existentes estão em pt-PT, nova entrada em pt-PT.
- **Templates canónicos são fonte de verdade.** Não improvisar estrutura de secções.
- **Se os ficheiros já existem, não sobrescrever** — usar Fluxo B (update) em vez de Fluxo A (init).

## Fonte de verdade

`HANDOVER-METHODOLOGY.md` (este documento-semente). Versão canónica: na raiz do repo actual; fallback em `~/Documentos/metodologias/HANDOVER-METHODOLOGY.md` ou em repo público do user.
```

### 11.3 Instalar a skill

```bash
mkdir -p ~/.cursor/skills-cursor/handover/templates
# Colar o SKILL.md (11.2) em ~/.cursor/skills-cursor/handover/SKILL.md
# Copiar os 4 templates para ~/.cursor/skills-cursor/handover/templates/
```

Reabrir o Cursor. A skill fica disponível em todos os projectos automaticamente.

### 11.4 Verificação

Abrir um projecto **novo** (sem `HANDOVER.md`). Iniciar sessão com o agente. Esperado: o agente menciona a skill logo no início e oferece inaugurar o protocolo.

---

## 12. Como construir agentes Handover fora do Cursor

Se usas **ChatGPT Custom GPT, Claude Projects, Gemini Gems, aider, Continue.dev** ou outros, a rule Cursor `.mdc` não carrega automaticamente. Há três estratégias:

### 12.1 System prompt para ChatGPT Custom GPT / Claude Project

Cria um Custom GPT ou Claude Project e cola o bloco abaixo como **instruções personalizadas**:

```text
Este projecto segue a metodologia HANDOVER de coordenação entre chats paralelos.

NO INÍCIO DE CADA SESSÃO, antes de qualquer outra acção, tu DEVES:
1. Ler HANDOVER.md (ou <app>/HANDOVER.md em monorepos) para conhecer trilhos em curso, decisões frescas, e ficheiros em voo activo por outros chats.
2. Ler o ADR numericamente mais alto em docs/adr/.
3. Correr `git status --short` e `git log --since='3 days ago' --oneline`.

Se o pedido do user tocar num ficheiro listado em «Ficheiros em voo activo» do HANDOVER com owner diferente do chat actual, PARA e pergunta confirmação antes de editar.

NO FIM DE CADA SESSÃO que tomou decisão material (escolha técnica com trade-off, pivot, mudança de contrato de API ou schema, feature flag, convenção global, mudança de fluxo UX partilhado), tu DEVES:
1. Adicionar entrada em «Decisões frescas» do HANDOVER.md no formato:
   "- YYYY-MM-DD — <decisão> — chat <chat-id ou nome> [, ADR-NNNN]"
2. Se for decisão arquitectural, criar docs/adr/NNNN-<slug>.md com 4 secções: Contexto, Decisão, Alternativas consideradas, Consequências.
3. Propor commit com tag DECISION(scope): se houver ADR, ou HANDOVER(scope): se for só actualização do HANDOVER. Exemplo:
   git commit -m "DECISION(auth): adoptar Auth.js v5 [ADR-0002]"

Arranjos triviais (typo, lint, formatação) NÃO exigem entrada no HANDOVER — só commit normal.

PRINCÍPIO-RAIZ: se uma decisão foi tomada em conversa e NÃO foi materializada num ficheiro do repositório, ela não existe para os próximos agentes. Escreve sempre.

Todos os templates canónicos (HANDOVER.md, session-handover.mdc, AGENTS.md, ADR inicial) estão no documento-semente HANDOVER-METHODOLOGY.md. Se o projecto ainda não tem os ficheiros, oferece inaugurar o protocolo a partir dos templates dessse documento.
```

Em Claude Projects, ainda podes anexar o `HANDOVER-METHODOLOGY.md` como **Knowledge** para o agente ter acesso aos templates completos.

### 12.2 Custom commands em aider

Adicionar ao `~/.aider.conf.yml`:

```yaml
read:
  - HANDOVER.md
  - AGENTS.md

# Em cada projecto, adicionar também (por path):
# project-files:
#   - docs/adr/
```

Ou usar `/read HANDOVER.md AGENTS.md docs/adr/` no início de cada sessão aider.

### 12.3 Continue.dev

Em `~/.continue/config.json`, adicionar como context provider:

```json
{
  "contextProviders": [
    {
      "name": "file",
      "params": {
        "paths": ["HANDOVER.md", "AGENTS.md", "docs/adr/"]
      }
    }
  ],
  "customCommands": [
    {
      "name": "handover-read",
      "prompt": "Lê HANDOVER.md e AGENTS.md e AF o ADR mais recente. Resume em 5 bullets o que cada chat activo está a fazer.",
      "description": "Leitura inicial de handover"
    },
    {
      "name": "handover-update",
      "prompt": "Pergunta-me sobre a decisão material tomada hoje (o quê, porquê, trilho, arquitectural, chat-id). Propõe entrada para «Decisões frescas» do HANDOVER.md + eventual ADR + commit DECISION(scope): ...",
      "description": "Assistente de registo de decisão"
    }
  ]
}
```

### 12.4 CLI genérico (zsh / bash wrapper)

Criar um pequeno wrapper que prepende o HANDOVER ao prompt de qualquer agente CLI:

```bash
#!/usr/bin/env bash
# ~/bin/askh — "ask with handover context"
# Usage: askh "minha pergunta ao agente"

PROMPT="$*"
CONTEXT="$(cat HANDOVER.md 2>/dev/null; echo '---'; cat AGENTS.md 2>/dev/null; echo '---'; git log --since='3 days ago' --oneline 2>/dev/null)"

# Substituir <agent-cli> pelo teu comando favorito (ex.: llm, aichat, etc.)
echo -e "Contexto do projecto:\n$CONTEXT\n\n---\n\nPedido:\n$PROMPT" | <agent-cli>
```

Tornar executável e colocar no `$PATH`. A partir daí, `askh "implementa login OTP"` envia o HANDOVER + AGENTS + git log recente **antes** do teu pedido real.

---

## 13. Cenários de recuperação

### Cenário A — Perdi o computador, tenho este ficheiro

1. Recuperar este ficheiro do backup off-site (email, gist, drive).
2. Abrir o novo projecto num editor qualquer (Cursor, VS Code, vim).
3. Seguir a secção [5 — Instalação manual num projecto novo](#5-instalação-manual-num-projecto-novo) passo-a-passo. Copiar os 3 templates para os caminhos correctos, substituir placeholders, fazer o primeiro commit.
4. **Opcional:** recrear a extensão speckit (secção 10) ou a skill Cursor (secção 11) se usares essas ferramentas.
5. Em 10 minutos o novo projecto tem a metodologia activa.

### Cenário B — Perdi o projecto antigo mas tenho o git remote

1. `git clone <remote>` recupera `HANDOVER.md`, `AGENTS.md`, rule e ADRs — **tudo está versionado**, por design.
2. `git log --grep='DECISION\|HANDOVER' --oneline` dá o sumário condensado do histórico de decisões.
3. Ler o `HANDOVER.md` recuperado + o ADR mais recente dá contexto suficiente para retomar.

### Cenário C — Tenho o ficheiro mas o LLM/Cursor não funciona

1. Os três ficheiros são **markdown puro**. Funcionam em qualquer editor.
2. A regra `session-handover.mdc` pode ser colada manualmente como contexto inicial de qualquer LLM (ChatGPT, Claude web, Gemini) — é só instrução em texto natural.
3. A convenção de commits é independente de ferramenta.

### Cenário D — Novo colaborador humano entra no projecto

1. Envia-lhe `AGENTS.md` (ponto de entrada).
2. Ele lê `HANDOVER.md` para estado actual.
3. `git log --grep='DECISION'` para histórico.
4. Em 15 minutos está orientado.

---

## 14. FAQ

**P: O que faço se não uso Cursor?**
R: O `.cursor/rules/session-handover.mdc` fica sem efeito automático, mas continua a ser **documentação válida em markdown**. Cola o seu conteúdo manualmente como system prompt/custom instruction em qualquer agente que uses (ChatGPT Custom GPT, Claude Projects, Gemini Gems). O `HANDOVER.md` + `AGENTS.md` + ADRs funcionam em qualquer IDE.

**P: Vale a pena para projectos solo (sem chats paralelos)?**
R: Sim — a trilha de decisões materiais via ADRs + `DECISION(...)` commits tem valor enorme para o **teu-eu-futuro** daqui a 6 meses. E se um dia trouxeres colaboradores, está tudo pronto.

**P: O `HANDOVER.md` cresce infinitamente?**
R: Não. A secção «Decisões frescas» tem janela de 7 dias por design. Entradas mais antigas devem ser **arquivadas** em `docs/handover-archive/YYYY-MM.md` (ou simplesmente apagadas se já estão cobertas por um ADR). A metodologia tem que viver — o ficheiro deve caber numa leitura de 2 minutos.

**P: Posso ter múltiplos `HANDOVER.md` num monorepo (um por app)?**
R: Sim, desde que a rule `session-handover.mdc` seja ajustada para listar todos os caminhos. Melhor estratégia em monorepos: **um HANDOVER.md global** na raiz com secções por app, a não ser que as apps sejam completamente independentes.

**P: Como faço *archaeology* de chats anteriores sem este protocolo?**
R: Se o Cursor/Claude guarda transcripts localmente (ex.: `~/.cursor/projects/<projecto>/agent-transcripts/`), podes pedir a um agente LLM "lê os transcripts das últimas 72h, destila decisões materiais e propõe entradas retrospectivas para `HANDOVER.md`". O comando `/speckit.handover.archaeology` (secção 10) automatiza isto em projectos speckit.

**P: Isto substitui RFCs / design docs / specs?**
R: Não. **ADRs** são só para decisões materiais curtas com trade-offs (1-2 páginas). Design docs detalhados continuam a viver em `docs/` ou num sistema externo (Notion, Confluence). O HANDOVER é **fotografia operacional** — não é documentação de produto.

**P: Como lido com decisões que são revertidas?**
R: Cria um ADR novo (não edites o antigo) com status `supersedes ADR-NNNN`. O ADR antigo ganha header `status: superseded by ADR-MMMM`. Entrada correspondente em «Decisões frescas» do HANDOVER explicando porquê.

**P: Posso automatizar a detecção de "decisão material"?**
R: Parcialmente — o `/speckit.handover.update` pode fazer heurísticas (ex.: detectar se o diff toca em `schema.prisma`, `.env.example`, mudanças em endpoints de API). Mas a decisão final é sempre humana. Falsos positivos aborrecem; falsos negativos perdem decisões.

---

## 15. Onde guardar este documento

Este ficheiro é o único que **não pode depender** do próprio projecto para sobreviver. Guarda-o em **pelo menos dois** destes sítios:

- ✉️ **Email a ti próprio** com assunto fácil de pesquisar (ex.: `[HANDOVER-METHODOLOGY] semente reutilizável — vN.M`).
- 🗂️ **Gist público ou privado no GitHub** (preferível privado se tiveres nomes de projecto mencionados) — `gist.github.com/<user>/<id>`.
- ☁️ **Google Drive / iCloud / Dropbox / OneDrive** numa pasta chamada `~Documentos/metodologias/`.
- 🏷️ **Repo GitHub autónomo** — sugestão: `github.com/<user>/handover-methodology` com este ficheiro como `README.md`. Podes também empurrar depois a extensão speckit para o mesmo repo.
- 📱 **Obsidian / Notion / Bear** — qualquer sistema de notas pessoal com sync.

**Versionamento:** quando fizeres alterações substanciais a este documento, incrementa o número de versão e a data no topo. Sugestão de semver simples: `v1.0` (adopção inicial) → `v1.1` (templates ajustados) → `v2.0` (estrutura mudou).

---

## 16. Licença e uso

Este documento e os templates que contém são livres para uso, modificação e distribuição, pessoal ou comercial, sem atribuição necessária (CC0 / Public Domain). Se achares útil, um ⭐ num eventual repo público seria apreciado mas não obrigatório.

Contribuições e melhorias são bem-vindas via PR se publicares o documento em repo autónomo.

---

## Meta

| Campo | Valor |
|---|---|
| **Versão** | v1.0 |
| **Data** | 2026-04-17 |
| **Autor original** | rkokubo + assistente Cursor |
| **Origem** | destilado da prática real no projecto ClinicaGestor (ver `clinicagestor/HANDOVER.md` para exemplo em produção) |

---

> **Última palavra:** esta metodologia é **barata** (3 ficheiros, uma regra, dois tags de commit) e **resiliente** (tudo em markdown versionado, zero infra). Se a disciplina cair, ela degrada graciosamente — os ficheiros ficam-se-lhes, esperando por retoma. Não tentes perfeição; tenta consistência.
