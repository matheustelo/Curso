# ADR-0007 — Repositório: monorepo pnpm + Turborepo

- **Status:** Aceito · **Data:** 2026-06-04 · **Confiança:** 92%

## Contexto
Backend (api), frontend (web) e worker compartilham tipos/contratos e schema de banco. Queremos fonte
única de verdade (DRY) e builds/testes rápidos.

## Opções consideradas
- **Monorepo (pnpm workspaces + Turborepo):** compartilhamento de tipos trivial, cache de
build/test/lint, pipelines simples. Mais leve/previsível que Nx para este porte.
- **Nx:** melhor para organizações enormes; overhead desnecessário aqui.
- **Polyrepo:** só faria sentido com deploys/times radicalmente independentes — não é o caso.

## Decisão
**Monorepo com pnpm workspaces + Turborepo.**

```
apps/ (api, web, worker)
packages/ (contracts, db, config, core)
```

## Justificativa
- `packages/contracts` (Zod) e `packages/db` (Drizzle) como fontes únicas de verdade → DRY ponta a ponta.
- Cache do Turborepo acelera CI; `packages/config` centraliza tsconfig/Biome/boundaries.

## Consequências
- Versão de Node/TS unificada; tooling centralizado; releases coordenados.

## Fontes
Relatório de arquitetura (estrutura de repositório).
