<!-- wiki-ingest: synced from clinicagestor/docs/adr/0006-episode-document-library-aggregated-view.md at 2026-05-09T18:04:15.346Z — edit upstream in monorepo, not here -->

# ADR-0006 — «Imagens e anexos do episódio» como biblioteca agregada de documentos clínicos

## Contexto

Ao longo de FR-031 (prescrições) e FR-032 (anexos manuais) e dos trilhos de assinatura legal (ADR-0003, ADR-0004, ADR-0005), o painel da consulta acumulou **5 superfícies independentes de documentos** que não conversam entre si do ponto de vista do utilizador:

| Categoria | Storage físico | UI de descoberta |
|---|---|---|
| Anexos manuais (foto, PDF, scan) | `ConsultationEpisodeAttachment` `source = USER_UPLOAD` | modal «Imagens e anexos» + bloco no painel |
| Episódio assinado digitalmente (Lacuna) | `ConsultationEpisodeAttachment` `source = LEGAL_SIGNATURE_LACUNA` | modal de anexos + selo na timeline + modal de assinatura |
| Prescrição finalizada (LOCKED) | `ConsultationEpisodeAttachment` `source = PRESCRIPTION_LOCKED` | modal de prescrições + modal de anexos |
| Pedido externo de exames TUSS | `ConsultationExternalProcedureRequest` (PDF on-demand, **sem anexo**) | lista própria na timeline + modal «Exames e procedimentos» |
| Roteiro do episódio (PDF) | gerado on-demand de `ConsultationEpisode`/`Revision` (**sem anexo**) | link no painel esquerdo |
| Prescrição em rascunho (DRAFT) | gerado on-demand de `Prescription` (**sem anexo**) | modal de prescrições |

Resultado para o utilizador final: **fragmentação de descoberta**. Um médico que abre a página de uma consulta e clica em «Imagens e anexos» vê só uma fatia (uploads + assinaturas + receitas finalizadas), e tem de se lembrar de abrir mais 3 modais distintos para confirmar se há prescrição em rascunho, pedido externo emitido, ou se o roteiro foi gerado em PDF. Um auditor a rever um episódio histórico precisa de andar entre cartões da timeline sem garantia de ter visto tudo.

A questão arquitectural é: **devemos centralizar o storage** (materializar pedidos externos, roteiro e prescrições DRAFT como `ConsultationEpisodeAttachment` para terem todas as fontes na mesma tabela) **ou centralizar apenas a vista** (manter o storage federado e construir uma vista agregada de leitura)?

## Decisão

**Centralizar a vista, não o storage.** O modal hoje chamado «Imagens e anexos do episódio» passa a ser a **biblioteca completa de documentos** do episódio, agregando 4 fontes distintas numa lista tipada por `kind`. O título do modal mantém-se («Imagens e anexos do episódio»).

Concretamente:

1. **Helper único de leitura** `lib/clinical/episode-document-library.ts` exporta `loadEpisodeDocumentLibrary(prisma, episodeId, consultationId)` que executa **4 queries em paralelo** e devolve `EpisodeDocumentRow[]` (união discriminada por `kind`):
   - `kind: "roteiro"` — entrada virtual sempre presente quando há `ConsultationEpisode`; aponta para a rota `/api/consultations/[id]/episodes/[episodeId]/roteiro-pdf`.
   - `kind: "prescription"` — uma entrada por `Prescription` do episódio (DRAFT ou LOCKED); aponta para a rota de PDF on-demand.
   - `kind: "external_procedure_request"` — uma entrada por `ConsultationExternalProcedureRequest`; aponta para a rota de PDF do pedido.
   - `kind: "attachment"` — entradas de `ConsultationEpisodeAttachment` filtradas por `source`. Os attachments com `source = PRESCRIPTION_LOCKED` / `PRESCRIPTION_PREVIEW` **são deliberadamente excluídos** desta categoria para evitar duplicação visual: o registo `Prescription` correspondente já é a entrada canónica e o PDF on-demand serve o mesmo blob lógico (com a vantagem de respeitar evolução do template).

2. **O modal `<ConsultationEpisodeAttachmentsModal>`** é redesenhado em **5 secções verticais** com headers iconográficos:
   - **Roteiro do episódio** (1 linha máximo)
   - **Prescrições médicas** (lista) — DRAFT mostra badge âmbar «Rascunho»; LOCKED mostra «Finalizada» (+ «Assinada (ICP)» se houver assinatura digital).
   - **Pedidos externos de exames** (lista) — cancelados marcados em vermelho.
   - **Assinatura digital do episódio** (lista) — entradas `LEGAL_SIGNATURE_*` em verde.
   - **Anexos manuais** (lista) — única secção **editável** (upload, comentários, delete).

   Cada item tem dois botões padronizados: **«Abrir»** (PDF/imagem em nova aba) e **«Ir para origem»** (atalha o modal especializado correspondente para criar/editar). A acção «Ir para origem» é implementada via:
   - Prescrições → deep-link `?openPrescription=<id>` + dispatch `CLINICAGESTOR_OPEN_PRESCRIPTIONS_EVENT` (consumido pelo modal de prescrições).
   - Pedidos de exames → dispatch `CLINICAGESTOR_OPEN_EXAMS_PROCEDURES_EVENT`.
   - Assinatura digital → `dispatchOpenLegalSignatureModal()`.

