<!-- wiki-ingest: synced from clinicagestor/docs/adr/0009-receivable-installment-operational-reconciliation.md at 2026-05-09T18:04:46.904Z — edit upstream in monorepo, not here -->

# ADR-0009 — Conciliação operacional por parcela (`installmentReconciledAt`)

## Estado

Aceite.

## Contexto

O relatório **Contas a receber** (parcelas de recebimento parcelado) precisa de um estado independente do pagamento da marcação: a consulta pode estar «paga» no sistema e uma parcela continuar «em aberto» até conferência com extrato ou convênio.

## Decisão

- Campo opcional **`ConsultationReceivableLine.installmentReconciledAt`** (`DateTime?`): **data civil** da conciliação no fuso do contratante (`startOf('day')` ao marcar como pago).
- Escrita apenas via acção servidor com **`finance.write`** e respectivo âmbito consolidado (`resolveConsolidatedFinanceReadScope`).
- **Reabrir:** `installmentReconciledAt := null`** com as mesmas permissões.
- **`AuditLog`** (`appendAuditEvent`): acções `finance.receivable_installment.reconcile` e `finance.receivable_installment.reopen` com `entityId` da linha CRL e metadados (`consultationId`, timestamps).
- Não substitui nem altera **`Consultation.paymentStatus`** nem fluxos do livro.

## Consequências

- Migração `20260428120000_crl_installment_reconciled_at`.
- UI em `/dashboard/finance/contas-a-receber`; hub Relatórios **accounts_receivable** aponta para esta rota.
- **Feedback de erro nas form actions** («Marcar pago» / «Reabrir»): em caso de falha (`{ ok: false }` da action servidor), redirect para o mesmo ecrã com query `crlReconcileErr=<mensagem>`; os filtros activos são preservados via campo oculto `returnQuery`. Sucesso faz redirect sem `crlReconcileErr` para limpar a URL após `revalidatePath`.
