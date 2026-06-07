# Console Super-Admin — Plataforma de Cursos SaaS Multitenant

- **Versão:** 1.0 · **Data:** 2026-06-07
- **Status:** Proposta para validação do coordenador de produto
- **Persona:** Super-Admin (SA) — staff interno (nós, a plataforma). Identidade global no schema `platform`.
- **Host:** `admin.app.com` (control plane, host dedicado — **proposta a validar com engenharia**, ver
  [IA #1](INFORMATION_ARCHITECTURE.md) / [OPEN_QUESTIONS #23](../OPEN_QUESTIONS.md)).
- **Documentos relacionados:** [RBAC_MATRIX.md](RBAC_MATRIX.md) (§3.1, condições C1–C3) ·
  [DATA_MODEL.md](../DATA_MODEL.md) (§1 control plane) · [ARCHITECTURE.md](../ARCHITECTURE.md) (§3, §6, §8) ·
  [MONETIZATION.md](MONETIZATION.md) (A.x billing/quotas) · [ANALYTICS_AND_DASHBOARDS.md](ANALYTICS_AND_DASHBOARDS.md) (§5.1) ·
  [BUSINESS_RULES_AND_STATES.md](BUSINESS_RULES_AND_STATES.md) · [INFORMATION_ARCHITECTURE.md](INFORMATION_ARCHITECTURE.md) (§2.6) ·
  [README.md](README.md) (glossário) · [CLAUDE.md](../../CLAUDE.md)

> **Princípio mestre (Regra nº1 + RBAC §2.2):** o SA opera **exclusivamente** sobre o control plane
> (`platform`). Para tocar **qualquer** dado de tenant, há **impersonação auditada** — **nunca** acesso
> silencioso. Toda ação sensível (provisionar, suspender, cancelar, impersonar, rotacionar chave,
> alterar plano/quota) grava `platform.audit_log`. O SA **não** lê tabelas `tenant_*` diretamente do
> console; o que vê do tenant são **agregados/contadores de uso** computados via `withTenant` por um
> serviço, sem expor conteúdo.

---

## Índice

1. [Convenções, fases e RBAC do console](#1-convenções-fases-e-rbac-do-console)
2. [Mapa de telas (control plane)](#2-mapa-de-telas-control-plane)
3. [Dashboard global — Saúde do SaaS](#3-dashboard-global--saúde-do-saas)
4. [Lista e gestão de tenants](#4-lista-e-gestão-de-tenants)
5. [Detalhe do tenant — visão geral, uso vs quota e saúde](#5-detalhe-do-tenant--visão-geral-uso-vs-quota-e-saúde)
6. [Provisionamento / onboarding (saga)](#6-provisionamento--onboarding-saga)
7. [Suspender / reativar / cancelar tenant (máquina de estados)](#7-suspender--reativar--cancelar-tenant-máquina-de-estados)
8. [Billing do SaaS (Stripe, faturas, inadimplência/dunning)](#8-billing-do-saas-stripe-faturas-inadimplênciadunning)
9. [Impersonação auditada (UX, audit_log, LGPD)](#9-impersonação-auditada-ux-audit_log-lgpd)
10. [Gestão de planos, quotas e feature flags](#10-gestão-de-planos-quotas-e-feature-flags)
11. [Observabilidade por tenant (métricas e incidentes)](#11-observabilidade-por-tenant-métricas-e-incidentes)
12. [Gestão de chaves e integrações por tenant (cifradas)](#12-gestão-de-chaves-e-integrações-por-tenant-cifradas)
13. [Suporte ao tenant (nível 2)](#13-suporte-ao-tenant-nível-2)
14. [Auditoria, super-admins e segurança do console](#14-auditoria-super-admins-e-segurança-do-console)
15. [Modelo de auditoria — vocabulário de `audit_log.action`](#15-modelo-de-auditoria--vocabulário-de-audit_logaction)
16. [Dependências e pontos para o coordenador](#dependências-e-pontos-para-o-coordenador)

---

## 1. Convenções, fases e RBAC do console

### 1.1 Legenda de fase

- **[MVP]** no lançamento · **[F2]** Fase 2 · **[F3]** Fase 3.

### 1.2 RBAC interno do console (papéis de SA)

`super_admins.role` distingue papéis internos da plataforma (não confundir com papéis de tenant). Proposta:

| Papel SA | Descrição | Pode |
|----------|-----------|------|
| **sa_owner** | Operador-chefe da plataforma | Tudo, incluindo gerir super-admins, planos e cancelamento/purge de tenant |
| **sa_ops** | Operações/provisionamento | Provisionar, retomar saga, suspender/reativar, ver billing, impersonar |
| **sa_support** | Suporte nível 2 | Impersonar (read-only por padrão), ver tenant/uso, abrir notas de suporte; **não** suspende/cancela nem altera planos |
| **sa_billing** | Financeiro do SaaS | Billing/faturas/dunning, ajustes de assinatura; **não** impersona conteúdo |
| **sa_readonly** | Auditoria/observabilidade | Apenas leitura de dashboards e `audit_log` |

> **MVP:** pode-se entregar com 2 papéis (`sa_owner`, `sa_ops`) e evoluir os demais na F2. Cada ação
> abaixo lista o papel mínimo. Defense in depth: o guard Fastify do control plane **revalida** o papel
> no use-case (ARCHITECTURE §6).

### 1.3 Regras transversais (todas as telas)

- **MFA obrigatório** para qualquer SA (RBAC dependência #5; `super_admins.mfa_enabled=true` é gate de login). [MVP]
- **Toda ação mutável** grava `platform.audit_log` com `actor_id` (SA), `actor_type='super_admin'`,
  `tenant_id` (quando aplicável), `action`, `metadata` (ver §15). [MVP]
- **Sem leitura silenciosa de dados de tenant.** Telas do console mostram dados do `platform` e
  **agregados de uso** (contadores via serviço de quotas). Ver §5.2. [MVP]
- **Step-up de autenticação** (re-MFA / confirmação por digitação do slug) para ações destrutivas:
  suspender, cancelar, purgar, rotacionar/expor chave, impersonar. [MVP]

---

## 2. Mapa de telas (control plane)

Alinhado a [IA §2.6](INFORMATION_ARCHITECTURE.md). Host `admin.app.com`:

```
admin.app.com
├─ /login ............................. Login da plataforma + MFA (step-up)            [MVP]
├─ / (Dashboard) ...................... Saúde do SaaS (MRR, tenants, quotas, incidentes) [MVP]  §3
├─ /tenants ........................... Lista/gestão de tenants                          [MVP]  §4
│   ├─ /tenants/novo ................. Criar tenant (dispara saga)                       [MVP]  §6
│   └─ /tenants/{id}
│       ├─ (visão geral) ............. Status, plano, uso/quota, saúde                   [MVP]  §5
│       ├─ /provisionamento .......... Progresso/retry da saga                           [MVP]  §6
│       ├─ /usuarios ................. Usuários do tenant → impersonar                   [MVP]  §9
│       ├─ /billing .................. Assinatura/faturas do SaaS (Stripe)               [MVP]  §8
│       ├─ /integracoes .............. Bunny / pagamento / LiveKit (chaves cifradas)     [MVP/F2] §12
│       ├─ /observabilidade .......... Métricas/erros/incidentes do tenant               [MVP]  §11
│       ├─ /suporte .................. Notas/tickets nível 2                             [F2]   §13
│       └─ /acoes .................... Suspender / reativar / cancelar                   [MVP]  §7
├─ /planos ............................ Planos do SaaS + quotas + feature flags          [MVP]  §10
├─ /faturamento ....................... Faturas/assinaturas do SaaS (global) + dunning   [MVP]  §8
├─ /provisionamento ................... Fila global de jobs de provisionamento           [MVP]  §6
├─ /observabilidade ................... Saúde/erros globais (links Sentry/OTel)          [MVP]  §11
├─ /auditoria ......................... Audit log global (impersonação/suspensões)       [MVP]  §14
└─ /super-admins ...................... Gestão da equipe interna + MFA                   [MVP]  §14
```

---

## 3. Dashboard global — Saúde do SaaS

> Fonte: [ANALYTICS §5.1](ANALYTICS_AND_DASHBOARDS.md) (dashboard Super-Admin, cross-tenant). Espelha
> aqui como tela operacional com **ações** (não só leitura). **[MVP]** widgets básicos; **[F2]** coortes/heatmaps.

| Widget | Métrica/fonte | Ação rápida | Fase |
|--------|---------------|-------------|------|
| MRR e crescimento | K12 (`saas_subscription_*`) | → /faturamento | MVP |
| Churn de tenants & MRR churn | K13 | → tenant em risco | F2 |
| GMV agregado por tenant | K10 | → detalhe do tenant | MVP |
| Tenants por status (active/provisioning/suspended/cancelled) | `platform.tenants.status` | filtro → /tenants | MVP |
| Sucesso de provisionamento + latência p95 | K23 (§3.11 analytics) | → /provisionamento | MVP |
| Uso de quota — tenants perto do limite | K24 (serviço de quotas) | → /tenants/{id} | MVP |
| Falhas de cobrança SaaS (dunning) | `saas_invoice_payment_failed` | → /faturamento | MVP |
| Impersonações ativas/recentes | `super_admin_impersonation_*` | → /auditoria | MVP |
| Saúde de entrega de vídeo (erros/buffering) | K22 | → /observabilidade | MVP |
| Incidentes abertos | Sentry/OTel (link) | → ferramenta externa | MVP |

- **RBAC:** leitura para todos os SA (`sa_readonly`+). Sem mutação direta. **Auditoria:** somente leitura
  (acesso ao dashboard não gera evento; cada drill-down que mute, sim).

---

## 4. Lista e gestão de tenants

**Rota:** `/tenants` · **Fase:** MVP · **Papel mínimo:** `sa_readonly` (ver) / `sa_ops` (agir)

### 4.1 Tabela principal

Colunas: **nome/slug**, **status** (badge: `provisioning | active | past_due[derivado] | suspended | cancelled`),
**plano** (Starter/Pro/Scale/Enterprise), **uso vs quota** (mini-barras: alunos ativos, storage, banda),
**saúde** (semáforo: provisionamento ok? cobrança em dia? erros recentes?), **MRR**, **GMV (30d)**,
**criado em**, **última atividade**.

- **Status `past_due/grace`** é **derivado** de `platform_subscriptions.status` (não é coluna em
  `tenants.status`) — exibido como sub-badge sobre `active` (MONETIZATION §A.5).
- **Filtros:** status, plano, inadimplência, quota estourada, com incidente aberto, sem 1ª venda (ativação).
- **Busca:** por nome, slug, domínio próprio, e-mail do owner.
- **Ações em lote [F2]:** exportar CSV, enviar aviso de quota.

### 4.2 Fonte de dados (sem violar isolamento)

- Linhas e status: `platform.tenants` + `platform_subscriptions` + `platform_plans`. [control plane]
- Uso corrente: **serviço de quotas** (DATA_MODEL §6.12) — alunos/cursos/equipe/afiliados contados
  via `withTenant` no schema do tenant; storage/banda via **API da Bunny**; limites via `platform_plans.limits`.
  **Nenhuma FK cross-schema**; a composição ocorre no use-case (Regra nº1).

- **Auditoria:** abrir a lista não audita. Criar tenant (§6), entrar em ações (§7), impersonar (§9) auditam.

---

## 5. Detalhe do tenant — visão geral, uso vs quota e saúde

**Rota:** `/tenants/{id}` · **Fase:** MVP · **Papel mínimo:** `sa_support` (ver)

### 5.1 Cabeçalho

Nome, slug, `schema_name`, status (com derivação past_due), plano + ciclo, owner (e-mail/nome —
do control plane via convite, ou via impersonação se vier do schema), datas, `custom_domain` (F2),
botão **Impersonar** (§9) e menu **Ações** (§7).

### 5.2 Painel de uso vs quota

Para cada quota de `platform_plans.limits` (`max_students`, `storage_gb`, `bandwidth_gb`, `max_courses`,
`max_team`, `max_affiliates`; **[F2]** `live_concurrent_participants`, `live_hours_month`):

| Coluna | Origem |
|--------|--------|
| Limite | `platform_plans.limits.<chave>` |
| Uso atual | serviço de quotas (Bunny p/ storage/banda; contagem no schema p/ o resto) |
| % e estado | `< 80%` ok · `80–100%` alerta · `> 100%` overage/bloqueio (política MONETIZATION §A.4) |

- "ilimitado" = ausência/`null` da chave (fair-use). Mostra alerta de abuso quando aplicável.
- **Ações:** "Conceder add-on/quota extra" e "Forçar recálculo de uso" (`sa_ops`+) → gera item de fatura
  Stripe (§8) e/ou atualiza override. **Auditoria:** `quota.override.granted`.

### 5.3 Saúde do tenant

- Provisionamento: último estado da saga (§6).
- Billing: status da assinatura SaaS, próxima fatura, falhas de cobrança.
- Pagamentos do tenant (B2C): recipient Pagar.me ativado? (sem recipient → checkout do tenant indisponível —
  MONETIZATION §6/26). Mostrar flag, não os dados.
- Entrega de vídeo: erros/buffering recentes (K22, agregado).
- Incidentes: links Sentry/OTel filtrados por `tenant_id` (tag).

- **Auditoria:** leitura não audita; é dado do control plane/agregado.

---

## 6. Provisionamento / onboarding (saga)

> Saga idempotente em `platform.provisioning_jobs` (ARCHITECTURE §3.4; DATA_MODEL §1). Executada por job
> no worker; o console **acompanha, retoma e reprocessa**.

### 6.1 Criar tenant — `/tenants/novo` · MVP · `sa_ops`

Formulário: nome, **slug** (valida unicidade e DNS de subdomínio), **plano inicial**, e-mail do **owner**
(para convite), ciclo de cobrança/trial, locale-base. Submeter → `INSERT tenant (status='provisioning',
idempotency_key)` e **dispara a saga**.

Passos da saga (espelham ARCHITECTURE §3.4), exibidos como timeline:

1. `tenant.insert` → `2. schema.create` → `3. migrations.run` → `4. seed` (papéis, owner do tenant, curso
   exemplo) → `5. bunny.library.create` (+ keys cifradas) → `5b. livekit.project.provision` **[F2]**
   (pós-onboarding, não bloqueia) → `6. routing.register + status=active + smoke_test`.

### 6.2 Acompanhar saga — `/tenants/{id}/provisionamento` e fila global `/provisionamento` · MVP

Por job (`provisioning_jobs`): `step`, `status`, `attempts`, `last_error`, `updated_at`, `idempotency_key`.

- **Estados visíveis:** `pending | running | completed | failed`. Tenant em `provisioning` até saga ok.
- **Ações (`sa_ops`):**
  - **Retomar/Reprocessar passo** — re-enfileira o passo falho (idempotente por `idempotency_key` +
    `IF NOT EXISTS`/migrations versionadas). `audit: tenant.provisioning.retried`.
  - **Reexecutar smoke test** — `audit: tenant.provisioning.smoke_retested`.
  - **Cancelar provisionamento** — aborta tenant que nunca ativou; se schema foi criado, oferece purge
    (§7). `audit: tenant.provisioning.aborted`.
- **Fila global:** lista jobs de todos os tenants, filtro por `step`/`status`, alerta de jobs presos
  (sem progresso > X min) e latência p95 (K23).

- **Analytics:** emite/consome `tenant_provisioning_*` (ANALYTICS §3.11). **Auditoria:** toda retomada/abort
  → `audit_log`.

---

## 7. Suspender / reativar / cancelar tenant (máquina de estados)

> **Máquina de estados de `tenants.status`** (DATA_MODEL §6.0; MONETIZATION §A.5). `past_due/grace` é
> **derivado** de `platform_subscriptions.status`, **não** é coluna. O console oferece transições manuais;
> as automáticas vêm do Stripe (§8).

```
provisioning ──(saga ok)──▶ active ──(suspensão manual / grace esgotado)──▶ suspended
                              ▲                                              │
                              └────────────(reativação / pagamento)─────────┘
   active/suspended ──(cancelamento)──▶ cancelled ──(janela de retenção)──▶ [DROP SCHEMA / purge]
```

**Rota:** `/tenants/{id}/acoes` · **Fase:** MVP

| Ação | Papel | Efeito no tenant | Efeito nos alunos | Estado | Auditoria |
|------|-------|------------------|-------------------|--------|-----------|
| **Suspender** | `sa_ops` | Painel admin em modo leitura; checkout do tenant off (novas vendas off); conteúdo preservado | **Política sensível** (decisão pendente): manter alunos pagantes ativos vs pausar acesso (MONETIZATION §A.5 / dep. #2). Default proposto: alunos pagantes mantêm acesso; bloqueia só painel/novas vendas | `active → suspended` | `tenant.suspended` (+ `reason`) |
| **Reativar** | `sa_ops` | Restaura acesso pleno; reabre checkout | Acesso normal | `suspended → active` | `tenant.reactivated` |
| **Cancelar** | `sa_owner` | Sem acesso ao painel; inicia janela de retenção | Sem acesso; **exportação disponível** (LGPD/lock-in) | `→ cancelled` | `tenant.cancelled` (+ `reason`, `requested_by`) |
| **Purgar (DROP SCHEMA)** | `sa_owner` | `DROP SCHEMA tenant_<slug> CASCADE` após retenção | Dados removidos | `cancelled → [purgado]` | `tenant.purged` |

**Regras-chave:**

- **Suspensão nunca apaga dados.** `DROP SCHEMA` só após `cancelled` + janela de retenção + oferta de
  exportação (ARCHITECTURE §3.6; PRD lock-in). [MVP]
- **Cancelar/purgar (RBAC C1):** o **owner solicita** o cancelamento; o **SA executa** o purge. O console
  registra `requested_by` (owner) quando aplicável; SA não dispara DROP sem solicitação válida ou política
  de inadimplência prolongada. [MVP]
- **Reativação automática** após pagamento volta a `active` via **webhook Stripe idempotente** (§8) — a
  transição manual aqui é para casos operacionais/exceções.
- **Step-up obrigatório** (re-MFA + digitar o slug) para suspender/cancelar/purgar.
- **Distinção de máquinas de estado:** inadimplência do **SaaS** (tenant→nós) ≠ inadimplência do **aluno**
  (aluno→tenant). Esta tela atua só na primeira (MONETIZATION §A.5; dep. #2).

- **Analytics:** `tenant_suspended` / `tenant_reactivated` (ANALYTICS §3.11).

---

## 8. Billing do SaaS (Stripe, faturas, inadimplência/dunning)

> Domínio **separado** do checkout dos alunos (README §3). **Stripe Billing** no control plane:
> `platform_subscriptions`, `platform_invoices` (DATA_MODEL §1). **Papel mínimo:** `sa_billing`/`sa_ops`.

### 8.1 Global — `/faturamento` · MVP

- **MRR/ARR**, faturas emitidas/pagas/pendentes, **falhas de cobrança (dunning)**, próximos vencimentos.
- **Lista de faturas** (`platform_invoices`): tenant, `amount_cents`, `currency`, `status`, `due_at`,
  `paid_at`. Filtros por status/tenant/período.
- **Painel de dunning:** tenants em `past_due` (Stripe), tentativa atual, fim da janela de grace (7–14d,
  MONETIZATION §A.3/§A.5), CTA "Suspender ao esgotar grace" (link §7).

### 8.2 Por tenant — `/tenants/{id}/billing` · MVP

- Assinatura: `plan_id`, ciclo, `status` (`trialing | active | past_due | canceled | expired`),
  `current_period_end`, `stripe_subscription_id` (link para o Stripe).
- Faturas do tenant + recibos (Resend).
- **Ações (`sa_billing`):**
  - **Trocar plano / aplicar proration** (upgrade imediato; downgrade no fim do período, bloqueado se
    excede quota de destino — MONETIZATION §A.4). `audit: saas.subscription.plan_changed`.
  - **Conceder crédito/cupom/exceção** (ex.: take rate negociado Enterprise, desconto). `audit:
    saas.billing.credit_granted` / `saas.takerate.override`.
  - **Reemitir/forçar cobrança / marcar como paga** (conciliação). `audit: saas.invoice.adjusted`.
  - **Iniciar/encerrar trial manualmente** (Scale/Enterprise via sales). `audit: saas.trial.adjusted`.

### 8.3 Reativação e idempotência

- Pagamento bem-sucedido → webhook Stripe (idempotente) → assinatura `active` → tenant volta a `active`
  automaticamente (sem ação manual). O console mostra o resultado; ações manuais são exceção.
- **Take rate** (§A.1) é operacionalizado **no split do checkout do aluno** (não na fatura Stripe). O
  console gerencia aqui apenas a **definição** de `take_rate_bps` por plano (§10) e overrides por tenant.

- **Analytics:** `saas_*` (ANALYTICS §3.12). **Auditoria:** toda alteração de assinatura/fatura/exceção.

---

## 9. Impersonação auditada (UX, audit_log, LGPD)

> **A função mais sensível.** RBAC §3.1 (🔶 C3) e §2.2: SA só toca dados de tenant **via impersonação**,
> **sempre** com `platform.audit_log`. **Sem acesso silencioso.** Espelha [USER_FLOWS] e ANALYTICS §3.1.

### 9.1 Entrada e UX

**Rota de entrada:** `/tenants/{id}/usuarios` (lista de usuários do tenant) ou botão **Impersonar** no
detalhe. **Fase:** MVP · **Papel mínimo:** `sa_support` (read-only) / `sa_ops` (escopo maior, ver dep. #5).

Fluxo:

1. SA seleciona o **usuário-alvo** e informa **motivo** (obrigatório; ex.: ticket #, diagnóstico) e
   **escopo** (default **read-only**; "full" exige `sa_ops`+ e step-up — dep. RBAC #5).
2. **Step-up de autenticação** (re-MFA). 
3. Sistema cria sessão de impersonação **com TTL curto** (ex.: 30–60 min, renovável com novo registro),
   escopada ao `tenant_id`/`user_id` alvo, **distinta** da sessão SA (não herda privilégios de control plane).
4. **Banner persistente e inconfundível** em toda a UI impersonada: "Você está como **{usuário}** no
   tenant **{slug}** — sessão auditada — [Encerrar]". Cor de alerta. Nunca esconde o estado.
5. Encerrar (ou expirar) volta ao console e fecha o registro.

### 9.2 Fronteiras de privacidade / LGPD

- **Sem acesso silencioso:** impossível ver conteúdo de tenant sem abrir impersonação registrada. O
  console em si só mostra agregados (§5.2).
- **Read-only por padrão:** ações de escrita exigem escopo "full" + papel + step-up; cada **ação mutável**
  dentro da impersonação gera seu próprio evento de auditoria (não só o início/fim).
- **Minimização:** banner sempre visível; TTL curto; motivo obrigatório; **notificação ao owner/admin do
  tenant** sobre a impersonação (transparência — recomendado, ver dep. #4).
- **PII:** o `audit_log.metadata` registra `target_user_id`/`tenant_id`, **não** conteúdo sensível visto.
  Analytics (`super_admin_impersonation_*`) é **sem PII** (ANALYTICS §3.1, §7).

### 9.3 Auditoria (obrigatória)

| Evento | `audit_log.action` | `metadata` |
|--------|--------------------|------------|
| Início | `tenant.impersonation.started` | `target_user_id`, `target_role`, `reason`, `scope` (`read_only`/`full`), `ttl_sec` |
| Ação mutável durante | `tenant.impersonation.action` | `domain`, `operation`, `entity_id` |
| Fim | `tenant.impersonation.ended` | `duration_sec`, `actions_count` |

- Espelha `super_admin_impersonation_started/ended` em PostHog (ANALYTICS §3.1) — fonte de verdade é o
  `audit_log` no Postgres.
- **Dashboard de impersonações** em `/auditoria` (§14) para revisão periódica.

---

## 10. Gestão de planos, quotas e feature flags

> Valores em `platform_plans.limits jsonb` (DATA_MODEL §6.11; MONETIZATION §A.2). **Papel mínimo:** `sa_owner`.

**Rota:** `/planos` · **Fase:** MVP

### 10.1 Editor de planos

Por plano (Starter/Pro/Scale/Enterprise): `name`, `price_cents`, `currency`, ciclo, e o `limits jsonb`:

| Chave | Tipo | Fase |
|-------|------|------|
| `max_students`, `storage_gb`, `bandwidth_gb`, `max_courses`, `max_team`, `max_affiliates` | int (ou ausente = ilimitado) | MVP |
| `take_rate_bps` | int (bps; `200` = 2,0% — sem float) | MVP |
| `features[]` | string[] (`order_bump`, `watermark`, `forum`, `api`, `live_classes`, `white_label`, ...) | MVP/F2 |
| `live_concurrent_participants`, `live_hours_month`, `live_recording_storage_gb` | int | F2 |

### 10.2 Feature flags

- Flags **booleanas por plano** vivem em `features[]` (entitlement por tier).
- **Overrides por tenant** (ex.: liberar `api` num Starter para um piloto): proposta de tabela/coluna de
  override no control plane (dep. #3) — sem isso, flags são só por plano. Cada override → `audit:
  plan.feature_override.set`.

### 10.3 Regras e segurança

- Alterar `take_rate_bps` ou quota **afeta cobrança/enforcement** → step-up + `audit:
  plan.updated` / `plan.takerate.changed` (RBAC §3.1; MONETIZATION dep. #1).
- O valor de quota/take rate **não** é consultado pelo use-case do tenant diretamente: é **injetado no
  `onRequest`** (resolvido via control plane) — Regra nº1 / DATA_MODEL §6.11. O console só edita a fonte.
- Mudança de quota **não** quebra tenants existentes retroativamente sem aviso: dispara recálculo e
  alertas (80/100%) via serviço de quotas.

- **Auditoria:** toda edição de plano/quota/flag/override.

---

## 11. Observabilidade por tenant (métricas e incidentes)

> ARCHITECTURE §10 (Pino/OTel/Sentry, `tenant_id` como tag/log). **Papel mínimo:** `sa_readonly`.

**Rotas:** `/observabilidade` (global) e `/tenants/{id}/observabilidade` · **Fase:** MVP

- **Métricas por tenant:** saúde de entrega de vídeo (K22 — `video_error/lesson_started`, % buffering),
  latência de API (OTel filtrado por tag `tenant_id`), volume de jobs/falhas (pg-boss), uso de quota (§5.2).
- **Incidentes:** lista de erros (Sentry) filtrada por `tenant_id`; links profundos para a ferramenta.
  **[F2]** registro interno de incidentes (status, severidade, postmortem) por tenant.
- **Alertas:** quota perto do limite (K24), falha de cobrança SaaS, jobs de provisionamento presos, pico
  de `video_error`. Surgem no Dashboard (§3) e na lista de tenants (semáforo de saúde).

- **Auditoria:** leitura não audita (são telemetria/agregados, sem dado de tenant). Abrir link externo
  segue a política da ferramenta.

---

## 12. Gestão de chaves e integrações por tenant (cifradas)

> Chaves Bunny/pagamento/LiveKit **nunca** no front (CLAUDE.md anti-padrão; RBAC C9). Cifradas em repouso
> (pgcrypto/KMS). **Papel mínimo:** `sa_ops` (rotacionar) / `sa_owner` (expor valor, se permitido).

**Rota:** `/tenants/{id}/integracoes` · **Fase:** MVP (Bunny/pagamento) · F2 (LiveKit)

| Integração | Onde | Campos | Ações no console |
|------------|------|--------|------------------|
| **Bunny Stream** | `platform.tenants.bunny_library_id`, `bunny_keys_encrypted` | library_id (visível), AccessKey/token key (cifradas) | Ver status, **rotacionar key**, recriar library (reprovisionar) |
| **Pagamento (Pagar.me/Asaas)** | recipient do produtor cifrado no **data plane**; recipient da plataforma em `platform.platform_payment_recipients` | recipient_id (mascarado), status KYC | Ver se ativado (sem recipient → checkout off), reabrir onboarding de pagamentos |
| **LiveKit [F2]** | `platform.tenants.live_keys_encrypted` (1 projeto/tenant) | api key/secret (cifradas) | Provisionar projeto, **rotacionar key** |

**Regras de segurança:**

- **Nunca exibir o valor em claro por padrão.** "Revelar" (se a política permitir) exige step-up + papel
  `sa_owner` e gera `audit: integration.key.revealed` (alto risco). Preferir **rotacionar** a revelar.
- **Rotacionar** gera nova chave, atualiza o cifrado e invalida a anterior. `audit:
  integration.key.rotated` (`provider`, `key_type`).
- **Recriar/reprovisionar** (Bunny library / LiveKit project) re-enfileira passo da saga (§6).
- **Recipient de pagamento do tenant** vive no **schema do tenant** (cifrado) — o console só mostra
  **status** (ativado/KYC), nunca o conteúdo; manipulação real é via impersonação/fluxo do owner (RBAC C9).

- **Auditoria:** rotação/exposição/reprovisionamento sempre auditados.

---

## 13. Suporte ao tenant (nível 2)

> Suporte interno escalado (níveis de SLA por plano — MONETIZATION §A.2). **Papel mínimo:** `sa_support`.

**Rota:** `/tenants/{id}/suporte` · **Fase:** F2 (MVP: suporte via impersonação + notas simples)

- **Notas/timeline de suporte** por tenant: interações, diagnósticos, ações tomadas. Vinculadas a
  ticket externo (Slack/helpdesk) se houver.
- **Ações de suporte comuns** (todas auditadas, dentro das fronteiras):
  - **Impersonar para diagnóstico** (§9) — caminho primário para "ver o que o cliente vê" sem acesso silencioso.
  - **Conceder add-on/quota temporária** (§5.2) ou **exceção de billing** (§8).
  - **Reprocessar provisionamento/saga** (§6) e **reenviar e-mails** (convite owner, recibo) via worker.
- **SLA visível** conforme o plano do tenant (comunidade/e-mail/chat/dedicado).

- **RBAC:** `sa_support` impersona read-only por padrão; suspender/cancelar/alterar plano **não** são de
  suporte (escala para `sa_ops`/`sa_owner`). **Auditoria:** cada ação de suporte.

---

## 14. Auditoria, super-admins e segurança do console

### 14.1 Audit log global — `/auditoria` · MVP · `sa_readonly`+

- Lê `platform.audit_log` (DATA_MODEL §1): `actor_id`, `actor_type`, `tenant_id`, `action`, `metadata`,
  `created_at`. Filtros por SA, tenant, `action`, período.
- **Visões dedicadas:** impersonações (§9), suspensões/cancelamentos (§7), rotações de chave (§12),
  alterações de plano/quota (§10).
- **Imutável/append-only** (recomendado): sem edição/exclusão pelo console. Exportável para revisão/compliance.

### 14.2 Gestão de super-admins — `/super-admins` · MVP · `sa_owner`

- CRUD de `super_admins` (`email`, `name`, `role`, `mfa_enabled`). Convite + obrigatoriedade de MFA no
  primeiro login. `audit: super_admin.created/updated/disabled`, `super_admin.role_changed`.
- **Princípio do menor privilégio:** atribuir o papel mínimo (§1.2). Revogar acesso desativa sessões.

### 14.3 Segurança transversal (NFR)

- MFA obrigatório; sessões do console **separadas** das de impersonação; step-up para ações destrutivas;
  IP/log de acesso; rate-limit no login. Alinhar a NON_FUNCTIONAL_REQUIREMENTS (segurança/LGPD).

---

## 15. Modelo de auditoria — vocabulário de `audit_log.action`

> Proposta de vocabulário canônico (`platform.audit_log.action`) para o console. `actor_type='super_admin'`.
> Schemas Zod em `packages/contracts` (DRY). Finalizar com a coordenação (dep. #2).

| Domínio | `action` | Papel mín. | Step-up |
|---------|----------|------------|---------|
| Tenant | `tenant.created` | `sa_ops` | — |
| Tenant | `tenant.provisioning.retried` / `.smoke_retested` / `.aborted` | `sa_ops` | — |
| Tenant | `tenant.suspended` / `tenant.reactivated` | `sa_ops` | sim (suspend) |
| Tenant | `tenant.cancelled` / `tenant.purged` | `sa_owner` | sim |
| Quota | `quota.override.granted` | `sa_ops` | — |
| Impersonação | `tenant.impersonation.started` / `.action` / `.ended` | `sa_support` | sim (início) |
| Billing SaaS | `saas.subscription.plan_changed` / `saas.invoice.adjusted` / `saas.billing.credit_granted` / `saas.takerate.override` / `saas.trial.adjusted` | `sa_billing` | — |
| Plano | `plan.updated` / `plan.takerate.changed` / `plan.feature_override.set` | `sa_owner` | sim |
| Integração | `integration.key.rotated` / `integration.key.revealed` / `integration.reprovisioned` | `sa_ops` / `sa_owner` | sim (reveal) |
| Suporte | `support.note.added` / `support.action.performed` | `sa_support` | — |
| Equipe SA | `super_admin.created` / `.updated` / `.disabled` / `.role_changed` | `sa_owner` | — |

---

## Dependências e pontos para o coordenador

1. **Host do console (`admin.app.com`):** confirmado como **proposta a validar com engenharia** (DNS,
   cookies/sessão, isolamento do middleware) — [IA #1](INFORMATION_ARCHITECTURE.md) /
   [OPEN_QUESTIONS #23](../OPEN_QUESTIONS.md). Não bloqueia o MVP do modelo de telas.

2. **Vocabulário de `audit_log.action`:** §15 é proposta. Precisa virar contrato canônico (Zod em
   `packages/contracts`) junto com os papéis de SA (§1.2). Cruza com RBAC e ANALYTICS (`super_admin_*`).

3. **Feature flags por tenant (override):** o modelo atual entitla por **plano** (`features[]`). Overrides
   por tenant (piloto/exceção) exigem nova tabela/coluna no control plane. Definir se entra no MVP ou F2.

4. **Impersonação — escopo e transparência (RBAC dep. #5):** confirmar (a) **read-only vs full** por papel
   de SA; (b) **TTL** da sessão; (c) **notificar owner/admin** do tenant sobre cada impersonação
   (transparência/LGPD recomendada). Confirmar exigência de MFA/step-up (já proposto obrigatório).

5. **Política de acesso dos alunos quando o tenant é suspenso (MONETIZATION dep. #2):** decisão sensível e
   pendente — alunos pagantes mantêm acesso (default proposto) ou perdem? Afeta a UX de §7 e o copy das
   páginas explicativas.

6. **Cancelar/purgar (RBAC C1) e janela de retenção:** confirmar **prazos de retenção LGPD** antes do
   `DROP SCHEMA` (dep. jurídica, README #24) e o fluxo "owner solicita → SA executa".

7. **Valores comerciais de plano/quota/take rate (OPEN_QUESTIONS #10/#11):** o editor de planos (§10) está
   pronto estruturalmente; faltam os **números** (incl. quotas de live F2) e a definição de **fee
   transacional sobre GMV** além da mensalidade. Sem isso, o enforcement de quota/take rate não fecha.

8. **Reconciliação Stripe ↔ control plane:** o billing SaaS (§8) assume webhooks idempotentes do Stripe
   como fonte de verdade. Confirmar com o agente de Pagamentos o mapeamento `platform_subscriptions.status`
   ↔ derivação de `past_due/grace` e as ações manuais de exceção (crédito/forçar cobrança).

9. **Serviço de quotas como dependência:** §4.2/§5.2/§10 dependem do **serviço de quotas central**
   (DATA_MODEL §6.12) já previsto. Confirmar onde mora (`packages/` vs módulo `identity`) e a frequência de
   leitura de uso (Bunny p/ storage/banda pode ter custo/latência de API).