3. **Timeline expandida inline** (decisão `expandir_inline`): cada cartão de episódio histórico passa a renderizar `<TimelineEpisodeDocumentLibraryInline>` — um resumo grid 2-col read-only com TODOS os documentos do episódio agrupados por categoria. Substitui a antiga lista isolada `<ConsultationExternalProcedureRequestsList>` que só mostrava pedidos externos. O botão «Documentos do episódio» (renomeado de «Anexos do episódio» — antes `TimelineEpisodeAttachmentsButton`) no canto do cartão continua a abrir o modal central.

4. **Endurecimento da eliminação:** `deleteConsultationEpisodeAttachmentAction` passa a rejeitar **também** `PRESCRIPTION_LOCKED` e `PRESCRIPTION_PREVIEW`, alinhando com a protecção já existente para `LEGAL_SIGNATURE_*`. Justificação: um PDF de receita finalizada faz parte do prontuário legal — para o remover, o médico tem de eliminar a `Prescription` na origem (server action `deletePrescriptionAction`), e a cascata de constraints do Prisma trata do attachment.

5. **Sem migração de schema.** Nenhuma tabela é alterada. O enum `ConsultationEpisodeAttachmentSource` mantém todos os valores actuais — incluindo `LEGAL_SIGNATURE_VIDAAS` (placeholder para integração VIDAAS API directa) e `PRESCRIPTION_PREVIEW` (sem caller actual) — documentados como **tech-debt assumida** porque remover um valor de enum Postgres exige `ALTER TYPE ... RENAME` + recriação, e o ganho operacional não justifica o risco em produção neste momento.

## Alternativas consideradas

### A — Materializar tudo em `ConsultationEpisodeAttachment`

Cada vez que um pedido externo for criado, gerar o PDF imediatamente, fazer upload e criar uma linha de attachment com novo `source = EXTERNAL_PROCEDURE_REQUEST`. Idem para roteiro (snapshot ao salvar revisão) e prescrição DRAFT (snapshot ao salvar). Resultado: a `ConsultationEpisodeAttachment` torna-se a única tabela canónica de documentos.

**Rejeitada** porque:
- **Custo de storage e regeneração:** PDFs de prescrição DRAFT mudam a cada edição — teríamos de re-uploadar a cada `updatePrescriptionItemAction`, criando dezenas de blobs órfãos por sessão. Idem roteiro: cada save cria nova revisão.
- **Perda de coerência visual:** quando o template do PDF (header da clínica, layout) for actualizado, o blob velho continua estagnado. PDFs gerados on-demand respeitam sempre a versão mais recente do template.
- **Migração pesada de dados existentes:** todo o histórico teria de ser reprocessado.
- **Duplicação de fonte de verdade:** os dados estruturados (`PrescriptionItem`, `ExternalProcedureRequestItem`) ficam desligados do PDF que os representa, complicando integrações futuras (e.g. envio de pedido por API ao laboratório).

### B — Manter status quo (sem agregação)

Aceitar que o utilizador navegue entre 3-4 modais distintos. **Rejeitada** porque é exactamente o problema que motivou esta ADR: a fragmentação confunde médicos e auditores e cria sensação de que documentos podem ter sido perdidos.

### C — Renomear o modal para «Documentos do episódio»

Mais alinhado com o conteúdo expandido. **Rejeitada** pelo utilizador (decisão `manter`) por continuidade visual: o termo «Imagens e anexos» já está estabelecido no fluxo e a mudança de label introduziria atrito de descoberta sem ganho funcional. O **subtítulo** do modal foi ajustado para «todos os documentos deste episódio» para sinalizar a expansão do escopo.

## Consequências

### Positivas

- **Descoberta única:** o utilizador tem garantia de que o modal «Imagens e anexos» (e a secção inline da timeline) mostra **tudo** o que o episódio produziu. Não há mais «será que esqueci de algum sítio?».
- **Atalhos contextuais preservados:** os botões «Prescrições», «Exames e procedimentos», «Assinar com Certificado» do left rail e left panel continuam a abrir os modais especializados — a centralização **não remove pontos de criação/edição**, apenas adiciona um ponto de leitura unificada.
- **Storage intocado:** zero migração, zero risco operacional. Schema actual é compatível.
- **Auditoria mais limpa:** a vista inline expandida na timeline reduz cliques para um auditor reconstruir a história clínica.

### Negativas / aceites

