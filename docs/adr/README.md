# Architecture Decision Records (ADRs)

Cada decisão arquitetural significativa é registrada aqui, com **contexto**, **opções consideradas**,
**decisão**, **justificativa**, **consequências** e **nível de confiança**. Formato inspirado em
Michael Nygard.

| ADR | Decisão | Status | Confiança |
|-----|---------|--------|-----------|
| [0001](0001-multitenancy-schema-per-tenant.md) | Multitenancy: schema-per-tenant | Aceito | 90% |
| [0002](0002-backend-fastify.md) | Backend: Fastify + Clean Architecture leve | Aceito | 88% |
| [0003](0003-orm-drizzle.md) | ORM: Drizzle (não Prisma) | Aceito | 88% |
| [0004](0004-contracts-zod.md) | Contratos/validação: Zod + type provider | Aceito | 88% |
| [0005](0005-frontend-nextjs.md) | Frontend: Next.js App Router/RSC | Aceito | 90% |
| [0006](0006-auth-better-auth.md) | Autenticação: Better-Auth (PoC) | Aceito c/ ressalva | 80% |
| [0007](0007-monorepo.md) | Repositório: monorepo pnpm + Turborepo | Aceito | 92% |
| [0008](0008-streaming-bunny.md) | Streaming: Bunny Stream | Aceito | 88% |
| [0009](0009-anti-piracy.md) | Anti-pirataria: estratégia faseada | Aceito | 85% |
| [0010](0010-payments-br.md) | Pagamentos: Pagar.me + Asaas (BR) | Aceito | 85% |
| [0011](0011-queue-pgboss.md) | Filas: pg-boss | Aceito | 82% |
| [0012](0012-supporting-platform.md) | Plataforma de apoio (R2/Resend/PostHog/OTel) | Aceito | 85% |
| [0013](0013-product-data-model-extensions.md) | Extensões do modelo de dados dirigidas por produto | Aceito | 85% |
| [0014](0014-analytics-provider-tracking-plan.md) | Port `AnalyticsProvider` + tracking plan em contracts | Aceito | 84% |
| [0015](0015-live-classes-interactive.md) | Aulas ao vivo interativas (LiveKit) + gravação→VOD; port `LiveProvider`; revisita ADR-0011 (sem Redis) | Aceito | 82% |

## ADRs pendentes (decisões estruturais a registrar quando adotadas)

> Levantados na integração dos docs de design/produto/ops/legal (2026-06-07). As decisões abaixo **ainda não
> estão maduras** (dependem de stakeholder/engenharia/jurídico) — por isso **não** foram criados ADRs agora; o
> default seguro está aplicado nos docs. Cada item vira **ADR** quando a decisão for tomada (numerar a partir de
> 0016). Rastreados em [OPEN_QUESTIONS](../OPEN_QUESTIONS.md) #32–#40 e no
> [Relatório de Consistência §4 (28–40)](../product/README.md).

| Decisão futura | Default atual (sem ADR ainda) | Gatilho do ADR | OQ |
|----------------|-------------------------------|----------------|----|
| **Port `EmailProvider`/templates** | Resend + React Email; enum `event→template_id` em `packages/contracts` | Formalizar a abstração e o catálogo i18n de e-mail | #32/#33 |
| **Port `SupportProvider` (build-vs-buy)** | Suporte **nativo** no MVP (DATA_MODEL §6.15) | Adotar Crisp/Intercom (dados de aluno saem do tenant → revisão LGPD) | #36 |
| **Pipeline de design tokens + `packages/ui`** | shadcn/Tailwind + tokens semânticos no app | Adotar Style Dictionary + Tokens Studio e pacote compartilhado | #34 |
| **Motor de busca externo** | `pg_trgm` por schema de tenant | Adotar Meilisearch/OpenSearch (índice isolado por tenant) | #38 |
| **Import/Export & Portabilidade** | Tabelas em DATA_MODEL §6.16 (proposta) | Consolidar migration + idempotência/retomada de lote | #39 |
| **Branding/domínio próprio + e-mail próprio** | Campos propostos (DATA_MODEL §6.18, tenant_settings) | Migration + SSL/ACME na edge + cookies multi-host (F2) | #32 |
| **CMP / port de consentimento** | Gating de analytics pendente | Escolher CMP (próprio vs terceiro) e implementar a port | #25/#40 |

> **Observação:** nenhum desses itens introduz FK cross-schema nem viola a Regra nº1; ports e flags são
> resolvidos por injeção no `onRequest` (ADR-0013).

## Como adicionar um ADR
1. Copie o formato de um ADR existente. 2. Numere sequencialmente. 3. Referencie ADRs que ele
substitui/complementa. 4. Atualize esta tabela. 5. Decisões que mudam o PRD devem refletir lá também.
