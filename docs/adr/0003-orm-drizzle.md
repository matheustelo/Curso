# ADR-0003 — ORM: Drizzle (não Prisma)

- **Status:** Aceito · **Data:** 2026-06-04 · **Confiança:** 88%

## Contexto
Schema-per-tenant exige **trocar o schema (`search_path`) dinamicamente por request** sobre um único
pool, mantendo type-safety e migrations gerenciáveis. Esta é a decisão de ORM mais condicionada pelo
modelo multitenant.

## Opções consideradas
- **Prisma:** DX excelente, dominante no corpus de IA. **Bloqueio documentado:** Prisma recomenda uma
única instância de `PrismaClient`, cada uma com **seu próprio pool**; trocar database/schema
dinamicamente sem recriar o client não é suportado, e instanciar um client por tenant "multiplica RAM e
esgota conexões". Multi-schema é estático no schema file. → **inadequado** ao nosso modelo.
- **Drizzle ORM:** SQL-first, type-safe, leve (~5KB), schema em TS. `db.withSchema(...)` /
`SET LOCAL search_path` triviais; ecossistema `drizzle-multitenant` (middleware Fastify, migrations
paralelas). Migrations via drizzle-kit, aplicáveis em loop por schema.
- **Kysely:** query builder type-safe puro, troca de schema trivial. Contra: sem migrations geradas e
sem relation-loading → mais verboso. Bom plano B.
- **TypeORM:** decorators/Active Record legados, type-safety inferior, em declínio. Descartado.

## Decisão
**Drizzle ORM** como camada primária de acesso a dados.

## Justificativa
- Troca de schema por tenant é **idiomática** (vs anti-padrão no Prisma).
- Type-safe ponta a ponta com schema único reaproveitado por todos os tenants.
- Leve e popular (atende os princípios); SQL-first → menos "mágica" para a IA errar.
- Confirmado **independentemente** pelos agentes de arquitetura e de multitenancy.

## Consequências
- Acesso a dados sempre via `withTenant` (transação + `SET LOCAL search_path`).
- Migrations: runner que itera sobre `platform` + `tenant_*` (idempotente, no CI).
- Relation-loading via API relacional do Drizzle (`db.query.x.findMany({ with })`).

## Fontes
Prisma docs (Connections & Pooling); Prisma GitHub #2443/#20920 (multi-tenant/datasource);
drizzle-multitenant; comparativos Drizzle/Kysely/Prisma. Ver relatórios de arquitetura e multitenancy.
