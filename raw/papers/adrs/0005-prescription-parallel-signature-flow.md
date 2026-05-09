<!-- wiki-ingest: synced from clinicagestor/docs/adr/0005-prescription-parallel-signature-flow.md at 2026-05-09T17:47:18.997Z — edit upstream in monorepo, not here -->

# ADR-0005 — Prescrição médica (FR-031, fase 1): assinatura digital como fluxo paralelo ao do episódio

## Contexto

- **FR-031** exige suporte a **prescrição médica** ligada ao episódio, com **catálogo de medicamentos**, **posologia editável**, **modelos** de documento personalizáveis e **impressão / PDF** para entrega ao paciente. Em fase 1 entregamos: receituário **SIMPLES** + **CONTROLE_ESPECIAL_BRANCA** (Portaria 344/98 lista C1, C4, C5, retinóides, antiretrovirais), com `DRAFT` → `LOCKED` lifecycle e PDF gerado on-demand.
- O **ADR-0003** definiu a separação **encerramento no sistema** (`signedAt`) vs **assinatura legal** (PAdES + metadata). O **ADR-0004** detalhou o fluxo PAdES local: client-side via Web PKI SDK, server-side via `preparePadesSignature` + `finalizePadesSignature`, com helper `applyLacunaSignatureCompletion` a aplicar tudo atomicamente sobre `ConsultationEpisode`.
- Para FR-031, **a prescrição também precisa de assinatura ICP-Brasil** (CFM Resolução 1.821/2007 + 2.299/2021): cada receituário individual é um documento legal autónomo, distinto do roteiro do episódio. Existem três cenários reais:
  1. Médico assina **só o episódio** (visita simples sem prescrição) → fluxo ADR-0004 inalterado.
  2. Médico assina **uma prescrição em particular** (e.g. paciente regressa para 2.ª via, ou episódio já encerrado mas precisa de receita controlada hoje) → precisa de assinar **só esse PDF**, sem reabrir o episódio.
  3. Médico assina **o episódio** com prescrições em rascunho → todas as prescrições `DRAFT` desse episódio devem **acompanhar** o lock (cascata), porque assinar o episódio implica que o médico se responsabiliza pelo conteúdo prescrito.

A questão arquitectural: **estendemos o fluxo Lacuna existente** (`/episodes/[episodeId]/legal-signature/lacuna/{prepare,finalize}`) para também aceitar `prescriptionId`, ou criamos **um par paralelo** dedicado a prescrição?

## Decisão

Criar um **par paralelo** de rotas, helper, modal e máquina de estados, **sem refactor** do fluxo de episódio. Concretamente:

1. **Rotas API novas, dedicadas a prescrição:**
   - `POST /api/consultations/[id]/prescription/[prescriptionId]/legal-signature/lacuna/prepare`
   - `POST /api/consultations/[id]/prescription/[prescriptionId]/legal-signature/lacuna/finalize`

   Estas rotas reutilizam **internamente** as primitivas de baixo nível `preparePadesSignature` / `finalizePadesSignature` (`lib/clinical/legal-signature/pades-local.ts`) — i.e. o **motor PAdES** é partilhado, apenas a **orquestração** é nova. A diferença é o que cada par faz a seguir:
   - O par do episódio chama `applyLacunaSignatureCompletion` → escreve em `ConsultationEpisode.legalSignatureStatus`/`signedAt` e cria anexo com `source = LEGAL_SIGNATURE`.
   - O par da prescrição chama `applyPrescriptionLacunaSignatureCompletion` (novo helper) → escreve em `Prescription.status = LOCKED` + `lockReason = SIGNED_INDIVIDUALLY` + `legalSignatureStatus`, e cria anexo com `source = PRESCRIPTION_LOCKED`.

2. **Permissão dedicada:** `prescription.sign` (não reutiliza `agenda.edit`). A separação permite à clínica delegar **assinatura de prescrição** sem dar permissão para reabrir / editar episódio. `prescription.write` (criar/editar/finalizar `DRAFT`) é distinta de `prescription.sign` (operação criptográfica que produz prova legal).

3. **Modal cliente paralelo:** `<ConsultationPrescriptionLegalSignatureModal>` é um novo componente, irmão de `<ConsultationEpisodeLegalSignatureModal>`. Aceita `prescriptionId` via `dispatchOpenPrescriptionLegalSignatureModal({ prescriptionId })`. Tem **a mesma máquina de estados visual** (intro → pki_loading → pki_certificates → pki_signing → completed/error) mas mira nos endpoints de prescrição. **Não** é uma extensão do modal do episódio.

