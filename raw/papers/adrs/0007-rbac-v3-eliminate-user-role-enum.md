<!-- wiki-ingest: synced from clinicagestor/docs/adr/0007-rbac-v3-eliminate-user-role-enum.md at 2026-05-09T17:55:22.749Z — edit upstream in monorepo, not here -->

# ADR-0007 — RBAC v3: eliminação do enum `UserRole` (OWNER/ADMIN/STAFF) a favor do eixo único `ClinicRole`

## Contexto

- A plataforma herdava dois eixos de perfil que coexistiam no mesmo `ClinicMember`:
  1. **Legacy** — enum `UserRole` com `OWNER`, `ADMIN`, `STAFF` (coluna `ClinicMember.role`, `NOT NULL`, default `STAFF`). Nasceu com o primeiro seed e nunca foi totalmente removido.
  2. **Moderno (RBAC v2 / v2.2)** — `ClinicRole` por clínica com quatro slugs sistema **`sistema`**, **`gestor`**, **`atendente`**, **`profissional`**, ligados a `Permission` via `ClinicRolePermission`, materializado pelo `bootstrapClinicRbac()` em cada clínica nova (ver ADR-0002 e snapshot `tests/rbac-bootstrap-snapshot.test.ts`).
- Consequências desta coexistência, observadas em produção e no seed:
  - **`lib/rbac/can.ts`** prioriza `ClinicRole` quando presente, mas cai num **fallback** (`legacyRoleAllows`) se `clinicRoleId` estiver nulo. O fallback mantinha `OWNER`/`ADMIN` com **todas** as permissões (incluindo `roteiro.*` e `prescription.*`), contrariando a regra de produto «conteúdo clínico é exclusivo de Profissional».
  - **`lib/finance/resolve-scope.ts`** concedia vista integral financeira a qualquer membro com `role` ∈ {`OWNER`,`ADMIN`}, sobrepondo-se ao `ClinicRole` efectivo.
  - O enum legacy aparecia em **UIs** (`app/dashboard/layout.tsx` exibia «Proprietário» / «Administrador» mesmo quando o papel real era Gestor), em **server actions** (setup/tenant, company, staff) como valor cosmético sem efeito RBAC real, e no **Excel template** com uma coluna `userRole*` obrigatória.
  - O bug concreto que motivou a revisão: um cronómetro de episódio corria há 53h sem poder ser parado; investigação revelou que o guard pedia `ClinicMember` com `clinicRoleId`, mas havia utilizadores seed em que apenas `role=OWNER` estava preenchido.
- Utilizador confirmou em chat `8f874d35` as três escolhas de design abaixo antes da implementação.

## Decisão

**Eliminar o enum `UserRole` e a coluna `ClinicMember.role` do modelo; tornar `ClinicMember.clinicRoleId` `NOT NULL`.** O único eixo de perfil passa a ser o `ClinicRole` da clínica.

Mapeamento de dados existentes (backfill da migração `20260420150000_clinic_member_drop_user_role`):

| `UserRole` legacy | novo `ClinicRole.slug` na mesma clínica |
|---|---|
| `OWNER` | `gestor` |
| `ADMIN` | `gestor` |
| `STAFF` (sem `clinicRoleId`) | `atendente` |
| (membros que já tinham `clinicRoleId`) | mantido |

Conceito operacional do **«dono da clínica»** pós-RBAC v3:

- «Dono» deixa de ser um papel no modelo de dados; passa a ser uma **propriedade operacional**: o utilizador é `ClinicMember` com papel **`gestor`** em **todas as clínicas da(s) empresa(s) que detém**. A `ClinicRole=gestor` concede `rbac.manage`, `tenant.structure.manage`, `finance.read` (vista integral), mas **não** concede `roteiro.*` nem `prescription.*` — preserva-se o sigilo clínico (regra RBAC v2.1).
- Se o dono também exercer clinicamente numa unidade, adiciona uma segunda `ClinicMember` com papel `profissional` nessa unidade (as permissões somam-se via escopo de clínica).

Consequências no código:

- `lib/rbac/can.ts` — removido o ramo `legacyRoleAllows`; ausência de `clinicRole` passa a ser **deny** explícito. `isPlatformMaintainerUserId()` continua a ser o único atalho para a conta de manutenção da plataforma.
- `lib/rbac/legacy-role-permissions.ts` e `tests/rbac-legacy.test.ts` — apagados.
- `lib/finance/resolve-scope.ts` — `memberHasFullFinanceView` passa a ser puramente `clinicRole.slug ∈ {sistema,gestor}`; `lib/reports/matrix-eligibility.ts` alinhado.
- `app/dashboard/layout.tsx` — `profileLabel` usa apenas `clinicRole?.name`. Caíram os ramos `Proprietário`/`Administrador`.
- Server actions (setup/tenant, company, staff) — deixam de escrever `role: UserRole.*`; criam/editam membros só via `clinicRoleId`.
- `prisma/seed.ts` — reordenado para correr `bootstrapClinicRbac()` **antes** dos `clinicMember.upsert`, garantindo `clinicRoleId` à criação (requisito do novo `NOT NULL`).
- `scripts/seed-excel/*` — coluna `userRole*` removida do template; `clinicRoleSlug` passa a obrigatório na importação.

