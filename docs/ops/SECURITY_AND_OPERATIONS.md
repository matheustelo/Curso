# Segurança e Operações — Plataforma de Cursos SaaS Multitenant

- **Versão:** 1.0 · **Data:** 2026-06-07 · **Status:** Proposto para revisão
- **Autor/perspectiva:** Staff Engineer (AppSec + SRE/DevOps)
- **Documentos relacionados:** [ARCHITECTURE.md](../ARCHITECTURE.md) · [DATA_MODEL.md](../DATA_MODEL.md) ·
  [PRD.md](../PRD.md) · [NON_FUNCTIONAL_REQUIREMENTS.md](../product/NON_FUNCTIONAL_REQUIREMENTS.md) ·
  [ADR-0001](../adr/0001-multitenancy-schema-per-tenant.md) · [ADR-0006](../adr/0006-auth-better-auth.md) ·
  [ADR-0010](../adr/0010-payments-br.md) · [ADR-0015](../adr/0015-live-classes-interactive.md) ·
  [CLAUDE.md](../../CLAUDE.md)

> **Premissa nº1 (CLAUDE.md §1):** o risco mais grave da plataforma é **vazamento cross-tenant**. Todo
> controle deste documento orbita esse risco. O `tenantId` é sempre derivado da **sessão/JWT** (fonte de
> verdade), nunca de header não autenticado, e todo acesso a dados passa por `withTenant` (`SET LOCAL
> search_path`). Vazamento cross-tenant é **P0 com meta zero** (PRD §6, NFR §1).

---

## Índice