4. **Cascata em três pontos quando o episódio é assinado:** novo helper `lockEpisodeDraftPrescriptions` (`lib/clinical/lock-prescription.ts`) percorre todas as prescrições `DRAFT` do episódio e aplica `LOCKED` + `lockReason = EPISODE_SIGNED` + cria anexo PDF. Pontos de invocação:
   - `POST /episodes/[episodeId]/legal-signature/lacuna/finalize` — após `applyLacunaSignatureCompletion` retornar `appliedLock=true`.
   - `finalizeConsultationEpisodeAction` (`app/dashboard/consultations/actions.ts`) — quando `shouldLockNow=true` (clínicas sem fornecedor de assinatura legal activo).
   - `signConsultationEpisodeAction` — fluxo manual de sign-only (sem PAdES).

   Wrapper `cascadeLockEpisodePrescriptionsSafe` envolve a cascata em try/catch que **loga mas nunca propaga**. Decisão consciente: **um episódio assinado sem cascata** é estado recuperável (cascata pode ser reexecutada manualmente no futuro), mas **rollback de uma assinatura legal já gravada** é arquitecturalmente catastrófico (perde-se a prova ICP-Brasil já criada com chave do médico que pode estar em token offline).

5. **Imutabilidade pós-`LOCKED`:** as 7 server actions de prescrição (`createPrescriptionDraftAction`, `addPrescriptionItemAction`, `updatePrescriptionItemAction`, `removePrescriptionItemAction`, `updatePrescriptionHeaderAction`, `lockPrescriptionAction`, `deletePrescriptionDraftAction`) rejeitam mutações em `LOCKED` redirecionando com `?err=prescription_locked`. PDF da prescrição `LOCKED` continua a ser gerado on-demand para coerência visual, mas **o PDF assinado/anexado** (gerado no momento do lock e armazenado no object storage como `ConsultationEpisodeAttachment` com `source=PRESCRIPTION_LOCKED`) fica **imutável** — esse é o documento legalmente vinculativo.

## Alternativas consideradas

### Alternativa A — Estender o par existente do episódio com `prescriptionId` opcional

**Descartada.** Implicaria que `applyLacunaSignatureCompletion` passasse a fazer um `if (prescriptionId) { ... } else { ... }`, branching que rapidamente cresce: o que acontece a `signedAt` quando se assina só uma prescrição mas o episódio nem sequer existe (cenário de prescrição standalone, fora do escopo desta fase mas planeado)? E quando o episódio está `LOCKED` e o médico precisa de assinar uma 2.ª via avulsa de uma prescrição já existente? Cada cenário introduz um ramo no helper que é **suposto** ser atómico e simples.

Pior: o fluxo de episódio é o **único** caminho que funciona em produção há semanas para vários consultórios. Refactor implica risco de regressão na operação core (encerramento do prontuário). O ganho — partilha de ~40 linhas de código duplicado entre dois helpers que de resto fazem coisas distintas — não compensa.

### Alternativa B — Modal único com seletor "tipo de documento" (episódio vs prescrição)

**Descartada.** O modal já tem máquina de estados não-trivial (intro → pki_loading → pki_not_installed → pki_certificates → pki_signing → completed → error) com transições assíncronas que envolvem o SDK Web PKI e duas chamadas HTTP. Adicionar uma dimensão "qual documento" multiplica os caminhos a testar e os modos de falha (e.g. utilizador troca o tipo a meio do fluxo após já ter o certificado seleccionado).

A duplicação visual entre os dois modais é absorvível — ambos partilham primitivas (`Button`, `LoadingBanner`, `friendlyErrorFromCode`) e a lógica do SDK Web PKI fica num pequeno helper interno (a especificidade está só no endpoint chamado e no shape do `complete`-state).

### Alternativa C — Assinar prescrição sempre como cascata do episódio (sem assinatura individual)

**Descartada.** Resolveria a duplicação eliminando o cenário 2 (assinar só uma prescrição). Mas viola FR-031 e a prática clínica real: receitas controladas (branca C1, e na fase 2 amarela / azul) costumam ser feitas **fora do contexto de visita** (renovação por telemedicina, 2.ª via para paciente que perdeu a receita, prescrição encadeada após exame externo). Forçar reabertura do episódio para emitir uma receita autónoma seria UX e legalmente incoerente.

