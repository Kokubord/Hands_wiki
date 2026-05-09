# Claude Code — vault Hands_wiki

This file is the **schema for coding agents** (Claude Code, Codex, etc.) working in this Obsidian vault alongside **`GROK.md`** (Portuguese product rules for Grok/Obsidian).

## Layers

1. **`raw/`** — Immutable sources in this workflow: mirrors from the ClinicaGestor monorepo (`repos`, `specs`, `articles`, …), clips, papers. Do not rewrite canonical mirrors here; fix upstream in Hands and re-sync via `wiki-ingest`.
2. **`wiki/`** — Compiled knowledge: `concepts`, `entities`, `queries`, `sources`, `topics`. You maintain summaries, links, and synthesis here per **`GROK.md`**.
3. **Root** — `index.md` (catalog), `log.md` (append-only timeline), `GROK.md` / this file (rules).

## Operations

- **Ingest:** New material lands under `raw/` → update relevant `wiki/` pages, backlinks, `index.md`, append `log.md`.
- **Query:** Answer from wiki first; cite paths under `raw/` or `wiki/`.
- **Lint:** Orphans, contradictions, stale claims — when asked.

## Language

- Wiki pages: Portuguese (aligned with `GROK.md`).
- This file: English (tooling convention).
