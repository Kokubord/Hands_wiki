<!-- wiki-ingest: synced from specs/001-clinicagestor-platform/plan.md at 2026-05-09T18:04:15.341Z — edit upstream in monorepo, not here -->

# Plano técnico: ClinicaGestor – Plataforma integrada para clínicas

**Branch**: `001-clinicagestor-platform`  
**Data**: 2026-04-11 (rev. **três fases** + MVP estendido)  
**Especificação**: [spec.md](./spec.md)  
**Constituição**: [.cursor/commands/speckit.constitution.md](../../.cursor/commands/speckit.constitution.md)

Este documento fixa a implementação em **três fases** alinhadas à **spec** (**FR**, **SC**, User Stories, **§6**–**§11**) e à **Constituição**. A **Fase 2** aqui corresponde ao **MVP estendido** (financeiro completo no produto + convênio + segmento + caixa projetado + transcrição), **depois** do núcleo operacional da **Fase 1** ter dado **vida ao modelo** (dados, RBAC, agenda, prontuário assinado). A antiga “Fase 2 – Advanced” passou a **Fase 3**.

Artefactos Speckit (`research.md`, `data-model.md`, `contracts/`, `quickstart.md`, [`tasks.md`](./tasks.md)) devem refletir esta divisão. **ADRs** desta feature: pasta [`adr/`](./adr/) (ex.: [**0001**](./adr/0001-fr022-billing-shell-phase1.md) — **FR-022** / cobrança Fase 1).

---

## Summary

O **ClinicaGestor** é uma aplicação **web** multi-tenant que integra **agenda**, **prontuário**, **financeiro** e **agentes de IA** com **LGPD**, **AuditLog** e **RBAC**. **Anti-refatoração**: secção **Princípios de implementação** (abaixo); **cobrança F1 vs livro**: [**ADR-0001**](./adr/0001-fr022-billing-shell-phase1.md).

- **Fase 1 – MVP núcleo**: auth forte, tenant, RBAC, muro, paciente Empresa, **CL + convênio + preços** (**FR-035**), agenda, painel com **FR-023** + **FR-022** em **casca + leitura** (sem POST livro), prontuário até assinatura/imutabilidade, WhatsApp agendamento, responsivo, backup mínimo, **E2E** núcleo + gate cobrança.

