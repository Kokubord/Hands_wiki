# wiki_hands - Knowledge Base do ClinicaGestor

Este vault Obsidian é mantido principalmente pelo Grok.

## Estrutura (adaptada ao ClinicaGestor)

- **`raw/`** — Fontes por tipo: `articles/`, `papers/`, `repos/`, `requirements/`, `scripts/`, `specs/`, mais `assets/`, `datasets/`, `notes/`, `transcripts/` (ver `raw/README.md`).
- **`wiki/`** — Síntese: `concepts/`, `entities/`, `queries/`, `sources/`, `topics/`.
- **`assets/`** — Media na raiz do vault (diagramas, imagens).
- **`GROK.md`** · **`claude.md`** — Regras (PT / EN).

## Sincronização determinística (monorepo Hands)

Na raiz do repositório **Hands** (não dentro deste cofre isolado):

```bash
npm run wiki:ingest
```

Depois faça commit e push neste repo (`wiki_hands`). Documentação do comando: ficheiro `scripts/wiki-ingest/README.md` na raiz do monorepo **Hands**.

## Como contribuir

Peça ao Grok:
- "Ingest novo plano"
- "Atualize o MOC Financeiro"
- "Faça linting da wiki"
