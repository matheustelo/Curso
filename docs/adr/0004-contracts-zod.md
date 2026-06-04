# ADR-0004 — Contratos e validação: Zod + fastify-type-provider-zod

- **Status:** Aceito · **Data:** 2026-06-04 · **Confiança:** 88%

## Contexto
Precisamos de validação de runtime + tipos compartilhados entre backend e frontend (DRY), suporte a
webhooks/clientes externos (Bunny, gateways, mobile futuro) e boa familiaridade da IA.

## Opções consideradas
- **tRPC:** type-safety automática front↔back sem codegen. Contra: acopla fortemente front a back,
atrita com REST/webhooks de terceiros e clientes externos.
- **OpenAPI gerado a partir do código:** ótimo para contratos públicos; adiciona pipeline de codegen.
- **Zod + contratos compartilhados:** schemas no `packages/contracts`, importados por back e front;
`fastify-type-provider-zod` valida request e tipa o handler; OpenAPI gerado a partir do Zod.
- **TypeBox:** mais rápido, JSON-Schema nativo do Fastify. Contra: menor familiaridade/legibilidade.

## Decisão
**Zod** + `fastify-type-provider-zod`, com **contratos compartilhados** em `packages/contracts`, e
**OpenAPI gerado do Zod** para documentação/clientes externos. Sem tRPC.

## Justificativa
- Fonte única de verdade (validação + tipo) → **DRY**; Zod é padrão de facto e dominante no corpus de IA.
- Boa parte do benefício do tRPC (tipos compartilhados) sem o acoplamento.
- OpenAPI abre porta para API pública (F2) e mobile sem comprometer a tipagem interna.

## Consequências
- Todo endpoint declara schemas Zod (input/output) reaproveitados no front.
- Reavaliar TypeBox **somente** se a validação virar gargalo medido em endpoints quentes.

## Fontes
Relatório de arquitetura (Zod vs tRPC vs TypeBox); docs fastify-type-provider-zod.
