# Modelo de Dados — Plataforma de Cursos SaaS Multitenant

- **Versão:** 1.3 · **Data:** 2026-06-07
- **Modelo de isolamento:** schema-per-tenant (ver [ADR-0001](adr/0001-multitenancy-schema-per-tenant.md))
- **Relacionados:** [ARCHITECTURE.md](ARCHITECTURE.md) · [PRD.md](PRD.md) · [docs/product/](product/README.md) · [ADR-0013](adr/0013-product-data-model-extensions.md) · [ADR-0014](adr/0014-analytics-provider-tracking-plan.md) · [ADR-0015](adr/0015-live-classes-interactive.md)

> **Changelog v1.3 (coordenação — integração dos docs de design/produto/ops/legal):** acrescentadas, como
> **proposta rastreável a consolidar por migration + ADR** (nada removido), as seções: **§6.15 — Suporte ao
> aluno** (`support_tickets`, `support_messages`, `kb_articles`), **§6.16 — Importação/Exportação**
> (`import_jobs`, `import_rows`, `export_jobs`), **§6.17 — Vocabulário de `platform.audit_log.action`** (console
> Super-Admin) e **§6.18 — Branding/white-label/domínio próprio** (novos campos em `tenant_settings` e
> `platform.tenants`). Acrescidos à §6.0 os status `support_tickets.status` e os status de `import_jobs`/
> `export_jobs`; novos campos em **§6.10** (`support_email`, `support_channel`, `support_widget_config`,
> `favicon_url`, `logo_dark_url`, `email_reply_to`, `whitelabel_full`, `onboarding_state`). Origem:
> [SUPPORT_DISCOVERY_SETTINGS](product/SUPPORT_DISCOVERY_SETTINGS.md), [DATA_IMPORT_EXPORT](product/DATA_IMPORT_EXPORT.md),
> [SUPER_ADMIN_CONSOLE](product/SUPER_ADMIN_CONSOLE.md), [BRANDING_WHITELABEL](design/BRANDING_WHITELABEL.md).
> **Toda tabela nova vive no schema do tenant, acessada via `withTenant`; nenhuma FK cruza schemas; cada uma
> exige teste de isolamento cross-tenant (gate CI).** Decisões estruturais (provider de suporte, motor de busca
> externo, migration de import/export) listadas em [OPEN_QUESTIONS](OPEN_QUESTIONS.md) #32–#40 — exigirão ADR ao
> serem adotadas.

> **Changelog v1.2 (coordenação de produto):** acrescentada a **§6.13 — Aulas ao vivo [F2]**
> (tabelas `live_sessions`, `live_attendance`, `live_chat_messages`, `live_bans`, `live_recording_events`),
> os status canônicos `live_sessions.status` e `live_sessions.recording_status` à §6.0, e a coluna
> `platform.tenants.live_keys_encrypted` (keys do LiveKit por tenant). Renumerados os antigos §6.13 →
> §6.14 (relacionamentos adicionais). Nada das versões anteriores foi removido. Decisão em
> [ADR-0015](adr/0015-live-classes-interactive.md) e spec em [LIVE_CLASSES.md](product/LIVE_CLASSES.md).

> **Changelog v1.1 (coordenação de produto):** acrescentada a **§6 — Extensões dirigidas por produto**
> (comentários/moderação, anti-seek, gamificação, quiz, ofertas/order_items/bundles, afiliados/atribuição,
> notificações, i18n, quotas/take rate, recipients de pagamento). Nada da v1.0 foi removido. Cada extensão
> está marcada por fase (MVP/F2/F3). Decisões registradas em [ADR-0013](adr/0013-product-data-model-extensions.md)
> e [ADR-0014](adr/0014-analytics-provider-tracking-plan.md).

Um único banco PostgreSQL com:
- **`platform`** — *control plane* (dados globais do SaaS);
- **`tenant_<slug>`** — *data plane* (um schema idêntico por tenant);
- **`public`** — apenas extensões (`pgcrypto`, `vector`, `pg_trgm`).

> Convenções: PKs `uuid` (default `gen_random_uuid()`); timestamps `created_at`/`updated_at` em UTC;
> soft-delete via `deleted_at` onde fizer sentido; valores monetários em **centavos** (`integer`) +
> `currency`. Nomes de tabela em `snake_case` plural.

---

## 1. Control plane — schema `platform`

```sql
-- Catálogo de tenants e roteamento
tenants(
  id uuid pk,
  slug text unique,             -- subdomínio: acme → acme.app.com
  name text,
  status text,                  -- provisioning | active | suspended | cancelled
  plan_id uuid fk -> platform_plans,
  schema_name text,             -- "tenant_acme"
  bunny_library_id text,        -- Video Library do tenant
  bunny_keys_encrypted bytea,   -- AccessKey/token key cifradas (pgcrypto/KMS)
  live_keys_encrypted bytea null, -- API key/secret do LiveKit cifradas (1 projeto LiveKit por tenant; F2 — ADR-0015)
  custom_domain text null,
  created_at, updated_at
)

platform_plans(                 -- planos do SaaS (tiers vendidos aos tenants)
  id uuid pk, name text, price_cents int, currency text,
  limits jsonb                  -- { max_students, storage_gb, features[] }
)

platform_subscriptions(         -- assinatura do tenant na plataforma (Stripe Billing)
  id uuid pk, tenant_id fk, plan_id fk,
  stripe_subscription_id text, status text, current_period_end timestamptz
)

platform_invoices(
  id uuid pk, tenant_id fk, amount_cents int, currency text,
  status text, due_at timestamptz, paid_at timestamptz null
)

super_admins(                   -- staff interno (nós)
  id uuid pk, email text unique, name text, role text, mfa_enabled bool
)

provisioning_jobs(              -- saga idempotente de onboarding
  id uuid pk, tenant_id fk, step text, status text,
  idempotency_key text unique, attempts int, last_error text, updated_at
)

audit_log(                      -- ações sensíveis (impersonação, suspensão...); vocabulário de `action` em §6.17
  id uuid pk, actor_id uuid, actor_type text, tenant_id uuid null,
  action text, metadata jsonb, created_at
)
```

