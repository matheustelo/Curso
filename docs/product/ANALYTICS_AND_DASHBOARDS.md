# Analytics & Dashboards — Plataforma de Cursos SaaS Multitenant

- **Versão:** 1.0
- **Data:** 2026-06-04
- **Status:** Proposta para implementação
- **Stack de analytics:** PostHog (ver [ADR-0012](../adr/0012-supporting-platform.md)) — eventos de produto/funis, self-host LGPD-friendly. Web analytics leve (Plausible/Umami) opcional para landings públicas.
- **Documentos relacionados:** [PRD.md](../PRD.md) (§6 Métricas) · [ROADMAP.md](../ROADMAP.md) · [DATA_MODEL.md](../DATA_MODEL.md) · [ARCHITECTURE.md](../ARCHITECTURE.md) · [CLAUDE.md](../../CLAUDE.md)

---

## Índice

1. [Princípios e governança da taxonomia](#1-princípios-e-governança-da-taxonomia)
2. [Modelo de identidade e propriedades padrão](#2-modelo-de-identidade-e-propriedades-padrão)
3. [Catálogo de eventos](#3-catálogo-de-eventos)
   - 3.1 [Autenticação & sessão](#31-autenticação--sessão)
   - 3.2 [Navegação & catálogo](#32-navegação--catálogo)
   - 3.3 [Checkout & pagamento](#33-checkout--pagamento)
   - 3.4 [Matrícula & acesso](#34-matrícula--acesso)
   - 3.5 [Consumo de vídeo](#35-consumo-de-vídeo)
   - 3.6 [Conclusão de aula & curso](#36-conclusão-de-aula--curso)
   - 3.7 [Quiz & avaliação](#37-quiz--avaliação)
   - 3.8 [Certificado](#38-certificado)
   - 3.9 [Comunidade & comentários](#39-comunidade--comentários)
   - 3.10 [Afiliado & comissão](#310-afiliado--comissão)
   - 3.11 [Onboarding de tenant (control plane)](#311-onboarding-de-tenant-control-plane)
   - 3.12 [Billing do SaaS (control plane)](#312-billing-do-saas-control-plane)
4. [Catálogo de métricas / KPIs (fórmulas)](#4-catálogo-de-métricas--kpis-fórmulas)
5. [Dashboards por papel](#5-dashboards-por-papel)
6. [Mapeamento KPI ↔ North Star / métricas do PRD](#6-mapeamento-kpi--north-star--métricas-do-prd)
7. [Implementação técnica & isolamento de tenant](#7-implementação-técnica--isolamento-de-tenant)
8. [Dependências e pontos para o coordenador](#8-dependências-e-pontos-para-o-coordenador)

---

## 1. Princípios e governança da taxonomia

| Princípio | Regra |
|-----------|-------|
| **Naming** | `snake_case`, padrão `objeto_ação` no passado (`course_purchased`, `lesson_completed`). Objeto primeiro, verbo depois → agrupa naturalmente no PostHog. |
| **Tempo verbal** | Eventos = fato consumado (`*_completed`, `*_started`). Sem gerúndio nem futuro. |
| **Granularidade** | 1 evento por intenção de negócio. Evitar evento genérico `click` com prop `type`; preferir eventos nomeados. Exceção: `heartbeat` (alto volume, agregável). |
| **Server vs client** | Eventos com consequência financeira/de acesso (`*_paid`, `*_refunded`, `enrollment_*`, `certificate_issued`) são emitidos **server-side** (fonte de verdade = webhook/use-case), nunca confiando só no browser. Eventos de UX/navegação podem ser client-side. |
| **Isolamento de tenant** | **TODO** evento carrega `tenant_id` (ver §2 e §7). Vazamento entre tenants em analytics é tratado com a mesma gravidade que vazamento de dados (Regra nº1 do CLAUDE.md). |
| **Versionamento** | Mudança incompatível de payload → novo evento ou sufixo `_v2`; nunca redefinir silenciosamente uma propriedade existente. |
| **Sem PII sensível** | Não enviar CPF, e-mail em claro, tokens de vídeo ou keys. Usar IDs (`user_id`, hash). PostHog configurado com mascaramento (LGPD). |
| **Tracking plan** | Este documento é o tracking plan canônico. Eventos novos passam por PR aqui antes de instrumentação. |

---

## 2. Modelo de identidade e propriedades padrão

### 2.1 Identidade
- **`distinct_id`** = `user_id` (UUID do schema do tenant) para usuários logados; antes do login, ID anônimo do PostHog, com `identify()` no `user_logged_in`/`user_signed_up`.
- **Group analytics (PostHog Groups):**
  - **group type `tenant`** → `tenant_id` (toda análise pode ser segmentada/filtrada por tenant; base do isolamento).
  - **group type `course`** → `course_id` (engajamento por curso, conclusão por curso).
- O mesmo e-mail pode existir em tenants diferentes (modelo workspace, ver ARCHITECTURE §6) → `distinct_id` é sempre **escopo de tenant** (`tenant_id + user_id`), nunca o e-mail.

### 2.2 Propriedades padrão (anexadas a TODO evento via wrapper de captura)
| Propriedade | Tipo | Origem | Obrigatória |
|-------------|------|--------|-------------|
| `tenant_id` | uuid | sessão/JWT (fonte de verdade) | **Sim** |
| `tenant_slug` | string | `platform.tenants` | Sim |
| `user_role` | enum (`owner`,`admin`,`instructor`,`affiliate`,`student`,`super_admin`,`anonymous`) | RBAC | Sim (quando logado) |
| `environment` | enum (`prod`,`staging`,`dev`) | config | Sim |
| `source` | enum (`web`,`pwa`,`worker`,`webhook`) | runtime | Sim |
| `request_id` | string | AsyncLocalStorage (OTel) | Recomendada |
| `app_version` | string | build | Recomendada |
| `locale` | string (`pt-BR`) | next-intl | Recomendada |

### 2.3 Propriedades de contexto recorrentes (quando aplicável)
`course_id`, `course_slug`, `module_id`, `lesson_id`, `lesson_type`, `enrollment_id`, `order_id`, `affiliate_id`, `coupon_code`, `quiz_id`, `certificate_id`, `plan_id`. Sempre IDs do schema do tenant.

---

## 3. Catálogo de eventos

> Convenção das tabelas: **Quando dispara** indica o gatilho; **(S)** = server-side, **(C)** = client-side. Todas as propriedades listadas acompanham as **propriedades padrão** da §2.2.

### 3.1 Autenticação & sessão

| Evento | Lado | Quando dispara | Propriedades específicas |
|--------|------|----------------|--------------------------|
| `user_signed_up` | S | Conta criada no tenant | `signup_method` (`password`,`oauth`), `invited` (bool), `inviter_role` |
| `user_logged_in` | S | Login bem-sucedido | `auth_method`, `is_first_login_today` |
| `user_login_failed` | S | Falha de credencial | `failure_reason` (`bad_credentials`,`locked`,`mfa_required`) |
| `user_logged_out` | C | Logout explícito | `session_duration_sec` |
| `password_reset_requested` | S | Pedido de reset | — |
| `password_reset_completed` | S | Senha redefinida | — |
| `super_admin_impersonation_started` | S | Super-admin entra como tenant (control plane) | `target_tenant_id`, `target_user_id`, `audit_log_id` |
| `super_admin_impersonation_ended` | S | Fim da impersonação | `target_tenant_id`, `duration_sec` |

### 3.2 Navegação & catálogo

| Evento | Lado | Quando dispara | Propriedades específicas |
|--------|------|----------------|--------------------------|
| `landing_page_viewed` | C | View da landing de venda de um curso | `course_id`, `utm_source`, `utm_medium`, `utm_campaign`, `affiliate_id` (de ref), `referrer` |
| `catalog_viewed` | C | View do catálogo do tenant | `filter_category`, `results_count` |
| `course_viewed` | C | Abertura da página de detalhe do curso | `course_id`, `is_enrolled` (bool), `price_cents` |
| `course_search_performed` | C | Busca no catálogo | `query_length`, `results_count` |
| `cta_clicked` | C | Clique no CTA de compra/matrícula | `course_id`, `cta_location` (`hero`,`pricing`,`sticky_bar`) |

### 3.3 Checkout & pagamento

> Estados de pedido alinhados a `orders.status` do DATA_MODEL: `pending | paid | refunded | chargeback`. Eventos financeiros são **server-side** (webhook do gateway = fonte de verdade).

| Evento | Lado | Quando dispara | Propriedades específicas |
|--------|------|----------------|--------------------------|
| `checkout_started` | C | Abre o checkout do curso | `course_id`, `amount_cents`, `currency`, `payment_method_intended`, `affiliate_id`, `coupon_code`, `has_order_bump` (F2) |
| `coupon_applied` | C | Cupom validado no checkout | `coupon_code`, `discount_cents`, `valid` (bool) |
| `payment_method_selected` | C | Aluno escolhe Pix/boleto/cartão | `payment_method` (`pix`,`boleto`,`card`), `installments` |
| `order_created` | S | `orders` row criada (`status=pending`) | `order_id`, `amount_cents`, `payment_method`, `provider` (`pagarme`,`asaas`), `affiliate_id`, `coupon_code` |
| `pix_qr_generated` | S | QR/copia-e-cola Pix gerado | `order_id`, `expires_at` |
| `boleto_generated` | S | Boleto emitido | `order_id`, `due_at` |
| `payment_authorized` | S | Cartão autorizado (pré-captura) | `order_id`, `installments` |
| `order_paid` | S | Webhook confirma pagamento (`status=paid`) | `order_id`, `amount_cents`, `payment_method`, `provider`, `affiliate_id`, `time_to_pay_sec` (created→paid) |
| `payment_declined` | S | Gateway recusa | `order_id`, `payment_method`, `decline_reason` (`insufficient_funds`,`fraud`,`expired_card`,`other`) |
| `payment_expired` | S | Pix/boleto não pago no prazo | `order_id`, `payment_method` |
| `order_refunded` | S | Reembolso (`status=refunded`) | `order_id`, `amount_cents`, `refund_reason`, `days_since_purchase` |
| `order_chargeback` | S | Chargeback (`status=chargeback`) | `order_id`, `amount_cents` |
| `cart_abandoned` | S | Job detecta `pending` sem conclusão após TTL (F2) | `order_id`, `course_id`, `amount_cents`, `payment_method_intended` |
| `subscription_started` | S | Assinatura do ALUNO ao conteúdo iniciada | `subscription_id`, `plan_ref`, `provider`, `billing_interval` |
| `subscription_renewed` | S | Renovação cobrada | `subscription_id`, `period_count` |
| `subscription_canceled` | S | Assinatura do aluno cancelada | `subscription_id`, `reason`, `lifetime_value_cents` |

### 3.4 Matrícula & acesso

> Estados de acesso alinhados a `enrollments.status`: `active | suspended | refunded | expired`. "Pagamento governa acesso" (PRD §4.3).

| Evento | Lado | Quando dispara | Propriedades específicas |
|--------|------|----------------|--------------------------|
| `enrollment_created` | S | Matrícula criada | `enrollment_id`, `course_id`, `source` (`purchase`,`manual`,`bulk`,`auto`), `order_id` (se compra) |
| `enrollment_activated` | S | Acesso liberado (`status=active`) | `enrollment_id`, `course_id`, `trigger` (`payment_paid`,`manual`,`bulk_import`) |
| `enrollment_suspended` | S | Acesso suspenso (inadimplência/chargeback) | `enrollment_id`, `course_id`, `reason` (`refund`,`chargeback`,`subscription_lapsed`,`admin`) |
| `enrollment_refunded` | S | Matrícula marcada refunded | `enrollment_id`, `course_id` |
| `enrollment_expired` | S | `expires_at` atingido / reconciliação | `enrollment_id`, `course_id` |
| `bulk_enrollment_imported` | S | Import CSV concluído | `course_id`, `rows_total`, `rows_succeeded`, `rows_failed` |
| `lesson_unlocked` | S | Drip libera aula (data fixa ou dias-após-matrícula) | `lesson_id`, `course_id`, `drip_rule` (`fixed_date`,`days_after_enroll`) |

### 3.5 Consumo de vídeo

> Base da North Star (watch-time). `video_heartbeat` é alto volume — throttle 5–15s no player (ARCHITECTURE §7) e agregação no PostHog. Conclusão exige ≥90% assistido **com** checagem anti-seek (tempo real ≥ duração × 0.8) — PRD §4.4.

| Evento | Lado | Quando dispara | Propriedades específicas |
|--------|------|----------------|--------------------------|
| `lesson_started` | C | Primeiro play de uma aula de vídeo na sessão | `lesson_id`, `course_id`, `module_id`, `video_guid`, `duration_seconds` |
| `video_played` | C | Play (incl. retomada) | `lesson_id`, `position_seconds`, `is_resume` (bool), `playback_rate` |
| `video_paused` | C | Pause | `lesson_id`, `position_seconds` |
| `video_seeked` | C | Seek (arrastar timeline) | `lesson_id`, `from_seconds`, `to_seconds`, `direction` (`forward`,`backward`) |
| `video_rate_changed` | C | Mudança de velocidade | `lesson_id`, `playback_rate` (0.5–2.0) |
| `video_quality_changed` | C | Mudança de qualidade HLS | `lesson_id`, `quality` |
| `video_caption_toggled` | C | Liga/desliga legenda | `lesson_id`, `enabled` (bool), `caption_lang` |
| `video_heartbeat` | C | A cada 5–15s de reprodução ativa | `lesson_id`, `position_seconds`, `watched_pct`, `seconds_watched_delta`, `playback_rate`, `is_buffering` |
| `video_buffering` | C | Stall de buffer (qualidade de entrega) | `lesson_id`, `buffer_duration_ms` |
| `video_error` | C | Erro de player/token | `lesson_id`, `error_code` (`token_expired`,`network`,`drm`,`other`) |
| `progress_synced` | S | `POST /progress` persiste em `lesson_progress` | `lesson_id`, `enrollment_id`, `position_seconds`, `watched_pct`, `status` |

### 3.6 Conclusão de aula & curso

| Evento | Lado | Quando dispara | Propriedades específicas |
|--------|------|----------------|--------------------------|
| `lesson_completed` | S | Regra de conclusão atingida (≥90% + anti-seek) ou aula texto/pdf marcada | `lesson_id`, `course_id`, `lesson_type`, `enrollment_id`, `watched_pct`, `completion_method` (`video_threshold`,`manual_mark`,`quiz_passed`) |
| `module_completed` | S | Todas as aulas do módulo concluídas | `module_id`, `course_id`, `enrollment_id` |
| `course_progress_milestone` | S | Cruzou marco de progresso (25/50/75%) | `course_id`, `enrollment_id`, `milestone_pct` |
| `course_completed` | S | Curso 100% concluído (e nota mínima quando houver) | `course_id`, `enrollment_id`, `days_to_complete` (enroll→complete), `total_watch_seconds` |
| `lesson_asset_downloaded` | S | Download de PDF/anexo | `lesson_id`, `asset_id`, `size_bytes` |

### 3.7 Quiz & avaliação

| Evento | Lado | Quando dispara | Propriedades específicas |
|--------|------|----------------|--------------------------|
| `quiz_started` | C | Tentativa iniciada | `quiz_id`, `course_id`, `lesson_id`, `attempt_number` |
| `quiz_submitted` | S | Respostas enviadas e corrigidas | `quiz_id`, `attempt_number`, `score`, `passed` (bool), `pass_score`, `duration_sec`, `questions_count` |
| `quiz_passed` | S | Aprovado (`score ≥ pass_score`) | `quiz_id`, `score`, `attempts_used` |
| `quiz_failed` | S | Reprovado | `quiz_id`, `score`, `attempts_remaining` |
| `quiz_question_answered` | C | Resposta por questão (drop-off de questão) | `quiz_id`, `question_id`, `position`, `is_correct` |

### 3.8 Certificado

| Evento | Lado | Quando dispara | Propriedades específicas |
|--------|------|----------------|--------------------------|
| `certificate_issued` | S | Worker gera PDF após `course_completed` | `certificate_id`, `course_id`, `enrollment_id`, `uuid_public` |
| `certificate_downloaded` | C | Aluno baixa o PDF | `certificate_id`, `course_id` |
| `certificate_shared` | C | Compartilhamento (link/social) | `certificate_id`, `channel` |
| `certificate_verification_viewed` | C | Página pública de verificação acessada | `uuid_public`, `is_valid` (bool), `is_revoked` (bool) |
| `certificate_revoked` | S | Admin revoga | `certificate_id`, `reason` |

### 3.9 Comunidade & comentários

| Evento | Lado | Quando dispara | Propriedades específicas |
|--------|------|----------------|--------------------------|
| `comment_posted` | S | Comentário criado em aula | `lesson_id`, `course_id`, `comment_id`, `is_reply` (bool), `parent_id` |
| `comment_replied` | S | Resposta (instrutor/aluno) | `lesson_id`, `comment_id`, `parent_id`, `replier_role` |
| `comment_deleted` | S | Remoção | `comment_id`, `deleted_by_role` |
| `comment_reported` | C | Denúncia de comentário (moderação) | `comment_id`, `reason` |
| `forum_topic_created` | S | F2 — tópico no fórum | `course_id`, `topic_id` |

### 3.10 Afiliado & comissão

> Comissão alinhada a `affiliate_commissions.status`: `pending | paid | reversed`.

| Evento | Lado | Quando dispara | Propriedades específicas |
|--------|------|----------------|--------------------------|
| `affiliate_link_generated` | S | Afiliado cria link | `affiliate_id`, `course_id`, `affiliate_code` |
| `affiliate_link_clicked` | C | Visita via link de afiliado | `affiliate_id`, `affiliate_code`, `course_id`, `utm_source` |
| `affiliate_attributed` | S | Pedido vinculado ao afiliado | `affiliate_id`, `order_id`, `attribution_model` (`last_click`) |
| `affiliate_commission_created` | S | Comissão gerada em `order_paid` (`status=pending`) | `affiliate_id`, `commission_id`, `order_id`, `amount_cents`, `commission_pct` |
| `affiliate_commission_paid` | S | Comissão paga (`status=paid`) | `commission_id`, `amount_cents`, `payout_batch_id` |
| `affiliate_commission_reversed` | S | Reembolso/chargeback estorna comissão | `commission_id`, `amount_cents`, `reason` |
| `split_executed` | S | Split de pagamento processado (co-produção/afiliado) | `order_id`, `recipients_count`, `total_split_cents` |

### 3.11 Onboarding de tenant (control plane)

> Eventos do schema `platform` — `group type tenant` é o próprio assunto. `tenant_id` continua presente.

| Evento | Lado | Quando dispara | Propriedades específicas |
|--------|------|----------------|--------------------------|
| `tenant_provisioning_started` | S | `INSERT tenant` (`status=provisioning`) | `tenant_id`, `plan_id`, `idempotency_key` |
| `tenant_provisioning_step_completed` | S | Cada passo da saga (schema, migrations, seed, bunny lib) | `tenant_id`, `step`, `attempt`, `duration_ms` |
| `tenant_provisioned` | S | `status=active` + smoke test ok | `tenant_id`, `provisioning_duration_sec` |
| `tenant_provisioning_failed` | S | Falha na saga | `tenant_id`, `failed_step`, `last_error` |
| `tenant_first_course_published` | S | Tenant publica 1º curso (ativação) | `tenant_id`, `days_since_provisioned` |
| `tenant_first_sale` | S | Tenant realiza 1ª venda (ativação) | `tenant_id`, `days_since_provisioned`, `amount_cents` |
| `tenant_suspended` | S | Suspensão (billing/admin) | `tenant_id`, `reason` (`billing_overdue`,`manual`,`abuse`) |
| `tenant_reactivated` | S | Reativação | `tenant_id` |
| `tenant_branding_configured` | S | Logo/cores/subdomínio definidos | `tenant_id`, `has_custom_domain` (bool) |
| `tenant_team_member_invited` | S | Convite a instrutor/admin | `tenant_id`, `invited_role` |

### 3.12 Billing do SaaS (control plane)

> Assinatura do **tenant na plataforma** (Stripe Billing, `platform.platform_subscriptions`). Separado do checkout dos alunos.

| Evento | Lado | Quando dispara | Propriedades específicas |
|--------|------|----------------|--------------------------|
| `saas_trial_started` | S | Trial do plano SaaS iniciado | `tenant_id`, `plan_id`, `trial_ends_at` |
| `saas_subscription_created` | S | Assinatura SaaS ativa (Stripe) | `tenant_id`, `plan_id`, `mrr_cents`, `billing_interval` |
| `saas_subscription_upgraded` | S | Upgrade de plano | `tenant_id`, `from_plan_id`, `to_plan_id`, `mrr_delta_cents` |
| `saas_subscription_downgraded` | S | Downgrade de plano | `tenant_id`, `from_plan_id`, `to_plan_id`, `mrr_delta_cents` |
| `saas_subscription_canceled` | S | Cancelamento (churn) | `tenant_id`, `plan_id`, `mrr_lost_cents`, `tenure_months`, `cancel_reason` |
| `saas_invoice_paid` | S | Fatura SaaS paga | `tenant_id`, `invoice_id`, `amount_cents` |
| `saas_invoice_payment_failed` | S | Falha de cobrança (risco de churn) | `tenant_id`, `invoice_id`, `attempt`, `amount_cents` |
| `saas_quota_threshold_reached` | S | Tenant atinge 80/100% de quota (alunos/storage) | `tenant_id`, `quota_type` (`students`,`storage`,`feature`), `usage_pct` |

---

## 4. Catálogo de métricas / KPIs (fórmulas)

> Eventos da §3 como fonte. "Tenant" = escopo via group analytics; KPIs do SaaS agregam todos os tenants (visão Super-Admin).

| # | KPI | Fórmula | Eventos / dados-fonte | Janela | Dono |
|---|-----|---------|-----------------------|--------|------|
| K1 | **North Star — Horas assistidas / aluno ativo / mês** | `Σ seconds_watched_delta / 3600 ÷ alunos ativos no mês` | `video_heartbeat` (delta), DAU/MAU de alunos | Mês | Todos |
| K2 | **Ativação do tenant (14d)** | `tenants que publicam ≥1 curso E fazem ≥1 venda em ≤14d ÷ tenants provisionados` | `tenant_first_course_published`, `tenant_first_sale`, `tenant_provisioned` | Coorte 14d | Super-Admin |
| K3 | **Ativação do aluno** | `alunos que concluem ≥1 aula em ≤7d da matrícula ÷ matrículas` | `enrollment_activated`, `lesson_completed` | Coorte 7d | Admin/Instrutor |
| K4 | **Conversão de checkout** | `order_paid ÷ checkout_started` | `checkout_started`, `order_paid` | Período | Admin |
| K5 | **Taxa de aprovação de pagamento** | `order_paid ÷ order_created` | `order_created`, `order_paid`, `payment_declined`, `payment_expired` | Período | Admin |
| K6 | **Taxa de conclusão de curso** | `course_completed (distintos) ÷ enrollment_activated (distintos)` | `enrollment_activated`, `course_completed` | Coorte | Admin/Instrutor |
| K7 | **Drop-off por aula** | `1 − (lesson_completed[n] ÷ lesson_started[n])` por aula ordenada | `lesson_started`, `lesson_completed`, `progress_synced` | Período | Instrutor |
| K8 | **Watch-time por aula** | `Σ seconds_watched_delta` por `lesson_id` (e média por aluno) | `video_heartbeat` | Período | Instrutor |
| K9 | **Retenção de alunos por coorte** | `% de alunos da coorte (mês de matrícula) ativos no mês N` | `enrollment_activated`, `video_played`/`lesson_started` como atividade | Coorte mensal | Admin |
| K10 | **GMV do tenant** | `Σ amount_cents de order_paid` (− refunds para GMV líquido) | `order_paid`, `order_refunded`, `order_chargeback` | Período | Admin/Super-Admin |
| K11 | **Take rate / receita da plataforma** | `MRR SaaS ÷ GMV dos tenants` (e fee se houver) | `saas_invoice_paid`, `order_paid` | Mês | Super-Admin |
| K12 | **MRR do SaaS** | `Σ mrr_cents de assinaturas SaaS ativas` | `saas_subscription_created/upgraded/downgraded/canceled` | Mês | Super-Admin |
| K13 | **Churn do SaaS (tenants)** | `tenants cancelados no mês ÷ tenants ativos no início do mês`; **MRR churn** = `mrr_lost ÷ MRR inicial` | `saas_subscription_canceled`, `tenant_suspended` | Mês | Super-Admin |
| K14 | **Ticket médio (AOV)** | `Σ amount_cents(order_paid) ÷ count(order_paid)` | `order_paid` | Período | Admin |
| K15 | **Comissão de afiliados** | `Σ amount_cents de affiliate_commission_created`; **paga** = status `paid`; **estorno** = `reversed` | `affiliate_commission_*` | Período | Admin/Afiliado |
| K16 | **Conversão de afiliado** | `affiliate_attributed (paid) ÷ affiliate_link_clicked` | `affiliate_link_clicked`, `affiliate_attributed`, `order_paid` | Período | Afiliado/Admin |
| K17 | **Taxa de reembolso** | `order_refunded ÷ order_paid` | `order_refunded`, `order_paid` | Período | Admin |
| K18 | **Carrinho abandonado** | `cart_abandoned ÷ checkout_started` (F2) | `checkout_started`, `cart_abandoned`, `order_paid` | Período | Admin |
| K19 | **Taxa de aprovação em quiz** | `quiz_passed ÷ quiz_submitted` | `quiz_submitted`, `quiz_passed` | Período | Instrutor |
| K20 | **Tempo médio até conclusão** | `média(days_to_complete)` | `course_completed` | Coorte | Instrutor |
| K21 | **Engajamento de comunidade** | `comment_posted por aluno ativo`; taxa de resposta do instrutor | `comment_posted`, `comment_replied` | Período | Instrutor/Admin |
| K22 | **Saúde de entrega de vídeo** | `video_error ÷ lesson_started`; `% heartbeats com is_buffering` | `video_error`, `video_buffering`, `video_heartbeat` | Período | Super-Admin/Admin |
| K23 | **Sucesso de provisionamento** | `tenant_provisioned ÷ tenant_provisioning_started`; latência p50/p95 | eventos §3.11 | Período | Super-Admin |
| K24 | **Uso de quota por tenant** | `usage_pct` por `quota_type` (alunos/storage) | `saas_quota_threshold_reached`, contagem de `enrollment_*` | Atual | Super-Admin |

---

## 5. Dashboards por papel

> Filtros globais por padrão em cada dashboard: período, e (exceto Super-Admin) escopo automático ao `tenant_id` da sessão. Filtros adicionais listados por widget.

### 5.1 Super-Admin — Saúde do SaaS (control plane, cross-tenant)

| Widget | Métrica-fonte | Visualização | Filtros |
|--------|---------------|--------------|---------|
| MRR e crescimento | K12 (`saas_subscription_*`) | Linha + número | Plano, intervalo de cobrança |
| Churn de tenants & MRR churn | K13 | Linha + número | Plano, motivo de cancelamento |
| Take rate | K11 (MRR ÷ GMV) | Número + tendência | — |
| GMV agregado de todos os tenants | K10 | Barra por tenant | Tenant, método de pagamento |
| Tenants ativos / provisionando / suspensos | `tenant_*` status | Funil/contadores | Status, plano |
| Ativação de tenant (14d) | K2 | Funil de coorte | Coorte de provisionamento |
| Sucesso de provisionamento + latência p95 | K23 | Funil + histograma | Step da saga |
| Uso de quota (alertas) | K24 | Tabela (tenants perto do limite) | Tipo de quota |
| North Star agregada (horas/aluno ativo/mês) | K1 | Linha | Tenant |
| Saúde de entrega de vídeo | K22 | Linha + tabela top erros | Tenant, error_code |
| Impersonações (auditoria) | `super_admin_impersonation_*` | Tabela | Super-admin, tenant |
| Falhas de cobrança SaaS | `saas_invoice_payment_failed` | Tabela/alerta | Tenant |

### 5.2 Admin do Tenant — Negócio (escopo do próprio tenant)

| Widget | Métrica-fonte | Visualização | Filtros |
|--------|---------------|--------------|---------|
| Receita (GMV bruto/líquido) | K10 | Linha + número | Curso, método de pagamento, afiliado |
| Ticket médio (AOV) | K14 | Número + tendência | Curso |
| Funil de checkout | K4 → `landing_page_viewed → checkout_started → order_created → order_paid` | Funil | Curso, método, cupom, afiliado, UTM |
| Taxa de aprovação de pagamento | K5 | Número + breakdown por método | Método, provider |
| Recusas por motivo | `payment_declined.decline_reason` | Barra | Método |
| Taxa de reembolso | K17 | Número + tendência | Curso |
| Carrinho abandonado (F2) | K18 | Funil/número | Curso, método |
| Alunos ativos (DAU/WAU/MAU) | atividade (`lesson_started`,`video_played`) | Linha | Curso |
| Novas matrículas por fonte | `enrollment_created.source` | Barra empilhada | Curso, fonte |
| Retenção por coorte | K9 | Tabela de coorte | Mês de matrícula, curso |
| Conclusão por curso | K6 | Barra por curso | Curso |
| Top afiliados (vendas e comissão) | K15, K16 | Tabela | Afiliado, curso |
| Comissões a pagar / pagas / estornadas | K15 (`affiliate_commission_*`) | Número + tabela | Status, afiliado |
| Assinaturas de alunos (MRR B2C, churn) | `subscription_*` | Linha | Plano |
| North Star do tenant | K1 | Linha | Curso |

### 5.3 Instrutor — Engajamento e qualidade do conteúdo (cursos próprios)

> Escopo automático: `instructor_id = user` (cursos onde é instrutor) dentro do tenant.

| Widget | Métrica-fonte | Visualização | Filtros |
|--------|---------------|--------------|---------|
| Watch-time por curso/aula | K8 (`video_heartbeat`) | Barra/heatmap | Curso, módulo |
| Drop-off por aula (curva de retenção) | K7 | Funil ordenado por aula | Curso |
| Heatmap de retenção dentro do vídeo (F2) | `video_heartbeat.position_seconds` | Heatmap por segundo | Aula |
| Conclusão de curso e tempo até concluir | K6, K20 | Número + histograma | Curso |
| Aprovação em quizzes / questões mais erradas | K19, `quiz_question_answered` | Barra + tabela | Quiz, questão |
| Alunos travados (sem progresso há N dias) | `progress_synced` ausente | Tabela | Curso |
| Engajamento de comunidade & SLA de resposta | K21 | Número + tabela | Curso |
| Velocidade de reprodução / uso de legenda | `video_rate_changed`, `video_caption_toggled` | Distribuição | Curso |
| Erros de player nas minhas aulas | K22 | Tabela | Aula, error_code |

### 5.4 Afiliado — Performance de divulgação (escopo do próprio `affiliate_id`)

| Widget | Métrica-fonte | Visualização | Filtros |
|--------|---------------|--------------|---------|
| Cliques nos meus links | `affiliate_link_clicked` | Linha | Curso, link/UTM |
| Conversão de afiliado | K16 | Funil (clique → atribuído → pago) | Curso |
| Comissão gerada / paga / estornada | K15 (`affiliate_commission_*`) | Número + tabela | Status, curso |
| Vendas atribuídas e ticket médio | `affiliate_attributed`, `order_paid` | Tabela | Curso |
| Top cursos que convertem | K16 por curso | Barra | — |

### 5.5 Aluno — Meu progresso (escopo do próprio `user_id`)

| Widget | Métrica-fonte | Visualização | Filtros |
|--------|---------------|--------------|---------|
| Progresso por curso matriculado | `progress_synced`, `course_progress_milestone` | Barra de progresso | Curso |
| Aulas concluídas / restantes | `lesson_completed` vs total | Contador | Curso |
| Continuar de onde parei | último `video_paused`/`progress_synced` | Card/CTA | — |
| Horas assistidas (minha) | K1 (escopo aluno) | Linha | Curso |
| Resultados de quizzes | `quiz_submitted`, `quiz_passed` | Tabela | Curso |
| Certificados emitidos | `certificate_issued`/`_downloaded` | Lista + download | — |
| Próxima aula liberada (drip) | `lesson_unlocked` | Card/contagem | Curso |

---

## 6. Mapeamento KPI ↔ North Star / métricas do PRD

> PRD §6 define a North Star e as métricas de suporte. Tabela cruza cada uma com os KPIs (§4).

| Métrica do PRD (§6) | KPIs que a medem |
|----------------------|------------------|
| **North Star — horas assistidas/aluno ativo/mês** | K1 (+ K8 watch-time como detalhe) |
| **Aquisição — tenants ativos, trial→pago SaaS** | K12, K23, `saas_trial_started`→`saas_subscription_created` |
| **Ativação — tenant publica ≥1 curso e ≥1 venda em 14d** | K2 (tenant); K3 (aluno) como complemento |
| **Receita — MRR SaaS, GMV dos tenants, take rate** | K12, K10, K11, K14 |
| **Retenção — churn de tenants, retenção de alunos por coorte, conclusão de curso** | K13, K9, K6, K20 |
| **Qualidade — custo de mídia/aluno ativo, vazamento cross-tenant (meta 0)** | K22 (proxy de entrega/erro); §7 (isolamento — guarda do "vazamento 0") |

KPIs adicionais de funil/operacionais que sustentam as métricas do PRD: K4, K5, K7, K15–K19, K24.

---

## 7. Implementação técnica & isolamento de tenant

Alinhado a CLAUDE.md (Regra nº1) e ARCHITECTURE §3/§10.

- **Wrapper único de captura** (`packages/analytics`, port `AnalyticsProvider` → impl PostHog) — **DRY** e DIP, igual a `JobQueue`/`VideoProvider`. Nenhum use-case chama o SDK do PostHog diretamente.
- **`tenant_id` injetado pelo wrapper**, lido do contexto de request (`request.tenant`, via `AsyncLocalStorage`) — **nunca** de header não autenticado. Eventos do worker leem `tenantId` do payload do job (mesmo padrão de `withTenant`). Isto torna impossível emitir evento sem tenant.
- **Group analytics PostHog:** `groupIdentify('tenant', tenant_id, …)` no provisionamento; toda captura associa o grupo `tenant` → permite isolamento e dashboards por tenant sem misturar.
- **Server-side primeiro** para eventos financeiros e de acesso (webhooks idempotentes = fonte de verdade), evitando dupla-contagem e fraude de client.
- **Volume / custo:** `video_heartbeat` agregado (delta de segundos) e com sampling configurável; considerar batch no PostHog. Evitar enviar heartbeat como evento individual de altíssima frequência sem throttle.
- **LGPD:** mascaramento de PII, `distinct_id` por tenant (sem e-mail em claro), respeito a consentimento (`consent` antes de tracking client-side), e suporte a deleção/exportação (alinhado ao direito ao esquecimento do PRD §3.9).
- **Teste de isolamento (gate de CI):** assim como dados, incluir teste que prova que evento emitido no contexto do tenant A nunca recebe `tenant_id` do tenant B e que filtros de dashboard por grupo `tenant` não vazam.
- **Reconciliação:** KPIs financeiros críticos (GMV, MRR, comissões) têm a fonte de verdade no Postgres; PostHog é camada de produto/funil. Dashboards financeiros executivos podem cruzar/validar contra o banco (job de reconciliação já previsto em ARCHITECTURE §8).

---

## 8. Dependências e pontos para o coordenador

Pontos que cruzam com outros agentes/documentos e precisam de alinhamento:

1. **UX de aprendizado (player / progresso):** os eventos `lesson_started`, `video_played/paused/seeked`, `video_heartbeat` e `progress_synced` dependem do contrato exato emitido pelo player (Bunny iframe no MVP; Vidstack na F2). É preciso confirmar com o agente de UX/Player o **throttle do heartbeat (5–15s)**, o cálculo de `watched_pct` e `seconds_watched_delta`, e a regra anti-seek (≥90% + tempo real ≥ duração×0.8 — PRD §4.4) para não divergir entre tracking de progresso (banco) e analytics (PostHog).

2. **Estados de pagamento:** os eventos da §3.3/§3.4 assumem a máquina de estados `orders.status` (`pending|paid|refunded|chargeback`) e `enrollments.status` (`active|suspended|refunded|expired`) do DATA_MODEL. Qualquer estado novo (ex.: `authorized`, `disputed`, `partial_refund`) ou transição de Pagar.me/Asaas precisa refletir aqui e no tracking plan. Confirmar nomes canônicos com o agente de Pagamentos/Webhooks.

3. **Pricing & billing SaaS:** K11 (take rate), K12 (MRR) e os eventos `saas_*` dependem da modelagem final de planos/quotas (`platform_plans.limits`) e do mapeamento Stripe Billing → eventos. Definir com o agente de Pricing: existe **fee transacional** sobre GMV (além da mensalidade)? Isso muda a fórmula de take rate.

4. **Afiliados/split:** o modelo de atribuição assumido é **last-click** (`affiliate_attributed`). Confirmar janela de atribuição e tratamento de co-produção (`splits`) com o agente de Afiliados — afeta K15/K16.

5. **Cart abandonment & order bump/upsell (F2):** `cart_abandoned`, `has_order_bump` dependem de funcionalidades de Fase 2. Os eventos estão definidos, mas só instrumentar quando a feature existir (evitar evento "morto").

6. **Consentimento/LGPD:** depende da camada de consent management (cookie banner) definida pelo agente de Compliance — o tracking client-side só dispara após consentimento. Confirmar fluxo.

7. **`packages/analytics` (port):** é uma nova dependência arquitetural. **Resolvido pela coordenação:**
   registrado em [ADR-0014](../adr/0014-analytics-provider-tracking-plan.md) — port `AnalyticsProvider`
   (impl PostHog, group analytics por tenant/course) + **tracking plan como schemas Zod em
   `packages/contracts`**. Heartbeats são **agregados** (não enviados crus ao PostHog); progresso/anti-seek
   é fonte de verdade no banco (`lesson_progress.real_watched_seconds`).