- **4 queries por episódio:** `loadEpisodeDocumentLibrary` faz 4 round-trips ao Postgres por episódio. Para a página da consulta com timeline grande (até 80 episódios), são até 320 queries adicionais. Mitigado por: (a) queries são `findMany` simples escopadas a `episodeId` com índices existentes, (b) execução em paralelo via `Promise.all` no nível do helper e do loop da page, (c) selecção mínima de campos. Se vier a tornar-se gargalo, considera-se uma única `$queryRaw` com `UNION ALL`.
- **Tech-debt enums zómbi:** `LEGAL_SIGNATURE_VIDAAS` e `PRESCRIPTION_PREVIEW` continuam no `ConsultationEpisodeAttachmentSource`. **Plano:** quando a integração VIDAAS API directa for retomada, `LEGAL_SIGNATURE_VIDAAS` ganha caller; se PRESCRIPTION_PREVIEW continuar sem uso 6 meses após esta ADR, criar migration de remoção.
- **Duplicação de UI entre modal e timeline-inline:** o componente `<TimelineEpisodeDocumentLibraryInline>` reimplementa parcialmente a renderização do modal (em layout mais compacto). Aceite porque os requisitos de densidade visual e interactividade são diferentes (timeline = read-only ultra-compacto; modal = scrollável com edição de uploads). Se divergirem mais, extrair primitivas partilhadas.

## Addendum (2026-04-26) — ICP: roteiro do episódio vs. documentos auxiliares

**Problema:** confundir **bloqueio do roteiro** (conteúdo clínico + PDF on-demand que o representa) com **possibilidade de assinatura ICP** de **outros** documentos do mesmo episódio (prescrição finalizada, pedido externo de exames, anexos manuais).

**Decisão de produto (actual):**

- **O que “bloqueia” com o encerramento/assinatura do episódio no sentido de roteiro** são o **conteúdo estruturado** do roteiro e a **geração coerente do PDF do roteiro** — regra alinhada a **imutabilidade pós-`signedAt` / pós-completar assinatura legal do episódio** nos fluxos já existentes do painel.
- **Não** se aplica a “esconder” o fluxo de assinatura ICP **por entidade** em **pedido externo**, **prescrição** (quando a regra da prescrição permitir) ou **anexo manual** só porque o episódio já foi encerrado ou o roteiro já está imutável: estes documentos têm **ciclo de assinatura próprio**; o utilizador pode concluir ou iniciar a assinatura noutro momento **depois** desse bloqueio, sujeito a `prescription.write` / `canPrescriptionSign` e a fornecedor ICP activo, conforme cada superfície.
- Nos diálogos **«Abrir documento»** a partir de **pedido externo** ou **anexo manual**, o botão **«Assinar com certificado»** (quando visível) continua a significar **somente a assinatura ICP do roteiro clínico do episódio** (Lacuna/VIDAAS), **não** o PDF do pedido externo nem o ficheiro do anexo — o texto de ajuda do popup explicita isso. O **layout** desses botões (linha com Cancelar + Abrir PDF + Reemitir; linha separada com Assinar) segue o padrão acordado na UI.

**Nota de fronteira:** a **imutabilidade** de PDFs já assinados (PAdES) e a **não remoção** de anexos com assinatura concluída permanecem requisitos de backend/contrato (FR-027 / política de anexos); este addendum restringe-se à **regra de gating e copy na biblioteca agregada**.

## Implementação

- **Novo:** `lib/clinical/episode-document-library.ts`, `components/timeline-episode-document-library-inline.tsx`.
- **Reescrito:** `components/consultation-episode-attachments-modal.tsx` (estrutura nova em 5 secções; mantém o evento global `CLINICAGESTOR_OPEN_EPISODE_ATTACHMENTS_EVENT` e o tipo `EpisodeAttachmentsOpenDetail`, mas o detail passa a ter `library: EpisodeDocumentRow[]` em vez de `attachments`).
- **Actualizado:** `components/timeline-episode-attachments-button.tsx` (recebe `library` em vez de `attachments`; label «Documentos do episódio»).
- **Actualizado:** `app/dashboard/(clinical-workspace)/consultations/[id]/page.tsx` (computa `episodeLibrariesByEpisodeId` via helper; passa `initialLibrary` ao Host; substitui lista isolada de pedidos externos por componente inline; remove import órfão de `ConsultationExternalProcedureRequestsList`).
- **Endurecido:** `deleteConsultationEpisodeAttachmentAction` em `app/dashboard/consultations/actions.ts` (rejeita `PRESCRIPTION_LOCKED` e `PRESCRIPTION_PREVIEW`).

## Referências

- ADR-0003 — Coexistência VIDAAS/sistema (separação encerramento vs assinatura).
- ADR-0004 — Lacuna Web PKI local (PAdES-B-B no episódio).
- ADR-0005 — Prescrição médica fase 1 + assinatura digital paralela.
- FR-031 — Prescrição médica.
- FR-032 — Anexos clínicos.