### Alternativa D — Adoptar fornecedor SaaS de e-prescription (Memed, NexoData, RxPRO)

**Descartada.** Reproduz a análise do ADR-0004 sobre VIDAAS / Clicksign: introduz custo recorrente, dependência externa para fluxo crítico do prontuário, e — crucialmente — **descola a posse do certificado** do médico para a plataforma. Perde-se a equivalência directa com PAdES qualificado no certificado pessoal do médico (Memed, por exemplo, assina em nome da plataforma com selo organizacional). Para clínicas que precisam de defensibilidade jurídica em controlado especial, queremos o certificado do próprio médico no PDF.

## Justificação para aceitar a duplicação controlada

A "duplicação" entre os dois pares (prepare/finalize, helper de completion, modal cliente) é **contida**, **rastreável** e **invariante**:

- **Contida:** a partilha está no nível certo — primitivas PAdES (`pades-local.ts`), ASN.1 (`cms-external-signature.ts`), formatação de erro (`friendlyErrorFromCode`), validação de certificado (`isCertificateUsable`). O que difere é a **orquestração de domínio**, que é exactamente o que **deve** divergir entre dois agregados (Episode vs Prescription).
- **Rastreável:** cada par tem nome explícito (`/episodes/.../legal-signature` vs `/prescription/.../legal-signature`), helper explícito (`applyLacunaSignatureCompletion` vs `applyPrescriptionLacunaSignatureCompletion`), modal explícito. Grep por "prescription" mostra todo o fluxo isolado.
- **Invariante:** quando a fase 2 trouxer mais tipos de receituário (AMARELA A1/A2/A3, AZUL B1/B2, ANTIMICROBIANOS), a evolução acontece **só** no helper / modal de prescrição, sem tocar no fluxo do episódio. Quando a fase 3 trouxer assinatura de **atestados** ou **declarações**, será um terceiro par paralelo com a mesma estrutura (third instance permite finalmente refactor para genérico via `legal-signature-target` polymorphism, se justificável).

Esta é a aplicação do princípio **"rule of three"**: aguardar o terceiro caso antes de generalizar. Generalizar prematuramente entre dois casos constrói uma abstracção contra um único exemplo de variação, quase sempre incorrecta para o terceiro.

## Cascata de lock: invariantes operacionais

A cascata `lockEpisodeDraftPrescriptions` tem três propriedades não negociáveis:

1. **Best-effort, nunca bloqueante.** Falha em prescrição individual produz `console.error` mas a função continua para as restantes e nunca relança. O endpoint principal (assinatura do episódio) recebe sempre sucesso se a parte do episódio teve sucesso.
2. **Idempotente.** Reexecutar a cascata sobre prescrições já `LOCKED` é no-op silencioso (o helper filtra por `status: "DRAFT"` no início). Recuperação manual sempre possível.
3. **Auditada como evento separado.** Emite `PRESCRIPTION_CASCADE_LOCKED_ON_EPISODE_SIGNED` com `metadata.lockedCount`, `metadata.attachmentIds[]`, `metadata.source` (`lacuna_finalize` | `finalize_episode` | `sign_episode`). Permite auditoria distinguir lock individual de lock por cascata, e reconstruir o "porquê" de cada `lockReason = EPISODE_SIGNED`.

## Segurança e conformidade

