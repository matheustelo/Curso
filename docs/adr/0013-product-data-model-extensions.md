# ADR-0013 — Extensões do modelo de dados dirigidas por produto

- **Status:** Aceito · **Data:** 2026-06-04 · **Confiança:** 85%
- **Complementa:** [ADR-0001](0001-multitenancy-schema-per-tenant.md) (isolamento), [ADR-0003](0003-orm-drizzle.md) (Drizzle), [ADR-0010](0010-payments-br.md) (pagamentos)
- **Relacionados:** [DATA_MODEL.md §6](../DATA_MODEL.md) · [docs/product/](../product/README.md)

## Contexto
A fase de detalhamento de produto (8 especialistas) revelou lacunas no DATA_MODEL v1.0 que precisavam de
decisão transversal antes da implementação: moderação de comentários, anti-seek de progresso, gamificação
idempotente, idempotência de quiz, modelagem de ofertas/order_items/bundles para order bump/upsell (F2),
atribuição de afiliado (cookie/janela/aprovação), preferências/central de notificações, campos
localizáveis (i18n), limite de dispositivos como quota e o mapeamento pricing → `platform_plans.limits` +
`take_rate_bps`. Várias dessas decisões cruzam a fronteira control plane × tenant e tocam a **Regra nº1**
(isolamento).

## Decisão
Estender o DATA_MODEL conforme [DATA_MODEL.md §6](../DATA_MODEL.md), **acrescentando** (nunca removendo)
tabelas/colunas marcadas por fase (MVP/F2/F3). Principais decisões:

1. **Vocabulário canônico de `status`** padronizado (§6.0). Inadimplência do SaaS (`past_due/grace`) é
   **derivada** do Stripe, sem nova coluna em `tenants.status`. Reembolso parcial **não** é estado de
   `orders` no MVP (tratado como ajuste financeiro).
2. **Anti-seek:** `lesson_progress.real_watched_seconds` (monotônico, deltas validados server-side),
   separado de `watched_pct`. Conclusão = `watched_pct ≥ 90 AND real_watched_seconds ≥ duration×0.8`.
3. **Comentários:** colunas de `resolved/hidden/deleted` + tabelas `comment_likes` e `comment_reports`.
4. **Quiz:** `attempt_number`, `status`, `idempotency_key` (dedup de submit), `scoring_policy`/`feedback_policy`.
5. **Gamificação (F2):** `gamification_xp_ledger` append-only e idempotente
   (`unique(user_id, source_type, source_event_id)`) → antifarming + reversão; agregado em
   `gamification_user_state` com `leaderboard_optout` (LGPD); `badges`/`user_badges`.
6. **Ofertas/checkout (F2):** `order_items` (orders passa a multi-item), `offers`, `bundles`/`bundle_items`,
   `orders.purchase_session_id`/`parent_order_id` para encadear upsell/downsell.
7. **Afiliados:** `affiliate_program_settings` (cookie window padrão **30 dias**, last-click,
   aprovação **manual** por padrão, auto-indicação bloqueada), `affiliate_clicks` (atribuição),
   `affiliate_course_commissions` (override por curso), recipient cifrado.
8. **Notificações (MVP):** `notifications` (central in-app) + `notification_preferences` (opt-out por
   canal/categoria, honra apenas P2/P3). Push = F2.
9. **i18n:** colunas `i18n jsonb` em `courses`/`lessons` preparadas já no MVP (evita migração destrutiva),
   com `default_locale`/`supported_locales` em `tenant_settings`. Ativação efetiva é F2.
10. **Quotas e take rate:** chaves de `platform_plans.limits` padronizadas, incluindo `take_rate_bps`
    (basis points). Limite de dispositivos (F2) é uma quota de plano, não coluna por usuário.

### Take rate sem violar isolamento (Regra nº1)
O cálculo de split ocorre no **schema do tenant**, mas o `take_rate_bps` mora no **control plane**. O
use-case de split **não** consulta `platform` diretamente. O valor é resolvido no `onRequest`
(lookup `platform.tenants → plan_id → platform_plans.limits.take_rate_bps`) e **injetado como valor** no
contexto da request (junto a `request.tenant`). Assim o use-case depende de um número, não de uma query
cross-schema — preserva DIP e o isolamento. O `recipient` da plataforma vive em
`platform.platform_payment_recipients`; os de produtor/afiliado, cifrados no data plane.

## Justificativa
- Mantém a **Regra nº1**: nenhuma FK cruza schemas; uso do tenant lido via `withTenant`, limites via
  control plane, composição no use-case.
- **Idempotência** (ledger de XP, quiz, milestones, webhooks) é coerente com a estratégia de webhooks já
  decidida (`payment_events.event_id`).
- **DRY:** o vocabulário de `status` vira enum/CHECK único reutilizado por `packages/contracts`.
- Colunas i18n preparadas cedo eliminam migração destrutiva futura (NFR-I18N-11/12).

## Consequências
- Migrations adicionam colunas/tabelas em **todos** os schemas `tenant_*` (runner em loop — ARCHITECTURE §3.5).
- Novos testes de **isolamento cross-tenant** para cada tabela nova que contém dados de tenant.
- Schemas Zod correspondentes em `packages/contracts` (fonte única de tipos).
- Itens marcados F2/F3 só ganham rotas/use-cases na respectiva fase (colunas podem existir antes).

## Pendências delegadas ao stakeholder (ver OPEN_QUESTIONS)
Prazos concretos de clearance de comissão, grace de inadimplência (aluno e SaaS), política de reembolso
parcial e take rate em reembolso, e tabela final de planos/quotas/preços do SaaS.
