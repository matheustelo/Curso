# ADR-0002 — Backend: Fastify + Clean Architecture leve

- **Status:** Aceito · **Data:** 2026-06-04 · **Confiança:** 88%

## Contexto
Backend Node/TypeScript leve, performático, type-safe e previsível para desenvolvimento guiado por IA,
com controle fino sobre a resolução de tenant (`search_path`) por request. Preferência declarada do
stakeholder: **Fastify**.

## Opções consideradas
- **Fastify** (preferência): throughput alto, validação por schema, plugins maduros (auth, cors,
multipart, swagger), footprint baixo, TS de primeira classe. Contra: não impõe arquitetura.
- **NestJS (on Fastify):** estrutura "enterprise" pronta e muito previsível para IA. Contra: mais
pesado, mais "mágica" (decorators/reflection), request-scoped DI custoso — atrita com a injeção de
conexão por tenant.
- **Hono:** ultraleve/edge. Contra: ecossistema server-side menos completo para este escopo.
- **AdonisJS:** produtivo full-stack. Contra: menos representado no corpus de IA; Lucid não favorece
multitenancy dinâmico.

## Decisão
**Fastify**, com arquitetura imposta por **convenção** (não por framework): Clean Architecture leve em
**vertical slices** (domain → application → infrastructure), DI **manual** via `composeModule()`.

## Justificativa
- A leveza e o controle explícito da troca de `search_path` por request são decisivos.
- Para IA, **convenção explícita e documentada** (CLAUDE.md + boundaries por lint + scaffolding) supera a
convenção implícita de um framework pesado.
- Mantém a preferência do stakeholder com fundamentos técnicos sólidos.

## Consequências
- É preciso **disciplina de convenções** (CLAUDE.md, ESLint boundaries, gerador de módulos) para
compensar a ausência de estrutura imposta.
- Plugins Fastify para auth, multipart (uploads), rate-limit, swagger/OpenAPI.

## Contraponto registrado
Se a equipe fosse grande/júnior e sem disciplina, NestJS-on-Fastify seria preferível pela estrutura
imposta. Como o desenvolvimento é guiado por IA com convenções documentadas e se valoriza leveza,
Fastify vence.

## Fontes
Benchmarks Fastify vs Express/Nest; docs NestJS performance (adapter Fastify). Ver relatório de
arquitetura.