- **Equivalência total ao ADR-0004.** A prescrição usa exactamente o mesmo motor PAdES-B-B local: `preparePadesSignature` + `finalizePadesSignature` + `node-forge` para CMS. A única diferença é o conteúdo do PDF (template de receita vs roteiro do episódio) e o que se actualiza no fim. Logo, **mesma validade legal ICP-Brasil** que o episódio.
- **Snapshots de medicamento.** `PrescriptionItem.nameSnapshot` / `dosageSnapshot` / `presentationSnapshot` são **gravados** no momento da criação da linha. Renomes ou inactivações posteriores no `MedicationCatalog` não corrompem a prescrição já assinada — o PDF re-renderizado a partir da BD continua a mostrar exactamente o que o médico assinou. FK do item ao catálogo é `Restrict` para impedir delete acidental do medicamento que ainda tem prescrições activas, e os snapshots cobrem o caso de inactivação (soft-delete).
- **Posologia híbrida.** Campo canónico para o PDF é `posologyText` (texto livre, sai literalmente). Campo opcional `posologyStructured` (JSONB com `{dose, doseUnit, route, frequency, duration, instructions}`) preserva a estrutura quando o médico usa o "modo guiado" do modal. Helper `lib/clinical/posology-format.ts` deriva `posologyText` do `posologyStructured` na hora da gravação. Esta separação permite (a) imprimir sempre uma string limpa, (b) habilitar futuras integrações (e-prescription, alertas DDI, exportação para SUS) sem migração destrutiva dos dados livres.
- **Receituário branco com duas vias.** Quando `type = CONTROLE_ESPECIAL_BRANCA`, o PDF gera **duas páginas idênticas** (paciente + farmácia, rótulo no rodapé), conforme exigência da Portaria 344/98 art. 56 § 4º. Implementado em `prescription-pdf.ts` ao invocar `generatePrescriptionPage` duas vezes na mesma `pdf-lib` instance.

## Consequências operacionais

- **Onboarding inalterado.** O médico já familiarizado com o fluxo Web PKI do episódio reconhece imediatamente o modal de prescrição (mesma UX, só muda o título). Primeira utilização ainda exige extensão Web PKI (link no modal); subsequentes silenciosas (A1) ou com PIN (A3).
- **Modal de prescrição auto-aberto via deep link.** Dois query params principais: `?openPrescriptions=1` (lista/chooser), `?openPrescription=<id>` (edit — vindo de `createPrescriptionDraftAction` ou, após lock manual, de `lockPrescriptionAction`). O PDF não abre automaticamente; o utilizador usa «Abrir PDF» no modal. Permite redirecionamento server-side limpo após server actions, sem prop-drilling de estado.
- **Item legacy do left rail removido.** O href "Prescrições → #roteiro-condutas" foi removido de `episodeWorkflowSteps` (`lib/dashboard/clinical-left-rail.ts`). Substituído por botão event-driven (`CLINICAGESTOR_OPEN_PRESCRIPTIONS_EVENT`) no bloco "Fluxo do episódio", paralelo ao botão "Exames e procedimentos" introduzido no chat `8f874d35`. Fluxo coerente: tudo que vive dentro do episódio abre como modal sobre o painel da consulta, não como navegação para anchor.
- **Sem custo recorrente novo.** Reusa todas as primitivas do ADR-0004 (web-pki SDK gratuito, node-forge, @signpdf, pdf-lib).
- **Testes:** smoke test PAdES (`scripts/smoke-pades-local.ts`) já cobre o motor partilhado. Testes específicos de prescrição (cascata + completion atomicidade) ficam pendentes até o runner vitest estabilizar (questão pré-existente do ambiente — ver ADR-0004).

## Status

**Aceite** — implementação presente nos commits que acompanham este ADR (chat `5dddc00f`). Ficheiros novos:

- `app/api/consultations/[id]/prescription/[prescriptionId]/legal-signature/lacuna/prepare/route.ts`
- `app/api/consultations/[id]/prescription/[prescriptionId]/legal-signature/lacuna/finalize/route.ts`
- `lib/clinical/legal-signature/apply-prescription-lacuna-completion.ts`
- `components/consultation-prescription-legal-signature-modal.tsx`
- `lib/clinical/lock-prescription.ts` (helpers `lockPrescriptionAndAttachPdf`, `lockEpisodeDraftPrescriptions`)
- `lib/clinical/prescription-pdf.ts`

Ficheiros modificados para integração:

- `app/dashboard/consultations/actions.ts` — invoca `cascadeLockEpisodePrescriptionsSafe` em finalize/sign manuais.
- `app/api/consultations/[id]/episodes/[episodeId]/legal-signature/lacuna/finalize/route.ts` — invoca cascata após `appliedLock=true`.
- `app/dashboard/(clinical-workspace)/consultations/[id]/page.tsx` — fetch de prescrições do episódio + monta modais + checks RBAC.
- `lib/dashboard/clinical-left-rail.ts`, `components/agenda-sidebar-column.tsx`, `components/clinical-workspace-shell.tsx` — botão event-driven de prescrições no left rail.

Rollback é viável endpoint-a-endpoint (cada par é independente do par do episódio), mas não está planeado.
