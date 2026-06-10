# Plataforma de Cursos Online (SaaS Multitenant)

> Plataforma SaaS de cursos online **multitenant** (schema-per-tenant em PostgreSQL), com
> streaming de vídeo via **Bunny.net**, checkout brasileiro (Pix/boleto/cartão) com afiliados e
> split, backend **Fastify** (Node/TypeScript) e frontend **Next.js**. Projetada para ser
> **leve**, **type-safe** e **desenvolvida com apoio de agentes de IA**, seguindo **SOLID, DRY e
> Clean Code** de forma não-negociável.

Este repositório contém, neste momento, a **especificação completa do produto e da arquitetura**
(fase de design). A implementação seguirá os documentos abaixo.

## 📚 Documentação

| Documento | Conteúdo |
|-----------|----------|
| [`docs/PRD.md`](docs/PRD.md) | **Product Requirements Document** — visão, personas, funcionalidades por domínio, regras de negócio, métricas. |
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | **Arquitetura técnica completa** — stack, multitenancy, camadas, integrações, segurança, infra. |
| [`docs/DATA_MODEL.md`](docs/DATA_MODEL.md) | **Modelo de dados** — control plane (`platform`) vs tenant data plane (`tenant_<slug>`). |
| [`docs/ROADMAP.md`](docs/ROADMAP.md) | **Roadmap MoSCoW** — MVP → Fase 2 → Fase 3, com escopo de cada fase. |
| [`docs/OPEN_QUESTIONS.md`](docs/OPEN_QUESTIONS.md) | Pontos residuais a validar (ex.: custo de egress BR, PoC de auth). |
| [`docs/GAP_ANALYSIS.md`](docs/GAP_ANALYSIS.md) | **Análise de gaps (red team)** — achados de 4 revisores adversariais (🔴/🟠/🟡), inconsistências entre docs e plano de correção em ondas. **Leitura obrigatória antes de implementar.** |
| [`docs/adr/`](docs/adr/) | **Architecture Decision Records** — cada decisão-chave registrada com contexto, opções e justificativa. |
| [`docs/product/`](docs/product/README.md) | **Especificação detalhada de produto** — jornadas, flows, IA, UX de aprendizado, user stories, regras de negócio/estados, RBAC, notificações, monetização, analytics, NFRs e os novos docs de e-mail, onboarding/ativação, suporte/descoberta/settings, console super-admin e import/export. Comece pelo [README de produto](docs/product/README.md). |
| [`docs/design/`](docs/design/DESIGN_SYSTEM.md) | **Design & white-label** — [DESIGN_SYSTEM](docs/design/DESIGN_SYSTEM.md) (tokens/componentes/a11y), [WIREFRAMES](docs/design/WIREFRAMES.md), [UX_WRITING](docs/design/UX_WRITING.md) e [BRANDING_WHITELABEL](docs/design/BRANDING_WHITELABEL.md). |
| [`docs/ops/`](docs/ops/SECURITY_AND_OPERATIONS.md) | **Segurança & operações** — backups/PITR (RPO/RTO), criptografia/KMS, rate-limit, CSP, DR/GameDay, FinOps. |
| [`docs/legal/`](docs/legal/COMPLIANCE.md) | **Compliance/LGPD** — papéis Controlador/Operador, retenção, CMP/consentimento, exportação/esquecimento, DPA. |
| [`CLAUDE.md`](CLAUDE.md) | **Guia para desenvolvimento guiado por IA** — convenções, camadas, regra de escopo por tenant. |

## 🎯 Decisões-chave (resumo)

| Tema | Decisão | ADR |
|------|---------|-----|
| Multitenancy | **Schema-per-tenant** (1 banco, 1 schema por tenant) | [ADR-0001](docs/adr/0001-multitenancy-schema-per-tenant.md) |
| Backend | **Fastify** + Clean Architecture leve (vertical slices) | [ADR-0002](docs/adr/0002-backend-fastify.md) |
| ORM | **Drizzle** (não Prisma) | [ADR-0003](docs/adr/0003-orm-drizzle.md) |
| Contratos/validação | **Zod** + `fastify-type-provider-zod` | [ADR-0004](docs/adr/0004-contracts-zod.md) |
| Frontend | **Next.js App Router + RSC** | [ADR-0005](docs/adr/0005-frontend-nextjs.md) |
| Autenticação | **Better-Auth** (self-hosted) | [ADR-0006](docs/adr/0006-auth-better-auth.md) |
| Repositório | **Monorepo pnpm + Turborepo** | [ADR-0007](docs/adr/0007-monorepo.md) |
| Streaming | **Bunny Stream** (1 library/tenant) | [ADR-0008](docs/adr/0008-streaming-bunny.md) |
| Anti-pirataria | **Faseada**: Token+MediaCage → watermark por aluno | [ADR-0009](docs/adr/0009-anti-piracy.md) |
| Pagamentos | **Pagar.me + Asaas (BR)** com split/afiliados | [ADR-0010](docs/adr/0010-payments-br.md) |
| Filas | **pg-boss** (sem Redis) | [ADR-0011](docs/adr/0011-queue-pgboss.md) |
| Plataforma de apoio | R2, Resend, PostHog, OTel/Sentry | [ADR-0012](docs/adr/0012-supporting-platform.md) |

## 🧱 Stack (visão rápida)

- **Runtime:** Node.js 22 LTS · TypeScript `strict` (ESM)
- **Backend:** Fastify · Drizzle ORM · Zod · pg-boss · Pino · OpenTelemetry
- **Frontend:** Next.js (App Router/RSC) · TanStack Query · shadcn/ui · Tailwind · next-intl
- **Banco:** PostgreSQL (schema-per-tenant) + PgBouncer + pgvector
- **Mídia:** Bunny Stream (vídeo) · Cloudflare R2 (arquivos/certificados)
- **Auth:** Better-Auth (cookies HttpOnly, RBAC por tenant)
- **Pagamentos:** Pagar.me / Asaas (BR) atrás de uma abstração `PaymentProvider`
- **Tooling:** pnpm + Turborepo · Biome · Vitest · Playwright · Testcontainers

## 🗂️ Estrutura de pastas planejada

```
curso/
├─ apps/
│  ├─ api/        # Fastify backend (vertical slices + Clean Architecture)
│  ├─ web/        # Next.js frontend
│  └─ worker/     # consumidor pg-boss (e-mails, certificados, webhooks Bunny)
├─ packages/
│  ├─ contracts/  # schemas Zod + tipos compartilhados (DRY)
│  ├─ db/         # schema Drizzle + TenantConnectionManager + migrations
│  ├─ config/     # tsconfig/biome bases + regras de boundaries
│  └─ core/       # utilitários puros (Result, errors, contexto de tenant)
├─ docs/
└─ CLAUDE.md
```

## 📊 Status

- ✅ Pesquisa de mercado e técnica concluída (4 agentes especializados)
- ✅ Decisões-chave consolidadas e formalizadas (PRD + Arquitetura + ADRs)
- ⏳ Próximo: scaffolding do monorepo e PoC de fluxo multitenant + auth + Bunny

---
_Documentação gerada na fase de design. Toda decisão está rastreada em ADRs e baseada em
evidências de pesquisa de mercado e benchmarks técnicos (ver fontes nos próprios ADRs)._
