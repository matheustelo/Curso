# ADR-0011 — Filas / background jobs: pg-boss

- **Status:** Aceito · **Data:** 2026-06-04 · **Confiança:** 82%

## Contexto
Precisamos de jobs assíncronos: e-mails, certificados (PDF), processamento de webhooks (Bunny/pagamento),
transcrição/IA, reconciliação. Princípio: stack **leve**. O trabalho pesado de mídia é da Bunny —
volume de jobs é moderado.

## Opções consideradas
- **pg-boss (fila no PostgreSQL):** usa o banco que já temos, garantias ACID, **sem Redis**. Teto de
throughput menor (centenas/s), mais que suficiente aqui.
- **BullMQ (Redis):** alto throughput, dashboards. Contra: **infra extra** (Redis), sem garantia
transacional com o DB (precisa de outbox) — conflita com "leve".

## Decisão
**pg-boss** no MVP, **atrás de uma port `JobQueue`** (DIP). Migração para BullMQ/Redis é localizada, se o
volume justificar no futuro.

## Justificativa
- Alinhado ao princípio de stack leve (sem Redis no MVP).
- Garantias ACID com o mesmo Postgres; orquestração simples.
- Conflito resolvido na convergência: o relatório de streaming sugeria Redis; convergimos em pg-boss +
abstração trocável.

## Consequências
- **Multitenant nas filas:** todo job carrega `tenantId`; o worker resolve o schema via `withTenant`.
- Idempotência dos handlers (especialmente webhooks).

## Fontes
Comparativo pg-boss vs BullMQ; relatórios de arquitetura e streaming.
