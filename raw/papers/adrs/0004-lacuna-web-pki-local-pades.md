<!-- wiki-ingest: synced from clinicagestor/docs/adr/0004-lacuna-web-pki-local-pades.md at 2026-05-09T18:04:15.346Z — edit upstream in monorepo, not here -->

# ADR-0004 — Assinatura legal Lacuna: Web PKI SDK (client-side) + PAdES-B-B local (server-side)

## Contexto

- O ADR-0003 define dois eixos distintos: **encerramento no sistema** (`signedAt`) e **assinatura legal** (PDF PAdES + metadata do fornecedor). A primeira integração planeada era **VIDAAS**; como fallback, foi iniciada uma segunda integração com **Lacuna Software** para garantir continuidade do fluxo enquanto credenciais VIDAAS não estavam operacionais.
- A **versão inicial** dessa integração Lacuna assentava em **Lacuna Rest PKI Core** (produto SaaS, endpoint `https://core.pki.rest`), com fluxo **Signature Sessions** (QR Code para app Web PKI mobile + callback/polling HTTP):
  - O nosso backend chamava `POST /api/v2/signature-sessions`, o médico lia um QR Code no app Lacuna, assinava via mobile, e o nosso backend fazia polling até `COMPLETED` para descarregar o PDF assinado.
- Na tentativa de obter credenciais reais para ambiente sandbox descobriu-se que **o produto «Rest PKI Core» é uma oferta SaaS comercial** que requer **provisioning owner-level pela administração Lacuna** para contas novas. O painel `https://core.pki.rest` fica literalmente em branco enquanto a conta não estiver provisionada, e não existe auto-serviço gratuito nem *tier* de desenvolvimento acessível sem falar com vendas.
- Paralelamente, o utilizador **reafirmou** o requisito de **assinatura qualificada ICP-Brasil** e confirmou que parte dos médicos da clínica já **tem certificado ICP-Brasil (A1/A3)** instalado na máquina de trabalho.

## Decisão

Abandonar **Rest PKI Core** como dependência obrigatória e substituir por uma arquitectura **100 % gratuita e auto-hospedada**, assente em duas peças:

1. **Client-side — Lacuna Web PKI SDK (package NPM `web-pki`):**
   - SDK gratuito fornecido pela Lacuna, exige uma vez a instalação local da extensão Web PKI no browser do médico (link oficial em `https://get.webpkiplugin.com/`).
   - Permite enumerar certificados ICP-Brasil instalados no SO / *smart-cards* da máquina (A1 e A3), ler o certificado em DER e calcular **assinatura RSA-SHA256 externa** sobre um `hash` qualquer que o nosso backend envie.
   - O modal `ConsultationEpisodeLegalSignatureModal` orquestra o fluxo: `init` → `listCertificates` → `readCertificate` → `signData`.

2. **Server-side — PAdES-B-B local (dois endpoints):**
   - `POST /api/consultations/:id/episodes/:episodeId/legal-signature/lacuna/prepare` — recebe o certificado DER do cliente, gera o PDF do roteiro, insere um placeholder PAdES com `@signpdf/placeholder-plain`, constrói os `signedAttributes` CMS (com `ESSCertIDv2` / `signingCertificateV2` para PAdES-B-B) e devolve ao cliente o *byte slice* a ser assinado (`dataToSignBase64`).
   - `POST /api/consultations/:id/episodes/:episodeId/legal-signature/lacuna/finalize` — recebe a assinatura RSA gerada pelo Web PKI, monta o envelope CMS `SignedData` completo com `node-forge` (`buildCmsSignedDataEnvelope`), injecta-o no `/Contents` do placeholder e grava o PDF assinado como `ConsultationEpisodeAttachment` vinculado ao `episodeId`.
   - A reentrada no fluxo ADR-0003 faz-se via `applyLacunaSignatureCompletion` — helper único responsável por: criar o anexo, actualizar `legalSignatureStatus = COMPLETED`, definir `legalSignatureCompletedAt`, aplicar o **bloqueio atómico no sistema** (`signedAt`) quando o encerramento foi diferido no finalize action, e emitir o evento de auditoria específico.

Consequências directas para a arquitectura:

