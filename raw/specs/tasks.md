# Tasks — ClinicaGestor (001-clinicagestor-platform)

**Branch**: `001-clinicagestor-platform`  
**Gerado a partir de**: [plan.md](./plan.md), [spec.md](./spec.md), [ADR-0001](./adr/0001-fr022-billing-shell-phase1.md), [Constituição](../../.cursor/commands/speckit.constitution.md)  
**Estratégia de testes / SC**: [plan.md#coverage-by-phase](./plan.md#coverage-by-phase) · checklist espelho em [checklists/requirements.md](./checklists/requirements.md)

**Legenda de status**: `To Do` (inicial) → `In Progress` → `Blocked` → `Done`

**Story points (SP)**: espelham o [plan.md](./plan.md); servem a priorização, não substituem estimativa em horas.

**Registo operacional (2026-04-29):** quatro ficheiros `*.plan` do Cursor (jornada clínica CTA, finalizar episódio atrasado, `signedAt` + episódio avulso UI, assinatura ICP + cabeçalho) foram **cancelados** após reversão do código — ver tabela em [plan.md#cursor-plans-cancelled](./plan.md#cursor-plans-cancelled); **não** abrir como tarefas Speckit.

---

## Convenção de cada tarefa

Cada entrada inclui: **ID**, **título**, **descrição**, **dependências**, **SP**, **critérios de aceite** (ligados à spec / ADR / SC), **status** (`To Do`). Onde útil, **decomposição sugerida** (Fase 1) quebra o trabalho em passos acionáveis sem alterar o ID canónico do plano.

---

# Fase 1 — MVP núcleo (operacional: “vida ao modelo”)

**Objetivo resumido**: agenda → painel (**FR-022** / **ADR-0001**) → episódio → prescrição → assinatura → imutabilidade; paciente Empresa (**FR-046**); **CL + convênio + preços** (**FR-035**); WhatsApp (**FR-018**); **sem** escritas no livro até F2.

**Ordem lógica sugerida** (IDs do plano): F1-01 → F1-02, F1-03 (paralelo após F1-01) → F1-04 → F1-05, F1-06 (paralelo após F1-04) → F1-07 → F1-08 → F1-09 → F1-10 (paralelo a F1-08 após F1-06) → F1-11 → F1-12 → F1-13 (paralelo possível após F1-08 + F1-02) → F1-14, F1-16 (cedo, baixo SP) → F1-15 → F1-17 → F1-18.

---

### F1-01 — Monorepo, Prisma, schema tenant, livro (só schema), object storage

- **Status**: Done
- **SP**: 10  
- **Dependências**: —  
- **Descrição**: Arranque do repositório alinhado a **§11**; PostgreSQL + Prisma; modelos **Empresa**, **Clínica**, utilizadores e contexto multi-tenant (`companyId` / `clinicId`). Modelar no schema entidades necessárias a **FR-036** / **FR-037** (livro, recebimento, parcelas, vínculo consulta/CL) com migrações **estáveis**; **sem** `POST`/mutations que persistam no livro até **F2-01** (**Princípio 1**, **ADR-0001**). Object storage: env + cliente ou **stub** para anexos futuros e **F2-09** (**Princípio 5**). Entregar **quickstart** reprodutível.  
- **Critérios de aceite**: Migrações aplicáveis de zero; documentação quickstart; código ou middleware impede escritas livro com flag `billingLedgerWritesEnabled === false`; nenhuma rota pública persiste recebimento em F1.  
- **Referências**: plan **Princípios 1, 5**; spec **FR-036**, **FR-037** (preparação); **§11**.

**Decomposição sugerida**

1. Tooling: Next.js 15, TS strict, ESLint/Prettier, estrutura `app/`, `lib/`, `e2e/` (vazios ou smoke).  
2. Prisma + Postgres: schema tenant base, seeds mínimos, política de migrações.  
3. Schema financeiro **read-only**: tabelas/FKs alinhadas a livro futuro; **zero** handlers de escrita.  
4. `billingLedgerWritesEnabled` global/tenant + util comum para gates em F1-09/F2-01.  
5. Object storage: variáveis `.env.example`, interface + implementação stub ou SDK cifrado.  
6. `README` / quickstart: subir DB, migrar, correr app.

---

### F1-02 — NextAuth v5, 2FA, clínica ativa (`activeClinicId`)

- **Status**: Done
- **SP**: 13  
- **Dependências**: F1-01  
- **Descrição**: Autenticação (**FR-003**); **2FA** (**FR-007**) com OTP (ex. WhatsApp conforme spec); sessão com **clínica ativa**; troca de clínica com limpeza de estado cliente (**FR-003**); UI indica Empresa + clínica ativa.  
- **Critérios de aceite**: Fluxo login + 2FA; troca `activeClinicId` sem vazar dados da clínica anterior; preparação para **SC-003** (testes negativos em F1-17).  
- **Referências**: **FR-003**, **FR-007**; US1.

**Decomposição sugerida**

1. NextAuth configurado (providers, callbacks tenant).  
2. Fluxo 2FA + recuperação conforme política mínima da spec.  
3. `activeClinicId` em sessão + UI seletor; invalidação de cache cliente ao trocar.  
4. Testes integração: dois utilizadores / duas clínicas (fixture).

---

### F1-03 — AuditLog

- **Status**: Done  
- **SP**: 8  
- **Dependências**: F1-01  
- **Descrição**: Registo consultável de operações sensíveis (**FR-005**, **FR-011**) com Empresa, clínica, ator; extensível a financeiro em F2.  
- **Critérios de aceite**: Mutações cobertas na F1 geram evento consultável em tempo compatível com **SC-002** (preparação; extensão F2 documentada).  
- **Referências**: **FR-005**, **FR-011**; **SC-002**.

**Decomposição sugerida**

1. Modelo `AuditLog` + serviço append-only.  
2. Integração em mutações clínicas/RBAC/paciente desde o primeiro fluxo.  
3. UI mínima ou API interna de consulta filtrada por tenant.

---

### F1-04 — RBAC catálogo e UI

- **Status**: Done  
- **SP**: 13  
- **Dependências**: F1-02  
- **Descrição**: Papéis e permissões (**FR-009**); UI de administração mínima; matriz de capacidades alinhada a **US1** (#4–#7).  
- **Critérios de aceite**: Cenários US1 verificáveis; permissões persistidas e avaliadas em middleware/server actions.  
- **Referências**: **FR-009**, **FR-010** (preparação muro); US1.

**Decomposição sugerida**

1. Modelo de roles/permissions + vínculo user↔clínica↔Empresa.  
2. Helpers `can(permission)` server-side.  
3. UI gestão de papéis (gestor clínica).  
4. Seeds: perfis típicos (atendente, profissional, gestor).

---

### F1-04b — Cadastro empresa / clínica, segmento obrigatório no CL, centro de custo

- **Status**: Done  
- **SP**: 8  
- **Dependências**: F1-01, F1-02, F1-04  
- **Descrição**: Utilizador sem memberships acede a `/setup/tenant` (middleware) e cria **Empresa** + primeira **Clínica** + segmento «Geral» + CL «Principal» + RBAC; página **Empresa e unidades** (`tenant.structure.manage`) para nome da empresa e novas unidades; modelo **CostCenter** com **centro de lucro obrigatório**; **ProfitCenter.segmentId** obrigatório (FK restrict); `session.update({ reloadClinics: true })` para JWT; `lib/tenant/bootstrap-clinic-rbac.ts` partilhado com o seed.  
- **Critérios de aceite**: Fluxo operacional sem depender do seed para empresa; CC e CL com vínculos obrigatórios; documentação SpecKit actualizada (spec.md + tasks).  
- **Referências**: spec §Clarifications 2026-04-11; §3 hierarquia.

---

### F1-05 — Muro financeiro (queries tenant + CL)

- **Status**: Done  
- **SP**: 13  
- **Dependências**: F1-04  
- **Descrição**: Camada **única** de leitura financeira (**FR-010**, **Princípio 2**): serviço `lib/finance/` (ou equivalente) que filtra por clínica ativa e **CL** autorizado; mesmo com livro vazio, APIs/UI não expõem totais de outros CL.  
- **Critérios de aceite**: Casos de teste **SC-004** definidos e a passar em CI para leituras implementadas em F1.  
- **Referências**: **FR-010**; **SC-004**.

**Decomposição sugerida**

1. Contrato de “contexto financeiro” (user, clinicId, CLIds permitidos).  
2. Query builders ou repositórios que aplicam filtro obrigatório.  
3. Testes negativos: user CL X não vê dados CL Y na mesma clínica.

---

### F1-06 — Paciente Empresa + consentimentos base

- **Status**: Done  
- **SP**: 8  
- **Dependências**: F1-04  
- **Descrição**: Cadastro único por **Empresa** (**FR-046**); busca/dedupe; consentimentos mínimos para LGPD na jornada F1 (**FR-012**–**FR-016** via F1-15 transversalmente).  
- **Critérios de aceite**: Unicidade por política (ex. CPF); merge/dedupe documentado; sem duplicar identidade cross-clínica inadvertidamente.  
- **Referências**: **FR-046**; US2/US3 dependências de paciente.

**Decomposição sugerida**

1. Modelo Paciente + vínculo Empresa; índices únicos.  
2. Fluxo criação/edição com AuditLog.  
3. Busca e reutilização na agenda (prepara F1-08).

---

### F1-07 — CL mínimo, convênio, preços planejados

- **Status**: Done  
- **SP**: 8  
- **Dependências**: F1-01, F1-04  
- **Descrição**: **CL** operacional para agenda/muro (**FR-034** parcial); cadastro **convênio** por clínica; **preços planejados** mínimos (**FR-035**) para particular/convênio na agenda e itens (**Princípio 4**).  
- **Nota (implementação no repo)**: modelos Prisma `Convenio` e `PlannedServicePrice`, `Clinic.pricingPolicy` (único por CL vs. por CL+profissional), permissões `pricing.read` / `pricing.manage`, página **`/dashboard/settings/pricing`** (política + CRUD), seed na Unidade Sul e testes de integração. **Agenda** grava `billingMode` + `convenioId` na consulta; **`lib/pricing/resolve-planned-price.ts`** expõe o preço planeado no **painel** da consulta (referência até F1-09 itens FR-023).  
- **Critérios de aceite**: Agenda pode marcar particular vs convênio; valores esperados visíveis no painel (referência planeado); base estável para **F2-05** sem novo cadastro de convênio.  
- **Referências**: **FR-034**, **FR-035**; **FR-018**.

**Decomposição sugerida**

1. CRUD CL (naturezas extra **MAY** F2).  
2. CRUD convênio por `clinicId`.  
3. Tabela ou vínculo de preços planejados (procedimento/tipo ato/convênio).  
4. Integração com F1-05 para restrição de leitura por CL.

---

### F1-08 — Agenda, fila, lista de espera, profissionais/slots

- **Status**: Done
- **SP**: 13  
- **Dependências**: F1-06, F1-07  
- **Descrição**: Agenda **FR-017**–**FR-021**, **FR-019**; profissionais alocados e slots (**FR-047**); lista de espera onde aplicável; estados e tipos de ato alinhados ao painel.  
- **Nota (implementação no repo)**: **`/dashboard/agenda`** com vista dia/semana (FullCalendar UTC, slots 20 min), calendário **por profissional** (`?member=`), criação de consulta, sobreposição, hover/duplo clique no calendário; **`WaitlistEntry`** (lista de espera por clínica, preferência de profissional/serviço opcionais, remoção com `agenda.edit`); **fila do dia** (consultas `CONFIRMED` / `WAITING` no dia UTC do contexto); **SC-001**: nota em `evidence/sc-001.md`, teste de integração de pré-requisitos (`sc001-agenda-smoke.integration.test.ts`) e cenário E2E temporal (`e2e/sc001-agenda-temporal.spec.ts`) com projetos `chromium`, `tablet` e `mobile`; **SC-005**: cobertura automática da visibilidade da lista de espera em `tests/waitlist.integration.test.ts` + nota em `evidence/sc-005.md`. **Pendente típico**: slots/recursos formais.  
- **Cadastro provisório + estado «Cadastro pendente»** (spec *Session cadastro provisório*, **FR-021**): marcação com paciente novo em **`ProvisionalPatientRegistration`** (código da consulta gravado no intermédio); ao carregar agenda ou painel, `applyPatientRegistrationPendingForClinic` promove `SCHEDULED`/`CONFIRMED` → **`PATIENT_REGISTRATION_PENDING`** no **dia UTC** da marcação; no painel, atalho **«Cadastro do paciente»** chama `finalizeProvisionalPatientRegistrationAction` (cria `Patient`, consulta → `CONFIRMED`, **apaga** o intermédio). Reagendar na agenda inclui este estado em `RESCHEDULABLE_ON_AGENDA`.  
- **Estado reconciliado**: **Done no escopo de homologação F1** (agenda/fila/lista de espera e estado cadastro pendente operacionais), com evidência automatizada em `tests/agenda.integration.test.ts`, `tests/waitlist.integration.test.ts` e `e2e/sc001-agenda-temporal.spec.ts` (chromium). Evoluções de agenda/slots no HANDOVER seguem como melhoria contínua, não bloqueador de fecho F1.
- **Critérios de aceite**: **SC-001** preparado (ensaio ou auto onde couber); **SC-005** opcional com critério documentado.  
- **Referências**: **FR-017**–**FR-021**, **FR-019**, **FR-047**; US2.

**Decomposição sugerida**

1. Modelo de slots + recursos humanos por clínica.  
2. UI calendário (dia/semana/mês) + criação de consulta com paciente Empresa.  
3. Estados da consulta alinhados a **FR-020** / painel (**FR-021**, incl. **Cadastro pendente** e fluxo intermédio).  
4. Lista de espera / reatribuição (âmbito **SC-005** se em âmbito).  
5. Sincronização **Cadastro pendente** + atalho conclusão + remoção do intermédio (ver nota F1-08).

---

### F1-09 — Painel da consulta (FR-022 / FR-023) — casca + leitura, sem livro

- **Status**: Done
- **SP**: 13  
- **Dependências**: F1-08  
- **Descrição**: Painel modal completo (**FR-022**): itens persistidos no **modelo final FR-023** (planeado + linhas); secção cobrança com **mesma estrutura** que spec; dados só leitura a partir da consulta; handoff atendimento; **`billingLedgerWritesEnabled: false`**; ações de recebimento → **403** código estável (**ADR-0001**); copy UX “próxima fase financeira” (ver plano — *Painel F1*).  
- **Nota (implementação no repo)**: painel em `app/dashboard/consultations/[id]/page.tsx` com código FR-020, dados clínicos/financeiros em leitura e shell de cobrança F1; itens FR-023 no modelo definitivo `ConsultationItem` (schema + migração) com CRUD no painel via server actions (`add/update/delete`) e auditoria (`CONSULTATION_ITEM_*`) cobertos por `tests/consultation-items.integration.test.ts` e `tests/consultation-item-actions.integration.test.ts`; endpoint `POST /api/billing/ledger` bloqueia escrita com `BILLING_LEDGER_WRITES_DISABLED` e contrato coberto em `tests/billing-ledger-route.test.ts`; evidência consolidada em `evidence/sc-008.md`.  
- **Critérios de aceite**: **ADR-0001** cumprido (UI + API); **FR-020**; **sem** DTO paralelo de itens (**Princípio 3**); E2E F1 cobre secção + bloqueio.  
- **Referências**: **FR-022**, **FR-023**, **FR-020**; **ADR-0001**; **SC-008** parcial.

**Revisão cruzada `plan.md` ↔ `tasks.md` (estado atual)**

- **Coberto**: **FR-020** (código da consulta no painel), shell de cobrança F1 + bloqueio de ledger em `POST /api/billing/ledger` com `403 BILLING_LEDGER_WRITES_DISABLED` (ADR-0001), itens FR-023 persistidos no modelo definitivo (`ConsultationItem`) com CRUD + auditoria.
- **Coberto**: critério “sem DTO paralelo de itens” (itens ligados diretamente à consulta no schema Prisma).
- **Coberto (testes)**: `tests/billing-ledger-route.test.ts`, `tests/consultation-items.integration.test.ts`, `tests/consultation-item-actions.integration.test.ts`, `tests/consultation-panel.integration.test.ts`, `e2e/f109-consultation-panel-ledger-block.spec.ts` e funil `e2e/f117-core-clinical-funnel.spec.ts` (com bloqueio de recebimento no âmbito F1-17).
- **Fecho**: critérios de aceite atendidos no escopo F1 (painel + bloqueio ledger + cobertura E2E mínima).

**Decomposição sugerida**

1. Layout painel: cabeçalho, paciente, profissional, itens, totais, área cobrança.  
2. CRUD itens de consulta (persistência **FR-023**).  
3. Secção cobrança: estados vazios/só leitura; mensagem acessível.  
4. Wire de recebimento: botões disabled + endpoint devolve **403** `BILLING_LEDGER_WRITES_DISABLED` (ou equivalente canónico).  
5. Testes API negativos + Playwright smoke painel (**parcialmente coberto por integração; E2E fica em F1-17**).

---

### F1-10 — Catálogo medicamentos, TUSS, prescrição base (FR-031)

- **Status**: Done
- **SP**: 13  
- **Dependências**: F1-06  
- **Descrição**: Catálogo medicamentos (campos **FR-035**/**FR-031**); TUSS para exames/procedimentos na baseline; busca e seleção para prescrição futura no episódio.  
- **Nota (implementação no repo)**: modelos Prisma `MedicationCatalog` e `ProcedureCatalog` (migração `20260414173000_f110_clinical_catalog`), página `app/dashboard/settings/clinical-catalog/page.tsx` com CRUD via server actions e auditoria, link em `MasterDataNav`; seed inicial da Unidade Sul (`M001`, `10101012`) e teste `tests/clinical-catalog.integration.test.ts`.  
- **Evolução (incremento atual)**: painel da consulta (`app/dashboard/consultations/[id]/page.tsx`) passou a consumir catálogo clínico ativo para seleção assistida no campo “Novo item” (datalist com medicamentos + procedimentos/TUSS) e bloco de referência visual do catálogo; ações do catálogo cobertas em `tests/clinical-catalog-actions.integration.test.ts`.  
- **Evolução (escopo tenant)**: catálogo clínico passou a ter escopo configurável em `Configurações > Empresa > Contratante` (`clinicalCatalogScope`: `COMPANY` ou `CONTRACTOR`). Em `CONTRACTOR`, catálogo único para todo o grupo (todas empresas/clínicas); em `COMPANY`, catálogo separado por empresa.  
- **Estado reconciliado**: **Done no escopo de homologação F1** (catálogo clínico base + CRUD + consumo no painel), com evidência automatizada em `tests/clinical-catalog.integration.test.ts` e `tests/clinical-catalog-actions.integration.test.ts`. Expansões CID-10/ANS-TUSS/sync permanecem trilho evolutivo sem bloquear o fecho da fase.
- **Critérios de aceite**: US3 #7–#10 verificáveis em UI ou API; dados por clínica.  
- **Referências**: **FR-031**, **FR-035**; US3.

**Decomposição sugerida**

1. Modelos catálogo + importação seed TUSS mínima.  
2. UI busca medicamento com atributos obrigatórios.  
3. Catálogo exames TUSS + pesquisa por código/termo.

---

### F1-11 — Episódio, timer, notas, anexos (sem transcrição IA)

- **Status**: Done
- **SP**: 13  
- **Dependências**: F1-09, F1-10  
- **Descrição**: Ciclo de episódio ligado à consulta (**FR-026**, **FR-029**, **FR-030**, **FR-032**); timer; anexos usando storage **F1-01**; **sem** agente transcrição (F2).  
- **Nota (implementação no repo)**: modelo `ConsultationEpisode` + `ConsultationEpisodeAttachment` (migração `20260414195500_f111_consultation_episode_base`), ações no painel da consulta para criar episódio, iniciar/parar timer, editar notas e anexos; **upload** `multipart/form-data` (`addConsultationEpisodeAttachmentAction`) com validação `lib/clinical/episode-attachment-policy.ts`, persistência via `createObjectStorageFromEnv()` (**F1-01** stub) e `sizeBytes`; UI `components/consultation-episode-attachments-block.tsx` no painel `app/dashboard/(clinical-workspace)/consultations/[id]/page.tsx`; auditoria `CONSULTATION_EPISODE_ATTACHMENT_*` no histórico de comandos do episódio; legado `fileName`+`fileUrl` em FormData ainda aceite para testes/fixtures. Teste `tests/consultation-episode.integration.test.ts`.  
- **Evolução (incremento atual)**: marcos de timer persistidos em `ConsultationEpisodeTimerEvent` (START/STOP), exibidos no painel da consulta para rastreio básico do atendimento.  
- **Backlog (agente de transcrição / NLP)**: extração automática de antecedentes a partir de um único texto corrido (repartir transcrito ou `historyAndAntecedents` em categorias) — ver **[backlog-transcription-agent.md](./backlog-transcription-agent.md)** (ANT-EXTRACT-01).  
- **Estado reconciliado**: **Done no escopo de homologação F1** (episódio, timer, notas e anexos base entregues), com evidência automatizada em `tests/consultation-episode.integration.test.ts`, `tests/consultation-panel.integration.test.ts` e cobertura E2E do funil em `e2e/f117-core-clinical-funnel.spec.ts`. Ajustes de prontuário/roteiro no HANDOVER seguem como evolução contínua.
- **Critérios de aceite**: Episódio criado a partir da consulta elegível; timer e marcos; anexos cifrados/stub conforme ambiente.  
- **Referências**: **FR-026**–**FR-032**; US3.

**Decomposição sugerida**

1. Estados “iniciar / em curso / …” alinhados à spec.  
2. Editor/conteúdo estruturado **FR-029**/**FR-030**.  
3. Upload anexo + ligação episódio + RBAC leitura.  
4. Timer persistido com AuditLog.

---

### F1-12 — Assinatura digital, imutabilidade, coerência com agenda

- **Status**: Done  
- **SP**: 13  
- **Dependências**: F1-11  
- **Descrição**: **FR-033** assinatura; bloqueio de edição **FR-027**/**FR-028**; reforço **SC-006**; agenda reflete estado pós-assinatura.  
- **Critérios de aceite**: **SC-006** automatizado: 100% tentativas de edição bloqueadas no tenant correto após assinatura.  
- **Referências**: **FR-033**, **FR-027**, **FR-028**; **SC-006**.
- **Nota 2026-04-26 (biblioteca agregada / ADR-0006):** distinga o **bloqueio do roteiro** (conteúdo + PDF do episódio) da possibilidade de o utilizador interagir com **assinaturas ICP noutros documentos** (prescrição, pedido externo, anexos) conforme regras **por entidade** e copy do produto. Ver addendum em [`clinicagestor/docs/adr/0006-episode-document-library-aggregated-view.md`](../../../clinicagestor/docs/adr/0006-episode-document-library-aggregated-view.md).
- **Nota 2026-04-27 (PAdES por documento):** Lacuna **prepare/finalize** para **anexo PDF** (atestado «Documentos e atestados») e pedido externo TUSS; fallback de leitura do PDF (backup local); nomes de ficheiro com Unicode; biblioteca com realce verde para anexo assinado. `HANDOVER.md`, manual `04-assinatura-digital.md`, PR **#1**.

**Decomposição sugerida**

1. Fluxo “assinar episódio” com validação de campos mínimos.  
2. Camada de domínio “locked” + testes de regressão em APIs.  
3. UI: edição desabilitada + mensagem clara.

**Evolução — assinatura legal fornecida (VIDAAS, extensível)**  
- **Status**: To Do  
- **Rastreio técnico**: [ADR-0003](../../clinicagestor/docs/adr/0003-legal-signature-vidaas-coexistence.md) · [Plano de implementação](../../clinicagestor/docs/plan-implementacao-assinatura-legal-vidaas.md)  
- **Nota**: Complementa **FR-033** com assinatura qualificada no **documento** (PDF PAdES via fornecedor); **não** substitui o encerramento no sistema (`signedAt`); convivência e integridade conforme ADR-0003. Implementação **faseada**; feature flag até sandbox/produção validados.

---

### F1-13 — Agente WhatsApp Agendamento

- **Status**: To Do  
- **SP**: 13  
- **Dependências**: F1-08, F1-02  
- **Descrição**: Integração Twilio (ou mock configurável); agente `scheduling` (**Constitution IV**); fluxos **FR-018**, **§6.4**; confirmação explícita; triagem pré-consulta opcional; AuditLog com origem integração.  
- **Critérios de aceite**: US2 #3–#8; sem criar consulta com mensagem ambígua; opt-in triagem conforme política.  
- **Referências**: **FR-018**; US2; **§6.4**.

**Decomposição sugerida**

1. Webhook + verificação de assinatura Twilio.  
2. Máquina de estados conversa → proposta de slot → confirmação.  
3. Persistência triagem simples ligada à consulta.  
4. Testes integração com conversas fixture.

---

### F1-14 — Reportar erro → canal humano (FR-056 parcial)

- **Status**: Done
- **SP**: 3  
- **Dependências**: F1-02  
- **Nota (repo)**: fluxo base entregue em `/dashboard/support` (`app/dashboard/support/page.tsx` + `actions.ts`), persistência `ErrorReport` (migration `20260424140000_f114_error_report`) e cobertura em `tests/error-report.integration.test.ts` com isolamento por clínica/tenant.
- **Descrição**: Fluxo mínimo de suporte: reporte de erro com âmbito tenant, sem agente **Suporte** completo (F3).  
- **Critérios de aceite**: Ticket ou canal registado por Empresa/clínica; **sem** fuga de dados entre tenants.  
- **Referências**: **FR-056** (parcial); US5 preparação.

---

### F1-15 — Responsividade e UX multi-dispositivo

- **Status**: In Progress  
- **SP**: 8  
- **Dependências**: F1-08, F1-09  
- **Descrição**: **FR-012**–**FR-016**, **FR-015**; paridade usável agenda + painel em desktop/tablet/mobile; acessibilidade básica (contraste, foco, não só cor).  
- **Nota (implementação no repo)**: checklist em `clinicagestor/docs/f115-responsive-accessibility-checklist.md` (breakpoints, rotas críticas incl. anexos do episódio, foco/teclado/cor); smoke E2E em viewport estreita em `e2e/f117-core-clinical-funnel.spec.ts` (bloco `F1-15 — painel em viewport estreita`).
- **Pendência para homologação**: preencher a secção **«Registo de revisão»** do checklist com evidência manual (data, revisor, âmbito e resultado) para `/dashboard/agenda` e `/dashboard/consultations/[id]` em smartphone/tablet/desktop; sem esse registo, manter como **In Progress**.
- **Critérios de aceite**: Checklist de breakpoints documentado; revisão manual ou visual regression em rotas críticas.  
- **Referências**: **FR-012**–**FR-016**; **FR-057** sensibilidade F1.

**Decomposição sugerida**

1. Layout responsivo agenda.  
2. Painel modal acessível em viewport pequena.  
3. Ajustes teclado/foco e legenda de estados (não só cor).

---

### F1-16 — Backup mínimo (FR-067)

- **Status**: Done  
- **SP**: 3  
- **Dependências**: F1-01  
- **Nota (repo)**: `clinicagestor/docs/backup-runbook.md` + `scripts/backup-postgres.sh` (pg_dump + gzip; variáveis `DATABASE_URL` / `BACKUP_DIR`).  
- **Descrição**: Política e runbook mínimo de backup (**FR-067**); automatização básica ou procedimento operacional claro.  
- **Critérios de aceite**: Runbook no repositório; evidência de job ou comando documentado.  
- **Referências**: **FR-067**; **§6.10** (mínimo F1).

---

### F1-17 — E2E Playwright — funil núcleo clínico + negativos

- **Status**: Done  
- **SP**: 13  
- **Dependências**: F1-12  
- **Nota (repo)**: funil feliz `e2e/f117-core-clinical-funnel.spec.ts` + negativos `e2e/f117-negatives.spec.ts` (SC-011 outra clínica, ledger GET sem sessão / sem `finance.read`); seed `seed-consultation-centro-1`; `e2e/global-setup.ts` garante a consulta Centro antes dos testes se o seed antigo faltar; `npm run test:e2e:ci` (Chromium).  
- **Descrição**: Suite **§6.11**: 2FA → agenda → painel (secção cobrança + tentativa recebimento bloqueada) → episódio → assinatura; cenários RBAC / cross-clínica negativos (**SC-003**, **SC-011**).  
- **Critérios de aceite**: **SC-003**, **SC-006**, **SC-011** cobertos nos casos definidos; **ADR-0001** validado em E2E.  
- **Referências**: **§6.11**; [plan#coverage-by-phase](./plan.md#coverage-by-phase).

**Decomposição sugerida**

1. Fixtures multi-tenant + utilizadores com papéis distintos.  
2. Spec funil feliz completo.  
3. Specs negativos (403 recebimento, episódio outra clínica bloqueado).  
4. Integração CI estável (timeouts, seeds).

---

### F1-18 — CI gates (unit + integração + E2E)

- **Status**: Done  
- **SP**: 5  
- **Dependências**: F1-17  
- **Nota (repo)**: `.github/workflows/clinicagestor-ci.yml` — Postgres serviço, `migrate deploy`, `db:seed`, lint, `tsc`, Vitest, Playwright Chromium (`npm run test:e2e:ci`).  
- **Descrição**: Pipeline obrigatório em PR: lint, typecheck, testes unitários, integração, E2E estável; merge bloqueado se falhar.  
- **Critérios de aceite**: Documentação em README; execução reprodutível local + CI.  
- **Referências**: plan **Gate PR**; **§6.11**.

---

# Fase 2 — MVP estendido (financeiro, convênio, segmento, caixa projetado, transcrição)

**Nota de ordenação**: executar na sequência lógica do plano (**F2-01** → **F2-07** → **F2-02** ∥ **F2-03** → **F2-04** → **F2-05** → **F2-06** → **F2-08**; **F2-09** pode sobrepor após **F2-01** com cuidado). IDs numéricos **não** indicam ordem cronológica.

---

### F2-01 — Livro, recebimentos, FR-023/036/037, ativar escritas cobrança

- **Status**: In Progress  
- **SP**: 13  
- **Dependências**: F1-09, F1-07  
- **Descrição**: Implementar **FR-036**, **FR-037**, **FR-023** (planeado×realizado); meios e híbrido; recibo; **`billingLedgerWritesEnabled: true`**; remover modo leitura da cobrança no painel; regressão **ADR-0001**.  
- **Nota (estado actual no repo, 2026-04-23)**:
  - `POST /api/consultations/[id]/receipt` já grava recebimentos no livro, incluindo parcelado com múltiplas linhas e vencimento por parcela (`dueDate` + `receiptInstallmentNumber`).
  - `POST /api/billing/ledger` deixou de ser placeholder `501` e já persiste lançamento genérico quando `BILLING_LEDGER_WRITES_ENABLED=true`.
  - Livro (`/dashboard/finance/livro-contabil`) já expõe coluna de vencimento e leitura enriquecida de dimensões.
  - Pendências para fechar F2-01 como `Done`: validar cobertura final de recibo/fluxo híbrido conforme SC-008 pleno e consolidar regressão E2E F2.
- **Critérios de aceite**: Fluxo US4 painel→livro; **SC-008** pleno com recebimento/recibo em smartphone conforme spec; audit trail financeiro.  
- **Referências**: **FR-036**, **FR-037**, **FR-023**; US4; **ADR-0001**.

---

### F2-07 — Segmento gerencial, EMPRESA_SEGMENTO, mapeamento CL↔Segmento

- **Status**: In Progress  
- **SP**: 13  
- **Dependências**: F2-01, F1-07, F1-04  
- **Descrição**: **Segmento**, cortes, reconciliação numérica preparatória (**SC-010**, **SC-012**); **antes** de relatórios agregados finos (**Princípio 6**).  
- **Nota (estado actual no repo, 2026-04-23)**:
  - Livro (`/dashboard/finance/livro-contabil`) foi confirmado como visão integral do registo de partidas para quem tem acesso ao relatório (sem filtro por CL/segmento).
  - A autorização por objeto (CL/segmento) fica reservada aos demais relatórios de receita/agregação.
  - Próxima lacuna para fechamento: consolidar reconciliação numérica por segmento (SC-010/SC-012) nos relatórios agregados com cobertura de teste.
- **Critérios de aceite**: **SC-010**; cenários **§2.1** A/B da spec em testes ou ensaio documentado.  
- **Referências**: **FR-034**, **FR-038** (preparação); **SC-010**, **SC-012**.

---

### F2-02 — Centro de custo, despesa, espelhamento FR-040 (sem FR-041)

- **Status**: In Progress (parcial no repo, 2026-04-30)  
- **SP**: 8  
- **Dependências**: F2-01, F1-07  
- **Descrição**: **CC**, despesas, vínculos CC→CL espelho simples (**FR-040**); **sem** **FR-041**.  
- **Nota (estado actual no repo, 2026-04-30)**:
  - **Entregue:** `ExpenseRubric` + UI **Rúbricas**; `ExpenseBudgetLine` com **data de vencimento** + UI **Planeamento** (`/dashboard/finance/planejamento`); `LedgerEntryKind.EXPENSE`; **`POST /api/payments/ledger`** (`lib/finance/create-expense-ledger-entry.ts`); espelho **CC→CL/segmento**; **Contas a pagar** com formulário e sugestões a partir do planeamento; migração `20260802120000_expense_planning_budget_line`.
  - **Pendente para fechar F2-02:** parcelamento / recorrência de **despesa**; relatório **planejado×realizado** exportável (pivot); critérios E2E e cobertura de testes API quando estabilizar.
- **Critérios de aceite**: Despesa com parcelas; espelho CL visível com regras de leitura/muro.  
- **Referências**: **FR-040**, **FR-035**; US4.

---

### F2-03 — Relatórios receita-first + export limitado (FR-054) + Segmento

- **Status**: In Progress  
- **SP**: 5  
- **Dependências**: F2-01, F2-07  
- **Descrição**: Primeiros relatórios financeiros; dimensão **Segmento** quando mapeado; export limitado **FR-054**; isolamento tenant.  
- **Nota (estado actual no repo, 2026-04-23)**:
  - `revenue_by_pc` deixou o estado de placeholder e já apresenta agregação de `RECEIPT` por centro de lucro + segmento.
  - O relatório aplica muro financeiro por escopo (full/restricted), mantendo isolamento por clínica e objeto para relatórios de receita.
  - Pendência para fechar: export limitado FR-054 (CSV/XLSX) e cobertura de teste dedicada aos cenários de segmentação.
- **Critérios de aceite**: **FR-054**; **FR-057(c)** onde aplicável; sem fuga cross-tenant.  
- **Referências**: **FR-054**, **FR-057**; **SC-007** (parcial até F2-04).

---

### F2-04 — Relatórios consolidados + export completos

- **Status**: To Do  
- **SP**: 8  
- **Dependências**: F2-02, F2-03  
- **Descrição**: Visão consolidada + exportações completas; performance aceitável no perfil do plano.  
- **Critérios de aceite**: **SC-007** (vista/export conforme spec).  
- **Referências**: **SC-007**; **FR-054**.

---

### F2-05 — Importação extrato convênio (manual + fila)

- **Status**: To Do  
- **SP**: 8  
- **Dependências**: F2-01, F1-07  
- **Descrição**: UI upload, fila de processamento, match manual (**FR-039** sem modelo IA obrigatório); reutiliza convênios **F1-07**; logs de inconsistências.  
- **Critérios de aceite**: Tenant-safe; reconciliação humana possível; dados não corrompem livro.  
- **Referências**: **FR-039**; US4.

---

### F2-06 — Agente IA conciliação + confirmação humana

- **Status**: To Do  
- **SP**: 13  
- **Dependências**: F2-05  
- **Descrição**: `reconciliation` agent; propostas + confirmação; **§6.3.1**; limites de payload e segurança.  
- **Critérios de aceite**: **SC-015**; testes de contrato entrada/saída; nunca gravar sem confirmação humana quando a spec exigir.  
- **Referências**: **§6.3.1**; **SC-015**.

---

### F2-08 — FR-038 fluxo de caixa projetado + SC-013

- **Status**: In Progress (parcial, 2026-04-30)  
- **SP**: 13  
- **Dependências**: F2-01, F2-02, F2-07  
- **Descrição**: Série única realizado+planeado; dia **D** híbrido; cortes por **Segmento** quando aplicável.  
- **Nota (repo 2026-04-30):** relatório **`cash_projection`** com agregação por **Dia / Semana ISO / Mês**, entradas (`RECEIPT`) e saídas (`EXPENSE` + planeamento não liquidado); doc [`clinicagestor/docs/finance/planejamento-despesas-e-fluxo-projetado.md`](../../clinicagestor/docs/finance/planejamento-despesas-e-fluxo-projetado.md). **Pendente:** receitas futuras a partir da agenda (Fase B), cortes por segmento finos, **SC-013** smoke.  
- **Critérios de aceite**: **SC-013** smoke automatizado ou ensaio documentado; sem dupla contagem (alinhado **SC-012**).  
- **Referências**: **FR-038**; **SC-013**.

---

### F2-09 — Agente transcrição + áudio + FR-068 + propostas FR-030

- **Status**: To Do  
- **SP**: 13  
- **Dependências**: F1-11, F1-02, F1-01  
- **Descrição**: `transcription` agent; storage áudio (reuso **F1-01**); retenção **FR-068**; propostas campo a campo com aceite/rejeição (**US3**).  
- **Critérios de aceite**: US3 #4–#5; nada grava em prontuário sem aceite humano explícito.  
- **Referências**: **FR-068**, **FR-030**; US3; **§6.5**.

---

### F2-10 — E2E estendido + smoke SC-013 / SC-015

- **Status**: To Do  
- **SP**: 8  
- **Dependências**: F2-06, F2-09, F2-08, F1-18  
- **Descrição**: Matriz alargada: recebimento híbrido→livro; conciliação manual + um fluxo IA; transcrição aceite/rejeite; smoke opcional **FR-038** (**SC-013**); smoke import **SC-015** onde aplicável.  
- **Critérios de aceite**: **§6.11** estendido; regressão **ADR-0001** + fluxos F2 críticos verdes.  
- **Referências**: [plan#coverage-by-phase](./plan.md#coverage-by-phase).

---

### F2-11 — Performance (FR-057) antecipada + `npm run perf`

- **Status**: **Done** (2026-04-30)  
- **SP**: 13  
- **Dependências**: F1-18 (**MAY**)  
- **Descrição**: Otimização **antes** do gate formal da Fase 3: bundle splitting (FullCalendar em carregamento adiado), eliminar `patient.findMany` global na agenda (lista de espera com pesquisa como «Nova marcação»), `loading.tsx` no segmento Prontuário, deduplicação de resolução de fuso (`React.cache`), índice Prisma `ConsultationEpisode.updatedAt`, script local Lighthouse+Playwright (`npm run perf`) com relatórios em `perf-reports/` (gitignored). **Gate SC-014 completo** continua associado a **F3-07** após **F3-06**.  
- **Nota (fecho 2026-04-30)** — entregue no código e documentado em `HANDOVER.md` / `clinicagestor/docs/performance-optimization-plan.md`: detalhe da marcação ao clique via API (payload de eventos reduzido); Fase 2 em `agenda-full-calendar` (`formatWallClockHmsInTimeZone` / `Intl`, iteração espelho sem `moment` no hot path, `statusEmojiChars` no servidor, cartão memoizado); overlay de «carregar detalhes» removido para não poluir a transição. **`npm run perf`** reprodutível; baseline Lighthouse em doc — rerregar após deploy para comparar TBT. O **gate formal SC-014** em **perfil de volume** e com **observabilidade (F3-06)** fica em **F3-07** (não confundir com o âmbito antecipado de F2-11).  
- **Critérios de aceite**: regressão funcional agenda/lista de espera; `npx tsc` limpo; documentação `plan.md`/`tasks.md`/`HANDOVER.md` alinhadas; script `perf` reprodutível com servidor Next + Postgres seed.  
- **Referências**: **FR-057**–**FR-061** (subset); **SC-014** (smoke local / gate formal volume → **F3-07**).

---

# Fase 3 — Advanced & otimização

---

### F3-01 — Migração US9 + pipeline staging

- **Status**: To Do  
- **SP**: 21  
- **Dependências**: F1-06, F2-01  
- **Descrição**: Carga migratória **FR-062**–**FR-066**, jobs **FR-061**; confirmação e relatórios de lote; staging equiparado.  
- **Critérios de aceite**: **SC-016**, **SC-017**, **SC-018**.  
- **Referências**: US9; **§6.9**.

---

### F3-02 — Rateio FR-041 (CC→CC, CL→CL) + simulação + estorno

- **Status**: To Do  
- **SP**: 13  
- **Dependências**: F2-02, F2-07  
- **Descrição**: Motor de rateio e UI; coerência com **FR-040**; **sem** cruzamento indevido CL↔CC.  
- **Critérios de aceite**: US4 #7–#9; testes de reconciliação numérica.  
- **Referências**: **FR-041**; US4.

---

### F3-03 — Agente follow-up + SC-009

- **Status**: To Do  
- **SP**: 8  
- **Dependências**: F1-13, F1-06  
- **Descrição**: `follow-up` agent; templates; opt-out cadastro Empresa (**US7**).  
- **Critérios de aceite**: **SC-009** — 100% opt-out respeitado em testes cobertos.  
- **Referências**: US7; **SC-009**.

---

### F3-04 — Agente suporte + FR-056 completo

- **Status**: To Do  
- **SP**: 8  
- **Dependências**: F1-14  
- **Descrição**: `support` agent; integração completa **FR-056** com limites de conteúdo clínico (**§6** agentes).  
- **Critérios de aceite**: US5; políticas de recusa/encaminhamento humano testadas.  
- **Referências**: US5; **FR-056**.

---

### F3-05 — US6 backlog pré-cirurgia

- **Status**: To Do  
- **SP**: 5  
- **Dependências**: F1-08  
- **Descrição**: Funcionalidades **FR-024**, **FR-025** para fila pré-cirúrgica.  
- **Critérios de aceite**: Cenários US6 cobertos em testes ou checklist QA.  
- **Referências**: US6; **FR-024**, **FR-025**.

---

### F3-06 — Observabilidade (métricas, tracing, logs, alertas)

- **Status**: To Do  
- **SP**: 8  
- **Dependências**: F1-01  
- **Descrição**: Instrumentação base; SLOs documentados; **MAY** overlap fim F2 (**Princípio 7**).  
- **Critérios de aceite**: Dashboards ou export Prometheus/OTel; runbook alertas.  
- **Referências**: plan **F3-06**; **§6** operação.

---

### F3-07 — Performance FR-057–FR-061 + tuning (gate formal SC-014)

- **Status**: To Do  
- **SP**: 13  
- **Dependências**: F3-06, **F2-11** (trabalho antecipado — não dispensa este gate com observabilidade)  
- **Descrição**: Baterias **SC-014**, **SC-015** em perfil de volume; tuning residual pós **F2-11**; export assíncrono **FR-061** se aplicável.  
- **Nota (2026-04-30)**: **F2-11** está **Done**; este item realiza-se **após** **F3-06** e quando as restantes dependências do backlog o permitirem — não exige antecipação em paralelo a todo o trabalho Fase 2/Fase 3 pendente.  
- **Critérios de aceite**: ≥90% iterações dentro dos limiares do plano ou exceção documentada.  
- **Referências**: **FR-057**–**FR-061**; **SC-014**, **SC-015**.

---

### F3-08 — Testes de carga + CI noturno

- **Status**: To Do  
- **SP**: 8  
- **Dependências**: F3-07  
- **Descrição**: Carga sintética; **SC-007** em volume; pipelines noturnos.  
- **Critérios de aceite**: Orçamentos e perfis documentados; relatórios arquivados.  
- **Referências**: **SC-007**; plan estimativa F3.

---

### F3-09 — Backup/DR + retenção (§6.10) + SC-019/020

- **Status**: To Do  
- **SP**: 8  
- **Dependências**: F1-16  
- **Descrição**: Ensaios de restore, monitorização de backups, alertas (**FR-069**–**FR-071** em operação conforme spec).  
- **Critérios de aceite**: **SC-019**, **SC-020** com evidências arquivadas.  
- **Referências**: **§6.10**; **FR-067**–**FR-071**.

---

### F3-10 — E2E matriz SC + hardening

- **Status**: To Do  
- **SP**: 13  
- **Dependências**: F3-01, F3-02, F3-03, F3-04  
- **Descrição**: Regressão alargada **SC-001**–**SC-020** declarada na matriz; fechar lacunas com automação ou ensaio.  
- **Critérios de aceite**: Matriz `SC × suite × fase` completa sem P1 por cobrir; **§6.11** declarado.  
- **Referências**: [plan#coverage-by-phase](./plan.md#coverage-by-phase); [checklists/requirements.md](./checklists/requirements.md).

---

## Backlog geral (ordenado por fase e dependência)

| # | ID | Título resumido | SP |
|---|-----|-----------------|-----|
| 1 | F1-01 | Monorepo, Prisma, tenant, schema livro RO, storage | 10 |
| 2 | F1-02 | NextAuth, 2FA, clínica ativa | 13 |
| 3 | F1-03 | AuditLog | 8 |
| 4 | F1-04 | RBAC catálogo + UI | 13 |
| 5 | F1-05 | Muro financeiro | 13 |
| 6 | F1-06 | Paciente Empresa | 8 |
| 7 | F1-07 | CL, convênio, preços planejados | 8 |
| 8 | F1-08 | Agenda, slots, profissionais (em curso no repo) | 13 |
| 9 | F1-09 | Painel consulta FR-022/023 sem livro | 13 |
| 10 | F1-10 | Catálogo medicamentos + TUSS | 13 |
| 11 | F1-11 | Episódio, timer, anexos | 13 |
| 12 | F1-12 | Assinatura + imutabilidade | 13 |
| 13 | F1-13 | Agente WhatsApp | 13 |
| 14 | F1-14 | Reportar erro (humano) | 3 |
| 15 | F1-15 | Responsivo FR-012–016 | 8 |
| 16 | F1-16 | Backup mínimo | 3 |
| 17 | F1-17 | E2E núcleo | 13 |
| 18 | F1-18 | CI gates | 5 |
| 19 | F2-01 | Livro + recebimentos + flag escritas | 13 |
| 20 | F2-07 | Segmento + mapeamentos | 13 |
| 21 | F2-02 | CC + despesa + espelho FR-040 | 8 |
| 22 | F2-03 | Relatórios receita-first | 5 |
| 23 | F2-04 | Relatórios consolidados + export | 8 |
| 24 | F2-05 | Import extrato convênio | 8 |
| 25 | F2-06 | IA conciliação | 13 |
| 26 | F2-08 | FR-038 + SC-013 | 13 |
| 27 | F2-09 | Transcrição + FR-068 | 13 |
| 28 | F2-10 | E2E estendido F2 | 8 |
| 29 | F2-11 | Performance FR-057 + npm run perf | 13 |
| 30 | F3-01 | Migração US9 | 21 |
| 31 | F3-02 | Rateio FR-041 | 13 |
| 32 | F3-03 | Follow-up | 8 |
| 33 | F3-04 | Suporte agente | 8 |
| 34 | F3-05 | US6 pré-cirurgia | 5 |
| 35 | F3-06 | Observabilidade | 8 |
| 36 | F3-07 | Performance | 13 |
| 37 | F3-08 | Carga + CI noturno | 8 |
| 38 | F3-09 | Backup/DR evidenciado | 8 |
| 39 | F3-10 | E2E matriz SC | 13 |

---

## Sugestão — primeiras 7 tarefas para iniciar desenvolvimento

1. **F1-01** — Base técnica e schema (bloqueia quase tudo).  
2. **F1-03** — AuditLog em paralelo logo após existir modelo de dados (facilita trilha desde o dia 1).  
3. **F1-02** — Auth + 2FA + clínica ativa (desbloqueia F1-04).  
4. **F1-04** — RBAC (desbloqueia muro, paciente, CL).  
5. **F1-16** — Backup/runbook mínimo (baixo SP, reduz risco operacional cedo).  
6. **F1-06** — Paciente Empresa (paralelo a F1-05/F1-07 após F1-04, conforme capacidade da equipa).  
7. **F1-05** ou **F1-07** — Muro (**F1-05**) vs CL/convênio (**F1-07**): se a equipa for uma só pessoa, preferir **F1-07** depois de **F1-05**; com duas frentes, **F1-07** em paralelo com **F1-05** após **F1-04**.

*(Opcional 8ª): **F1-14** após **F1-02** para canal de feedback interno durante construção.*

---

## Critérios de “Fim de Fase”

### Fase 1 — fechada quando

- Função **agenda → painel → episódio → prescrição (catálogo) → assinatura → imutável** demonstrável em **produção interna** ou staging.  
- **FR-022** no painel com **ADR-0001** integral (UI + **403** + copy); **FR-023** itens no modelo definitivo.  
- **Dispensa formal nesta homologação**: **F1-13 / FR-018 (Agente WhatsApp Agendamento)** fica fora do fecho da Fase 1 por decisão de produto. O agendamento mantém fluxo **manual assistido pela recepção** até replaneamento explícito da automação conversacional.
- **Rastreabilidade da dispensa**: registar no handover/ata de homologação a data da decisão e o plano de entrada do F1-13 na fase posterior, sem bloquear o arranque da **Fase 2**.
- **F1-17** e **F1-18** verdes: E2E núcleo + gates CI.  
- **SC P1 F1** na [matriz](./plan.md#coverage-by-phase): **SC-002**, **SC-003**, **SC-004**, **SC-006**, **SC-011** satisfeitos ou com dispensa registada; **SC-008** em modo **parcial** aceite; **SC-001**/**SC-005** conforme decisão MAY.  
- **Sem** **FR-039** import extrato, **sem** UI segmento gerencial, **sem** **FR-038** pleno, **sem** transcrição IA.

### Fase 2 — fechada quando

- **`billingLedgerWritesEnabled === true`** e fluxo **US4** painel→livro com **SC-008** pleno.  
- **F2-07** entregue **antes** de considerar relatórios finais estagnados; **F2-03**/**F2-04** com dimensão segmento.  
- **FR-039** + fila + **F2-06** IA com confirmação humana e **SC-015** endereçado.  
- **F2-08** com smoke **SC-013** acordado.  
- **F2-09** transcrição com **FR-068** e aceite/rejeição por campo.  
- **F2-10** verde; matriz **SC** F2 atualizada no artefacto de rastreio.

### Fase 3 — fechada quando

- **F3-01** migração com **SC-016**–**SC-018**.  
- **F3-02** **FR-041** em produção/staging com testes de reconciliação.  
- Agentes **F3-03**, **F3-04** com **US7**/**US5** e **SC-009** onde aplicável.  
- **F3-07**/**F3-08** com **SC-014**, **SC-015**, **SC-007** volume conforme plano.  
- **F3-09** com evidências **SC-019**/**SC-020**.  
- **F3-10**: declaração explícita de cobertura **SC-001**–**SC-020** + **§6.11** sem lacunas P1 não justificadas.

---

## Matriz de rastreio `SC × suite × fase` (placeholder)

Preencher e manter junto de QA/CI (obrigatório antes de fechar cada fase). Referência de âmbito: [plan.md#coverage-by-phase](./plan.md#coverage-by-phase).

| SC | Suite / artefacto | Fase alvo | Automatizado? | Notas |
|----|-------------------|-----------|-----------------|-------|
| SC-001 | e2e + int agenda (`e2e/sc001-agenda-temporal.spec.ts`, `tests/sc001-agenda-smoke.integration.test.ts`) | 1 → 3 | Parcial | Ver `evidence/sc-001.md` |
| SC-002 | unit+int / audit UI | 1 | Parcial | Trilha de auditoria já usada em mutações agenda/painel; ampliar matriz por módulo |
| SC-005 | int lista de espera (`tests/waitlist.integration.test.ts`) | 1 | Sim (escopo funcional) | Ver `evidence/sc-005.md`; falta E2E UI |
| SC-008 | int painel+ledger gate (`tests/consultation-items.integration.test.ts`, `tests/consultation-item-actions.integration.test.ts`, `tests/billing-ledger-route.test.ts`) | 1 → 2 | Parcial F1 | Ver `evidence/sc-008.md`; pleno após F2-01 |
| … | … | … |  | |
| SC-020 | ops evidence | 3 |  | |

*(Copiar linhas SC-002 … SC-019 do checklist [requirements.md](./checklists/requirements.md) e associar paths de teste reais quando o código existir.)*

---

## Manutenção deste ficheiro

- Qualquer alteração a IDs, dependências ou SP: **atualizar primeiro** [plan.md](./plan.md), depois sincronizar `tasks.md` e [checklists/requirements.md](./checklists/requirements.md).  
- Commits que fecham tarefas: referir **ID** (ex. `F1-09`) na mensagem para rastreio.
