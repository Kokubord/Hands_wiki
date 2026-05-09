<!-- wiki-ingest: synced from specs/001-clinicagestor-platform/checklists/requirements.md at 2026-05-09T17:55:22.746Z — edit upstream in monorepo, not here -->

# Specification Quality Checklist: ClinicaGestor – Plataforma integrada para clínicas

**Purpose**: Validate specification completeness and quality before proceeding to planning  
**Created**: 2026-04-11  
**Feature**: [spec.md](../spec.md)

## Content Quality

- [x] Requisitos e **SC** sem vinculação obrigatória a fornecedor; **§11** da spec documenta **stack preferida** como referência explícita (exceção acordada para alinhar plano e Constituição)
- [x] Focused on user value and business needs
- [x] Written for non-technical stakeholders
- [x] All mandatory sections completed

## Requirement Completeness

- [x] No [NEEDS CLARIFICATION] markers remain
- [x] Requirements are testable and unambiguous
- [x] Success criteria are measurable
- [x] Success criteria are technology-agnostic (no implementation details)
- [x] All acceptance scenarios are defined
- [x] Edge cases are identified
- [x] Scope is clearly bounded
- [x] Dependencies and assumptions identified

## Feature Readiness

- [x] All functional requirements have clear acceptance criteria
- [x] User scenarios cover primary flows
- [x] Feature meets measurable outcomes defined in Success Criteria
- [x] Detalhe de implementação confinado a **§11** e notas de plano; restantes secções de produto sem stack imposta

## Checklist SC × fase (rastreio com plano técnico)

