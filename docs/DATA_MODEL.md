# Modelo de Dados — Plataforma de Cursos SaaS Multitenant

- **Versão:** 1.1 · **Data:** 2026-06-04
- **Modelo de isolamento:** schema-per-tenant (ver [ADR-0001](adr/0001-multitenancy-schema-per-tenant.md))
- **Relacionados:** [ARCHITECTURE.md](ARCHITECTURE.md) · [PRD.md](PRD.md) · [docs/product/](product/README.md) · [ADR-0013](adr/0013-product-data-model-extensions.md) · [ADR-0014](adr/0014-analytics-provider-tracking-plan.md)

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

audit_log(                      -- ações sensíveis (impersonação, suspensão...)
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
  updated_at timestamptz
)
```

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

### 6.13 Relacionamentos adicionais (texto)
```
[ dentro de cada schema tenant_<slug> ]
lesson_comments 1───* comment_likes ; lesson_comments 1───* comment_reports
users 1───* gamification_xp_ledger ; users 1───1 gamification_user_state ; users *───* badges (user_badges)
enrollments 1───* enrollment_milestones
orders 1───* order_items ; orders *───1 bundles (via order_items) ; bundles 1───* bundle_items *───1 courses
affiliates 1───* affiliate_clicks ; affiliates *───* courses (affiliate_course_commissions)
users 1───* notifications ; users 1───* notification_preferences
tenant 1───1 tenant_settings ; tenant 1───1 affiliate_program_settings
[ control plane ]
platform.platform_payment_recipients (recipient da plataforma)
```
