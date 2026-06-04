# ADR-0010 — Pagamentos: Pagar.me + Asaas (BR) com afiliados/split

- **Status:** Aceito · **Data:** 2026-06-04 · **Confiança:** 85%

## Contexto
Decisão do stakeholder: **checkout BR completo + afiliados/split**. Necessário Pix, boleto, cartão,
recorrência, cupons e **split de pagamento** (afiliados/co-produção). Separar do billing do SaaS.

## Opções consideradas
- **Pagar.me (Stone):** split robusto (marketplace), Pix/boleto/cartão/recorrência, boa API.
- **Asaas:** forte em cobrança recorrente Pix/boleto de baixo custo.
- **Mercado Pago:** amplo, mas docs densas.
- **Stripe:** melhor DX/Connect, mas BR sem boleto e Pix mais limitado — reservado para internacional.
- **Hotmart/Eduzz:** plataformas (não gateways) — úteis só se quisermos o ecossistema de afiliados deles.

## Decisão
**Pagar.me como gateway principal** (por causa do split robusto p/ afiliados/co-produção) +
**Asaas** para Pix/boleto e cobrança recorrente de baixo custo. Ambos atrás de uma abstração
**`PaymentProvider`** (interface SOLID), selecionável por estratégia/feature flag.
**Billing do SaaS** (tenant → plataforma) fica em **Stripe Billing** no control plane, domínio separado.

## Justificativa
- Split de pagamento é requisito central (afiliados) → Pagar.me é a escolha mais forte no BR.
- Asaas complementa em custo de Pix/boleto recorrente.
- Abstração evita lock-in e permite somar Stripe (global) no futuro sem tocar no domínio.

## Consequências
- Webhooks de pagamento com **verificação de assinatura + idempotência** (`payment_events.event_id`).
- **Máquina de estados de acesso** (aprovado → libera; reembolso/chargeback → suspende) +
**reconciliação periódica** com o gateway.
- Nunca armazenar dados de cartão (PCI no gateway).

## Fontes
Comparativos de gateways BR (2026); docs Pagar.me/Asaas/Stripe. Ver relatório de streaming/integrações.
