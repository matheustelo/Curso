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

## Como adicionar um ADR
1. Copie o formato de um ADR existente. 2. Numere sequencialmente. 3. Referencie ADRs que ele
substitui/complementa. 4. Atualize esta tabela. 5. Decisões que mudam o PRD devem refletir lá também.