- **Fase 2 – MVP estendido**: ativação livro (**FR-023**/**FR-036**/**FR-037**), **Segmento** cedo, **CC** + **FR-040** (sem **FR-041**), relatórios + **FR-054**, importação extrato **FR-039** + IA conciliação, **FR-038**/**SC-013**, transcrição **FR-068**, **performance / FR-057** (**F2-11**, antecipada face à Fase 3), **E2E** alargado.

- **Fase 3 – Advanced**: migração **US9**, **FR-041**, agentes follow-up/suporte, **US6**, observabilidade, performance/carga (**SC-014**/**SC-015**), backup/DR (**SC-019**/**SC-020**), matriz **E2E**/**§6.11**.

**Stack preferida** (**§11** da spec): Next.js 15, React 19, TypeScript strict, Tailwind + shadcn/ui, Prisma, PostgreSQL, NextAuth v5, Twilio, Stripe + PIX, Vercel AI SDK.

---

## Nota de planeamento — fase intermédia e alinhamento à spec

**Racional**: separar risco **livro/reconciliação** de **prontuário imutável + tenant**; demos quinzenais com valor clínico em F1; F2 concentra fecho financeiro e IA quando o modelo/API estão estáveis; **Constitution IV**: F1 agendamento → F2 conciliação + transcrição → F3 follow-up + suporte.

**FR-022 / cobrança**: decisão e critérios completos em [**ADR-0001**](./adr/0001-fr022-billing-shell-phase1.md) (UI F1 + `billingLedgerWritesEnabled`; **F2-01** ativa escritas; **SC-008** em pleno após F2; E2E F1 = secção presente + **403** estável em mutações livro).

---

## Technical Context

| Dimensão | Decisão |
|----------|---------|
| **Linguagem** | TypeScript (strict), Node.js LTS |
| **Aplicação** | Monólito modular Next.js |
| **Persistência** | PostgreSQL + Prisma; `companyId` / `clinicId` |
| **Auth** | NextAuth v5; **2FA** (**FR-007**); OTP WhatsApp |
| **Ficheiros / áudio** | Object storage cifrado; retenção **FR-068** a partir da Fase 2 (transcrição) |
| **Testes** | Ver [**Estratégia de cobertura por fase**](#coverage-by-phase); **§6.11** |
| **Performance** | **FR-057** como guia em F1/F2; trabalho **F2-11** (bundle/query/`npm run perf`); gates formais **SC-014**/**SC-015** em volume na **Fase 3** (**F3-07**/**F3-08**) salvo **exceção** documentada |

---

<a id="coverage-by-phase"></a>

## Estratégia de cobertura por fase (testes + SC)

Objetivo: **três decisões explícitas** em cada fase — o que **bloqueia merge (PR)**, o que **bloqueia release** da fase, e o que corre em **pipeline noturno/semanal**. Critérios **SC** da [spec](./spec.md) são **portas de aceite**: cada SC MUST estar **automatizado**, **ensaio documentado** (critério + evidência) ou **explicitamente fora de âmbito** da fase com **justificação** e data de cobertura.

### Definições (gates)

| Gate | Significado |
|------|-------------|
| **PR** | Obrigatório em **todo** merge para `main` (ou política equivalente): **unit + integração** do que mudou + **suites estáveis** listadas para a fase. |
| **Release fase** | Obrigatório antes de declarar a fase “fechada”: **E2E** da fase + **SC** marcados **P1** para essa fase **verdes** ou com **dispensa** registada. |
| **Noturno / semanal** | **Carga**, **E2E longo**, **baterias FR-057** pesadas, ensaios **SC-001**/UX — **não** bloqueiam PR salvo política explícita. |

### Pirâmide por fase (resumo)

| Camada | Fase 1 | Fase 2 | Fase 3 |
|--------|--------|--------|--------|
| **Unit** | Domínio tenant, RBAC helpers, AuditLog, imutabilidade, queries muro | Livro, parcelas, espelho **FR-040**, SQL relatórios, fila import, validadores | Rateio **FR-041**, jobs migração, políticas agentes |
| **Integração** | API + DB isolamento clínica/Empresa; negativos cross-tenant | Endpoints financeiros + filas; export; IA **contrato** (**§6.3.1**) | Pipeline migração staging; webhooks |
| **E2E** | **Um funil** crítico (ver **F1-17**) + RBAC negativos | **F2-10** — múltiplos fluxos financeiros + conciliação + transcrição | Regressão **matriz SC** (**F3-10**) |
| **Carga / SRE** | — | Smoke **FR-057** **MAY** em CI | **F3-08** — **SC-014**, **SC-015**, **SC-007** em perfil de volume |

### SC — âmbito mínimo por fase

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
| **SC-014** / **SC-015** | Fora do gate F1 | Smoke **MAY** (**F2-11** local); gate completo **preferencialmente F3** | **P1** (**F3-07** / **F3-08**) |
| **SC-016**–**SC-018** | — | — | **P1** migração (**F3-01**) |
| **SC-019** / **SC-020** | — | — | **P1** evidência operacional (**F3-09**) |

**Regra**: qualquer SC **P1** sem automação MUST ter **procedimento de ensaio** (passos, dados, evidência arquivada) até existir teste — evita “verde falso”.

### Rastreabilidade

Manter **uma matriz** `SC × (unit \| int \| e2e \| manual) × fase` no artefacto de trabalho (**`tasks.md`** ou checklist de QA), atualizada quando **F1-18**, **F2-10** ou **F3-10** mudarem âmbito.

---

## Constitution Check

| Princípio | Como se cumpre |
|-----------|----------------|
| **I – Domínio** | F1 = núcleo dos três módulos **em fluxo**; F2 = fecho financeiro/spec **FR-022**/**US4**; F3 = escala e adoção. |
| **II – LGPD / AuditLog / imutabilidade** | Transversal; **SC-006** desde F1. |
| **III – RBAC** | **FR-009**, **FR-010** desde F1 (muro com queries seguras mesmo com livro vazio). |
| **IV – Cinco agentes** | F1: **WhatsApp Agendamento**; F2: **Conciliação** + **Transcrição**; F3: **Follow-up** + **Suporte** (`/lib/ai/agents/`). |
| **V – Agenda** | **FR-017**–**FR-022**; cobrança no painel **ADR-0001** (casca F1, livro F2). |
| **VI–VII** | **§11**, responsividade, **FR-054** na **Fase 2**. |

### Complexity Tracking (agentes e spec FR-022)

| Desvio | Porquê | Mitigação |
|--------|--------|-----------|
| **Livro / recebimentos** só na Fase 2 | Reduz acoplamento agenda–financeiro na primeira onda; foco em **imutabilidade** e **tenant**. | **Princípio 1** + **ADR-0001** (evitar repetir texto do ADR aqui). |
| **Rateio FR-041** na Fase 3 | Dependência de **CC→CL** estável e **Segmento** (F2). | F3 após **F2-02**/**F2-07**. |

---

## Project Structure

*(Igual à versão anterior: `app/`, `lib/db`, `lib/rbac`, `lib/audit`, `lib/ai/agents/`, `jobs/`, `e2e/`.)*

**Agentes por fase**: `scheduling.ts` (F1); `reconciliation.ts`, `transcription.ts` (F2); `follow-up.ts`, `support.ts` (F3).

---

## Princípios de implementação (reúso técnico / anti-refatoração)

1. **Schema financeiro cedo, mutações tarde**: em **F1-01**, modelar no Prisma as entidades necessárias a **FR-036** / **FR-037** (livro, recebimento, parcelas, vínculo à **consulta** e **CL**) com migrações **estáveis**; **proibir** `POST`/mutações que persistam no livro até **`billingLedgerWritesEnabled`** e **F2-01** (**ADR-0001**). Reduz migrações grandes na transição F1→F2.

2. **Leitura financeira única**: um serviço partilhado (ex.: camada `lib/finance/` com queries tenant-aware) alimenta o **muro** (**FR-010**), o painel (F1 vazio / F2 preenchido) e relatórios — **sem** duplicar regras de isolamento só na UI.

3. **Itens de consulta alinhados a FR-023**: persistir em F1 **linhas de item** + **valor planejado** no mesmo modelo que **F2-01** usará para **planejado × realizado**; evita “DTO provisório” e refator do painel.

4. **Convênio e preços na F1**: **F1-07** inclui **cadastro de convênio** (por clínica) e **preços planejados** mínimos (**FR-035**) para alimentar agenda (**FR-018**), itens e **F2-05** (importação de extrato referencia o mesmo cadastro).

5. **Object storage em F1-01**: variáveis de ambiente + cliente/SDK (ou stub) para ficheiros **cifrados** — reutilizado por **anexos** (**FR-032**) e, na Fase 2, por **áudio** / transcrição (**F2-09**), sem novo tipo de deploy só na F2.

6. **Segmento antes dos relatórios agregados**: na Fase 2, **F2-07** segue **logo após F2-01** (ver tabela) para que **F2-03**/**F2-04** já projetem dimensão **Segmento** opcional — evita reescrever SQL de relatórios.

7. **Observabilidade em overlap**: **F3-06** MAY iniciar-se no **final da Fase 2** (paralelo a **F2-09**/**F2-10**) para instrumentar antes da **F3-07** (performance) e **maximizar** reúso de pipeline CI.