1. **Dependências externas eliminadas.** Removidas as variáveis `REST_PKI_ENDPOINT`, `REST_PKI_ACCESS_TOKEN` e `LEGAL_SIGNATURE_LACUNA_WEBHOOK_URL`; apagados `lib/clinical/legal-signature/lacuna-rest-pki.ts` e as rotas `/lacuna/start`, `/lacuna/status`, `/lacuna/callback`.
2. **Feature flag `legalSignatureLacunaEnabled` mantém-se** como único *kill switch* por clínica. O helper `hasLacunaRuntimeCredentials()` é conservado apenas por compatibilidade de call-sites e passa a devolver sempre `true` (marcado `@deprecated`).
3. **Novas dependências server-side (gratuitas, MIT/Apache):** `node-forge` para ASN.1/CMS, `@signpdf/placeholder-plain` + `@signpdf/utils` para placeholder PAdES, `pdf-lib` configurado com `useObjectStreams: false` para produzir *xref table* clássica compatível com `@signpdf/placeholder-plain`.
4. **Ficheiros novos:**
   - `lib/clinical/legal-signature/cms-external-signature.ts` — ASN.1 / PKCS#7 puro.
   - `lib/clinical/legal-signature/pades-local.ts` — orquestrador `preparePadesSignature` + `finalizePadesSignature`.
   - `scripts/smoke-pades-local.ts` — smoke test `tsx` que valida o pipeline fim-a-fim (preparar → assinar com chave RSA local de teste → finalizar → verificar `ByteRange`, parse CMS, digest e assinatura).
5. **UX para o médico.** O fluxo QR Code mobile deixa de existir; o médico assina directamente na máquina onde está o certificado ICP-Brasil. Primeira utilização: instalar a extensão/app Web PKI (link claro no modal quando o SDK não detecta a extensão).

## Alternativas consideradas

- **Manter Rest PKI Core** → descartado por requerer contrato comercial e não ter tier gratuito acessível para sandbox.
- **Migrar para Rest PKI legacy (`pki.rest`)** → descartado: o legacy não suporta Signature Sessions e tem um modelo de integração diferente, com *depreciação anunciada* pela Lacuna.
- **Outro fornecedor SaaS (Clicksign, DocuSign, Contraktor, BRy/ValidCertificadora cloud)** → todos têm custo por assinatura e, crucialmente, **não entregam assinatura qualificada ICP-Brasil com o certificado do próprio médico** — apenas assinaturas avançadas em nome da plataforma ou em cloud HSM pago.
- **Implementação pura server-side com HSM/cloud signing** → inviável sem infra-estrutura própria de HSM ICP-Brasil; não resolve o requisito de assinatura no certificado pessoal do médico.

A solução escolhida é a única que mantém **(a) custo zero**, **(b) assinatura qualificada ICP-Brasil** no certificado do próprio médico e **(c) arquitectura 100 % sob nosso controlo**.

### Trade-off de UX assumido

Abrimos mão do fluxo **QR Code + app mobile Web PKI** (exclusivo do Rest PKI Core, pago) em troca de **zero custo mensal** e **total conformidade ICP-Brasil / CFM** com assinatura qualificada no certificado do próprio médico. Em contrapartida exige-se, na primeira utilização, a instalação única (≈30 s) da extensão/app Web PKI gratuita na máquina do médico — onboarding sinalizado de forma explícita no modal `ConsultationEpisodeLegalSignatureModal`. Subsequentes assinaturas no mesmo computador são silenciosas (A1) ou pedem apenas o PIN (A3).

### Adenda — Caminho A (A1/A3 local) vs. Caminho B (VIDAAS via VIDaaS Connect)

A arquitectura Web PKI SDK + PAdES-B-B local serve **dois públicos diferentes** através de um único fluxo técnico unificado, apenas com exigências de onboarding distintas:

| Dimensão | Caminho A — Certificado local (A1/A3) | Caminho B — VIDAAS cloud via VIDaaS Connect |
|---|---|---|
| **Quem serve** | Médicos com certificado A1 (ficheiro no SO) ou A3 (token USB / smartcard) já instalado no computador onde atendem. | Médicos que usam VIDAAS mobile da Valid (certificado cloud, normalmente assinatura via QR Code + telemóvel) e ainda assim querem assinar directamente no browser da ClinicaGestor. |
| **Software adicional no cliente** | Apenas a extensão Web PKI (gratuita, `get.webpkiplugin.com`). | Extensão Web PKI **+** VIDaaS Connect (ponte local gratuita da Valid que expõe o certificado cloud como se fosse um A3 local). |
| **UX de assinatura** | Silenciosa (A1) ou com PIN (A3). Sem passos no telemóvel. | Autorização biométrica no app VIDAAS mobile via push, acionado pelo VIDaaS Connect. Computador recebe a assinatura automaticamente assim que o médico aprova no telemóvel. |
| **Código do lado ClinicaGestor** | **Zero diferença.** O Web PKI SDK abstrai a origem do certificado; `prepare`/`finalize` tratam ambos os casos de forma idêntica. | **Zero diferença.** Ver acima. |
| **Onboarding** | 1 passo (instalar Web PKI). | 2 passos (instalar Web PKI + instalar VIDaaS Connect e fazer login). |
| **Validade legal** | ICP-Brasil qualificada. | ICP-Brasil qualificada (o certificado VIDAAS é ICP-Brasil emitido pela Valid AC). |
| **Custo para o médico** | Custo do próprio certificado A1/A3 (já pago; ADR-0004 não introduz custo novo). | Custo do próprio certificado VIDAAS (já pago) + VIDaaS Connect gratuito. |
| **Custo para a clínica** | Zero. | Zero. |

