# ADR-0014 — Port `AnalyticsProvider` + tracking plan em `packages/contracts`

- **Status:** Aceito · **Data:** 2026-06-04 · **Confiança:** 84%
- **Complementa:** [ADR-0012](0012-supporting-platform.md) (PostHog), [ADR-0004](0004-contracts-zod.md) (Zod)
- **Relacionados:** [docs/product/ANALYTICS_AND_DASHBOARDS.md](../product/ANALYTICS_AND_DASHBOARDS.md) · [ARCHITECTURE.md §10](../ARCHITECTURE.md)

## Contexto
O detalhamento de analytics definiu um **tracking plan canônico** (catálogo de eventos + propriedades
padrão) com PostHog como destino (ADR-0012, group analytics por `tenant`/`course`). Faltava: (a) tratar
analytics como dependência arquitetural sob uma **port** (SOLID/DIP), e (b) decidir onde vive o contrato
dos eventos para ser reutilizado por back e front sem duplicação (DRY) e respeitando a **Regra nº1**.

## Decisão
1. **Port `AnalyticsProvider`** (interface SOLID) em `packages/core`/`shared`, com impl PostHog em
   infraestrutura. Eventos com consequência financeira/de acesso (`*_paid`, `*_refunded`,
   `enrollment_*`, `certificate_issued`) são emitidos **server-side** (fonte de verdade = use-case/webhook);
   eventos de UX podem ser client-side. Troca de provedor não toca use-cases (DIP, como `VideoProvider`/
   `PaymentProvider`/`JobQueue`).
2. **Tracking plan como schemas Zod em `packages/contracts`** — o catálogo de eventos do
   ANALYTICS_AND_DASHBOARDS vira a fonte única de tipos dos payloads (DRY back/front). Eventos novos
   passam por PR no doc + schema.
3. **Isolamento:** todo evento carrega `tenant_id`; `distinct_id` é sempre escopo de tenant
   (`tenant_id + user_id`), nunca o e-mail. Vazamento em analytics tem a mesma gravidade da Regra nº1.
   Captura client-side só dispara **após consentimento** (LGPD).
4. **Heartbeats de vídeo:** alto volume. Decisão: o heartbeat alimenta `lesson_progress`
   (banco, fonte de verdade de progresso/anti-seek) e é **agregado** para watch-time; **não** se envia
   cada heartbeat cru ao PostHog. Emite-se evento amostrado/agregado (`video_heartbeat` com throttle
   5–15s do lado do progresso; agregação de watch-time para os dashboards). North Star (horas assistidas)
   deriva de `lesson_progress.real_watched_seconds`.
5. **Sem PII sensível** nos eventos (sem CPF, e-mail em claro, tokens de vídeo, keys).

## Justificativa
- Coerência com as demais ports do projeto → previsível para agentes de IA.
- Schemas Zod compartilhados evitam drift entre instrumentação e backend.
- Agregar heartbeats no banco protege custo/volume e mantém o progresso como fonte de verdade única.

## Consequências
- Nova pasta de schemas de eventos em `packages/contracts`.
- O wrapper de captura injeta propriedades padrão (`tenant_id`, contexto) automaticamente.
- Dashboards (PostHog) consomem group analytics por `tenant`/`course`.
- Definir em produto se existe **fee transacional sobre GMV** além da mensalidade (muda fórmula de take
  rate) — registrado em OPEN_QUESTIONS.