8. **Roteiro vs. assinaturas em documentos auxiliares (2026-04-26):** a **biblioteca agregada** do episódio (**ADR-0006**) apresenta prescrição, pedido externo, anexos e roteiro numa única vista. A regra de **bloqueio / imutabilidade** que afecta o **roteiro** (campos + PDF on-demand) **não** remove por si a possibilidade de o utilizador ainda **assinar com ICP** outros documentos cuja lógica de assinatura seja **independente** (fornecedor activo, RBAC, estado do PDF da prescrição, etc.); o CTA «Assinar com certificado» a partir de **pedido externo** ou **anexo** continua a designar a assinatura do **PDF do roteiro**, com copy explícita no UI. Mantém-se **SC-006** para o objecto imutável após conclusão conforme o domínio de cada anexo.

---

## Estimativa global

| Fase | SP (faixa) | DI indicativos |
|------|------------|----------------|
| **Fase 1** | 80–108 | 10–15 semanas (2–3 devs) |
| **Fase 2** (MVP estendido) | 82–112 | 10–15 semanas |
| **Fase 3** (Advanced) | 68–95 | 8–14 semanas |

---

# Fase 1 – MVP núcleo (operacional: “vida ao modelo”)

**Objetivo**: percurso **agenda → painel → episódio → prescrição → assinatura → imutabilidade**; **paciente único Empresa**; **CL + convênio + preços** (**FR-035**); **2FA**, **RBAC**, **AuditLog**, **UI responsiva**, **WhatsApp** agendamento. **Painel**: **FR-022** casca + leitura (**ADR-0001**). **Fora de âmbito F1**: **FR-039**, UI **Segmento**, **FR-038** pleno, transcrição IA, escritas livro.

