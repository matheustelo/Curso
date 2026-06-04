# Arquitetura Técnica — Plataforma de Cursos SaaS Multitenant

- **Versão:** 1.0 · **Data:** 2026-06-04 · **Status:** Aprovado (≥95%)
- **Relacionados:** [PRD.md](PRD.md) · [DATA_MODEL.md](DATA_MODEL.md) · [ADRs](adr/) · [CLAUDE.md](../CLAUDE.md)

Princípios não-negociáveis: **SOLID, DRY, Clean Code**, stack **leve**, codebase **previsível e
type-safe** otimizada para **desenvolvimento guiado por IA**.

---

## 1. Visão geral da stack

| Camada | Tecnologia | ADR |
|--------|-----------|-----|
| Runtime / Linguagem | Node.js 22 LTS · TypeScript `strict` (ESM) | — |
| Backend | **Fastify** (Clean Architecture leve, vertical slices) | [0002](adr/0002-backend-fastify.md) |
| ORM / query layer | **Drizzle ORM** | [0003](adr/0003-orm-drizzle.md) |
| Validação / contratos | **Zod** + `fastify-type-provider-zod` (+ OpenAPI gerado) | [0004](adr/0004-contracts-zod.md) |
| Frontend | **Next.js** (App Router/RSC) · TanStack Query · shadcn/ui · Tailwind · next-intl | [0005](adr/0005-frontend-nextjs.md) |
| Autenticação | **Better-Auth** (self-hosted, cookies HttpOnly, RBAC por tenant) | [0006](adr/0006-auth-better-auth.md) |
| Banco | **PostgreSQL** (schema-per-tenant) + PgBouncer + pgvector | [0001](adr/0001-multitenancy-schema-per-tenant.md) |
| Filas / jobs | **pg-boss** (no Postgres) | [0011](adr/0011-queue-pgboss.md) |
| Streaming | **Bunny Stream** (1 Video Library por tenant) | [0008](adr/0008-streaming-bunny.md) |
| Arquivos / certificados | **Cloudflare R2** (egress zero) | [0012](adr/0012-supporting-platform.md) |
| E-mail transacional | **Resend** | [0012](adr/0012-supporting-platform.md) |
| Observabilidade | **Pino** + **OpenTelemetry** + **Sentry** | [0012](adr/0012-supporting-platform.md) |
| Pagamentos | **Pagar.me** (principal, split) + **Asaas** (Pix/boleto recorrente) | [0010](adr/0010-payments-br.md) |
| Repo / tooling | Monorepo **pnpm + Turborepo** · **Biome** · **Vitest/Playwright/Testcontainers** | [0007](adr/0007-monorepo.md) |

---

## 2. Diagrama de alto nível

```
                         ┌──────────────────────────────────────────┐
                         │              CLIENTES (browsers)            │
                         │   acme.app.com        escola2.app.com       │
                         └───────────────────┬────────────────────────┘
                                             │ HTTPS
                         ┌───────────────────▼────────────────────────┐
                         │   FRONTEND — Next.js (App Router / RSC)      │
                         │   middleware.ts resolve tenant (subdomínio)  │
                         │   TanStack Query · shadcn/ui · next-intl     │
                         │   Better-Auth client (cookies HttpOnly)      │
                         └───────────────────┬────────────────────────┘
                                             │ REST tipado (Zod) + OpenAPI
                         ┌───────────────────▼────────────────────────┐
                         │     BACKEND — Fastify (containers)           │
                         │  onRequest: resolve tenant → search_path     │
                         │  [http adapter] → [use-case] → [port]        │
                         │  Auth (Better-Auth) · Pino · OTel · RBAC     │
                         └───┬───────────────────┬──────────────┬──────┘
                             │                   │              │
            ┌────────────────▼──────┐   ┌────────▼─────────┐  ┌─▼────────────────┐
            │ PostgreSQL (1 banco)   │   │ Bunny Stream     │  │ Pagar.me / Asaas  │
            │  schema "platform"     │   │ 1 library/tenant │  │ (checkout/split)  │
            │  schema "tenant_acme"  │   │ token + webhooks │  └─┬─────────────────┘
            │  schema "tenant_esc2"  │   └────────┬─────────┘    │ webhook pagamento
            │  (via PgBouncer)       │            │ webhook ready │
            └────────────┬───────────┘           │              │
                         │  ┌──────────────────────▼──────────────▼─────┐
                         └─▶│   WORKER — pg-boss (fila no Postgres)       │
                            │   e-mails · certificados · webhooks · IA    │
                            │   reusa o resolvedor de tenant (search_path)│
                            └─────────────────────────────────────────────┘
                       Apoio: Cloudflare R2 (arquivos/certificados) · Resend (e-mail)
                              PostHog (analytics) · pgvector (RAG/IA)
```