Migração de esquema (`20260420150000_clinic_member_drop_user_role/migration.sql`):
1. Backfill `ClinicMember.clinicRoleId` por regra de conversão, tocando apenas linhas `NULL`.
2. `DO $$` de salvaguarda aborta se sobrar alguma linha sem `clinicRoleId` (sinal claro de que `bootstrapClinicRbac` não correu numa clínica — corrigir e repetir).
3. `ALTER TABLE ... SET NOT NULL` em `clinicRoleId`.
4. `DROP COLUMN "role"`.
5. `DROP TYPE "UserRole"`.

## Alternativas consideradas

### Alternativa A — Manter `UserRole` como atalho, reforçando `ClinicRole` (status quo RBAC v2.2)

**Descartada.** Precisávamos de escrever e manter regras em dois sítios (fallbacks `legacyRoleAllows`, overrides em `memberHasFullFinanceView`) para uma semântica sobreposta. O risco de divergência é o bug que nos trouxe aqui: um utilizador tem acesso de `OWNER` legacy mesmo sem ter `ClinicRole=gestor` nessa clínica, ou vice-versa.

### Alternativa B — Migrar `OWNER` para um papel-template novo com permissões clínicas também

**Descartada.** Violaria a matriz do **RBAC v2.1** (`roteiro.*` e `prescription.*` exclusivas de Sistema+Profissional; ver `tests/rbac-bootstrap-snapshot.test.ts`, invariantes). O dono que também atende clinicamente adiciona uma segunda `ClinicMember` como `profissional` na unidade onde atende — mais explícito, preserva sigilo e auditoria.

### Alternativa C — `ClinicMember.clinicRoleId` continuar `NULL`able com deny implícito

**Descartada.** Deixar o campo `NULL`able reabre exactamente o buraco que a v3 fecha (membros sem papel, dependentes de atalhos por `UserRole`). Tornar `NOT NULL` é o reforço de invariante que torna o modelo auto-documentado.

## Consequências

### Positivas
- Modelo único, alinhado com a matriz aprovada pelo utilizador (Print RBAC v2.2). `can()` fica trivial de auditar.
- Preservação do sigilo clínico: dono global pode gerir tudo financeira/estruturalmente, mas **não** acede a `roteiro`/`prescription` sem ser Profissional numa unidade concreta.
- Snapshot `tests/rbac-bootstrap-snapshot.test.ts` continua a ser a fotografia canónica dos 4 papéis — nenhuma das invariantes RBAC v2.1 muda.
- Matriz de testes simplificada: o fallback legacy e as suas fixtures (`tests/rbac-legacy.test.ts`, fixtures OWNER em `matrix-eligibility.test.ts`) deixam de ser necessários.

### Negativas / trade-offs
- Migração requer que **todas as clínicas tenham os 4 papéis sistema** antes do backfill. Em produção, isto é garantido pelo `bootstrapClinicRbac()` correr em cada `createInitialTenantAction` / `createCompanyAction` / `createClinicForCompanyAction`. Para clínicas históricas pré-RBAC v2 a migração aborta com erro explícito — correr `npx tsx scripts/sync-rbac-permissions.ts` antes.
- Quem actualmente tinha **apenas** `UserRole=OWNER` sem `ClinicRole` passa automaticamente a `gestor`, perdendo por defeito o acesso a conteúdo clínico (não o tinha legitimamente de qualquer forma; tinha-o pelo fallback legacy). Se o dono é também médico, tem de criar explicitamente a segunda `ClinicMember` como `profissional` na unidade onde atende.
- Conta de manutenção da plataforma (`PLATFORM_MAINTAINER_EMAIL`) não foi tocada — continua a ser detectada via `isPlatformMaintainerUserId()` no cabeçalho de `can()`, ortogonal ao RBAC de clínica.

### Abertas (seguimentos não-bloqueantes)
- Revisar utilizadores migrados em produção após deploy (script de auditoria: listar `ClinicMember` onde o utilizador actual assinou algo nos últimos 90 dias sem ter `roteiro.*` na `ClinicRole` efectiva).
- Documentação de utilizador em `docs/manuais/` para explicar o conceito de «dono = Gestor em todas as clínicas».
- Limpar migração `20260411220000_init` está fora de escopo (migrações aplicadas ficam congeladas; futuras correcções ao enum já não existem porque o tipo foi dropado).
