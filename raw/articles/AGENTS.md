<!-- wiki-ingest: synced from AGENTS.md at 2026-05-09T18:04:46.899Z — edit upstream in monorepo, not here -->

# AGENTS.md — guia para agentes LLM e contribuidores deste repositório

Este ficheiro é o **primeiro ponto de contacto** para qualquer contribuidor — humano ou LLM — que abre este monorepo. Mantenha-o curto e prático.

## Estrutura do repositório

```text
/
├── clinicagestor/         # Aplicação principal (Next.js 15 + Prisma + Postgres)
│   ├── HANDOVER.md        # ⭐ fotografia operacional viva — LEIA PRIMEIRO
│   ├── docs/
│   │   ├── adr/           # Architectural Decision Records (fonte de verdade das decisões)
│   │   └── manuais/       # (proposto) manuais de utilizador final
│   ├── app/, components/, lib/, prisma/, tests/, e2e/, scripts/
│   └── README.md
├── specs/001-clinicagestor-platform/   # Especificação formal (spec.md, plan.md, tasks.md)
├── wiki_hands/            # Submodule Git → Kokubord/Hands_wiki (vault Obsidian Markdown)
├── scripts/wiki-ingest/   # `npm run wiki:ingest` — espelho determinístico para o vault
├── .cursor/
│   ├── rules/             # Regras carregadas automaticamente em cada sessão de agente
│   └── commands/
└── AGENTS.md              # (este ficheiro)
```

## Leitura mínima antes de agir

Para qualquer agente que inicie uma sessão neste repositório, por esta ordem:

1. `clinicagestor/HANDOVER.md` — estado dos trilhos em curso e decisões frescas.
2. ADR numericamente mais alto em `clinicagestor/docs/adr/` — último contexto arquitectural.
3. `git status --short` + `git log --since="3 days ago" --oneline` — trabalho recente em voo.
4. Regras em `.cursor/rules/*.mdc` — são carregadas automaticamente, mas vale a pena lê-las uma vez para saber o que o agente já «sabe». Em especial: **`dev-database-no-production-route.mdc`** — dev local **não** aponta para produção/Neon como atalho; produção só via **GitHub + deploy**.

## Convenção de commit-messages

Baseada em **Conventional Commits**, com dois tags adicionais próprios deste projecto:

| Prefixo | Quando usar | Exemplo |
|---|---|---|
| `feat(scope):` | Nova funcionalidade utilizador | `feat(agenda): adicionar filtro por convênio` |
| `fix(scope):` | Correcção de bug | `fix(login): preservar callbackUrl após OTP` |
| `docs(scope):` | Só documentação | `docs(adr): adicionar ADR-0004 sobre PAdES local` |
| `test(scope):` | Testes novos ou refactor de testes | `test(pades): smoke end-to-end com chave RSA` |
| `refactor(scope):` | Refactor sem mudar comportamento | `refactor(agenda): extrair day-bounds` |
| `chore(scope):` | Infra, deps, build | `chore(deps): bumpar node-forge para 1.3.x` |
| **`DECISION(scope):`** | Decisão material (tecnologia, pivot, contrato de API, RBAC) — **obrigatório referenciar ADR** | `DECISION(signature): pivot Rest PKI Core → Web PKI local [ADR-0004]` |
| **`HANDOVER(scope):`** | Actualização exclusivamente do `HANDOVER.md` e afins (regras, este ficheiro) | `HANDOVER(docs): destilar decisões dos transcripts 14-17 Abr` |

Um `git log --grep='DECISION\|HANDOVER'` deve dar, em qualquer momento, o histórico condensado de decisões materiais do projecto — este é o motivo dos dois tags próprios.

**Regra de ouro:** commits `DECISION(...)` **obrigam** a um ADR (novo ou aditamento ao existente). Commits sem ADR que tomem decisão material devem ser evitados.

## Para agentes LLM em chats paralelos

Este projecto tem múltiplas sessões de agente a correr em paralelo, frequentemente ao longo de vários dias. Para não pisar o trabalho dos outros:

1. Respeite o protocolo em `.cursor/rules/session-handover.mdc` (é `alwaysApply: true`, portanto já está no seu system prompt).
2. Antes de editar qualquer ficheiro listado na tabela «Ficheiros em voo activo» do HANDOVER, **confirme** com o utilizador.
3. No fim da sua sessão, escreva a entrada de «Decisões frescas» no HANDOVER **antes** de encerrar. Se não escrever, a decisão perde-se.
4. Se encontrar **duas decisões antigas sobre o mesmo tema** que se contradizem, **prevalece a mais recente**; actualize o texto antigo com *superseded* ou *Antes/Depois* (ver `HANDOVER-METHODOLOGY.md`).

## Para contribuidores humanos

- Português europeu nas docs (alinhado com o estilo existente nos ADRs e ficheiros em `docs/`).
- Nada de segredos em commits (`.env` deve permanecer git-ignored; só `.env.example` é versionado).
- Antes de abrir PR, correr `npm run test` e `npx tsx scripts/smoke-pades-local.ts` (quando aplicável).
