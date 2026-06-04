# Modelo de Dados — Plataforma de Cursos SaaS Multitenant

- **Versão:** 1.0 · **Data:** 2026-06-04
- **Modelo de isolamento:** schema-per-tenant (ver [ADR-0001](adr/0001-multitenancy-schema-per-tenant.md))
- **Relacionados:** [ARCHITECTURE.md](ARCHITECTURE.md) · [PRD.md](PRD.md)

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
