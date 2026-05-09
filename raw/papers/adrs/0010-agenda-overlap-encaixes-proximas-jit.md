<!-- wiki-ingest: synced from clinicagestor/docs/adr/0010-agenda-overlap-encaixes-proximas-jit.md at 2026-05-09T17:47:19.000Z — edit upstream in monorepo, not here -->

# ADR-0010 — Agenda: encaixes (double-booking), slots livres, «Próximas consultas» e just-in-time

## Estado

Aceite (2026-04-29).

## Contexto

A agenda bloqueava sobreposição entre marcações `AGENDA` do mesmo profissional (excepto `CANCELLED`), impedindo encaixes e reutilização do tempo após falta (`NO_SHOW`). A lista «Próximas consultas» cortava consultas com `startsAt` antes do instante actual, ocultando atrasos ainda não iniciados. A política temporal rejeitava qualquer início no passado sem `agendaAllowPastSlots`, mesmo dentro da **mesma célula** da grelha horária que o «agora» (encaixe just-in-time).

## Decisão

1. **Sobreposição:** deixa de haver validação de conflito temporal entre marcações na criação, reagendamento por arrastar e `PATCH` quick-edit. `CANCELLED` e `NO_SHOW` continuam documentados como estados que **não** ocupam slot para **busy hints** (`findNextFreeAgendaSlotAction`).
2. **UI:** FullCalendar com `slotEventOverlap` activo para visualizar várias marcações no mesmo intervalo.
3. **«Próximas consultas»:** o conjunto inclui (no período da vista) marcações futuras **e** consultas em atraso (`startsAt` &lt; agora) que ainda passam no filtro de estado/episódio; ordenação `startsAt`, desempate `consultationNumber`.
4. **Just-in-time:** com `agendaAllowPastSlots` falso, permite-se `startsAt` ligeiramente no passado se `startsAt` e «agora» caem na mesma célula da grelha (`slotStepMinutes`, fuso do contratante), alinhado ao modal de criação (`slotStepMinutes` no form).

## Consequências

- Dois doentes podem estar agendados no mesmo intervalo; o fluxo clínico de episódio **não** foi alterado por este ADR.
- Relatórios que assumem exclusividade temporal por profissional devem ser revistos produto a produto.