**Fonte normativa dos SC**: definições e métricas em [spec.md § critérios de sucesso](../spec.md) (lista **SC-001**–**SC-020**). **Gates (PR / release / noturno), pirâmide de testes e regra P1**: [plan.md — Estratégia de cobertura por fase](../plan.md#coverage-by-phase).

### Âmbito mínimo por fase (espelho do plano)

Manter esta tabela alinhada a [plan.md#coverage-by-phase](../plan.md#coverage-by-phase) quando a estratégia mudar.

| SC | Fase 1 (PR / release) | Fase 2 | Fase 3 |
|----|------------------------|--------|--------|
| **SC-001** | **MAY**: baseline manual ou ensaio cronometrado documentado | Refinar com UX se necessário | Manter na matriz |
| **SC-002** | **P1** (AuditLog em mutações cobertas) | Estender a financeiro + filas | Completo |
| **SC-003** | **P1** (cross-Empresa + cross-clínica negativos) | Incluir APIs/financeiro/agentes em âmbito F2 | Reforço migração |
| **SC-004** | **P1** (muro) | Manter regressão | Manter |
| **SC-005** | **MAY** (lista de espera) | Idem | Idem |
| **SC-006** | **P1** (imutabilidade pós-assinatura) | Regressão | Manter |
| **SC-007** | Fora do gate F1 | **P1** (relatórios + export conforme entrega) | **P1** + volume (**F3-08**) |
| **SC-008** | **Parcial F1**: UI painel + **bloqueio** mutações livro (**ADR-0001**); **pleno** após **F2-01** | **P1** pós-**F2-01** | Manter |
| **SC-009** | — | — | **P1** (**F3-03**) |
| **SC-010** | — | **P1** pós-**F2-07** | Manter |
| **SC-011** | **P1** (prontuário cross-clínica) | Regressão | Manter |
| **SC-012** | — | **P1** pós-**F2-07** | Manter |
| **SC-013** | — | **P1** smoke pós-**F2-08** (série temporal **FR-038**) | Manter |
| **SC-014** / **SC-015** | Fora do gate F1 | Smoke **MAY**; gate completo **preferencialmente F3** | **P1** (**F3-07** / **F3-08**) |
| **SC-016**–**SC-018** | — | — | **P1** migração (**F3-01**) |
| **SC-019** / **SC-020** | — | — | **P1** evidência operacional (**F3-09**) |

**Regra**: SC marcados **P1** sem automação MUST ter **procedimento de ensaio** documentado até existir teste (ver plano).

### Confirmação ao fechar fase

- [ ] **Fase 1**: todos os **P1** da coluna Fase 1 satisfeitos **ou** com dispensa registada; matriz `SC × suite` atualizada
- [ ] **Fase 2**: idem para coluna Fase 2 + regressão dos P1 herdados de F1
- [ ] **Fase 3**: idem para coluna Fase 3 + **SC-014**/**SC-015**/**SC-007** em volume conforme plano
- [ ] **Rastreabilidade**: artefacto único (**`tasks.md`** ou checklist QA) com `SC × (unit | int | e2e | manual) × fase` referenciado no [plano](../plan.md#coverage-by-phase)

### Estado factual já documentado (2026-04)

- [x] **SC-001** com evidência de integração + E2E temporal base/multi-dispositivo (`evidence/sc-001.md`)
- [x] **SC-005** funcional em integração (lista de espera por profissional/status) (`evidence/sc-005.md`)
- [x] **SC-008 parcial F1**: painel FR-022 com itens FR-023 persistidos + bloqueio de ledger (ADR-0001) (`evidence/sc-008.md`)

---

## Notes

- Validação inicial na criação da spec; após refinamentos em clarificação (multi-clínica, isolamento, UX responsiva, estrutura, **cenários A/B**, **auth/RBAC/AuditLog/LGPD** em US1, §6.1 e §6.2), `spec.md` segue alinhado a este checklist.
- **§11** consolida stack preferida e integrações; **§8**/**§9** ligam capacidades de produto a essa referência; **FR**/**SC** permanecem agnósticos de fornecedor salvo capacidades explícitas (2FA, não repúdio, etc.).
- **Performance**: **FR-057**–**FR-061**, **SC-014**–**SC-015** e **§11.9**; **SC-007** alinhado a vista interativa vs exportação assíncrona (**FR-061**).
- **Migração**: **User Story 9**, **§6.9** (**FR-062**–**FR-066**), **SC-016**–**SC-018**; MVP em ficheiros; conectores API fora de **§10**.
- **Backup / DR / retenção**: **§6.10** (**FR-067**–**FR-071**), **SC-019**–**SC-020**; valores **RPO**/**RTO** e detalhe de infra no **`/speckit.plan`**.
- **Testes e qualidade**: **§6.11** (tipos de teste, prioridades, **CI** gates, mapeamento **SC**); estratégia por fase e tabela **SC** em [plan.md#coverage-by-phase](../plan.md#coverage-by-phase) e secção **Checklist SC × fase** acima.
- **2026-04-26 — Prontuário / ADR-0006:** **SC-006** (imutabilidade) aplica-se por **objecto** (roteiro assinado, anexo PAdES, etc.). A **biblioteca agregada** não confunde bloqueio do **roteiro** com a **impossibilidade** de assinar outros documentos noutro momento; ensaios manuais ou E2E futuros devem cobrir o diálogo «Abrir documento» (pedido externo / anexo) e a mensagem «refere-se ao roteiro clínico».
- **Plano / painel**: **ADR-0001** (`specs/001-clinicagestor-platform/adr/0001-fr022-billing-shell-phase1.md`) — **FR-022** secção cobrança em F1 sem escritas no livro; flag `billingLedgerWritesEnabled`. **plan.md**: Princípios anti-refatoração; **FR-035** em **F1-07**; **F2-07** antes de **F2-03**/**F2-04**.
- **Evidências correntes F1**: `evidence/sc-001.md` (agenda temporal + pré-requisitos), `evidence/sc-005.md` (lista de espera funcional), `evidence/sc-008.md` (painel FR-022/FR-023 + bloqueio ledger ADR-0001).
- Próximo passo sugerido: `/speckit.plan` para arquitetura e plano técnico.
