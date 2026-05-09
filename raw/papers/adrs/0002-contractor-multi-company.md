<!-- wiki-ingest: synced from clinicagestor/docs/adr/0002-contractor-multi-company.md at 2026-05-09T17:47:18.994Z — edit upstream in monorepo, not here -->

# ADR-0002 — `Contractor` e empresas jurídicas múltiplas

## Contexto

Um contratante pode ter **várias empresas legais** (`Company`) e várias unidades (`Clinic`). Profissionais podem atender em clínicas de **empresas diferentes** desde que pertençam ao **mesmo** contratante. Sem um agrupamento explícito, o risco é misturar convites entre clientes distintos ou impedir legitimamente o multi-empresa.

## Decisão

- Introduzir o modelo **`Contractor`** (contratante / grupo comercial) com `id`, `name`, `code` (único global).
- Cada **`Company`** tem **`contractorId`** obrigatório (`Restrict` on delete).
- Regra de convite (`addStaffMemberAction`): se o email já existe, o utilizador só pode ser associado a uma nova clínica se **todas** as suas `ClinicMember` actuais forem de empresas com o **mesmo** `contractorId` que a clínica alvo; caso contrário, erro `user_other_contractor`. O **super-admin da plataforma** ignora esta verificação (manutenção / correção de dados).
- Pacientes e catálogo de serviços permanecem **por `Company`** (FR-046); partilha de pacientes entre empresas do mesmo contratante fica para evolução explícita (política de dados).

## Consequências

- Novo bootstrap de tenant (`/setup/tenant`): cria `Contractor` + primeira `Company` + `Clinic`.
- Migração: empresas existentes recebem um `Contractor` legado (`CTR-LEGACY`); o seed reatribui ao contratante demo quando aplicável.
- Códigos automáticos de contratante: prefixo **`T`** + sufixo numérico (`allocateContractorAutocode`), alinhado a `E`/`U`/… existentes.

## UI — nova empresa no mesmo grupo

Quem tem **`tenant.structure.manage`** em **`/dashboard/settings/company`** pode submeter **Nova empresa jurídica (mesmo contratante)** (`createCompanyUnderContractorAction`): cria `Company` com o mesmo `contractorId`, primeira `Clinic`, segmento «Geral», CL «Principal», RBAC e `ClinicMember` do actor como gestor; depois `?fresh=1` recarrega a lista de clínicas na sessão.

## Status

Aceite e implementado no repositório (migração `20260420120000_contractor_and_company_fk` + seed multi-empresa + acção acima).