**Decisão operacional:** a ClinicaGestor **não privilegia** nenhum dos dois caminhos; ambos estão disponíveis simultaneamente e são apresentados lado-a-lado no modal (dois banners informativos, um por cada público). O médico escolhe o seu caminho consoante o certificado que já possui. O manual do utilizador [`docs/manuais/medico/04-assinatura-digital.md`](../manuais/medico/04-assinatura-digital.md) documenta ambos os fluxos.

**Consequência arquitectural:** nenhum código novo foi escrito especificamente para suportar VIDAAS — o Caminho B é servido como **efeito colateral gratuito** da decisão de assentar sobre o Web PKI SDK. Quando a API directa VIDAAS da Valid estiver disponível (credenciais comerciais da clínica), será adicionada como **terceiro fornecedor** no picker, não como substituto de nenhum dos caminhos actuais. Ver `plan-implementacao-assinatura-legal-vidaas.md` (parcialmente superseded por este ADR).

## Segurança e conformidade

- **Assinatura qualificada ICP-Brasil.** O certificado usado é o próprio A1/A3 do médico; a chave privada **nunca** sai da máquina dele. O nosso backend vê apenas o certificado público em DER e a assinatura RSA-SHA256 já pronta. Isto mantém equivalência com PAdES-B-B aceite em auditoria (MP 2.200-2/2001, Lei 14.063/2020).
- **Integridade do PDF.** O `ByteRange` é calculado pós-placeholder sobre os bytes reais do PDF; qualquer alteração posterior invalida o digest e a assinatura.
- **Atomicidade do encerramento.** O `signedAt` só é escrito dentro de `applyLacunaSignatureCompletion` na mesma transacção do anexo e do `legalSignatureStatus = COMPLETED`. Falha a meio → episódio continua **encerrado mas editável** (garantindo que não há bloqueio sem prova legal gravada).
- **Auditoria.** Reaproveita os eventos definidos em ADR-0003 (`CONSULTATION_EPISODE_LEGAL_SIGNATURE_COMPLETED`), com `metadata.provider = "LACUNA"`, `attachmentId`, `subjectCn` e *thumbprint* SHA-256 do certificado.
- **Timeline.** Entrada adicional «XX:XX – Assinatura digital concluída com Lacuna Web PKI – Sistema» registada junto da linha de encerramento, para transparência para o médico.

## Consequências operacionais

- **Onboarding do médico** documentado no modal: primeira utilização pede instalação da extensão Web PKI (um link, ~30 segundos). Subsequentes utilizações são silenciosas.
- **Sem custo recorrente.** Nenhuma dependência de subscrição Lacuna, Valid ou outro fornecedor enquanto o flag VIDAAS não for reactivado.
- **VIDAAS continua em preparação.** O selector de fornecedor (`resolveActiveLegalSignatureProviders`) mostra ambos os cartões para transparência ao médico, conforme pedido expresso; VIDAAS fica desactivado com nota técnica até as credenciais Valid chegarem.
- **Testes:** `scripts/smoke-pades-local.ts` valida o pipeline PAdES com chave de teste; `tests/lacuna-signature-completion.integration.test.ts` valida a atomicidade do `applyLacunaSignatureCompletion`. Testes `vitest` dos módulos `pades-local`/`cms-external-signature` ficam pendentes até o runner estar saudável (questão pré-existente do ambiente).

## Status

**Aceite** — implementação presente nos commits que acompanham este ADR. Rollback é possível revertendo o par de rotas `/prepare`+`/finalize` e restaurando `/start`+`/status` a partir de histórico git, mas não está planeado — a arquitectura Rest PKI Core deixa de ser alvo.
