# ADR-0001 — Multitenancy: schema-per-tenant em PostgreSQL

- **Status:** Aceito · **Data:** 2026-06-04 · **Confiança:** 90%

## Contexto
Requisito do stakeholder: multitenant com **separação no nível de banco de dados**, em PostgreSQL.
Escala esperada: **dezenas de tenants (até ~100)** em 1–3 anos. Projeto guiado por IA (previsibilidade)
e leve. Há duas leituras de "separação por banco": database-per-tenant e schema-per-tenant.

## Opções consideradas
1. **Database-per-tenant (silo):** isolamento físico máximo, backup/restore por tenant nativo. Contra:
exige **um pool de conexões por banco** → pressão sobre `max_connections`; migrations em N bancos.
2. **Schema-per-tenant (bridge):** 1 banco, 1 schema por tenant. Isolamento lógico forte
(`DROP SCHEMA` p/ LGPD), **um único pool** trocando `search_path`, onboarding/migrations simples.
3. **Shared schema + RLS (pool):** não atende "separação por banco"; descartado.

## Decisão
**Schema-per-tenant** (escolha do stakeholder), com control plane em um schema dedicado `platform`.
Com dezenas de tenants, é o ponto ideal: isolamento forte, **um pool + PgBouncer**, migrations em loop
triviais, custo baixo. Arquitetura permite, no futuro, promover um tenant a banco dedicado se necessário
(sem reescrever o domínio, pois o Drizzle suporta ambos).

## Justificativa
- Um único pool de conexões (vs N pools do database-per-tenant) — decisivo para simplicidade e custo.
- `DROP SCHEMA tenant_x CASCADE` dá argumento forte de LGPD/auditoria.
- Onboarding (`CREATE SCHEMA` + migrations) é rápido e idempotente.
- Padrão recomendado pela AWS (bridge) para equilíbrio custo/isolamento.

## Consequências
- **Padrão de ouro obrigatório:** todo acesso em transação com `SET LOCAL search_path` (compatível com
PgBouncer transaction pooling; evita vazamento cross-tenant). Centralizado em `withTenant`.
- Migrations rodam em `platform` + todos os `tenant_*` (job idempotente).
- Testes de isolamento cross-tenant são gate de CI.
- Atenção ao número total de objetos no catálogo (irrelevante em dezenas de tenants).

## Riscos e mitigação
| Risco | Mitigação |
|-------|-----------|
| Vazamento via search_path em transaction pooling | `SET LOCAL` por transação + nomes qualificados + testes |
| Estouro de conexões | PgBouncer + pool pequeno por instância |
| Migration parcial entre schemas | Migrations versionadas/idempotentes + tracking por schema |

## Fontes
AWS SaaS Lens (Silo/Pool/Bridge); AWS Bridge model; Arkency (Postgres schemas); PgBouncer transaction
pooling / `SET LOCAL search_path`. Ver relatório de pesquisa de multitenancy.