---

## 2. Data plane — schema `tenant_<slug>` (idêntico para todos os tenants)

### 2.1 Identidade e equipe
```sql
users(
  id uuid pk, email text, name text,
  role text,                    -- owner | admin | instructor | affiliate | student
  password_hash text null,      -- gerenciado pelo Better-Auth
  status text, created_at, updated_at,
  unique(email)                 -- único DENTRO do schema do tenant
)
sessions(...)                   -- tabelas do Better-Auth (por tenant)
```

### 2.2 Catálogo de conteúdo
```sql
courses(
  id uuid pk, title text, slug text, description text,
  cover_url text, instructor_id fk -> users,
  status text,                  -- draft | published | archived
  price_cents int null, currency text, pricing_type text, -- one_time | subscription | free
  created_at, updated_at, unique(slug)
)
modules(
  id uuid pk, course_id fk, title text, position int
)
lessons(
  id uuid pk, module_id fk, title text, position int,
  type text,                    -- video | text | pdf | quiz | live
  content jsonb,                -- texto rico / config
  video_guid text null,         -- GUID do vídeo no Bunny
  duration_seconds int null,
  status text,                  -- draft | published
  drip_release_at timestamptz null,        -- liberação por data fixa
  drip_days_after_enroll int null          -- liberação por dias após matrícula
)
lesson_assets(                  -- PDFs/anexos (armazenados no R2)
  id uuid pk, lesson_id fk, filename text, storage_key text, size_bytes bigint
)
```

### 2.3 Matrícula, acesso e progresso
```sql
enrollments(
  id uuid pk, user_id fk, course_id fk,
  status text,                  -- active | suspended | refunded | expired
  source text,                  -- purchase | manual | bulk | auto
  enrolled_at, expires_at null,
  unique(user_id, course_id)
)
lesson_progress(
  id uuid pk, enrollment_id fk, lesson_id fk,
  position_seconds int, watched_pct numeric,
  status text,                  -- not_started | in_progress | completed
  completed_at timestamptz null
)
```

### 2.4 Avaliações e certificados
```sql
quizzes(id uuid pk, lesson_id fk null, course_id fk, title text,
        pass_score numeric null, max_attempts int null, time_limit_sec int null)
quiz_questions(id uuid pk, quiz_id fk, type text, prompt text, options jsonb, answer jsonb, position int)
quiz_attempts(id uuid pk, quiz_id fk, user_id fk, score numeric, answers jsonb,
              started_at, submitted_at)
certificates(
  id uuid pk, enrollment_id fk, user_id fk, course_id fk,
  uuid_public text unique,      -- usado na URL de verificação + QR
  hash text,                    -- SHA256(student+course+issued_at+secret)
  storage_key text,             -- PDF no R2
  issued_at timestamptz, revoked bool default false
)
```

### 2.5 Monetização (checkout dos alunos)
```sql
orders(
  id uuid pk, user_id fk, course_id fk null,
  amount_cents int, currency text, status text,    -- pending | paid | refunded | chargeback
  provider text,                                    -- pagarme | asaas
  provider_order_id text, payment_method text,      -- pix | boleto | card
  coupon_id fk null, affiliate_id fk null,
  created_at, paid_at null
)
subscriptions(                  -- assinatura do ALUNO ao conteúdo do tenant
  id uuid pk, user_id fk, plan_ref text, status text,
  provider text, provider_subscription_id text, current_period_end timestamptz
)
coupons(id uuid pk, code text unique, type text, value int, valid_until timestamptz, max_uses int, uses int)
payment_events(                 -- idempotência de webhooks
  id uuid pk, provider text, event_id text unique, payload jsonb, processed_at timestamptz
)
```

### 2.6 Afiliados e split
```sql
affiliates(
  id uuid pk, user_id fk, code text unique, commission_pct numeric, status text
)
affiliate_commissions(
  id uuid pk, affiliate_id fk, order_id fk, amount_cents int, status text, -- pending | paid | reversed
  created_at, paid_at null
)
splits(                         -- regras de divisão (co-produção/afiliado)
  id uuid pk, course_id fk null, recipient_ref text, percent numeric
)
```

### 2.7 Comunidade
```sql
lesson_comments(id uuid pk, lesson_id fk, user_id fk, body text, parent_id fk null, created_at)
-- F2: forum_topics, forum_posts, groups, events
```

### 2.8 IA (Fase 2/3)
```sql
lesson_transcripts(id uuid pk, lesson_id fk, language text, vtt_url text, text text)
lesson_embeddings(id uuid pk, lesson_id fk, chunk text, embedding vector(1536))  -- pgvector
```

---

## 3. Diagrama de relacionamentos (texto)