### Painel da consulta em F1 — experiência sem livro (**FR-022**)

Objetivo de UX: o utilizador **não** interpreta o produto como “ainda não instalaram o financeiro”, mas como **“recebimentos e recibos ativam na próxima fase”** — alinhado ao risco já listado em **Riscos principais** (modo leitura vs falha).

| Aspeto | Comportamento (F1) |
|--------|---------------------|
| **Layout** | **Mesma** secção de cobrança que a spec prevê (**FR-022**): cabeçalho, itens, totais, área de pagamento/recibo — **sem** omitir blocos. |
| **Dados visíveis** | **Leitura** a partir da **consulta** e **itens** (**FR-023**): valores **planeados**, particular/convênio, estado operacional da consulta quando existir **sem** depender do livro. |
| **Ações bloqueadas** | Registar recebimento, estorno, recibo definitivo no livro: **desativados** ou **403** com código estável — **proibido** “clique mudo”. |
| **Copy / acessibilidade** | Mensagem curta e repetível (ex.: *“Registo de recebimentos na clínica ativa fica disponível na atualização financeira”*) + **ADR-0001** § decisão 4. |
| **O que já pode funcionar** | Lembrete WhatsApp, navegação painel, itens, triagem pré-consulta, tudo o que **não** persista no livro — ver **SC-008** parcial na [estratégia de cobertura](#coverage-by-phase). |

Detalhe técnico e política de API: [**ADR-0001**](./adr/0001-fr022-billing-shell-phase1.md) (`billingLedgerWritesEnabled`).

### Ordem recomendada (Fase 1)

1. Fundamentos + Auth + AuditLog  
2. RBAC + clínica ativa + muro financeiro  
3. Paciente Empresa + **CL + convênio + preços planejados** (**F1-07**) + profissionais  
4. Agenda + lista de espera + **painel** (itens / planejado no modelo definitivo **FR-023**)  
5. Prontuário (episódio, timer, **FR-029**–**FR-032**, **FR-031**) + assinatura (**FR-033**) + imutabilidade (**FR-027**, **FR-028**)  
6. Agente WhatsApp + triagem pré-consulta opcional  
7. Responsivo + LGPD mínimo + backup mínimo  
8. **E2E** núcleo clínico + CI gates  

### Cadastro provisório de paciente e estado «Cadastro pendente» (F1)

- **Problema**: evitar **duplicados** no cadastro **Empresa** ao marcar **paciente novo** antes de existir ficha definitiva.
- **Modelo**: tabela intermédia com **nome/contactos**, **número e id da consulta** na clínica; consulta com `patientId` vazio até conclusão.
- **Estado `PATIENT_REGISTRATION_PENDING`**: atribuição **automática** no **dia UTC** da marcação ao abrir **agenda** ou **painel** (`applyPatientRegistrationPendingForClinic`); só então o **atalho** no painel chama a action de **concluir cadastro**.
- **Conclusão**: cria **Patient** (autocode), consulta passa a **CONFIRMED** (ou política equivalente), **remove** o registo intermédio; **AuditLog** na operação.
- **Transições**: ver `lib/agenda/status-transitions.ts` e **FR-021** na spec (*Session cadastro provisório*).

### Tarefas – Fase 1

| ID | Entrega | Dependências | SP | Critérios de aceite |
|----|---------|----------------|-----|---------------------|
| **F1-01** | Monorepo + Prisma + schema tenant + **livro/recebimento** (tabelas/FKs, **sem** mutations API até F2) + **object storage** (env/SDK ou stub para anexos/áudio futuro) | — | 10 | Migrações reprodutíveis; **quickstart**; **Princípio 1** (schema antecipado) |
| **F1-02** | NextAuth + 2FA + `activeClinicId` (**FR-003**, **FR-007**) | F1-01 | 13 | **SC-003** (preparação); troca clínica |
| **F1-03** | **AuditLog** (**FR-005**, **FR-011**) | F1-01 | 8 | **SC-002** |
| **F1-04** | RBAC catálogo + UI (**FR-009**) | F1-02 | 13 | US1 #4–#7 |
| **F1-05** | Muro financeiro (**FR-010**) | F1-04 | 13 | **SC-004** (queries seguras) |
| **F1-06** | Paciente Empresa (**FR-046**) + consentimentos base | F1-04 | 8 | Unicidade / dedupe |
| **F1-07** | **CL** (**FR-034** parcial: mínimo para agenda/muro; naturezas administrativas/completas **MAY** F2) + **convênio** + **preços planejados** (**FR-035**) | F1-01, F1-04 | 8 | Particular/convênio na agenda; base **F2-05** |
| **F1-08** | Agenda **FR-017**–**FR-021** + **FR-019**; **profissionais** alocados à unidade e slots (**FR-047** onde aplicável) | F1-06, F1-07 | 13 | **SC-001**; **SC-005** opcional |
| **F1-09** | **Painel consulta FR-022**: **itens persistidos** (mesmo modelo **FR-023**), planeamento, **secção cobrança** (UI completa, só leitura); handoff atendimento; **`billingLedgerWritesEnabled: false`** + APIs recebimento → **403** | F1-08 | 13 | **FR-020**; **ADR-0001**; **sem** DTO paralelo de itens |
| **F1-10** | Catálogo medicamentos + TUSS + **FR-031** | F1-06 | 13 | US3 #7–#10 |
| **F1-11** | Episódio + timer + **FR-029**/**FR-030**/**FR-032** | F1-09, F1-10 | 13 | **FR-026** |
| **F1-12** | Assinatura + imutabilidade + coerência agenda (**FR-033**, **FR-027**, **FR-028**) | F1-11 | 13 | **SC-006** |
| **F1-13** | **Agente WhatsApp** (**FR-018**, §6.4) | F1-08, F1-02 | 13 | US2 #3–#8 |
| **F1-14** | Reportar erro → canal humano (**FR-056** parcial) | F1-02 | 3 | Âmbito tenant |
| **F1-15** | Responsivo **FR-012**–**FR-016** + **FR-015** | F1-08, F1-09 | 8 | Paridade multi-dispositivo |
| **F1-16** | Backup mínimo **FR-067** | F1-01 | 3 | Runbook |
| **F1-17** | **E2E** Playwright: 2FA → agenda → painel (**secção cobrança** + tentativa recebimento bloqueada) → episódio → assinatura + negativos RBAC | F1-12 | 13 | **§6.11**; **SC-003**, **SC-006**, **SC-011**; **ADR-0001** |
| **F1-18** | CI gates (unit + integração + E2E estável) | F1-17 | 5 | Merge bloqueado se falhar |

**Dependência crítica**: **F1-12** (imutabilidade) — testes contínuos **SC-006**.

---

# Fase 2 – MVP estendido (financeiro, convênio, segmento, caixa projetado, transcrição)

**Objetivo**: cumprir **US4** no essencial; **ativar escritas no livro** no painel (**`billingLedgerWritesEnabled: true`**, **FR-036**/**FR-037**/**FR-023**) completando **FR-022** face a recebimentos e recibos; **importação convênio** (**FR-039**) primeiro **sem IA**, depois **com Agente de conciliação**; **Segmento** + **SC-010**/**SC-012**; **FR-038** + **SC-013**; **Agente de transcrição** + **FR-068** (áudio/retenção).

### Ordem recomendada (Fase 2)

*(Ordenada para **reúso de schema** e **menos refactor** de relatórios.)*

1. **Receita planejada** + **recebimentos** + **livro** + **híbrido** + **recibo** (**FR-023**, **FR-036**, **FR-037**) — ativa **`billingLedgerWritesEnabled`**.  
2. **Segmento** + mapeamentos **CL↔Segmento** (**F2-07**) — **logo após** o livro, **antes** dos relatórios agregados.  
3. **CC** + despesa + **espelhamento CC→CL** (sem **FR-041**).  
4. **Relatórios**: **receita-first** + export limitado; depois consolidados + **FR-054** completo (já com dimensão segmento).  
5. **Importação arquivo convênio** + fila + matches manuais (reutiliza convênios **F1-07**).  
6. **Agente IA conciliação** (§6.3.1).  
7. **FR-038** + **SC-013**.  
8. **Agente transcrição** + **FR-068** (paralelizável com 5–7 após **F2-01**).  
9. **E2E** estendido (incl. smoke opcional **SC-013**).

### Tarefas – Fase 2

**Ordem das linhas** = **sequência sugerida de execução** (grafo real está na coluna **Dependências**). O prefixo numérico (**F2-03** vs **F2-07**) **não** implica cronologia: **F2-07** (Segmento) **antes** de **F2-03**/**F2-04** (relatórios), conforme **Princípio 6** e **F2-03** → depende de **F2-07**.

| ID | Entrega | Dependências | SP | Critérios de aceite |
|----|---------|----------------|-----|---------------------|
| **F2-01** | **FR-023** + **FR-036** + **FR-037** (meios, híbrido, livro, planejado×realizado) + **`billingLedgerWritesEnabled: true`** + remoção de modo leitura cobrança | F1-09, F1-07 | 13 | US4 fluxo painel→livro; **SC-008** com recebimento/recibo; regressão **ADR-0001** |
| **F2-07** | **Segmento** + **EMPRESA_SEGMENTO** + mapeamento **CL↔Segmento** + cortes + **SC-012** | F2-01, F1-07, F1-04 | 13 | **SC-010**; §2.1 A/B |
| **F2-02** | **CC** + despesa + espelho **FR-040** simples (sem **FR-041**) | F2-01, F1-07 | 8 | Despesa + parcelas CL |
| **F2-03** | Relatórios **receita-first** + export limitado (**FR-054**); dimensão **Segmento** quando mapeado | F2-01, F2-07 | 5 | Isolamento tenant; **FR-054**; vista interativa **FR-057(c)** quando aplicável |
| **F2-04** | Relatórios consolidados + export completos | F2-02, F2-03 | 8 | **SC-007** (vista/export) |
| **F2-05** | UI importação convênio + fila + match manual (**FR-039** sem modelo) | F2-01, F1-07 | 8 | Reutiliza **convênio** de **F1-07**; log inconsistências; tenant |
| **F2-06** | **Agente IA conciliação** + confirmação humana | F2-05 | 13 | **SC-015**; §6.3.1 |
| **F2-08** | **FR-038** fluxo de caixa projetado + **SC-013** | F2-01, F2-02, F2-07 | 13 | Série única; híbrido em **D**; cortes por **Segmento** quando aplicável |
| **F2-09** | **Agente transcrição** + armazenamento áudio + **FR-068** + propostas **FR-030** | F1-11, F1-02, F1-01 | 13 | US3 #4–#5; reutiliza storage **F1-01** |
| **F2-10** | **E2E** estendido: recebimento híbrido → livro; conciliação (manual + 1 fluxo IA); transcrição aceite/rejeite; **smoke opcional FR-038** (**SC-013**) | F2-06, F2-09, **F2-08**, F1-18 | 8 | **§6.11**; smoke **SC-015**; matriz alargada |

**Encadeamento lógico (resumo)**: `F2-01` → `F2-07` → (`F2-02` em paralelo com `F2-03` após ambos terem pré-requisitos: `F2-03` só precisa **F2-01+F2-07`; `F2-02` precisa **F2-01**) → `F2-04` → cadeia importação `F2-05`→`F2-06` → `F2-08` → `F2-09` pode **overlap** com importação/FR-038 após **F2-01** (ver ordem recomendada).

**Nota de implementação (repo, 2026-04-26):** existe **MVP operacional** de **Contas a Pagar** com cadastro de **rúbricas** (`ExpenseRubric`), UI em `/dashboard/finance/contas-a-pagar`, `POST /api/payments/ledger` (despesa com `rubricId` obrigatório) e relatório preliminar em `/dashboard/finance/relatorios/despesas`. Isto **não** encerra **F2-02** na totalidade dos critérios de aceite (parcelas de despesa, orçamento formal e planejado×realizado completos); ver estado **In Progress (parcial)** em `tasks.md`.

**Dependência crítica**: **F2-08** exige **F2-07** quando o **fluxo projetado** incorporar agregações por **Segmento** (**FR-038** / **FR-034**).

---

# Fase 3 – Advanced & otimização

**Objetivo**: adoção (**User Story 9**), **rateios FR-041**, agentes **Follow-up** e **Suporte**, **US6**, **observabilidade**, **performance** (**SC-014**, **SC-015**), **carga**, **backup/DR** evidenciado (**SC-019**, **SC-020**), matriz de testes alinhada a **todos** os **SC**.

### Ordem recomendada (Fase 3)

1. Migração legado (**FR-062**–**FR-066**, jobs **FR-061**)  
2. **Rateios FR-041** + reconciliação com **FR-040**  
3. **Agente follow-up** + **SC-009**  
4. **Agente suporte** + **FR-056** completo  
5. **US6** pré-cirurgia (**FR-024**, **FR-025**)  
6. Observabilidade + SLOs  
7. Performance + testes de carga  
8. **§6.10** completo (evidências **SC-019**/**SC-020**, **FR-068**–**FR-071** em operação)  
9. **E2E** completo + regressão **SC-001**–**SC-020**

### Tarefas – Fase 3

| ID | Entrega | Dependências | SP | Critérios de aceite |
|----|---------|----------------|-----|---------------------|
| **F3-01** | **Migração US9** + pipeline staging/confirmação | F1-06, F2-01 | 21 | **SC-016**–**SC-018** |
| **F3-02** | **Rateio FR-041** CC→CC e CL→CL + simulação + estorno | F2-02, F2-07 | 13 | US4 #7–#9; sem cruzamento CL↔CC |
| **F3-03** | **Agente follow-up** + templates + opt-out | F1-13, F1-06 | 8 | **US7**, **SC-009** |
| **F3-04** | **Agente suporte** + integração **FR-056** | F1-14 | 8 | **US5** |
| **F3-05** | **US6** backlog pré-cirurgia | F1-08 | 5 | **FR-024**, **FR-025** |
| **F3-06** | Observabilidade (métricas, tracing, logs, alertas) | F1-01 | 8 | SLOs documentados; **MAY** iniciar em **paralelo** ao fim da Fase 2 (**Princípio 7**) |
| **F3-07** | Performance **FR-057**–**FR-061** + tuning | F3-06 | 13 | **SC-014**, **SC-015** |
| **F3-08** | Testes de **carga** + CI noturno | F3-07 | 8 | Orçamentos no plano |
| **F3-09** | **Backup/DR** + retenção operacional (**§6.10**) | F1-16 | 8 | **SC-019**, **SC-020** |
| **F3-10** | **E2E** matriz **SC** + hardening | F3-01–F3-04 | 13 | **§6.11** cobertura declarada |

---

## Roadmap (alto nível)

```text
F1: schema livro (sem POST) · CL+convênio+preços · agenda · painel FR-023 · prontuário · WhatsApp
F2: POST livro · segmento · CC/espelho · relatórios+extrato+IA · FR-038 · transcrição
F3: US9 · FR-041 · agentes · US6 · observ. · perf/carga · backup/DR
```

---

## Riscos principais

| Fase | Risco | Impacto | Mitigação |
|------|-------|---------|-----------|
| **1** | Utilizadores confundem “modo leitura” com falha do produto | Médio | **ADR-0001**: mensagens explícitas na UI; formação na demo; **F2-01** o mais cedo possível na Fase 2. |
| **1** | Imutabilidade / cross-tenant | Crítico | **SC-006**, **SC-003** em CI. |
| **2** | **FR-038** + segmento: dupla contagem | Alto | **SC-012**, **SC-013**; revisão com negócio. |
| **2** | IA conciliação / payload | Crítico | **§6.3.1**; testes de contrato. |
| **3** | Migração corrupção dados | Crítico | **FR-063** staging; **SC-016**–**SC-018**. |

---

## Entregas incrementais sugeridas

### Fase 1 (quinzenal)

| Incremento | Tarefas-alvo | Valor demonstrável |
|------------|----------------|-------------------|
| **S1–S2** | F1-01–F1-05 | Auth 2FA, tenant, RBAC, muro, AuditLog |
| **S3–S4** | F1-06–F1-09 | Paciente, **CL + convênio + preços**, agenda, painel + **itens FR-023** + **ADR-0001** |
| **S5–S6** | F1-10–F1-12 | Prescrições, episódio, **assinatura + imutabilidade** |
| **S7–S8** | F1-13–F1-16 | WhatsApp, responsivo, backup, reportar erro |
| **S9–S10** | F1-17–F1-18 | **E2E** núcleo + **CI** gates |

### Fase 2 (quinzenal ou mensal)

| Incremento | Tarefas-alvo | Valor |
|------------|----------------|-------|
| **E1** | F2-01 | Livro + recebimentos + **painel** spec-completo (**ADR-0001** fechado) |
| **E2** | F2-07, F2-02 | **Segmento** + CC/espelho (**reordenação** para relatórios) |
| **E3** | F2-03, F2-04 | Relatórios **receita-first** + consolidados **FR-054** |
| **E4** | F2-05, F2-06 | Importação extrato + **IA** conciliação |
| **E5** | F2-08 | **FR-038** + **SC-013** |
| **E6** | F2-09, F2-10 | Transcrição + **E2E** estendido (**SC-013** smoke) |

---

## Mapeamento spec → fase

| Área | Fase 1 | Fase 2 | Fase 3 |
|------|--------|--------|--------|
| **US1** | ✓ | — | — |
| **US2** | ✓ (agente WhatsApp) | — | — |
| **US3** | ✓ (sem transcrição) | Transcrição **FR-068** | — |
| **US4** | Painel + itens **FR-023** + **FR-035** convênio/preços + **FR-022** cobrança **UI**; **sem livro** (**ADR-0001**) | ✓ livro + importação extrato + segmento + **FR-038** | **FR-041** |
| **US5** | Suporte humano | — | Agente suporte |
| **US6** | — | — | ✓ |
| **US7** | — | — | Follow-up |
| **US8** | — | Consolidados + **Segmento** | — |
| **US9** | — | — | ✓ |
| **§6.10** | **FR-067** mínimo | Retenção áudio (**FR-068**) com transcrição | **SC-019**/**SC-020**, **FR-069**–**FR-071** |
| **§6.11** | Funil **F1-17** + **F1-18** (ver [estratégia de cobertura](#coverage-by-phase)) | **F2-10** | **F3-08** + **F3-10** matriz **SC** |

---

## Próximos passos (Speckit)

1. **`tasks.md`** criado — manter **F1-xx**, **F2-xx**, **F3-xx** e a **matriz SC × suite × fase** ([cobertura](#coverage-by-phase)) atualizados em cada sprint.  
2. Manter **`research.md`** / **`data-model.md`** coerentes com a **divisão em três fases**.  
3. Implementar **`billingLedgerWritesEnabled`** conforme [**ADR-0001**](./adr/0001-fr022-billing-shell-phase1.md); rever ADR apenas se a spec ou o risco de produto mudarem.

---

<a id="cursor-plans-cancelled"></a>

## Planos auxiliares Cursor (ficheiros `.plan`) — cancelados

Explorações no **Cursor** que geram ficheiros `*.plan` (por vezes só no *workspace storage* do IDE, **não** versionados). As entradas abaixo foram **criadas, o trabalho associado foi revertido** e o estado **não** deve ser reaberto como pendente do Speckit — registo de **cancelado** para alinhamento entre agentes e humanos.

| Ficheiro / identificador | Estado |
|--------------------------|--------|
| `jornada_clínica_cta_d15cc9ea.plan` | **Cancelado** (2026-04-29) |
| `finalizar_episódio_atrasado_3b169149.plan` | **Cancelado** (2026-04-29) |
| `signedat_+_episódio_avulso_ui_5921f56b.plan` | **Cancelado** (2026-04-29) |
| `assinatura_icp_+_cabeçalho_16318f63.plan` | **Cancelado** (2026-04-29) |