1. [Threat model (STRIDE) — foco em vazamento cross-tenant](#1-threat-model-stride--foco-em-vazamento-cross-tenant)
2. [Segurança aplicada](#2-segurança-aplicada)
3. [Catálogo de erros (Problem Details RFC 9457) e versionamento de API](#3-catálogo-de-erros-problem-details-rfc-9457-e-versionamento-de-api)
4. [Backup & Disaster Recovery (runbook)](#4-backup--disaster-recovery-runbook)
5. [Ambientes & configuração](#5-ambientes--configuração)
6. [FinOps / modelo de custo consolidado](#6-finops--modelo-de-custo-consolidado)
7. [Dependências e pontos para o coordenador](#dependências-e-pontos-para-o-coordenador)

---

## 1. Threat model (STRIDE) — foco em vazamento cross-tenant

### 1.0 Ativos, atores e superfícies de ataque

**Ativos (ordenados por criticidade):**
1. **Isolamento de dados entre tenants** (catálogo `tenant_*`, control plane `platform`).
2. **Credenciais por tenant cifradas** (`bunny_keys_encrypted`, `live_keys_encrypted`,
   `pagarme_recipient_id_encrypted`, `platform_payment_recipients`).
3. **Sessões/identidade** (cookies Better-Auth, super-admin global, impersonação).
4. **Conteúdo protegido** (vídeos Bunny, gravações de live, certificados/PDFs no R2).
5. **Fluxo financeiro** (orders, splits, webhooks de pagamento, recipients).

**Atores de ameaça:** aluno autenticado malicioso (tenant A querendo dados de B); instrutor/admin de um
tenant; afiliado; atacante externo não autenticado; insider (super-admin); provedor terceiro comprometido
(webhook spoofado).

**Superfícies (foco do enunciado):** auth/sessão · webhooks (Bunny, LiveKit, Pagar.me/Asaas) · uploads
(TUS/pre-signed) · URLs/tokens assinados (vídeo, R2, sala live) · impersonação super-admin.

### 1.1 STRIDE resumido — tabela por ameaça × superfície

Legenda de risco: 🔴 crítico (P0/P1) · 🟠 alto · 🟡 médio.

| # | STRIDE | Ameaça | Superfície | Risco | Controles |
|---|--------|--------|-----------|-------|-----------|
| T-01 | **S**poofing | Forjar `tenant_id` via subdomínio/header para ler dados de outro tenant | Auth/sessão | 🔴 | `tenant_id` vem do **claim JWT assinado**; subdomínio apenas sugere e é validado contra o claim (ARCHITECTURE §3.2). Hook `onRequest` rejeita divergência (`403 tenant-mismatch`). |
| T-02 | **S**poofing | Webhook falso (sem ser do provedor) cria matrícula/libera vídeo | Webhooks | 🔴 | **HMAC sobre bytes brutos** + comparação em tempo constante; rejeitar antes de parsear JSON. Allowlist de IP quando o provedor publicar faixas. |
| T-03 | **S**poofing | Roubo de sessão / fixação de cookie | Auth/sessão | 🟠 | Cookies **HttpOnly+Secure+SameSite=Lax**; rotação de session id no login; binding por device fingerprint (F2, §7.1 NFR). |
| T-04 | **T**ampering | `search_path` "vazado" no transaction pooling do PgBouncer expõe schema errado | Banco/multitenancy | 🔴 | **`SET LOCAL search_path` por transação** (transacional, compatível com PgBouncer); nomes qualificados; `withTenant` único; **teste de isolamento cross-tenant como gate de CI** (ADR-0001). |
| T-05 | **T**ampering | Manipular payload de webhook (valor pago, status) após assinatura | Webhooks | 🟠 | Assinatura cobre o corpo inteiro; **reconciliação periódica** com o gateway (ARCHITECTURE §8) como fonte de verdade, não o webhook. |
| T-06 | **T**ampering | Anti-seek burlado (inflar progresso para concluir sem assistir) | Player/progresso | 🟡 | `real_watched_seconds` monotônico, deltas validados server-side; conclusão = `watched_pct≥90 AND real_watched_seconds≥duration*0.8` (DATA_MODEL §6.2). |
| T-07 | **R**epudiation | Super-admin nega ter impersonado/alterado tenant | Impersonação | 🟠 | **`platform.audit_log`** append-only (actor, ação, tenant, metadata, timestamp); banner visível; 2FA obrigatório (NFR-SEC-09). |
| T-08 | **R**epudiation | Falta de trilha em ações financeiras/moderação | Admin tenant | 🟡 | Audit log por tenant para ações sensíveis (suspensão, reembolso, ban, exclusão LGPD). |
| T-09 | **I**nfo disclosure | **Aluno A lê curso/aluno/pedido do tenant B** (o risco nº1) | Multitenancy | 🔴 | `withTenant` obrigatório; **nenhuma FK cruza schemas**; chave de cache/edge inclui `tenantId` (NFR §1); ZERO query crua sem search_path; teste de isolamento por tabela nova. |
| T-10 | **I**nfo disclosure | URL assinada de vídeo/R2 vazada permite acesso fora da sessão | URLs assinadas | 🟠 | **TTL curto (1–12h)**; token = `SHA256(key+videoId+expires)`; entitlement revalidado a cada assinatura; restrição de referer + MP4 progressivo off (ARCHITECTURE §7, NFR-SEC-10/11). |
| T-11 | **I**nfo disclosure | Keys Bunny/LiveKit/pagamento expostas no frontend ou em logs | Secrets | 🔴 | Keys **cifradas em repouso** (pgcrypto/KMS); **nunca** chegam ao front (upload via TUS/pre-signed; tokens gerados no backend); scrubbing de PII/secrets em logs/Sentry (NFR-OBS-02/07). |
| T-12 | **I**nfo disclosure | Mensagens de erro/Problem Details vazam schema, query ou existência de outro tenant | API | 🟡 | Problem Details sem stack/detalhe interno em prod; `detail` genérico; nunca revelar existência de outro tenant (NFR-LGPD-17). |
| T-13 | **I**nfo disclosure | Cache/CDN/ISR servindo conteúdo de um tenant para outro | Edge/cache | 🟠 | Chave de cache **sempre** inclui `tenantId`+`locale` (NFR-PERF-27); service worker não cacheia conteúdo de outro tenant (NFR-COMP-09). |
| T-14 | **D**oS | Brute force em login/reset; flood de webhooks; abuso de assinatura de token | Auth/webhooks | 🟠 | Rate limiting por IP+identidade+tenant; backoff progressivo; webhooks respondem 200 rápido e empurram trabalho pesado ao worker (NFR-PERF-20). |
| T-15 | **D**oS | Upload gigante / abuso de storage estoura quota e custo | Uploads | 🟡 | Limites de tamanho/tipo no pre-sign; quota por plano (storage_gb/bandwidth_gb); alertas de custo (§6). |
| T-16 | **D**oS | Exhaustão de conexões Postgres por tenant | Banco | 🟡 | **Um pool + PgBouncer** (sem pool por tenant — vantagem do schema-per-tenant); pool pequeno por instância (ADR-0001). |
| T-17 | **E**levation | Aluno vira admin via parâmetro/role forjada | Authz/RBAC | 🔴 | RBAC em guard único Fastify **e** revalidado no use-case (defense in depth, ARCHITECTURE §6); role vem do registro no schema do tenant, não do request. |
| T-18 | **E**levation | Admin de tenant alcança control plane (`platform`) ou outro tenant | Authz | 🔴 | `platform` só acessível por super-admin; admin de tenant nunca recebe search_path de `platform`; take_rate resolvido no `onRequest` e injetado como valor, sem o use-case consultar `platform` (DATA_MODEL §6.11). |
| T-19 | **E**levation | Impersonação abusada para escalar sem rastro | Impersonação | 🟠 | 2FA + audit log + banner + escopo temporal da sessão de impersonação (NFR-SEC-09, NFR-LGPD-16). |
| T-20 | **S/T** | Replay de webhook (reprocessar evento para duplicar matrícula/cobrança) | Webhooks | 🟠 | **Idempotência** por `event_id` unique (`payment_events`, `live_recording_events`); reprocesso é no-op (NFR-REL-12). |

### 1.2 Controles transversais nucleares (resumo)

1. **`withTenant` é a única porta de dados** — lint/boundary impede query crua; PR checklist exige.
2. **Teste de isolamento cross-tenant** como gate de CI para toda tabela nova com dado de tenant (inclui
   `live_*`, NFR §11).
3. **Claim assinado > subdomínio > header** na resolução de tenant.
4. **HMAC + idempotência** em todos os webhooks; **reconciliação** como rede de segurança.
5. **Defense in depth de authz**: guard + use-case.
6. **Secrets cifrados, nunca no front, scrubbing em telemetria.**

---

## 2. Segurança aplicada

### 2.1 Rate limiting / anti-abuso

- **Camadas:** edge (Vercel/Cloudflare) para volumetria bruta + `@fastify/rate-limit` no backend para
  limites por rota/identidade/tenant. Chave de limite = `tenantId + (userId | IP) + rota`.
- **Limites sugeridos (ajustar com dados reais):**
  - Login / reset de senha: **5/min por IP+e-mail**, depois backoff exponencial e CAPTCHA (NFR-SEC-05).
  - Assinatura de token de vídeo/live: **30/min por usuário** (protege custo e abuso de share).
  - Checkout / criação de pedido: **10/min por usuário**; idempotency key impede duplo-clique (NFR-REL-14).
  - Webhooks de entrada: limite alto + dedup por `event_id` (o controle real é HMAC+idempotência).
  - APIs de listagem: paginação keyset, `pageSize ≤ 100` (NFR-PERF-22).
- **Anti-enumeração:** respostas genéricas em login/reset ("se o e-mail existir, enviaremos instruções")
  sem revelar existência da conta (NFR-SEC-06).
- **Store do rate-limit:** pg-boss/Postgres no MVP (sem Redis — ADR-0011); se virar gargalo, trocar por
  store dedicado via ADR.

### 2.2 Proteção contra account takeover (ATO)

- Cookies de sessão **HttpOnly+Secure+SameSite** (Better-Auth); token nunca exposto a JS (NFR-SEC-01).
- **Senha forte** com feedback de força + bloqueio de senhas vazadas conhecidas (k-anonymity/HIBP range)
  quando viável (NFR-SEC-07); hashing **argon2** (plano B de auth também usa argon2).
- **Reautenticação** para mudança de senha/e-mail; **notificação por e-mail** após alteração de credencial
  (detecção de invasão, NFR-SEC-04).
- **Reset** com token de uso único e expiração curta; invalida sessões ao trocar senha (NFR-SEC-06).
- **2FA (TOTP)**: obrigatório para super-admin (MVP) e Owner do tenant; recomendado para admin; opcional
  para aluno. Códigos de backup (NFR-SEC-08).
- **Logout de todos os dispositivos** + lista de sessões ativas (F2, NFR-SEC-03).

### 2.3 Gestão de secrets (keys por tenant, cifragem e rotação)

- **Secrets de plataforma** (DB URL, HMAC de webhooks, chave-mestra de cifragem, billing Stripe) no
  **secret manager** do provedor (Fly/Railway/Vercel env encrypted), nunca no repo.
- **Keys por tenant** (Bunny AccessKey/token, LiveKit API key/secret, recipient de pagamento) **cifradas em
  repouso** no control plane (`bunny_keys_encrypted`, `live_keys_encrypted`,
  `platform_payment_recipients.recipient_id_encrypted`) via **envelope encryption**: chave-mestra (KMS) cifra
  uma DEK; a DEK cifra as keys do tenant (pgcrypto). A chave-mestra **nunca** vive no banco.
- **Acesso a keys de tenant** só no backend/worker, dentro do contexto do tenant; **decifragem just-in-time**
  para emitir token/chamar API; valor decifrado nunca logado.
- **Rotação:**
  - Chave-mestra (KMS): rotação anual ou sob suspeita; re-encrypt das DEKs (re-key offline).
  - Keys de tenant (Bunny/LiveKit): rotacionáveis por tenant sem downtime (provisionar nova, atualizar
    coluna cifrada, invalidar antiga) — útil em offboarding de funcionário do tenant.
  - HMAC de webhook: suportar **duas chaves ativas** durante rotação (aceitar ambas por janela).
- **Detecção de vazamento:** secret scanning no CI (gitleaks/`run_secret_scanning`) e push protection.

### 2.4 CSP e cabeçalhos de segurança

- **HTTPS obrigatório + HSTS** (`max-age` longo, `includeSubDomains`, `preload`) (NFR-SEC-14).
- **CSP** restritiva, compatível com **iframe do player Bunny**, **WebRTC do LiveKit** e pixels consentidos:
  - `default-src 'self'`; `frame-src` + `media-src` para domínios Bunny; `connect-src` para LiveKit
    (wss) e APIs; `script-src` sem `unsafe-inline` (usar nonce/hash); `img-src` self + CDN do tenant.
  - Pixels (Meta/GA4) só após consentimento (consent gating real — NFR-LGPD-02).
- Demais headers: `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`,
  `Permissions-Policy` (restringir câmera/microfone exceto na sala live), `X-Frame-Options`/`frame-ancestors`
  para evitar clickjacking (NFR-SEC-15).

### 2.5 Validação (Zod) e CORS

- **Validação:** schemas **Zod** em `packages/contracts` como fonte única (DRY, CLAUDE.md); `fastify-type-
  provider-zod` valida body/query/params/headers na borda. Rejeição → `400 validation-error` com
  `errors[]` por campo. Nunca confiar em input do cliente para `tenantId`/`role`.
- **Sanitização** de conteúdo rico (texto de aula, comentários, chat de live) contra XSS armazenado
  (NFR-SEC-16); allowlist de tags.
- **CORS:** origem restrita aos domínios do tenant (subdomínio + custom domain validado em
  `platform.tenants.custom_domain`); `credentials: true` apenas para origens conhecidas; **nunca** `*` com
  credenciais. **CSRF** protegido em mutações (SameSite + token quando necessário, NFR-SEC-16).

### 2.6 Proteção de webhooks (HMAC / idempotência)

Padrão único para Bunny, LiveKit e Pagar.me/Asaas (ARCHITECTURE §7/§8/§9):

1. Ler **bytes brutos** do corpo (sem parse prévio).
2. Verificar **HMAC** com a chave do provedor, **comparação em tempo constante**; rejeitar `401` se falhar.
3. Conferir timestamp/anti-replay quando o provedor fornecer (janela curta).
4. **Dedup por `event_id`/`egress_id`** (`payment_events.event_id` unique, `live_recording_events.event_id`
   unique). Evento já processado → `200` no-op.
5. Responder **200 rápido** (≤200ms, NFR-PERF-20) e **enfileirar** (pg-boss) o trabalho pesado, com
   `tenantId` no payload; o worker mapeia `VideoLibraryId`/`recipient`→tenant.
6. **Reconciliação periódica** com o gateway corrige divergências/perdas (NFR-REL-13).

### 2.7 Least privilege no PostgreSQL

- **Roles separadas:**
  - `app_runtime` (API/worker): `USAGE`/`SELECT/INSERT/UPDATE/DELETE` nos schemas `tenant_*` e `platform`,
    **sem** `CREATE`/`DROP SCHEMA` nem DDL.
  - `migrator` (job de migrations em CI): role com DDL, usada só no pipeline.
  - `provisioner` (saga de onboarding): pode `CREATE SCHEMA` + rodar migrations; idealmente um job
    dedicado, não a role de runtime.
- **`platform`** acessível apenas onde necessário; o runtime de um tenant não recebe search_path de
  `platform` para dados sensíveis (take_rate injetado como valor, DATA_MODEL §6.11).
- **PgBouncer** em **transaction pooling**; `SET LOCAL` por transação (não `SET` de sessão).
- **`public`** só com extensões (`pgcrypto`, `vector`, `pg_trgm`); sem objetos de negócio.
- Credenciais de DB no secret manager, rotacionáveis; conexões TLS.

---

## 3. Catálogo de erros (Problem Details RFC 9457) e versionamento de API

### 3.1 Formato base

Um **único error handler Fastify** mapeia a hierarquia `DomainError` (`packages/core`) para
**`application/problem+json`** (RFC 9457). Use-cases lançam erros de domínio; **nunca** tratam HTTP
(CLAUDE.md). Campos: `type` (URI estável, doc do erro), `title`, `status`, `detail` (genérico em prod),
`instance` (path), `requestId`/`traceId` (correlação, NFR-OBS-05), e extensões específicas (`errors[]`,
`retryAfter`).

```json
{
  "type": "https://errors.app.com/payment-required",
  "title": "Pagamento necessário",
  "status": 402,
  "detail": "Matrícula inativa para este curso.",
  "instance": "/api/v1/courses/abc/lessons/xyz/play-token",
  "requestId": "01J...","traceId": "4bf..."
}
```

> **Higiene:** `detail` nunca expõe schema, query, stack ou **existência de outro tenant** (T-12,
> NFR-LGPD-17). Stack só em log estruturado (Pino) + Sentry com scrubbing.

### 3.2 Catálogo de tipos de erro

| `type` (slug) | HTTP | Classe de domínio | Quando |
|---------------|------|-------------------|--------|
| `validation-error` | 400 | `ValidationError` (Zod) | Body/query/params inválidos; inclui `errors[]`. |
| `unauthorized` | 401 | `UnauthorizedError` | Sessão ausente/expirada/inválida. |
| `webhook-signature-invalid` | 401 | `WebhookSignatureError` | HMAC inválido (não vaza detalhe). |
| `forbidden` | 403 | `ForbiddenError` | Autenticado sem permissão (RBAC). |
| `tenant-mismatch` | 403 | `TenantMismatchError` | Claim de tenant ≠ subdomínio (T-01). |
| `not-found` | 404 | `NotFoundError` | Recurso inexistente **no schema do tenant** (mesmo corpo para "existe em outro tenant"). |
| `method-not-allowed` | 405 | — | Método não suportado. |
| `conflict` | 409 | `ConflictError` | Violação de unicidade (ex.: matrícula duplicada). |
| `gone` | 410 | `GoneError` | Token/recurso expirado (URL assinada vencida). |
| `payment-required` | 402 | `EntitlementError` | Sem matrícula/pagamento ativo ao pedir play-token. |
| `unprocessable-entity` | 422 | `BusinessRuleError` | Regra de negócio violada (ex.: max_attempts de quiz). |
| `rate-limited` | 429 | `RateLimitError` | Excedeu limite; inclui `retryAfter`. |
| `quota-exceeded` | 429/402 | `QuotaExceededError` | Estourou quota de plano (alunos/storage/banda — §6). |
| `payment-failed` | 402/502 | `PaymentError` | Falha no gateway (Pagar.me/Asaas). |
| `provider-unavailable` | 502/503 | `ProviderError` | Bunny/LiveKit/Resend indisponível (graceful — NFR-REL-05). |
| `idempotency-replay` | 200/409 | — | Webhook/operação já processada (no-op idempotente). |
| `internal-error` | 500 | `InternalError` | Inesperado; só `requestId` ao usuário (NFR-REL-09). |
| `service-unavailable` | 503 | — | Manutenção/degradação; `Retry-After`. |

> Erros de domínio adicionais herdam de `DomainError` e mapeiam para o `type` mais próximo; o handler tem
> um fallback `internal-error` para qualquer erro não-mapeado (fail-safe, sem vazar detalhe).

### 3.3 Versionamento de API

- **Prefixo de versão na URL:** `/api/v1/...` — explícito, cacheável, simples para o front e para o
  OpenAPI gerado (Zod → OpenAPI, ARCHITECTURE §1).
- **Compatibilidade:** mudanças aditivas (novos campos opcionais) não quebram versão; mudanças breaking →
  **`/api/v2`** com janela de depreciação anunciada (header `Deprecation`/`Sunset`).
- **Contratos:** `packages/contracts` versiona os schemas Zod; front e back consomem a mesma fonte (DRY).
- **Webhooks:** versionados pelo provedor; nosso endpoint tolera campos extras (forward-compatible) e
  valida só o necessário.

---

## 4. Backup & Disaster Recovery (runbook)

### 4.1 Metas (do NFR)

- **RPO ≤ 24h** e **RTO ≤ 4h** (NFR-REL-15). Caminho crítico de receita (checkout/webhook de pagamento)
  priorizado; PITR busca **RPO efetivo de minutos** para o Postgres.

| Componente | Estratégia | RPO | RTO |
|-----------|-----------|-----|-----|
| PostgreSQL (todos os schemas) | Snapshot diário + **WAL/PITR** contínuo (Neon/Supabase/Fly PG) | ~minutos (PITR) | ≤ 4h |
| Vídeos Bunny | Master files retidos pela Bunny (não re-hospedamos) | herda Bunny | herda Bunny |
| Gravações de live (R2) | Master MP4 em R2 (`<tenantId>/<sessionId>/...`) com versionamento | ≤ 24h | ≤ 4h |
| Arquivos/certificados (R2) | R2 com versionamento + lifecycle | ≤ 24h | ≤ 4h |
| Secrets/config | Secret manager (versionado) + IaC no repo | n/a | minutos |

### 4.2 Backup do Postgres (e a dimensão por tenant)

- **Backup físico** do banco inteiro (snapshot gerenciado + WAL archiving para PITR). Como é **1 banco /
  schema-per-tenant**, o backup físico cobre **todos os tenants** de uma vez.
- **Backup lógico por tenant** (granular, para restore seletivo e portabilidade/LGPD):
  `pg_dump --schema=tenant_<slug>` agendado (job pg-boss), enviado cifrado ao R2. Permite restaurar **um
  tenant** sem tocar nos demais — vantagem direta do modelo (ADR-0001).
- `platform` (control plane) tem dump lógico próprio (catálogo de tenants, mapeamento de keys).
- Backups **cifrados em repouso**, retenção (ex.: diário 30d, semanal 12s); restore testado (§4.5).

### 4.3 Runbook de restauração

**A) Restore total (perda do banco):**
1. Provisionar nova instância Postgres + PgBouncer.
2. Restaurar do snapshot mais recente; aplicar **PITR** até o ponto-alvo (antes do incidente).
3. Apontar `DATABASE_URL` (secret manager) para a nova instância; subir API/worker.
4. Rodar `db:migrate` (idempotente) para garantir consistência de schema.
5. **Smoke test** + **teste de isolamento cross-tenant** antes de liberar tráfego.

**B) Restore de um único tenant (corrupção/erro lógico isolado):**
1. Restaurar o snapshot/dump em um banco **scratch**.
2. `pg_dump --schema=tenant_<slug>` do scratch → `pg_restore` para um schema temporário
   `tenant_<slug>_restore` no banco de produção.
3. Validar dados; **swap** (renomear schema) em janela de manutenção do tenant; auditar.
4. `DROP SCHEMA` do temporário. Demais tenants **intocados**.

**C) Exclusão de tenant (LGPD/offboarding):** `DROP SCHEMA tenant_<slug> CASCADE` (ARCHITECTURE §3.6) —
manter o último backup lógico cifrado pelo período legal antes do descarte definitivo.

### 4.4 Mídia (Bunny / R2)

- **Bunny:** os masters ficam na Bunny; nosso "backup" é o **mapeamento** `lessons.video_guid` →
  reprocessável. Perder o banco mas manter a Bunny permite re-associar via `library_id`.
- **R2:** versionamento + replicação; gravações de live e certificados re-geráveis quando aplicável
  (certificado = `hash` determinístico, pode ser re-emitido).
- **Pipeline de live:** se a reingestão VOD falhar, o master em R2 permite reprocessar
  (`live.recording.ingest`, ADR-0015).

### 4.5 Teste de DR

- **GameDay trimestral:** restaurar produção em ambiente isolado, medir RTO real, validar PITR (RPO),
  rodar teste de isolamento cross-tenant pós-restore.
- **Restore de tenant único** testado em staging a cada mudança relevante de schema.
- Checklist de DR versionado; resultados registrados; alertas se backup diário falhar.

---

## 5. Ambientes & configuração

### 5.1 Ambientes

| Ambiente | Backend/Worker | Frontend | Banco | Propósito |
|----------|----------------|----------|-------|-----------|
| **dev** | local (tsx) / containers | Next local | Postgres local / Testcontainers | Desenvolvimento |
| **staging** | Fly/Railway (réplica de prod) | Vercel preview | Postgres gerenciado separado | QA, e2e, ensaios de migration/DR |
| **prod** | Fly/Railway | Vercel | Postgres gerenciado + PgBouncer | Produção |

- **Paridade dev/prod**: mesma imagem de container; diferenças só por env. Providers externos em **modo
  sandbox/test** em dev/staging (Pagar.me/Asaas test, Bunny library de teste, LiveKit project de teste).

### 5.2 Gestão de env/secrets

- **Schema de env validado por Zod** no boot (fail-fast se faltar variável). Sem secrets no repo;
  `.env.example` documenta chaves (sem valores).
- Secrets por ambiente no secret manager do provedor; **chave-mestra de cifragem** e HMAC de webhooks
  exclusivos de prod. Rotação conforme §2.3.

### 5.3 Feature flags (por tenant / fase)

- **Flags por fase** (MVP/F2/F3) e **por tenant/plano**: derivadas de `platform_plans.limits.features[]`
  (DATA_MODEL §6.11) — ex.: `order_bump`, `watermark`, `forum`, `live`, `api`. Resolvidas no `onRequest`
  junto do contexto do tenant e injetadas como valores (sem o use-case consultar `platform`).
- **Flags operacionais** (kill-switch, rollout gradual) em config central; permitem desligar um provider
  ou feature sem deploy. Avaliação determinística e logada.
- Uso: habilitar live/IA por tenant, ativar gateway alternativo (estratégia Pagar.me↔Asaas), gating de
  beta.

### 5.4 Migrations em CI (multitenant)

- **Pipeline:** GitHub Actions + Turborepo → Biome → typecheck → Vitest (Testcontainers, inclui
  **isolamento cross-tenant**) → build → deploy → **job de migrations multitenant**.
- **Runner** aplica a mesma definição Drizzle em **loop sobre `platform` + todos os `tenant_*`**,
  idempotente, com **tracking por schema** (ARCHITECTURE §3.5). Falha em um schema não corrompe os demais
  (transacional por schema).
- Migrations **versionadas e aditivas** preferencialmente (expand → migrate → contract) para permitir
  rollback sem perda; mudanças destrutivas exigem ADR + plano de janela.
- Provisionamento de novo tenant roda migrations no schema recém-criado (saga idempotente, ARCHITECTURE
  §3.4).

### 5.5 Release / rollback

- **Estratégia:** deploy de containers com **health checks**; rollout gradual onde o provedor suportar;
  **rollback de aplicação** = redeploy da imagem anterior (artefato imutável por commit).
- **Banco:** migrations **forward-compatible** (a versão N-1 do app roda com o schema N) → permite rollback
  do app sem rollback de migration. Reverter schema só via migration de contração planejada.
- **Frontend:** Vercel preview por PR + promote; rollback instantâneo para deploy anterior.
- **Kill-switch** via feature flag para isolar feature problemática sem rollback completo.

---

## 6. FinOps / modelo de custo consolidado

### 6.1 Custos variáveis principais (ordenados por risco)

| # | Custo | Driver | Risco | Mitigação |
|---|-------|--------|-------|-----------|
| **1** | **Bunny egress (BR)** — entrega de vídeo VOD | GB transferidos (alunos BR, mobile) | 🔴 **risco nº1** (PRD §7) | **Cap de resolução adaptativo**; quota `bandwidth_gb` por plano; alerta de egress (NFR-OBS-06); anti-share (TTL curto, entitlement). |
| **31** | **LiveKit** — participante-minuto + recording/egress [F2] | minutos × participantes + gravação | 🟠 **risco #31** | Limite de concorrência por sala/plano (NFR-LIVE-09, teto 100); `recording_enabled` opt-in; simulcast/SVC reduz banda; gravação reentra no VOD (sem dupla infra). |
| 3 | **R2** — storage de arquivos/gravações/certificados | GB armazenados (egress zero) | 🟡 | Egress zero é a vantagem; controlar **storage** via lifecycle + quota `storage_gb`. |
| 4 | **E-mail (Resend)** | volume de e-mails transacionais/marketing | 🟡 | Preferências de notificação (opt-out P2/P3, DATA_MODEL §6.9); batproteção anti-flood. |
| 5 | **Infra (Fly/Railway/Postgres)** | instâncias + storage + conexões DB | 🟡 | Pool pequeno + PgBouncer; autoscale com teto; right-sizing. |
| 6 | **Pagamentos** (Pagar.me/Asaas) | % por transação + split | 🟢 | Repassado via **take_rate_bps** do plano; Asaas para Pix/boleto de baixo custo. |
| 7 | **IA** (transcrição/embeddings) [F2/F3] | tokens/minutos processados | 🟡 | Gating por plano/feature; processar sob demanda; cache de embeddings. |

### 6.2 Alertas de custo

- **Alerta de egress de vídeo** (Bunny) por tenant e global — gatilho central (NFR-OBS-06, PRD §7); avisar
  ao se aproximar da quota do plano antes de overage.
- Alertas de **storage R2**, **minutos LiveKit**, **volume de e-mail** e **custo de infra** com thresholds
  por ambiente; budget alerts no provedor de cloud.
- **Dashboard de unit economics**: custo por tenant vs receita (take rate + plano) para identificar tenant
  deficitário.

### 6.3 Como as quotas de plano protegem a margem

- `platform_plans.limits` define `max_students`, `storage_gb`, `bandwidth_gb`, `max_courses`, `max_team`,
  `max_affiliates`, `take_rate_bps`, `features[]` (DATA_MODEL §6.11).
- **Serviço de quotas** (§6.12 DATA_MODEL) compõe limites (control plane) + uso corrente (contagens via
  `withTenant`; storage/banda via API Bunny) **sem FK cross-schema**, e autoriza/bloqueia conforme política
  de overage (MONETIZATION). Erro `quota-exceeded` (§3.2) quando estoura.
- **Banda de vídeo** (driver de custo nº1) é quota de primeira classe: o plano limita egress, alinhando
  custo Bunny ao preço cobrado e protegendo a margem. **`take_rate_bps`** repassa custo de pagamento;
  quotas de live/IA isolam os custos variáveis de F2/F3.

---

## Dependências e pontos para o coordenador

1. **Provedor gerenciado de Postgres (RPO/RTO):** confirmar Neon/Supabase/Fly PG com **PITR** e restore
   dentro de **RTO ≤ 4h** (cruza NFR-REL-15 §7 do NFR e ARCHITECTURE §12). Definir retenção de backup.
2. **KMS / envelope encryption:** escolher o KMS (provedor de cloud vs pgcrypto puro) para a chave-mestra
   que cifra `*_encrypted`. Define o procedimento de rotação (§2.3) — falta a decisão de ferramenta.
3. **Stack de rate-limit sem Redis:** validar que `@fastify/rate-limit` com store Postgres/in-memory atende
   o volume; se não, abrir ADR para revisar o "sem Redis" (ADR-0011).
4. **CSP definitiva:** consolidar a allowlist real de domínios Bunny + LiveKit (wss) + pixels consentidos;
   depende da config final de player e do CMP de consentimento (NFR-LGPD-08, item 8 do NFR §12).
5. **Política de retenção concreta (logs/financeiro/progresso):** validação jurídica/LGPD para fechar os
   prazos usados em backup/scrubbing (cruza NFR-LGPD-11 e item 4 do NFR §12) — falta o número oficial.
6. **2FA obrigatório para Owner/Admin do tenant:** decisão de produto/segurança (obrigatório vs
   recomendado) com impacto em onboarding (cruza NFR-SEC-08, item 6 do NFR §12).
7. **Quotas de banda/live como parâmetro de plano:** confirmar que `bandwidth_gb` e limites de live entram
   em `platform_plans.limits` no MVP/F2 (cruza FinOps §6.3, NFR §12 itens 2 e 3) — protege o risco de custo
   nº1 e #31.
8. **Versão de API e janela de depreciação:** confirmar `/api/v1` e a política de Sunset com o time de
   frontend (consumo via `packages/contracts`).
9. **GameDay de DR:** agendar o primeiro ensaio de restore total + restore por tenant em staging; definir
   responsável e cadência (§4.5).