```
platform.tenants 1───* platform.provisioning_jobs
platform.tenants *───1 platform.plans
platform.tenants 1───1 platform.subscriptions ──* platform.invoices

[ dentro de cada schema tenant_<slug> ]
users 1───* courses (como instructor)
courses 1───* modules 1───* lessons 1───* lesson_assets
courses 1───* enrollments *───1 users
enrollments 1───* lesson_progress *───1 lessons
courses 1───* quizzes 1───* quiz_questions ; quizzes 1───* quiz_attempts *───1 users
enrollments 1───1 certificates
users 1───* orders ; orders *───1 coupons ; orders *───1 affiliates
affiliates 1───* affiliate_commissions *───1 orders
lessons 1───* lesson_comments *───1 users
lessons 1───1 lesson_transcripts 1───* lesson_embeddings
```

---

## 4. Índices e considerações de performance

- `enrollments(user_id, course_id)` unique; índices em `lesson_progress(enrollment_id)`,
`orders(status, created_at)`, `payment_events(event_id)` unique (idempotência).
- `pg_trgm` para busca textual (cursos/aulas); `pgvector` (ivfflat/hnsw) para RAG na F3.
- Como o schema é por tenant, os índices são por schema — volumes pequenos por tenant, ótimo plano de
query. Atenção ao número total de objetos no catálogo (dezenas de tenants → sem problema).

---

## 5. Regras de integridade de tenant

- **Nenhuma FK cruza schemas.** Cada schema é autossuficiente; a ligação ao tenant é o próprio schema.
- O control plane (`platform`) **não** referencia tabelas de tenant por FK — apenas guarda `schema_name`.
- Todo acesso a dados de tenant passa por `withTenant(tenantId, fn)` que aplica
`SET LOCAL search_path TO tenant_<slug>, public;` dentro da transação.

---

## 6. Extensões dirigidas por produto (v1.1)

> Consolidação das lacunas levantadas pelos especialistas de produto (LEARNING_EXPERIENCE_UX,
> BUSINESS_RULES_AND_STATES, MONETIZATION, NOTIFICATIONS_MATRIX, ANALYTICS, NFR). Decisões em
> [ADR-0013](adr/0013-product-data-model-extensions.md). Todas as colunas existentes da v1.0 permanecem;
> aqui acrescentamos campos/tabelas e **fixamos o vocabulário canônico de `status`**.

### 6.0 Vocabulário canônico de `status` (padronização)

Onde o DATA_MODEL v1.0 usava `text` genérico, ficam padronizados (CHECK ou enum lógico):

| Coluna | Valores canônicos | Fase |
|--------|-------------------|------|
| `tenants.status` (platform) | `provisioning \| active \| suspended \| cancelled` | MVP |
| `platform_subscriptions.status` | `trialing \| active \| past_due \| canceled \| expired` | MVP |
| `subscriptions.status` (aluno) | `trialing \| active \| past_due \| canceled \| expired` | MVP |
| `enrollments.status` | `active \| suspended \| refunded \| expired` | MVP |
| `orders.status` | `pending \| paid \| refunded \| chargeback` | MVP |
| `lesson_progress.status` | `not_started \| in_progress \| completed` | MVP |
| `affiliate_commissions.status` | `pending \| paid \| reversed` | MVP |
| `affiliates.status` | `pending \| active \| blocked` | MVP |
| `live_sessions.status` | `scheduled \| lobby \| live \| ended \| canceled` | F2 |
| `live_sessions.recording_status` | `none \| recording \| processing \| ready \| failed` | F2 |
| `support_tickets.status` (§6.15) | `open \| pending \| resolved \| closed` | MVP |
| `kb_articles.status` (§6.15) | `draft \| published` | MVP |
| `import_jobs.status` (§6.16) | `uploaded \| validating \| preview_ready \| committing \| partially_done \| done \| failed \| canceled` | MVP |
| `import_rows.status` (§6.16) | `valid \| invalid \| created \| updated \| skipped \| error` | MVP |
| `export_jobs.status` (§6.16) | `queued \| running \| ready \| expired \| failed` | MVP |
| `custom_domain_status` (platform.tenants, §6.18) | `none \| pending \| verifying \| active \| failed` | F2 |

> O estado de inadimplência do **SaaS** (`past_due/grace`) é **derivado** de `platform_subscriptions.status`
> (Stripe), **sem** nova coluna em `tenants.status` (MONETIZATION §A.5). Reembolso parcial **não** é estado
> de `orders` no MVP (decisão: tratado como ajuste financeiro/nota; ver OPEN_QUESTIONS).

### 6.1 Conteúdo — status de vídeo e i18n [MVP / F2]
```sql
-- ALTER lessons: status de processamento do vídeo Bunny (MVP)
lessons.video_status text     -- none | queued | processing | ready | failed  (default 'none')

-- i18n de conteúdo do tenant (F2; coluna preparada no MVP para evitar migração destrutiva — NFR-I18N-11/12)
-- Estratégia escolhida: coluna jsonb de overrides por locale, mantendo as colunas-base como PT-BR default.
courses.i18n  jsonb null      -- { "es": { "title": "...", "description": "..." }, "en": {...} }
lessons.i18n  jsonb null      -- { "es": { "title": "...", "content": {...} }, ... }
-- locale-base e locales suportados ficam em branding/config do tenant (tenant_settings, abaixo)
```

