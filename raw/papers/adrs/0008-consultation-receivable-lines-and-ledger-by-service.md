<!-- wiki-ingest: synced from clinicagestor/docs/adr/0008-consultation-receivable-lines-and-ledger-by-service.md at 2026-05-09T17:45:44.518Z — edit upstream in monorepo, not here -->

# ADR-0008 — Contas a receber (`ConsultationReceivableLine`) e livro por serviço

## Estado

Aceite.

## Contexto

O livro (`LedgerEntry`) concentrava recebimentos na marcação sem detalhe por linha de serviço; adiantamentos geravam linhas de livro imediatas, misturando «caixa» e «análise por serviço».

## Decisão

1. **`ConsultationReceivableLine`** — sub-ledger de caixa por marcação: uma linha por movimento real (à vista ou cada parcela), incluindo **adiantamentos** sem escrituração analítica obrigatória no livro na mesma operação.
2. **`ADVANCE`** — grava-se **apenas** em `ConsultationReceivableLine` com `pendingConsultationLedgerSettlement`, **sem** criar `LedgerEntry`.
3. **`PAYMENT_OTHER`** — (*regra vigente nos aditamentos mais recentes; não dupla escritura em cada pagamento parcial*) ver §14–15 em baixo; historicamente esta linha descrevia CRL + livro proporcional por pesos **em cada** recebimento — **substituído** por CRL até quitação e livro só na **quitação integral**.
4. **Quitação com adiantamento prévio** — quando o total recebido cobre o planeado e já existia CRL `ADVANCE`: uma CRL para o caixa deste POST e **`LedgerEntry` por item** ao valor **planeado** de cada linha (não proporcional ao último pagamento); invariante de somas na transacção; limpeza do flag `pendingConsultationLedgerSettlement` nas linhas `ADVANCE`.
5. **`LedgerEntry`** — campos opcionais `consultationItemId`, `ledgerCompanyServiceId`, `ledgerServiceCodeSnapshot`, `ledgerServiceNameSnapshot` para relatórios e pivot por serviço.
6. **Relatório pivot** — adiantamentos do período agregam **CRL** `ADVANCE` (novo) e mantêm **linhas `LedgerEntry` `ADVANCE` legadas**; serviço no livro: prioridade snapshot / item / serviço da marcação.
7. **Livro contábil (UI)** — filtros por querystring, ordenação por coluna, `narrative` em linha única com tooltip; listagem **Parcelas futuras** para vencimentos `>=` hoje.
8. **Recebimento** — `PAID` automático quando `advancePaidMinor` atinge o total planeado dos itens.

### Aditamento (2026-04-28) — conciliação operacional e relatório «Contas a receber»

9. **`installmentReconciledAt`** na CRL usa-se também para movimentos **sem** número de parcela (à vista): data civil de conciliação no fuso do contratante.
10. **À vista (`AT_SIGHT`)** — ao gravar o recebimento, preencher automaticamente «conciliado» (`installmentReconciledAt` no início do dia civil); **parcelado (`INSTALLMENTS`)** — cada linha CRL nasce **em aberto** até «Marcar pago» na UI (extrato / convênio).
11. **Relatório Financeiro «Contas a receber»** — lista **todas** as linhas CRL no âmbito financeiro (não só parcelas); o relatório dashboard **«Recebimento»** mantém-se **agregado por consulta**, com **valor real** = soma das CRL (`amountMinor`), coluna **CRL (resumo)** e painel/hover (`GET …/receivable-lines`) a partir de CRL — **não** usar o livro como fonte desses campos.
12. **Exposição na UI/API** — coluna **Serviço** no relatório Livro contábil usa `ledgerServiceCodeSnapshot` / `ledgerServiceNameSnapshot` (e `consultationItemId` / `ledgerCompanyServiceId` na base); painel marcação lista CRL (`receivable-lines`), não o livro — ver §16.
13. **Código CRL (Contas a receber)** — `companyReceivableLineSequence`: sequencial **por empresa** (`companyId`), no **mesmo âmbito** que `LedgerEntry.companyLedgerSequence` (um contador para o livro, outro para a CRL; visualização idêntica: `formatCompanyLedgerSequence`, sem prefixo).

### Aditamento (2026-04-27) — livro só na quitação integral; painel marcação mostra CLR

14. **`PAYMENT_OTHER` parcial** — enquanto após o POST `advancePaidMinor < Σ plannedAmountMinor`: criar **apenas** linhas **`ConsultationReceivableLine`** para o montante recebido (à vista ou cada parcela deste POST); **não** criar **`LedgerEntry`** para esse incremento.
15. **Quitação integral neste POST** (`prevAdvance < Σ planeado` e `nextAdvance ≥ Σ planeado`, com itens planeados): após registar CRL do incremento, criar **`LedgerEntry` por `ConsultationItem`** ao **`plannedAmountMinor`** (com snapshots de serviço / `consultationItemId`), limpar **`pendingConsultationLedgerSettlement`** nas CRL **ADVANCE** da marcação; invariante **`sum(LedgerEntry)`** no POST **=** `plannedTotalMinor`; soma **acumulada** CRL da marcação **≥** planeado (admite sobrepagamento).
16. **Painel «Cobrança e recibo» na marcação** — botão **«Valores recebidos»** lista **`GET /api/consultations/[id]/receivable-lines`** (não o livro). O relatório Financeiro → Recebimento mantém o mesmo contrato CRL no detalhe expandido.

### Aditamento (2026-04-27) — referência de grupo e estorno conjunto CRL + livro

17. **`receiptPostingGroupId`** (UUID) — presente em todas as linhas **`ConsultationReceivableLine`** e nas **`LedgerEntry`** `RECEIPT` criadas no **mesmo POST** de recebimento; permite estorno conjunto (caixa + receita por serviço na quitação integral). **Exibição:** primeiros 8 caracteres do UUID como «Ref. grupo» no quadro Valores recebidos.
18. **Estorno pela CRL** — `POST /api/consultations/[id]/receivable-lines/[lineId]/storno`: cria linhas CRL de contrapartida (`amountMinor` negativo, `reversesReceivableLineId`); se existir grupo, também **`ADJUSTMENT`** no livro para cada `RECEIPT` com o mesmo `receiptPostingGroupId`; ajusta **`advancePaidMinor`**, **`paymentStatus`**, reabre linhas de item quando deixa de estar **Pago**, repõe **`pendingConsultationLedgerSettlement`** nas CRL **ADVANCE** quando o total deixa de cobrir o planeado.
19. **Estorno só pelo livro** — para lançamentos `RECEIPT` com **`receiptPostingGroupId`**, o estorno directo em **`POST …/ledger-entries/[id]/storno`** é **recusado** (409); usar o fluxo «Valores recebidos» para manter CRL e livro alinhados.

## Consequências

- Migração `20260710120000_consultation_receivable_line_ledger_item`.
- Histórico com só `LedgerEntry` `ADVANCE` continua visível nos relatórios; novos adiantamentos não duplicam no livro.
- No **painel clínico da marcação**, **«Contas a receber (CLR)»** lista **`ConsultationReceivableLine`** (`receivable-lines`). No **relatório dashboard Recebimento**, o detalhe expandido/hover também usa **`receivable-lines`**. **`LedgerEntry`** neste fluxo aparece só na **quitação integral** (regra §14–15).
