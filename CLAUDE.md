# CLAUDE.md — Guia para Desenvolvimento Guiado por IA

Este projeto é desenvolvido com forte apoio de agentes de IA. Este arquivo é a **fonte de verdade de
convenções**. Leia-o antes de gerar ou alterar código. Princípios **SOLID, DRY e Clean Code** são
**não-negociáveis**.

> Documentos de referência: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) ·
> [docs/PRD.md](docs/PRD.md) · [docs/DATA_MODEL.md](docs/DATA_MODEL.md) · [docs/adr/](docs/adr/)

---

## ⛔ Regra nº 1 — Escopo de tenant (a mais importante)

O modelo é **schema-per-tenant** (1 banco PostgreSQL, 1 schema por tenant `tenant_<slug>` + schema
`platform`). **Vazar dados entre tenants é o bug mais grave possível.**

- **TODO** acesso a dados de um tenant DEVE passar por `withTenant(tenantId, fn)` (em `packages/db`),
que abre uma transação e executa `SET LOCAL search_path TO tenant_<slug>, public;`.
- **NUNCA** use nomes de tabela não-qualificados fora de uma transação com `search_path` setado.
- **NUNCA** confie em estado de sessão herdado do PgBouncer.
- O `tenantId` vem da **sessão/JWT** (fonte de verdade), nunca de um header não autenticado.
- Dados globais do SaaS ficam no schema `platform` (control plane) e **não** têm FK para schemas de tenant.
- Todo novo recurso de dados precisa de um **teste de isolamento cross-tenant** (gate de CI).

---

## 🧱 Arquitetura (Clean Architecture leve, vertical slices)

Cada feature é um **módulo** em `apps/api/src/modules/<feature>/` com 3 camadas e dependência **para
dentro** (`domain ← application ← infrastructure`):

- **domain/** — entidades, value objects, regras puras. **Proibido** importar Fastify/Drizzle/Bunny.
- **application/** — use-cases (1 classe = 1 caso de uso → SRP). Dependem de **ports** (interfaces) → DIP.
  - `application/ports/*.ts` — interfaces (ex.: `CourseRepository`).
- **infrastructure/** — implementações: repositórios Drizzle, clientes externos, rotas Fastify (adapters).
- `<feature>.module.ts` — `composeModule()` faz a **DI manual** (sem container mágico).

Replique o padrão do módulo vizinho. Use o gerador de scaffold quando existir (`pnpm gen module`).

---

## 📛 Convenções de nomenclatura e arquivos

- Arquivos `kebab-case`; classes/tipos `PascalCase`; 1 conceito por arquivo.
- Sufixos explícitos: `*.entity.ts`, `*.usecase.ts`, `*.repository.ts` (port) /
`*.drizzle.repository.ts` (impl), `*.routes.ts`, `*.schema.ts` (Zod), `*.module.ts`, `*.test.ts`.
- Erros de domínio estendem `DomainError` (em `packages/core`). Um único error handler Fastify os mapeia
para Problem Details (RFC 9457). **Não** trate erros HTTP dentro dos use-cases.

---

## 🔁 DRY — onde reaproveitar

- **Contratos/validação:** `packages/contracts` (schemas Zod) é a **única** fonte de tipos partilhados
entre back e front. Não duplique tipos de DTO.
- **Acesso a dados de tenant:** sempre `withTenant` (`packages/db`).
- **Filas:** sempre pela port `JobQueue` (impl pg-boss). Jobs carregam `tenantId` no payload.
- **Vídeo:** sempre pela port `VideoProvider` (impl Bunny). Pagamentos pela port `PaymentProvider`.

---

## 🧪 Testes

- **Vitest** (unit/integration) + **Playwright** (e2e) + **Testcontainers** (Postgres real).
- Unit: use-cases com ports mockados. Integration: repositórios contra Postgres real em schema efêmero.
- **Obrigatório:** teste provando que tenant A não enxerga dados do tenant B.

---

## 🛠️ Comandos (a definir no scaffolding; manter atualizados)

```bash
pnpm install            # instala deps do monorepo
pnpm dev                # sobe api + web + worker
pnpm test               # Vitest (unit/integration)
pnpm test:e2e           # Playwright
pnpm lint               # Biome (lint + format check)
pnpm format             # Biome format
pnpm typecheck          # tsc --noEmit
pnpm db:migrate         # roda migrations em platform + todos os schemas tenant_*
pnpm gen module         # scaffold de um novo módulo (3 camadas + teste)
```

---

## 📐 TypeScript

`strict: true` + `noUncheckedIndexedAccess` + `exactOptionalPropertyTypes` + `verbatimModuleSyntax` +
`isolatedModules` + `NodeNext`. Imports type-only explícitos (`import type`). ESM em todo o repo.

---

## ✅ Checklist antes de abrir PR

- [ ] Acesso a dados de tenant passa por `withTenant` (nada de query crua sem `search_path`).
- [ ] Use-case depende de ports, não de implementações concretas.
- [ ] Schemas Zod em `packages/contracts` reutilizados (sem duplicar tipos).
- [ ] Erros via `DomainError` (sem `res.status(...)` espalhado).
- [ ] Testes: unit do use-case + integration + **isolamento cross-tenant** quando tocar dados.
- [ ] `pnpm lint && pnpm typecheck && pnpm test` verdes.
- [ ] Decisão arquitetural nova? Adicione um **ADR** em `docs/adr/`.

---

## 🚫 Anti-padrões (não faça)

- Prisma para acesso a dados (escolhemos Drizzle — ver ADR-0003).
- Lógica de negócio em controllers/rotas.
- Acesso direto a `tenant_*` sem `withTenant`.
- Redis/BullMQ no MVP (usamos pg-boss — ADR-0011) sem antes trocar a decisão por ADR.
- Expor AccessKey/keys da Bunny ou de pagamento no frontend.
- Adicionar dependência pesada sem justificativa (mantenha a stack leve).
