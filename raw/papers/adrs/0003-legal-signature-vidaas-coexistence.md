<!-- wiki-ingest: synced from clinicagestor/docs/adr/0003-legal-signature-vidaas-coexistence.md at 2026-05-09T17:47:18.995Z — edit upstream in monorepo, not here -->

# ADR-0003 — Assinatura legal (VIDAAS) convivendo com encerramento no sistema

## Contexto

- O produto já possui **encerramento / bloqueio operacional** do episódio (`ConsultationEpisode.signedAt`, `signedByUserId`, `AuditLog`, imutabilidade de edição — alinhado a **FR-027** / **SC-006** no fluxo actual).
- Esse mecanismo **não** constitui, por si só, **assinatura digital qualificada** com valor probatório equivalente a **PAdES + certificado** via fornecedor (ex. **VIDAAS** / Valid).
- A plataforma deve suportar **vários fornecedores** de assinatura legal ao longo do tempo; a **primeira integração** será **VIDAAS** (QR Code + app VIDAA S), em ambiente controlado (sandbox → produção).
- **FR-033** exige não repúdio com meio aceite na jurisdição/política; **FR-032** prevê anexos por episódio; o modelo **`ConsultationEpisodeAttachment`** já possui **`episodeId`**.

## Definições (glossário obrigatório na UI e na documentação técnica)

| Termo | Significado | Persistência |
|--------|--------------|--------------|
| **Encerramento no sistema** | O profissional conclui o fluxo no PE; o episódio deixa de aceitar edição no aplicativo. | `signedAt`, `signedByUserId`, eventos de auditoria existentes. |
| **Assinatura legal (VIDAAS)** | Assinatura criptográfica do **documento** (PDF PAdES) pelo fornecedor; prova no ficheiro. | PDF como **anexo** do episódio + campos de estado/metadata do fornecedor (ver plano de dados). |

**Regra de integridade:** o utilizador **nunca** deve interpretar só `signedAt` como “assinado com certificado ICP/VIDAAS”. O valor legal da assinatura formal reside no **PDF assinado** + **metadados do fornecedor** + **evento de auditoria** específico.

## Decisão

1. **Manter** o fluxo actual de encerramento (`signConsultationEpisodeAction` / finalização com `signAfterFinalize`) **inalterado em semântica de bloqueio**; renomear **rótulos na UI** onde ainda sugiram “certificado digital” sem PDF assinado (evitar falsa equivalência legal).
2. **Acrescentar** um eixo separado **“assinatura legal”** com:
   - estados próprios (pendente / concluída / falha / expirada);
   - `provider` discriminado (`VIDAAS` na primeira fase);
   - conclusão materializada por **anexo** `ConsultationEpisodeAttachment` (mesma tabela e RBAC já previstos para anexos).
3. **Não** usar `signedAt` interno como substituto de `legalSignatureCompletedAt`.
4. **Fornecedores múltiplos:** interface de código (`LegalSignatureProvider` ou equivalente) e configuração por **Empresa/clínica** (feature flag + credenciais), começando por **VIDAAS** apenas.
5. **Auditoria:** novo tipo de evento (ex. `CONSULTATION_EPISODE_LEGAL_SIGNATURE_COMPLETED`) **separado** de `CONSULTATION_EPISODE_SIGNED`, com `metadata` incluindo `provider`, `transactionId`, `attachmentId` (idempotência e prova).

## Consequências

- Migração Prisma em **fase própria** (campos novos com defaults seguros; ver plano de implementação).
- Novos endpoints e segredos (VIDAAS) — **nunca** no cliente; filas ou handlers com idempotência.
- **UI:** botão/modal VIDAAS + popup “outros certificadores em breve”; timeline com selo e “Ver anexos” — sujeito a **aprovação de produto** onde existir comentário de *layout oficial* (regra do repositório).
- Testes: integração + E2E mínimos por fase; regressão **SC-006** (imutabilidade) obrigatória após cada alteração no domínio do episódio.

## Status

**Proposto** — implementação faseada conforme `docs/plan-implementacao-assinatura-legal-vidaas.md`. Nenhuma integração VIDAAS em produção até credenciais, DPA e documentação oficial da Valid estarem definidos no plano de rollout.
