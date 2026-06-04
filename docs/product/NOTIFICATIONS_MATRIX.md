# Matriz de Notificações e Comunicação

- **Versão:** 1.0 · **Data:** 2026-06-04
- **Status:** Proposto para revisão do coordenador
- **Relacionados:** [PRD.md](../PRD.md) · [ARCHITECTURE.md](../ARCHITECTURE.md) · [BUSINESS_RULES_AND_STATES.md](BUSINESS_RULES_AND_STATES.md) · [RBAC_MATRIX.md](RBAC_MATRIX.md) · [DATA_MODEL.md](../DATA_MODEL.md)

> Matriz de **Evento × Canal × Destinatário × Gatilho × Prioridade × Template**. E-mail transacional via
> **Resend**; in-app sempre disponível; **push (Web Push/PWA)** é F2. Notificações são disparadas por
> transições das máquinas de estado (BUSINESS_RULES) e processadas via `JobQueue` (pg-boss) com
> `tenantId` no payload. Templates respeitam o **branding/i18n do tenant** e a base WCAG AA.

---

## Índice

- [0. Convenções, canais e prioridades](#0-convenções-canais-e-prioridades)
- [1. Eventos transacionais — Pagamento e acesso](#1-eventos-transacionais--pagamento-e-acesso)
- [2. Eventos transacionais — Aprendizado e conteúdo](#2-eventos-transacionais--aprendizado-e-conteúdo)
- [3. Eventos transacionais — Afiliados e financeiro](#3-eventos-transacionais--afiliados-e-financeiro)
- [4. Eventos de conta e segurança](#4-eventos-de-conta-e-segurança)
- [5. Eventos de plataforma / Tenant (control plane)](#5-eventos-de-plataforma--tenant-control-plane)
- [6. Eventos de engajamento e ciclo de vida](#6-eventos-de-engajamento-e-ciclo-de-vida)
- [7. Webhooks de saída (integração do tenant)](#7-webhooks-de-saída-integração-do-tenant)
- [8. Placeholders comuns dos templates](#8-placeholders-comuns-dos-templates)
- [Dependências e pontos para o coordenador](#dependências-e-pontos-para-o-coordenador)

---

## 0. Convenções, canais e prioridades

**Canais:** `E-mail` (Resend, transacional) · `In-app` (central de notificações) · `Push` (Web
Push/PWA — **F2**).

**Prioridades:**
- **P0 Crítico** — segurança/dinheiro/acesso (envio imediato, não suprimível por preferências).
- **P1 Alto** — transacional importante (envio imediato; respeita opt-out só onde for marketing).
- **P2 Médio** — engajamento/relacional (pode agrupar/digerir; respeita preferências).
- **P3 Baixo** — marketing/ciclo de vida (opt-in/opt-out; sujeito a anti-spam e LGPD).

**Regras gerais:**
- Disparo a partir de **transições idempotentes** (sem duplicar e-mail em reprocessamento de webhook).
- **Tenant suspenso** (BUSINESS_RULES §1): comunicações a alunos pausadas, exceto avisos de
  indisponibilidade; comunicação ao **owner** sobre o estado do tenant continua.
- Marketing (P3) exige consentimento (LGPD) e link de descadastro; transacional (P0–P1) não.
- Templates parametrizados por **branding/i18n do tenant**; placeholders na §8.
- Push só após permissão do navegador (F2); fallback para in-app/e-mail.

Legenda destinatários: **ST** Aluno · **IN** Instrutor · **OW/AD** Owner/Admin · **AF** Afiliado ·
**SA** Super-Admin.

---

## 1. Eventos transacionais — Pagamento e acesso

| Evento | E-mail | In-app | Push (F2) | Destinatário | Gatilho (transição) | Prio | Template / placeholders |
|--------|:------:|:------:|:---------:|--------------|---------------------|:----:|--------------------------|
| Compra aprovada | ✅ | ✅ | ✅ | ST | Pedido `pending→paid` (§5) | P0 | `purchase_approved` · `{student_name,course_title,order_id,amount,access_url}` |
| Nova venda recebida | ✅ | ✅ | ✅ | OW/AD, IN(autor) | Pedido `→paid` | P1 | `sale_received` · `{course_title,buyer_name,amount,net_amount}` |
| Pagamento recusado | ✅ | ✅ | — | ST | Cartão recusado / `pending` falho (§10.1) | P1 | `payment_failed` · `{course_title,reason,retry_url}` |
| Pagamento pendente (Pix/boleto) | ✅ | ✅ | — | ST | Pedido `pending` (Pix/boleto emitido) | P1 | `payment_pending` · `{course_title,method,pix_code/boleto_url,expires_at}` |
| Pix/boleto expirado | ✅ | ✅ | — | ST | `pending→expirado` (§10.1) | P2 | `payment_expired` · `{course_title,new_checkout_url}` |
| Matrícula confirmada / acesso liberado | ✅ | ✅ | ✅ | ST | Matrícula `→active` (§4) | P0 | `enrollment_active` · `{course_title,access_url,start_here}` |
| Matrícula manual/cortesia | ✅ | ✅ | — | ST | Matrícula manual/bulk/auto `→active` | P1 | `enrollment_granted` · `{course_title,granted_by,access_url}` |
| Reembolso processado | ✅ | ✅ | — | ST | Pedido `→refunded` (§5/§10.2) | P0 | `refund_processed` · `{course_title,amount,refund_id}` |
| Reembolso emitido (aviso ao tenant) | ✅ | ✅ | — | OW/AD | Pedido `→refunded` | P1 | `refund_issued_admin` · `{course_title,buyer_name,amount}` |
| Chargeback aberto | ✅ | ✅ | — | OW/AD | Pedido `→chargeback` (§5) | P0 | `chargeback_opened` · `{order_id,buyer_name,amount}` |
| Acesso suspenso (inadimplência/chargeback) | ✅ | ✅ | — | ST | Matrícula `→suspended` (§4) | P1 | `access_suspended` · `{course_title,reason,resolve_url}` |
| Acesso expirado | ✅ | ✅ | — | ST | Matrícula `→expired` (§4) | P2 | `access_expired` · `{course_title,renew_url}` |
| Assinatura: cobrança falhou (dunning) | ✅ | ✅ | — | ST | `subscriptions active→past_due` (§6) | P1 | `sub_payment_failed` · `{plan,attempt,update_card_url,grace_until}` |
| Assinatura: renovada | ✅ | ✅ | — | ST | `subscriptions →active` (renovação) | P2 | `sub_renewed` · `{plan,period_end,amount}` |
| Assinatura: cancelada | ✅ | ✅ | — | ST | `subscriptions →canceled` | P1 | `sub_canceled` · `{plan,access_until}` |
| Trial terminando | ✅ | ✅ | ✅ | ST | `subscriptions trialing` (T-N dias) | P2 | `trial_ending` · `{plan,trial_end,upgrade_url}` |
| Cupom aplicado/inválido | — | ✅ | — | ST | Checkout (cupom validado/§10.6) | P3 | `coupon_feedback` · `{code,discount/reason}` |

---

## 2. Eventos transacionais — Aprendizado e conteúdo

| Evento | E-mail | In-app | Push (F2) | Destinatário | Gatilho | Prio | Template / placeholders |
|--------|:------:|:------:|:---------:|--------------|---------|:----:|--------------------------|
| Vídeo pronto (encoding ok) | — | ✅ | — | IN, OW/AD | Vídeo `processing→ready` (§3) | P1 | `video_ready` · `{lesson_title,course_title,publish_url}` |
| Falha no processamento de vídeo | ✅ | ✅ | — | IN, OW/AD | Vídeo `→failed` (§3/§10.5) | P1 | `video_failed` · `{lesson_title,error,reupload_url}` |
| Nova aula liberada (drip) | ✅ | ✅ | ✅ | ST | Drip atingido (§10.4) | P2 | `lesson_unlocked` · `{lesson_title,course_title,lesson_url,release_at}` |
| Curso publicado (para inscritos/lista) | ✅ | ✅ | ✅ | ST(opt-in) | Curso `draft→published` (§2) | P2 | `course_published` · `{course_title,landing_url}` |
| Nova resposta em comentário | ✅ | ✅ | ✅ | ST/IN (autor do tópico) | Novo `lesson_comments` filho | P2 | `comment_reply` · `{replier_name,lesson_title,thread_url}` |
| Nova dúvida em aula (para instrutor) | — | ✅ | ✅ | IN(autor curso) | Novo comentário em aula do curso | P2 | `new_question` · `{student_name,lesson_title,thread_url}` |
| Comentário moderado/removido | ✅ | ✅ | — | ST(autor) | Moderação (RBAC §3.10) | P2 | `comment_moderated` · `{lesson_title,reason}` |
| Quiz corrigido / resultado | — | ✅ | — | ST | `quiz_attempts` submetido (§3.6) | P2 | `quiz_result` · `{quiz_title,score,passed}` |
| Marco de progresso (50/100%) | — | ✅ | ✅ | ST | `lesson_progress`/curso concluído | P3 | `progress_milestone` · `{course_title,pct}` |
| Certificado emitido | ✅ | ✅ | ✅ | ST | Certificado `→issued` (§7) | P1 | `certificate_issued` · `{course_title,cert_url,verify_url,qr}` |
| Certificado revogado | ✅ | ✅ | — | ST | Certificado `→revoked` (§7) | P1 | `certificate_revoked` · `{course_title,reason}` |

---

## 3. Eventos transacionais — Afiliados e financeiro

| Evento | E-mail | In-app | Push (F2) | Destinatário | Gatilho | Prio | Template / placeholders |
|--------|:------:|:------:|:---------:|--------------|---------|:----:|--------------------------|
| Afiliação aprovada | ✅ | ✅ | — | AF | `affiliates.status→active` | P1 | `affiliate_approved` · `{program,links_url,materials_url}` |
| Afiliação recusada/suspensa | ✅ | ✅ | — | AF | `affiliates.status→rejected/suspended` | P1 | `affiliate_status` · `{reason}` |
| Nova comissão gerada | ✅ | ✅ | ✅ | AF | Comissão `→pending` (§8) | P2 | `commission_earned` · `{course_title,amount,order_id}` |
| Comissão paga | ✅ | ✅ | — | AF | Comissão `pending→paid` (§8) | P1 | `commission_paid` · `{amount,period,payout_ref}` |
| Comissão revertida | ✅ | ✅ | — | AF | Comissão `→reversed` (§8/§10.2) | P1 | `commission_reversed` · `{amount,reason,order_id}` |
| Repasse de co-produção (F2) | ✅ | ✅ | — | IN/produtor | Split executado (§3.9) | P1 | `coproduction_payout` · `{course_title,amount,period}` |

---

## 4. Eventos de conta e segurança

| Evento | E-mail | In-app | Push (F2) | Destinatário | Gatilho | Prio | Template / placeholders |
|--------|:------:|:------:|:---------:|--------------|---------|:----:|--------------------------|
| Boas-vindas / confirmação de conta | ✅ | — | — | ST/IN/AF | Cadastro de usuário | P1 | `welcome` · `{name,verify_url,tenant_brand}` |
| Verificação de e-mail | ✅ | — | — | qualquer | Cadastro/alteração de e-mail | P0 | `email_verify` · `{verify_url,expires_at}` |
| Redefinição de senha | ✅ | — | — | qualquer | Solicitação de reset | P0 | `password_reset` · `{reset_url,expires_at,ip}` |
| Senha alterada | ✅ | ✅ | — | qualquer | Senha trocada | P0 | `password_changed` · `{when,ip,support_url}` |
| Novo login/dispositivo (F2) | ✅ | ✅ | — | qualquer | Login de novo dispositivo/IP | P1 | `new_login` · `{device,ip,location,when}` |
| Convite para equipe | ✅ | — | — | IN/AD | Convite (RBAC §3.3) | P1 | `team_invite` · `{inviter,role,accept_url}` |
| Alteração de papel | ✅ | ✅ | — | usuário-alvo | Papel alterado (RBAC §3.3) | P1 | `role_changed` · `{new_role,changed_by}` |
| Conta suspensa/reativada | ✅ | ✅ | — | usuário-alvo | Suspensão/reativação de usuário | P1 | `account_status` · `{status,reason}` |
| Impersonação iniciada (transparência) | — | ✅ | — | OW/AD do tenant | SA impersona (RBAC C3) | P1 | `impersonation_notice` · `{actor,when}` (config — ver Dependências) |
| Dados exportados prontos (LGPD) | ✅ | ✅ | — | solicitante | Export concluída (§3.12) | P1 | `data_export_ready` · `{download_url,expires_at}` |
| Conta/dados deletados (LGPD) | ✅ | — | — | solicitante | Deleção concluída | P1 | `data_deleted` · `{when}` |

---

## 5. Eventos de plataforma / Tenant (control plane)

| Evento | E-mail | In-app | Push (F2) | Destinatário | Gatilho | Prio | Template / placeholders |
|--------|:------:|:------:|:---------:|--------------|---------|:----:|--------------------------|
| Tenant provisionado (boas-vindas) | ✅ | ✅ | — | OW | Tenant `provisioning→active` (§1) | P1 | `tenant_welcome` · `{owner_name,admin_url,subdomain}` |
| Falha de provisionamento | ✅ | — | — | SA | Saga falha (§1/§10.11) | P0 | `provisioning_failed` · `{tenant,step,error}` |
| Fatura do SaaS emitida | ✅ | ✅ | — | OW | `platform.invoices` criada | P1 | `saas_invoice` · `{amount,due_at,pay_url}` |
| Cobrança do SaaS falhou | ✅ | ✅ | — | OW | Invoice `overdue` (dunning) | P0 | `saas_payment_failed` · `{amount,grace_until,update_url}` |
| Tenant suspenso (inadimplência SaaS) | ✅ | ✅ | — | OW | Tenant `active→suspended` (§1/§10.8) | P0 | `tenant_suspended` · `{reason,reactivate_url}` |
| Tenant reativado | ✅ | ✅ | — | OW | Tenant `suspended→active` | P1 | `tenant_reactivated` · `{when}` |
| Plano/quota próximo do limite | ✅ | ✅ | — | OW/AD | Quota (alunos/storage) ~80–100% | P2 | `quota_warning` · `{resource,used,limit,upgrade_url}` |
| Quota excedida (ação bloqueada) | ✅ | ✅ | — | OW/AD | Ação excede quota (§10.11) | P1 | `quota_exceeded` · `{resource,limit,upgrade_url}` |
| Cancelamento de tenant agendado | ✅ | ✅ | — | OW | Tenant `→cancelled` | P0 | `tenant_canceled` · `{access_until,export_url,purge_at}` |

---

## 6. Eventos de engajamento e ciclo de vida

| Evento | E-mail | In-app | Push (F2) | Destinatário | Gatilho | Prio | Template / placeholders |
|--------|:------:|:------:|:---------:|--------------|---------|:----:|--------------------------|
| Sequência de boas-vindas pós-matrícula | ✅ | — | — | ST | Matrícula `→active` (D0/D1/D3) | P3 | `onboarding_drip` · `{course_title,next_step}` |
| Lembrete de inatividade | ✅ | ✅ | ✅ | ST | Sem progresso há N dias | P3 | `inactivity_nudge` · `{course_title,resume_url}` |
| Carrinho abandonado (F2) | ✅ | — | — | ST/lead | Checkout iniciado sem `paid` | P3 | `abandoned_cart` · `{course_title,resume_checkout_url,coupon?}` |
| Conquista/badge (F2 gamificação) | — | ✅ | ✅ | ST | Badge concedido | P3 | `badge_earned` · `{badge_name,points}` |
| Resumo semanal do instrutor | ✅ | ✅ | — | IN, OW/AD | Agendado (semanal) | P3 | `weekly_digest` · `{sales,enrollments,completion,top_course}` |
| Live agendada / lembrete (F2) | ✅ | ✅ | ✅ | ST | Evento de live criado / T-1h | P2 | `live_reminder` · `{title,when,join_url}` |
| Recomendação de curso (F3) | ✅ | ✅ | — | ST(opt-in) | Recomendador | P3 | `course_recommendation` · `{courses[]}` |

---

## 7. Webhooks de saída (integração do tenant)

Canal técnico (não humano) — PRD §3.15. Entregues a endpoints do tenant (CRM/automação) via
`JobQueue`, com assinatura HMAC, `event_id` para idempotência, retry + backoff.

| Evento (webhook) | Gatilho | Payload-chave | Prio |
|------------------|---------|---------------|:----:|
| `order.paid` | Pedido `→paid` (§5) | `{order_id,user,course,amount,affiliate?}` | P0 |
| `order.refunded` | Pedido `→refunded` | `{order_id,amount}` | P1 |
| `order.chargeback` | Pedido `→chargeback` | `{order_id}` | P1 |
| `enrollment.created` | Matrícula `→active` (§4) | `{user,course,source}` | P1 |
| `enrollment.suspended/expired` | Matrícula `→suspended/expired` | `{user,course,status}` | P1 |
| `course.completed` | Curso 100% concluído | `{user,course,completed_at}` | P1 |
| `certificate.issued` | Certificado `→issued` (§7) | `{user,course,verify_url}` | P2 |
| `subscription.updated` | Mudança de estado (§6) | `{user,plan,status}` | P1 |
| `commission.updated` | Comissão muda de estado (§8) | `{affiliate,order,status,amount}` | P2 |

---

## 8. Placeholders comuns dos templates

Disponíveis a (quase) todos os templates, resolvidos por tenant/destinatário:

- **Tenant/marca:** `{tenant_name}`, `{tenant_logo_url}`, `{brand_primary_color}`, `{tenant_url}`,
  `{support_email}`, `{locale}`.
- **Destinatário:** `{recipient_name}`, `{recipient_email}`, `{recipient_role}`.
- **Curso/aula:** `{course_title}`, `{course_url}`, `{lesson_title}`, `{lesson_url}`,
  `{instructor_name}`.
- **Comercial:** `{amount}`, `{currency}`, `{order_id}`, `{payment_method}`, `{net_amount}`.
- **Sistema:** `{cta_url}`, `{expires_at}`, `{unsubscribe_url}` (somente P3), `{year}`, `{event_id}`.

Regras de template: respeitar `{locale}` (next-intl/i18n), contraste/labels WCAG AA, rodapé com
identificação do tenant e (em marketing) link de descadastro; transacional não traz descadastro.

---

## Dependências e pontos para o coordenador

1. **Preferências/opt-out:** o DATA_MODEL não tem tabela de preferências de notificação por usuário.
   Recomendo `notification_preferences(user_id, channel, category, enabled)` para honrar opt-out de P2/P3
   sem afetar P0/P1 (decisão de modelagem/migration).
2. **Central de notificações in-app:** sem tabela hoje. Sugiro `notifications(id,user_id,type,payload,
   read_at,created_at)` no data plane (escopo tenant).
3. **Push (F2):** confirmar provedor (Web Push nativo vs OneSignal/Novu, citado no PRD §3.10) e armazenamento
   de subscriptions/tokens.
4. **Aviso de impersonação ao tenant:** decidir se OW/AD são notificados quando SA impersona
   (transparência) ou se fica apenas no `audit_log` (RBAC C3).
5. **Janela/dunning de assinatura:** parâmetros de `trial_ending` (T-N dias) e número de e-mails de
   dunning alinhados com BUSINESS_RULES (dep. 5 de lá).
6. **Catálogo canônico de `event types`:** consolidar os nomes de evento (transacionais + webhooks de
   saída) num enum único reutilizado em `packages/contracts` (DRY) — base para templates e webhooks.
7. **Throttling/digest:** definir agrupamento de P2/P3 (ex.: resumo diário de comentários) para evitar
   ruído e custo de e-mail.
8. **i18n de templates:** confirmar que o conjunto inicial é PT-BR com arquitetura pronta para ES/EN
   (PRD §3.11).