### 6.2 Progresso — anti-seek [MVP]
```sql
-- ALTER lesson_progress: tempo REAL assistido acumulado, separado de watched_pct (PRD §4.4)
lesson_progress.real_watched_seconds int not null default 0
-- Regra de conclusão: watched_pct >= 90 AND real_watched_seconds >= duration_seconds * 0.8
-- Atualizado de forma monotônica pelos heartbeats (deltas validados server-side; seek não infla).
lesson_progress.last_heartbeat_at timestamptz null
```

### 6.3 Comunidade — moderação, curtidas, denúncias [MVP]
```sql
-- ALTER lesson_comments
lesson_comments.resolved_at timestamptz null     -- dúvida marcada como resolvida
lesson_comments.resolved_by uuid null            -- fk -> users
lesson_comments.hidden_at  timestamptz null      -- moderação (oculto)
lesson_comments.hidden_by  uuid null             -- fk -> users
lesson_comments.deleted_at timestamptz null      -- soft-delete pelo autor/moderador

comment_likes(                                   -- curtidas (idempotente por par)
  comment_id uuid fk, user_id uuid fk,
  created_at timestamptz,
  primary key (comment_id, user_id)
)
comment_reports(                                  -- denúncias
  id uuid pk, comment_id uuid fk, reporter_id uuid fk,
  reason text, status text,                       -- open | reviewed | dismissed
  created_at timestamptz, reviewed_by uuid null, reviewed_at timestamptz null
)
```

### 6.4 Avaliações — tentativas e idempotência [MVP base / F2 avançado]
```sql
-- ALTER quiz_attempts: numeração e idempotência (PRD §3.3 F2; base já no MVP)
quiz_attempts.attempt_number int not null default 1   -- sequencial por (quiz_id, user_id)
quiz_attempts.status text not null default 'in_progress' -- in_progress | submitted | graded
quiz_attempts.idempotency_key text null               -- dedup de submit; unique(quiz_id,user_id,idempotency_key)
quiz_attempts.passed boolean null                      -- derivado de score >= quiz.pass_score
-- Política de nota (maior vs última) e feedback ficam em quiz config:
quizzes.scoring_policy text null default 'highest'    -- highest | last  (F2)
quizzes.feedback_policy text null default 'after_submit' -- after_submit | after_close | none (F2)
-- unique(quiz_id, user_id, attempt_number)
```

### 6.5 Gamificação — ledger idempotente [F2]
```sql
gamification_xp_ledger(                            -- append-only, idempotente (antifarming/reversão)
  id uuid pk, user_id uuid fk, amount int,         -- pode ser negativo (reversão)
  source_type text,                                -- lesson_completed | quiz_passed | daily_login | ...
  source_event_id text,                            -- chave do fato de origem
  created_at timestamptz,
  unique(user_id, source_type, source_event_id)    -- impede dupla contagem
)
gamification_user_state(                            -- agregado materializado (XP/nível) por usuário
  user_id uuid pk, xp_total int not null default 0, level int not null default 1,
  leaderboard_optout boolean not null default false, -- privacidade (LGPD)
  updated_at timestamptz
)
badges(id uuid pk, code text unique, title text, description text, icon_url text, criteria jsonb)
user_badges(user_id uuid fk, badge_id uuid fk, awarded_at timestamptz, primary key(user_id, badge_id))
```

### 6.6 Marcos/celebrações — idempotência [MVP/F2]
```sql
enrollment_milestones(                             -- evita recelebrar marcos (25/50/75/100%, 1ª aula...)
  enrollment_id uuid fk, milestone text,           -- first_lesson | pct_25 | pct_50 | pct_75 | completed
  reached_at timestamptz,
  primary key (enrollment_id, milestone)
)
```

### 6.7 Monetização — ofertas, order_items, bundles [F2]
```sql
-- order_items: orders v1.0 carrega 1 course_id; promovemos para multi-item (order bump/bundle/upsell)
order_items(
  id uuid pk, order_id uuid fk, course_id uuid fk null, bundle_id uuid fk null,
  kind text,                                        -- main | order_bump | upsell | bundle
  amount_cents int, currency text, created_at timestamptz
)
-- correlação de sessão de compra para encadear upsell/downsell pós-compra à venda original
orders.purchase_session_id text null                -- agrupa orders de uma mesma jornada de checkout
orders.parent_order_id uuid null                    -- upsell aponta para a order principal

offers(                                             -- order bump / upsell configuráveis pelo admin
  id uuid pk, type text,                            -- order_bump | upsell | downsell
  target_course_id uuid fk null, trigger_course_id uuid fk null,
  price_cents int, currency text, active boolean, position int, created_at timestamptz
)
bundles(id uuid pk, title text, slug text unique, price_cents int, currency text, status text, created_at)
bundle_items(bundle_id uuid fk, course_id uuid fk, position int, primary key(bundle_id, course_id))
```

