<!-- wiki-ingest: synced from clinicagestor/docs/adr/0011-supplier-and-accounts-payable-lines.md at 2026-05-09T18:04:15.348Z — edit upstream in monorepo, not here -->

# ADR-0011 — Fornecedor e linhas operacionais de Contas a pagar

## Estado

Aceite (2026-04-30).

## Contexto

O relatório **Contas a pagar** não deve depender apenas do livro para movimentos operacionais de caixa (parcelas, conciliação bancária), tal como **Contas a receber** usa **CRL** em paralelo ao livro.

## Decisão

1. **Cadastro `Supplier`** por **`companyId`**, código interno automático com prefixo **`F`** e comprimento alinhado ao paciente (`allocateSupplierAutocode`).
2. **`LedgerEntry.supplierId`** em **`EXPENSE`** — obrigatório nas escrituras novas (validação em `createExpenseLedgerEntry`).
3. **`AccountsPayableLine`** — tabela operacional com sequência por empresa, `installmentReconciledAt`, estorno por par (`reversesAccountsPayableLineId`), ligação opcional **`ledgerEntryId`** ao livro quando o compromisso liquida uma despesa registada.
4. **Lançamento à vista** (popup de planeamento): gera **`EXPENSE`** + linha CAP **`PAYMENT` / `AT_SIGHT`** com conciliação na data civil do vencimento no fuso do contratante.
5. **Estorno:** `POST /api/finance/accounts-payable-lines/[lineId]/storno` — contrapartida na CAP + **`ADJUSTMENT`** no livro quando existia `ledgerEntryId`.

## Consequências

- Cadastro de fornecedores toca a superfície **Cadastros** (`masterdata.edit`).
- Migração: `20260430220000_supplier_accounts_payable_lines`.