---

## 3. Modelo de multitenancy (schema-per-tenant) — decisão central

> **Decisão:** 1 banco PostgreSQL. Um **schema por tenant** (`tenant_<slug>`) + um schema de
> **control plane** (`platform`). Escala-alvo: dezenas de tenants (até ~100). Ver [ADR-0001](adr/0001-multitenancy-schema-per-tenant.md).

### 3.1 Organização de schemas
- **`platform`** (control plane): catálogo de tenants, planos do SaaS, assinaturas/faturas do SaaS,
super-admins, jobs de provisionamento, mapa de roteamento, mapeamento `tenant → Bunny library`.
- **`tenant_<slug>`** (data plane): usuários do tenant, cursos, módulos, aulas, matrículas, progresso,
pedidos, assinaturas dos alunos, certificados, afiliados, comunidade.
- **`public`**: somente extensões compartilhadas (`pgcrypto`, `vector`, `pg_trgm`).

### 3.2 Resolução de tenant (request → schema)
1. **Identificação:** subdomínio (`acme.app.com`) é a fonte primária; resolvido contra `platform.tenants`.
2. **Autoridade:** o **claim do JWT/sessão** (`tenant_id`) é a fonte de verdade final — o subdomínio
apenas sugere; o claim assinado confirma (impede troca de tenant via header).
3. **Aplicação:** hook `onRequest` do Fastify resolve o tenant, valida contra a sessão e decora a
request com o contexto (`request.tenant`).

### 3.3 Conexões e o "padrão de ouro" anti-vazamento
- **Um único pool** de conexões compartilhado + **PgBouncer** à frente (pool da app pequeno, 5–10
conexões por instância). Não há pool por tenant — vantagem decisiva do schema-per-tenant.
- **Padrão obrigatório:** **todo** acesso a dados ocorre dentro de uma transação que começa com
`SET LOCAL search_path TO tenant_<slug>, public;`. `SET LOCAL` é transacional e **compatível com o
transaction pooling** do PgBouncer — evita o vazamento cross-tenant clássico (search_path "vazando"
entre conexões backend reaproveitadas).
- **Nunca** confiar em estado de sessão herdado do pooler nem usar nomes não-qualificados fora de uma
transação com `search_path` setado.
- Implementação centralizada em `packages/db` (`withTenant(tenantId, fn)`), usada por todos os
repositórios — **DRY** e auditável.

### 3.4 Provisionamento de tenant (saga idempotente)
Executado por um job no worker, com estado em `platform.provisioning_jobs`:
1. `INSERT` tenant (`status='provisioning'`, `idempotency_key`).
2. `CREATE SCHEMA IF NOT EXISTS tenant_<slug>;`
3. Rodar **migrations** no schema (Drizzle migrator com `search_path`).
4. **Seed** (papéis, admin do tenant, curso de exemplo).
5. Criar **Bunny Video Library** do tenant e guardar `library_id` + keys cifradas.
6. Registrar roteamento + `status='active'` + **smoke test**.
Idempotência: `IF NOT EXISTS`, migrations versionadas, retry com estado persistido.

### 3.5 Migrations multitenant
- Uma única definição de schema Drizzle (igual para todos os tenants).
- Runner aplica as migrations **em loop sobre todos os schemas `tenant_*`** + no `platform` (job de CI
dedicado, idempotente, com tracking por schema). Com dezenas de tenants isso é trivial e rápido.

### 3.6 LGPD / exclusão
- Exclusão de tenant: `DROP SCHEMA tenant_<slug> CASCADE` (forte para auditoria).
- Direito ao esquecimento de aluno: deleção/anonimização dentro do schema do tenant.

---

## 4. Arquitetura do backend (Fastify) — Clean Architecture leve

### 4.1 Filosofia
Vertical slices por feature, com 3 camadas internas e **regra de dependência para dentro**
(domain ← application ← infrastructure). DI **manual leve** (sem container mágico) → previsível p/ IA.

```
domain  →  application (use-cases)  →  infrastructure (db, http, bunny, pagamentos)
        ↖__________ dependência aponta para dentro (Dependency Rule) __________↗
```

- **Domain:** entidades, value objects, regras puras. **Zero** imports de framework/ORM.
- **Application:** use-cases (1 classe = 1 caso de uso → **SRP**), dependem de **ports** (interfaces) → **DIP**.
- **Infrastructure:** repositórios Drizzle, clientes Bunny/pagamento, rotas Fastify (adapters).
- **Composição:** `composeModule()` por módulo monta as dependências (DI manual).