### 6.8 Afiliados — atribuição (cookie/janela/aprovação) e split [MVP]
```sql
-- ALTER affiliates: override de comissão por curso e dados de recipient
affiliates.pagarme_recipient_id_encrypted bytea null  -- recipient do afiliado (cifrado)

affiliate_program_settings(                         -- config do programa por tenant (1 linha)
  id uuid pk, enabled boolean not null default false,
  default_commission_pct numeric,                   -- padrão do tenant
  cookie_window_days int not null default 30,        -- janela de atribuição (last-click)
  approval_policy text not null default 'manual',    -- manual | auto
  self_referral_allowed boolean not null default false,
  updated_at timestamptz
)
affiliate_course_commissions(                        -- override de pct por curso (alternativa a splits)
  affiliate_id uuid fk, course_id uuid fk, commission_pct numeric,
  primary key (affiliate_id, course_id)
)
affiliate_clicks(                                    -- atribuição last-click (cookie/janela)
  id uuid pk, affiliate_id uuid fk, course_id uuid fk null,
  visitor_id text,                                   -- id anônimo first-party
  clicked_at timestamptz, expires_at timestamptz     -- clicked_at + cookie_window_days
)
-- splits v1.0 mantido; recipient da plataforma e do produtor são resolvidos no use-case de split
-- a partir do take_rate_bps do plano (control plane) + recipients abaixo.
```

### 6.9 Notificações — preferências e central in-app [MVP]
```sql
notifications(                                       -- central in-app (escopo tenant)
  id uuid pk, user_id uuid fk, type text, payload jsonb,
  read_at timestamptz null, created_at timestamptz
)
notification_preferences(                            -- opt-out por canal/categoria (honra só P2/P3)
  user_id uuid fk, channel text,                     -- in_app | email   (push = F2)
  category text,                                     -- replies | instructor | new_lesson | certificate | marketing | ...
  enabled boolean not null default true,
  primary key (user_id, channel, category)
)
-- push_subscriptions (Web Push) = F2
```

### 6.10 Configurações do tenant (branding/i18n/políticas) [MVP]
```sql
tenant_settings(                                     -- 1 linha por tenant (data plane); branding + políticas
  id uuid pk,
  logo_url text null, primary_color text null, secondary_color text null,
  default_locale text not null default 'pt-BR', supported_locales text[] not null default '{pt-BR}',
  -- políticas configuráveis (defaults globais sensatos; tenant pode sobrescrever)
  refund_revokes_certificate boolean not null default true,  -- revogar certificado em reembolso
  student_dunning_grace_days int not null default 7,          -- carência inadimplência do ALUNO
  affiliate_clearance_days int not null default 14,           -- garantia antes de pending→paid
  public_catalog_enabled boolean not null default true,
  -- v1.3 (proposta): suporte ao aluno (SUPPORT_DISCOVERY_SETTINGS §1) — fonte da coluna do placeholder {support_email}
  support_email text null,                                    -- suporte ao aluno; null → fallback e-mail do owner
  support_channel text not null default 'native',             -- native | widget (Crisp/Intercom em F2 via SupportProvider)
  support_widget_config jsonb null,                           -- config do widget externo (F2)
  -- v1.3 (proposta): branding/white-label (BRANDING_WHITELABEL §9.2) — hoje só logo_url/primary/secondary existem
  favicon_url text null,                                      -- favicon do tenant (escopo MVP da IA §2.4; sem coluna até aqui)
  logo_dark_url text null,                                    -- variante de logo p/ fundo escuro
  email_reply_to text null,                                   -- reply-to do e-mail transacional white-label (EMAIL_TEMPLATES §dep.2)
  whitelabel_full boolean not null default false,             -- remoção da marca da plataforma (gated por feature do plano)
  -- v1.3 (proposta): ativação (ONBOARDING_ACTIVATION §3.2) — estado é DERIVADO; jsonb só p/ itens dispensados/ordem
  onboarding_state jsonb null,                                -- { dismissed:[...], order:[...] } (opcional; progresso real é derivado)
  -- v1.3 (proposta, LGPD): contato de privacidade do próprio tenant (COMPLIANCE §dep.6) — cada tenant é Controlador
  privacy_contact_email text null,
  updated_at timestamptz
)
```
> **v1.3:** os campos acima são **propostas** vindas dos novos docs; consolidar via migration + ADR (não há
> coluna hoje além de `logo_url/primary_color/secondary_color`). Sem `favicon_url`/`support_email`, o favicon
> (MVP) e a fonte de `{support_email}` (NOTIFICATIONS §8) não teriam onde persistir. Booleanos como
> `whitelabel_full` são consumidos por **injeção no `onRequest`** (ADR-0013), nunca consultados direto pelo
> use-case. `onboarding_state` é **opcional**: o progresso do checklist é **derivado** de eventos/queries reais
> via `withTenant`, sem nova tabela (ONBOARDING_ACTIVATION §3.2).

### 6.11 Control plane — quotas, take rate e recipient da plataforma [MVP]
```sql
-- platform_plans.limits jsonb (padronização de chaves — MONETIZATION §A.2):
-- {
--   "max_students": int, "storage_gb": int, "bandwidth_gb": int,
--   "max_courses": int, "max_team": int, "max_affiliates": int,
--   "take_rate_bps": int,          -- ex.: 200 = 2,0% (basis points, sem float)
--   "features": ["order_bump","watermark","forum","api", ...]
-- }
-- "ilimitado" = ausência da chave numérica ou valor null (fair-use).

-- recipient da PLATAFORMA (Pagar.me) vive no control plane; recipients de produtor/afiliado no data plane.
platform_payment_recipients(
  id uuid pk, provider text,                         -- pagarme
  recipient_id_encrypted bytea, created_at timestamptz
)

-- O take_rate_bps chega ao use-case de split SEM violar isolamento: resolvido no onRequest junto com
-- request.tenant (lookup em platform.tenants → plan_id → platform_plans.limits.take_rate_bps) e injetado
-- como valor no contexto. O use-case (no schema do tenant) NÃO consulta o schema platform diretamente.
-- Ver ADR-0013 §"take rate sem violar isolamento".
```

