# GROK.md — Regras do Vault ClinicaGestor

**Este arquivo é o System Prompt permanente deste vault.**

## Contexto do Projeto

- Nome: **ClinicaGestor (Hands)**
- Propósito: Sistema completo para clínicas de saúde (agenda, prontuário, financeiro, assinatura digital, etc.)
- Foco actual: Fase 1 avançada + Fase 2 (Financeiro — CRL vs Livro, recebíveis, pivot)

## Localização no disco (referência)

- **Ubuntu / WSL:** o vault é a pasta **`wiki_hands/`** dentro do clone do monorepo Hands:
  - **`/home/rkokubo/projetos/Hands/wiki_hands`** (ajusta `rkokubo` / `projetos` se o teu caminho for diferente).
- **Windows (Obsidian):** usar um **clone separado** deste mesmo repositório Git num caminho **NTFS local** (ex. `C:\Users\…\wiki_hands`), **não** abrir via `\\wsl$\…` — evita erros de file watcher.

## Relação com o monorepo Hands

- Este vault é (ou será) um **repositório Git próprio**, ligado ao Hands como **git submodule** em `wiki_hands/`.
- Cópias literais dos docs canónicos e derivados programáveis vêm de **`scripts/wiki-ingest/`** no repo Hands (`wiki-mirror.manifest`, `wiki-derived`, templates) — não reinventar caminhos aqui; seguir o que o submodule trouxe após `git pull`.

## Estrutura do Vault (modelo adaptado — não é o minimalismo Karpathy)

Inspirado na ideia *raw / wiki* (fontes → síntese), mas **mais granular** para o ClinicaGestor:

| Área | Função |
|------|--------|
| **`raw/`** | Fontes imutáveis **neste fluxo de edição**: espelhos do repo (`repos`, `specs`, `papers`, …), clips, papers, datasets, transcripts. Subpastas típicas: `articles/`, `assets/`, `datasets/`, `notes/`, `papers/`, `repos/`, `requirements/`, `scripts/`, `specs/`, `transcripts/`. |
| **`wiki/`** | Wiki compilada: `concepts/`, `entities/`, `queries/`, `sources/`, `topics/` (Agenda, Assinatura, Financeiro, Prontuário, …). |
| **`assets/`** (raiz) | Media e diagramas para leitura na wiki. |
| **`raw/assets/`** | Brutos / anexos ligados a fontes em `raw/`. |
| Raiz | `index.md`, `log.md`, `README`, **`GROK.md`**, **`claude.md`** (schema para Claude Code, inglês). |

### Duas faixas de ingestão (compatível com estratégia Karpathy + pipeline Hands)

1. **Ingest repo-sync (determinístico)** — actualiza `raw/` e blocos/páginas marcados gerados em `wiki/` conforme manifests no repo Hands. **Não editar à mão** o que estiver só dentro de blocos `<!-- BEGIN GENERATED -->` … `<!-- END GENERATED -->` ou pastas convencionadas como só-saída-do-script (documentadas no Hands).
2. **Ingest exploratório (agente)** — novo ficheiro em `raw/` (clip, nota, transcript) + conversa: o agente actualiza `wiki/` (concepts, entities, backlinks), `index.md`, `log.md`, como no padrão “LLM Wiki”.

**Conflito:** se uma página mistura as duas faixas, mantém-se texto livre **fora** dos marcadores GENERATED; o pipeline só substitui o interior dos marcadores.

## Regras de Comportamento

1. **Ingestão**
   - Processar novos documentos a partir de **`raw/`** e reflectir na **`wiki/`** quando for ingest exploratório.
   - Após sync do submodule Hands, **assumir** que `raw/repos`, `raw/specs`, etc., já podem ter sido actualizados pelo gerador — priorizar reconciliação na wiki e no `index.md`, não duplicar edição manual em `raw/` para corrigir verdade canónica (isso corrige-se no **repo Hands**).

2. **Manutenção**
   - Manter **`wiki/index.md`** (catálogo por categorias, links, linha de resumo) e **`log.md`** (append-only; prefixos tipo `## [YYYY-MM-DD] ingest | …` para grep/`tail`).
   - Backlinks e MOCs onde fizer sentido.
   - **Lint** quando pedido: contradições, páginas órfãs, conceitos mencionados sem página, links quebrados.

3. **Idioma e Estilo**
   - Páginas da wiki em **português**; tom técnico, claro e profissional (alinhar com documentação do produto em PT).

4. **Temas Prioritários**
   - Financeiro (CRL vs Livro, adiantamentos, recebíveis)
   - Prontuário e episódio clínico
   - Assinatura digital
   - Agenda e estado das consultas
   - RBAC e multi-clínica

## Comandos Úteis

- «Ingest novo plano»
- «Actualiza o index da wiki»
- «Lint da wiki»
- «Cria página sobre CRL vs Livro»

---

**Última actualização:** 2026-05-08