### 4.2 Estrutura de pastas (`apps/api`)
```
apps/api/src/
├─ main.ts                      # bootstrap
├─ app.ts                       # registra plugins + módulos
├─ plugins/
│  ├─ tenant.plugin.ts          # onRequest → resolve tenant + search_path
│  ├─ auth.plugin.ts            # Better-Auth + guards RBAC
│  ├─ error-handler.plugin.ts   # erros de domínio → Problem Details (RFC 9457)
│  └─ observability.plugin.ts   # Pino + OTel + requestId/tenantId
├─ modules/
│  ├─ courses/
│  │  ├─ domain/                # course.entity.ts, course.errors.ts
│  │  ├─ application/
│  │  │  ├─ ports/course.repository.ts      # interface (DIP)
│  │  │  ├─ create-course.usecase.ts        # 1 use-case = 1 classe (SRP)
│  │  │  └─ list-courses.usecase.ts
│  │  ├─ infrastructure/
│  │  │  ├─ course.drizzle.repository.ts    # implementa o port
│  │  │  └─ course.routes.ts                # adapter HTTP (schemas Zod)
│  │  ├─ courses.module.ts                  # composeModule()
│  │  └─ __tests__/
│  ├─ enrollments/   ├─ payments/   ├─ video/   ├─ certificates/
│  ├─ community/     ├─ affiliates/ └─ identity/ (provisionamento + super-admin)
└─ shared/                       # ports/utilitários transversais
```

### 4.3 Tratamento de erros
Hierarquia de erros de domínio (`DomainError`, `NotFoundError`, `ForbiddenError`,
`PaymentError`...) → **um único** error handler Fastify mapeia para HTTP/RFC 9457 (Problem Details).
**DRY + Clean.**

---

## 5. Frontend (Next.js)

- **App Router + RSC.** Páginas públicas (landing/catálogo por tenant) → SSG/ISR; área logada
(player, progresso) → RSC + SSR; partes interativas (player, quiz) → Client Components.
- **Multitenancy:** `middleware.ts` resolve tenant por subdomínio e injeta no contexto; branding/tema
por tenant via CSS variables.
- **Data fetching:** TanStack Query (estado de servidor no client) + `fetch` em RSC para carga inicial.
- **Estado:** Zustand apenas para o pouco estado global de UI; o resto é TanStack Query + `useState`.
- **UI:** shadcn/ui + Tailwind. **i18n:** next-intl. **Auth:** cliente Better-Auth (mesma lib do back).
- **Estrutura:** `app/` (rotas) + `features/` (vertical slices) + `components/ui` (shadcn) + `lib/`.

---

## 6. Autenticação e autorização

- **Better-Auth** self-hosted no backend Fastify. Sessões via **cookies HttpOnly + Secure**.
- **Usuários por tenant:** os usuários (alunos/instrutores/admin do tenant) vivem no **schema do
tenant**; o mesmo e-mail pode existir em tenants diferentes (modelo "workspace"). Super-admins vivem
no `platform`.
- **RBAC por tenant:** papéis `owner`, `admin`, `instructor`, `affiliate`, `student`. Autorização
aplicada em **guards/hooks Fastify** (ponto único → DRY) e revalidada nos use-cases (defense in depth).
- **Super-admin:** identidade global no `platform`, com **impersonação auditada**.
- **Risco/PoC:** os exemplos do Better-Auth assumem DB único; faremos **PoC** do fluxo
auth + resolução de tenant + `search_path` antes de escalar. Plano B: auth próprio no Fastify
(`@fastify/jwt` + `@fastify/cookie` + argon2). Ver [ADR-0006](adr/0006-auth-better-auth.md).

---

## 7. Integração de vídeo (Bunny Stream)

- **Isolamento:** **1 Video Library por tenant** (AccessKey, Pull Zone e token key próprios),
provisionada via API no onboarding. Mapeamento `tenant → library_id + keys cifradas` no `platform`.
- **Upload:** **pre-signed + TUS (resumable)** direto do browser ao edge da Bunny (API key nunca no
front; sem consumir banda do backend).
- **Reprodução segura:** backend valida **entitlement** (matrícula ativa) → gera **Embed Token**
(`SHA256(key + videoId + expires)`, TTL curto 1–12h) → front renderiza o player.
- **Webhooks "vídeo pronto":** endpoint Fastify verifica **HMAC** (bytes brutos, comparação em tempo
constante) → responde 200 → **enfileira** job → worker mapeia `VideoLibraryId → tenant` e atualiza o
status. Idempotente.
- **Tracking de progresso:** player.js emite `timeupdate` (throttle 5–15s) → `POST /progress` →
`lesson_progress`. Conclusão: ≥90% com checagem anti-seek.
- **Anti-pirataria (faseado):** ver [ADR-0009](adr/0009-anti-piracy.md). `VideoProvider` é uma **port**
abstrata (SOLID) — permite trocar/estender (player próprio, DRM Enterprise) sem tocar nos use-cases.

---