### 6.12 Quotas como serviço (control plane + uso corrente)
- **Serviço de quotas** central (em `packages/` ou módulo `identity`/control plane) lê
  `platform_plans.limits` e o **uso corrente** (alunos ativos e nº de cursos/equipe/afiliados contados no
  schema do tenant; storage/banda consultados na **API da Bunny**). Autoriza/bloqueia ações conforme a
  política de overage (MONETIZATION §A.4). **Não** introduz FK cross-schema: o uso do tenant é lido via
  `withTenant`; os limites, via control plane; a decisão é composta no use-case.

### 6.13 Aulas ao vivo [F2]

> Spec completa em [LIVE_CLASSES.md §15](product/LIVE_CLASSES.md); decisão em
> [ADR-0015](adr/0015-live-classes-interactive.md). Todas as tabelas vivem no **schema do tenant**, acessadas
> via `withTenant`. **Nenhuma FK cruza schemas.** Cada tabela com dado de tenant exige **teste de isolamento
> cross-tenant** (gate de CI, CLAUDE.md). A gravação **reusa** `lessons.video_guid`/`lessons.video_status` e
> `lesson_progress` (anti-seek) — **sem** tabela nova de VOD (DRY). A aula ao vivo é uma `lessons.type='live'`
> (valor já previsto em §2.2). As keys do LiveKit por tenant ficam em `platform.tenants.live_keys_encrypted`
> (control plane), cifradas — **1 projeto LiveKit por tenant** (isolamento análogo à Video Library da Bunny).

```sql
live_sessions(
  id uuid pk,
  lesson_id uuid fk -> lessons,            -- aula type='live' (1 aula ↔ 1..N sessões)
  course_id uuid fk -> courses,            -- desnormalizado p/ escopo/consulta (mesmo schema)
  host_user_id uuid fk -> users,           -- instrutor host
  title text,
  mode text,                               -- interactive | host_mostly        (broadcast = F3)
  scheduled_start_at timestamptz,
  scheduled_end_at timestamptz null,
  timezone text not null default 'America/Sao_Paulo',
  started_at timestamptz null,             -- room_started
  ended_at timestamptz null,               -- room_finished
  status text not null default 'scheduled',
    -- scheduled | lobby | live | ended | canceled  (vocabulário canônico §6.0)
  recording_enabled boolean not null default true,
  recording_status text not null default 'none',
    -- none | recording | processing | ready | failed  (espelha lessons.video_status na fase de VOD)
  recording_provider_ref text null,        -- egress_id do provedor (idempotência/observabilidade)
  recording_master_key text null,          -- chave do MP4 master no R2 (<tenantId>/<sessionId>/...)
  room_provider_ref text null,             -- nome/sid da sala no provedor
  chat_locked boolean not null default false,
  attendance_completes_lesson boolean not null default false, -- toggle (LIVE_CLASSES §8)
  peak_participants int not null default 0,
  created_at timestamptz, updated_at timestamptz, deleted_at timestamptz null
)
-- índices: (lesson_id), (course_id, scheduled_start_at), (status)

live_attendance(
  id uuid pk,
  live_session_id uuid fk -> live_sessions,
  user_id uuid fk -> users,
  enrollment_id uuid fk -> enrollments null,   -- null p/ host/staff
  role text,                                    -- host | participant
  joined_at timestamptz not null,
  left_at timestamptz null,                     -- fechado por participant_left/room_finished
  created_at timestamptz
)
-- índices: (live_session_id), (user_id), (live_session_id, user_id)
-- 1 linha por intervalo de presença (reconexão = novo intervalo); attended_seconds é derivado

live_chat_messages(
  id uuid pk,
  live_session_id uuid fk -> live_sessions,
  user_id uuid fk -> users,
  body text,
  kind text not null default 'chat',            -- chat | question (Q&A) | system
  answered_at timestamptz null,                 -- Q&A: marcada como respondida pelo host
  answered_by uuid null,                         -- fk -> users
  hidden_at timestamptz null,                    -- moderação (espelha comment_* §6.3)
  hidden_by uuid null,                           -- fk -> users
  created_at timestamptz
)
-- índices: (live_session_id, created_at), (live_session_id, kind)
-- chat efêmero usa o data channel do provedor; persistência aqui é p/ histórico/replay/moderação

live_bans(
  id uuid pk,
  course_id uuid fk -> courses null,             -- ban por curso (todas as sessões) ...
  live_session_id uuid fk -> live_sessions null, -- ... ou por sessão
  user_id uuid fk -> users,
  banned_by uuid fk -> users,
  reason text null,
  created_at timestamptz
)
-- a emissão de token de sala consulta live_bans (defense in depth)

live_recording_events(                            -- idempotência de webhooks live (espelha payment_events)
  id uuid pk,
  provider text,                                  -- livekit (ou outro)
  event_id text unique,                           -- egress_id/event id do provedor (dedup)
  live_session_id uuid null,
  payload jsonb,
  processed_at timestamptz null,
  created_at timestamptz
)
```

