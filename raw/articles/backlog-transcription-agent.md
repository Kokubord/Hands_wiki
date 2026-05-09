<!-- wiki-ingest: synced from specs/001-clinicagestor-platform/backlog-transcription-agent.md at 2026-05-09T17:47:18.988Z — edit upstream in monorepo, not here -->

# Backlog — agente de transcrição / NLP (ClinicaGestor)

Itens **fora do escopo** das entregas atuais do painel/prontuário, a tratar quando existir pipeline de transcrição ou decisão clínica explícita.

**Nota de rastreio**: **ANT-EXTRACT-01** não é a tarefa **F1-13** (agente WhatsApp / agendamento). A extração de antecedentes fica neste backlog até haver agente de transcrição ou fluxo dedicado; **F1-13** permanece no plano de produto como integração Twilio / `scheduling`.

---

## ANT-EXTRACT-01 — Extração automática de antecedentes a partir de texto corrido

**Status**: To Do  
**Contexto**: Hoje o resumo «Antecedentes Pessoais» espelha **entrada manual por categoria** (`historyAndAntecedents` para clínicos; `antecedentsSurgical`, `antecedentsFamily`, `antecedentsHabits`, `antecedentsAllergies`, `antecedentsMedications` para o resto). Ver `lib/clinical/personal-antecedents-summary.ts`.

**Pedido pendente**: «Extração» automática a partir de um **único** texto corrido (ex.: repartir só `historyAndAntecedents` ou texto transcrito em várias categorias).

**Porque não foi feito**: Exige heurísticas, eventual **NLP** e **decisões clínicas** (o que conta como «familiar» vs «clínico», normalização de medicamentos, etc.). O desenho actual é entrada manual por categoria + resumo espelhado no painel.

**Próximos passos sugeridos** (quando houver dono de produto/clínico):

1. Definir se a fonte é só ditado/transcrição ou também import de notas livres.
2. Esquema de saída validável (obrigatório revisão humana antes de persistir?).
3. Prototipo: classificador por secções ou LLM com schema fixo + testes com casos reais anonimizados.

**Referências de código**: `ConsultationEpisode` / `ConsultationEpisodeRevision` (campos de antecedentes); `components/consultation-patient-antecedents-panel.tsx` (entrada por categoria no painel); `components/consultation-episode-roteiro-form.tsx` (roteiro estruturado — não substitui os cinco campos de antecedentes pessoais).