## 8. Pagamentos (BR + afiliados/split)

- **Abstração `PaymentProvider`** (interface SOLID) desacopla o domínio do gateway.
- **Pagar.me** como principal (split robusto p/ afiliados/co-produção); **Asaas** para Pix/boleto e
cobrança recorrente de baixo custo. Seleção por estratégia/feature flag.
- **Checkout próprio** (Pix/boleto/cartão, cupom; order bump/upsell na F2).
- **Webhooks de pagamento:** verificação de assinatura + **idempotência** (chave do evento) →
máquina de estados de acesso (aprovado → libera; reembolso/chargeback → suspende).
- **Reconciliação periódica** com o gateway (job agendado) para corrigir divergências.
- **Billing do SaaS** (tenant paga a plataforma): **Stripe Billing** no control plane; status da
assinatura ativa/suspende o tenant. Domínio **separado** do checkout dos alunos.

---

## 9. Background jobs / filas

- **pg-boss** (fila no próprio PostgreSQL — sem Redis, alinhado a "leve"). Casos: e-mails,
certificados (Puppeteer/pdfme), processamento de webhooks (Bunny/pagamento), transcrição/IA,
reconciliação. Volume moderado (Bunny faz o trabalho pesado de mídia).
- **Interface `JobQueue` (port):** troca para BullMQ/Redis no futuro sem tocar nos use-cases (**DIP**).
- **Multitenant nas filas:** todo job carrega `tenantId` no payload; o worker resolve o schema igual ao
request HTTP (`withTenant`).

---

## 10. Observabilidade, logging e erros

- **Pino:** logs JSON estruturados; **sempre** com `tenantId` + `requestId` (via `AsyncLocalStorage`).
- **OpenTelemetry:** traces/metrics (Fastify + pg + chamadas Bunny/pagamento).
- **Sentry:** exceções front e back, com `tenantId` como tag.
- **Erros:** Problem Details (RFC 9457) em um único handler.

---

## 11. Testes

- **Vitest** (unit/integration) + **Playwright** (e2e). **Testcontainers** para Postgres real.
- **Unit:** use-cases e domínio com ports mockados (a arquitetura DIP torna isto trivial).
- **Integration:** repositórios Drizzle + Fastify contra Postgres real; cada teste cria um **schema de
tenant efêmero**, roda migrations e faz teardown.
- **Teste de isolamento (crítico):** provar que dados do tenant A **nunca** aparecem para o tenant B
(o bug mais perigoso do modelo). É um gate de CI.

---

## 12. Infraestrutura e deploy

- **Frontend:** Vercel (App Router/RSC, preview deploys).
- **Backend + worker:** containers em **Fly.io** ou **Railway** (API e worker como processos da mesma
imagem). 
- **PostgreSQL gerenciado** (Fly Postgres / Neon / Supabase) + **PgBouncer/Supavisor**.
- **CI/CD:** GitHub Actions + Turborepo cache → Biome → typecheck → Vitest (Testcontainers) → build →
deploy → **job de migrations multitenant** (loop idempotente sobre schemas).
- **Segredos:** secret manager; keys de tenant (Bunny/pagamento) **cifradas em repouso** (pgcrypto/KMS).

---

## 13. Como a arquitetura serve a SOLID / DRY / Clean Code

- **S (SRP):** 1 use-case por arquivo/classe; controllers só adaptam HTTP; repositórios só persistem.
- **O (Open/Closed):** novas features = novos módulos/use-cases; novos providers via novos adapters.
- **L (Liskov):** ports definem contratos; implementações substituíveis (Drizzle, Pagar.me→Asaas, Bunny).
- **I (ISP):** ports pequenos e focados (`CourseRepository`, `JobQueue`, `VideoProvider`, `PaymentProvider`).
- **D (DIP):** use-cases dependem de interfaces, não de Drizzle/Fastify/Bunny → testes triviais.
- **DRY:** `packages/contracts` (Zod) como fonte única de tipos/validação front+back; um error handler;
um `withTenant`; um resolvedor de tenant; factories de teste reutilizadas.
- **Clean Code:** convenções explícitas + Biome (formatação determinística) + boundaries por lint +
ADRs + `CLAUDE.md`. **Previsibilidade máxima para agentes de IA.**

---

## 14. Configuração de TypeScript (resumo)

`tsconfig.base.json` com `strict: true`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`,
`verbatimModuleSyntax`, `isolatedModules`, `module/moduleResolution: NodeNext`. Estes flags forçam
código sem ambiguidade — o que mais ajuda agentes de IA a acertarem. Dev com `tsx`; build com `tsup`.

---

_Detalhes e justificativas de cada decisão estão nos [ADRs](adr/). O modelo de dados completo está em
[DATA_MODEL.md](DATA_MODEL.md)._