### 6.14 Relacionamentos adicionais (texto)
```
[ dentro de cada schema tenant_<slug> ]
lesson_comments 1───* comment_likes ; lesson_comments 1───* comment_reports
users 1───* gamification_xp_ledger ; users 1───1 gamification_user_state ; users *───* badges (user_badges)
enrollments 1───* enrollment_milestones
orders 1───* order_items ; orders *───1 bundles (via order_items) ; bundles 1───* bundle_items *───1 courses
affiliates 1───* affiliate_clicks ; affiliates *───* courses (affiliate_course_commissions)
users 1───* notifications ; users 1───* notification_preferences
tenant 1───1 tenant_settings ; tenant 1───1 affiliate_program_settings
lessons 1───* live_sessions *───1 users (host) ; live_sessions 1───* live_attendance *───1 users
live_sessions 1───* live_chat_messages *───1 users ; live_sessions 1───* live_bans *───1 users
live_sessions 1───* live_recording_events
support_tickets 1───* support_messages ; support_tickets *───1 users (requester/assignee)   [v1.3, §6.15]
kb_articles (standalone; índice pg_trgm para busca)                                          [v1.3, §6.15]
import_jobs 1───* import_rows ; import_jobs/export_jobs *───1 users (created_by/requested_by) [v1.3, §6.16]
[ control plane ]
platform.platform_payment_recipients (recipient da plataforma)
platform.tenants.live_keys_encrypted (keys do LiveKit por tenant; sem FK cross-schema)
platform.support_tickets (suporte B2B Nível 2; tenant_id sem FK cross-schema)                [v1.3, §6.15]
platform.audit_log.action (vocabulário canônico do console SA — §6.17)                        [v1.3]
platform.tenants.{custom_domain_status, custom_domain_verify_token, email_sender_domain}      [v1.3, §6.18]
```

### 6.15 Suporte ao aluno — KB e tickets nativos [MVP]

