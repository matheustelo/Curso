# ADR-0006 — Autenticação: Better-Auth (self-hosted)

- **Status:** Aceito com ressalva (PoC) · **Data:** 2026-06-04 · **Confiança:** 80%

## Contexto
Auth multitenant: usuários por tenant (aluno/instrutor/admin), super-admins globais, RBAC por tenant,
sessões seguras. Sem vendor lock-in caro por MAU. Identidade alinhada ao isolamento por schema.

## Opções consideradas
- **Auth.js (NextAuth):** sem multitenancy B2B/RBAC nativo; acoplado ao Next.
- **Lucia:** em modo manutenção desde 2025; autores recomendam migrar. Descartado.
- **Clerk/WorkOS:** organizations + RBAC prontos, porém pagos e com dados de identidade fora do nosso
banco (atrita com isolamento e custo).
- **Keycloak:** poderoso (realms), porém pesado operacionalmente (Java) — conflita com "leve".
- **Better-Auth:** TS-first, self-hosted, plugins `organization` + RBAC, 2FA/passkeys/magic links;
gerencia próprio schema.

## Decisão
**Better-Auth** self-hosted no backend Fastify; sessões via **cookies HttpOnly + Secure**. Usuários
vivem no **schema do tenant**; super-admins no `platform`. RBAC: `owner/admin/instructor/affiliate/student`.

## Ressalva / PoC obrigatório
Os exemplos do Better-Auth assumem DB único. Faremos uma **PoC** validando auth + resolução de tenant +
`search_path` ponta a ponta **antes de escalar**. **Plano B robusto:** auth próprio no Fastify
(`@fastify/jwt` + `@fastify/cookie` + argon2). Por isso a confiança é 80% (não maior).

## Consequências
- Autorização aplicada em guards/hooks Fastify (ponto único, DRY) e revalidada nos use-cases.
- Super-admin com **impersonação auditada** (`platform.audit_log`).

## Fontes
Comparativos Better-Auth vs Lucia vs NextAuth (2026); aviso de manutenção do Lucia.