> Origem: [SUPPORT_DISCOVERY_SETTINGS §1/§2](product/SUPPORT_DISCOVERY_SETTINGS.md). **Proposta** a consolidar
> por migration. Tabelas no **schema do tenant** (Nível 1 — aluno↔tenant), via `withTenant`. O suporte B2B
> (Nível 2 — tenant↔plataforma) vive em `platform.support_tickets` (control plane), **sem FK cross-schema**.
> **Build nativo no MVP**; widget externo (Crisp/Intercom) atrás da port `SupportProvider` é **F2** e, por
> mover dados de aluno para fora do tenant, **exige ADR + revisão LGPD** (OPEN_QUESTIONS #36).

```sql
support_tickets(
  id uuid pk, requester_id uuid fk -> users,
  subject text, status text,             -- open | pending | resolved | closed  (§6.0)
  priority text not null default 'normal',-- low | normal | high
  context jsonb null,                     -- { course_id?, order_id?, lesson_id? } (botão "Preciso de ajuda" contextual)
  assignee_id uuid null fk -> users,
  created_at timestamptz, updated_at timestamptz, resolved_at timestamptz null
)
support_messages(
  id uuid pk, ticket_id uuid fk -> support_tickets, author_id uuid fk -> users,
  body text, is_staff boolean,            -- distingue resposta da equipe vs aluno
  created_at timestamptz
)
kb_articles(
  id uuid pk, slug text unique, category text,
  title text, body text, status text,     -- draft | published  (§6.0)
  created_at timestamptz, updated_at timestamptz
  -- índice GIN/pg_trgm sobre (title, body) p/ busca (§6.0 da busca; ver §6.16-busca abaixo)
)
```
- **Eventos novos** `support_ticket_*` e categoria `support` em `notification_preferences` (NOTIFICATIONS_MATRIX
  dep. #6) — registrar no enum canônico de eventos em `packages/contracts`.
- **Busca (`pg_trgm`):** índices GIN/GiST criados nas migrations de cada `tenant_*` **e** ao provisionar novos
  tenants. FTS `tsvector` PT-BR a confirmar (MVP vs F2). Motor externo (Meilisearch/OpenSearch) exige **ADR** e
  índice **isolado por tenant** (Regra nº1) — OPEN_QUESTIONS #38.
- **Isolamento:** PII de tickets/mensagens fora de logs; **teste cross-tenant obrigatório** (tenant A não vê
  tickets/artigos de B).

### 6.16 Importação / Exportação e portabilidade [MVP base / F2 avançado]

> Origem: [DATA_IMPORT_EXPORT §2.1](product/DATA_IMPORT_EXPORT.md). **Proposta** a consolidar por migration +
> **ADR "Import/Export & Portabilidade"** (OPEN_QUESTIONS #39). Tabelas no **schema do tenant**, via
> `withTenant`. Arquivos (origem/relatórios/artefatos) em **Cloudflare R2** sempre sob prefixo `<tenantId>/`,
> com **URL assinada (TTL curto)** e expiração. MVP cobre **alunos + matrículas + estrutura de curso**;
> **vídeo/progresso/pedidos históricos e conectores por API são F2**.

```sql
import_jobs(
  id uuid pk,
  kind text,                              -- students | courses | enrollments | progress | orders | videos
  source_provider text null,              -- hotmart | kiwify | eduzz | teachable | generic_csv
  status text,                            -- §6.0 (uploaded..done|failed|canceled)
  source_file_key text,                   -- CSV/ZIP no R2: <tenantId>/imports/<jobId>/source.csv
  mapping jsonb, options jsonb, totals jsonb,
  report_file_key text null,              -- relatório de erros (CSV) no R2
  created_by uuid fk -> users,            -- Owner/Admin
  created_at, updated_at, finished_at timestamptz null
)
import_rows(                              -- 1 linha por registro (idempotência + auditoria do lote)
  id uuid pk, job_id uuid fk -> import_jobs,
  row_number int, natural_key text,        -- ex.: lower(trim(email)) | course.slug | (email|course_slug)
  status text,                            -- §6.0 (valid|invalid|created|updated|skipped|error)
  errors jsonb null,                       -- [{ field, code, message }]
  unique(job_id, row_number)
)
export_jobs(
  id uuid pk,
  scope text,                             -- tenant | student
  subject_ref text null,                  -- user_id quando scope=student
  format text,                            -- zip_csv | json
  status text,                            -- §6.0 (queued|running|ready|expired|failed)
  artifact_key text null,                 -- R2: <tenantId>/exports/<jobId>/export.zip
  expires_at timestamptz null,            -- link expira (TTL 24–72h a confirmar)
  requested_by uuid fk -> users,
  created_at, finished_at timestamptz null
)
```
- **Idempotência:** `import_jobs.id` é a `idempotency_key` do lote (retoma do checkpoint); a `natural_key` da
  linha torna a reaplicação no-op (reupload do mesmo CSV após falha parcial → linhas já `created` viram
  `skipped`). Commit em chunks (~500 linhas/tx).
- **Validação** via Zod em `packages/contracts` (DRY); **não importamos senhas** (aluno define por link/social).
- **Esquecimento/offboarding (LGPD):** export gera artefato no R2; o `DROP SCHEMA` do offboarding deve **também**
  purgar **R2** e **Bunny Library** do tenant (não só o Postgres) — ver COMPLIANCE §dep.4/§9 e OPEN_QUESTIONS #24.
- **Eventos novos** `import_ready`/`import_done`, `export_ready`, `data_erasure_done` → registrar na
  NOTIFICATIONS_MATRIX e no enum de eventos.
- **Isolamento:** prefixo R2 por tenant; nada de chave de export reutilizável entre tenants; **teste cross-tenant
  obrigatório**.

### 6.17 Control plane — vocabulário de `platform.audit_log.action` [MVP]

> Origem: [SUPER_ADMIN_CONSOLE §15](product/SUPER_ADMIN_CONSOLE.md). **Proposta** a virar **contrato canônico
> Zod** em `packages/contracts` (DRY), junto com os papéis de SA (`sa_ops | sa_support | sa_billing |
> sa_owner`). A tabela `platform.audit_log` já existe (§1); aqui se padroniza o domínio do campo `action`
> (`actor_type='super_admin'`). Toda ação sensível grava `audit_log`; impersonação é **sempre auditada**.

| Domínio | `action` (valores canônicos) | Step-up MFA |
|---------|------------------------------|-------------|
| Tenant | `tenant.created` · `tenant.provisioning.retried` · `.smoke_retested` · `.aborted` · `tenant.suspended` · `tenant.reactivated` · `tenant.cancelled` · `tenant.purged` | suspend/cancel/purge |
| Quota | `quota.override.granted` | — |
| Impersonação | `tenant.impersonation.started` · `.action` · `.ended` | início |
| Billing SaaS | `saas.subscription.plan_changed` · `saas.invoice.adjusted` · `saas.billing.credit_granted` · `saas.takerate.override` · `saas.trial.adjusted` | — |
| Plano | `plan.updated` · `plan.takerate.changed` · `plan.feature_override.set` | sim |
| Integração | `integration.key.rotated` · `integration.key.revealed` · `integration.reprovisioned` | reveal/reprov |
| Suporte | `support.note.added` · `support.action.performed` | — |
| Equipe SA | `super_admin.created` · `.updated` · `.disabled` · `.role_changed` | — |

> Overrides de feature-flag por tenant (piloto/exceção) exigiriam nova tabela/coluna no control plane —
> decidir MVP vs F2 (SUPER_ADMIN_CONSOLE dep. #3).

### 6.18 Control plane + branding — domínio próprio e e-mail próprio [MVP favicon / F2 domínio]

> Origem: [BRANDING_WHITELABEL §9.2](design/BRANDING_WHITELABEL.md). **Proposta** a consolidar por migration +
> ADR. Os campos de **branding/white-label** ficam em `tenant_settings` (data plane — ver §6.10); os que
> **governam roteamento/verificação** ficam em `platform.tenants` (control plane). **Sem FK cross-schema.**

```sql
-- platform.tenants (control plane) — PROPOSTA (apenas o que governa roteamento/verificação de domínio/e-mail)
custom_domain_status text null,           -- none | pending | verifying | active | failed  (§6.0)  [F2]
custom_domain_verify_token text null,     -- TXT de verificação de propriedade do domínio          [F2]
email_sender_domain text null             -- domínio verificado p/ From próprio (DKIM/SPF) — Pro só após domínio active [F2]
```
- **MVP:** apenas `tenant_settings.favicon_url`/`logo_dark_url`/`email_reply_to` (§6.10) — favicon e reply-to
  white-label. **Domínio próprio + SSL** (ACME na edge) e **e-mail de domínio próprio** são **F2**
  (engenharia: sessão Better-Auth válida em `slug.app.com` **e** `custom_domain`; 301 ao trocar host).
- **Booleanos/flags** (`whitelabel_full`, `custom_domain`, `email_sender_domain`) consumidos por **injeção no
  `onRequest`** (ADR-0013), nunca pelo use-case direto.
- **MVP de envio de e-mail** = **domínio compartilhado verificado** (From com `{tenant_name}`); domínio próprio
  por tenant = F2 (OPEN_QUESTIONS #32).
