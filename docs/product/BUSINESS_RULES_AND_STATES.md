# Regras de Negócio e Máquinas de Estado

- **Versão:** 1.0 · **Data:** 2026-06-04
- **Status:** Proposto para revisão do coordenador
- **Relacionados:** [PRD.md](../PRD.md) · [DATA_MODEL.md](../DATA_MODEL.md) · [ARCHITECTURE.md](../ARCHITECTURE.md) · [ROADMAP.md](../ROADMAP.md) · [RBAC_MATRIX.md](RBAC_MATRIX.md) · [NOTIFICATIONS_MATRIX.md](NOTIFICATIONS_MATRIX.md)

> Este documento é a fonte de verdade das **regras de negócio**, **máquinas de estado** e **edge cases**.
> Expande as 7 regras centrais do [PRD §4](../PRD.md). Todo estado citado mapeia para colunas `status`
> do [DATA_MODEL.md](../DATA_MODEL.md). Os estados/transições aqui descritos devem ser implementados em
> entidades de domínio (`*.entity.ts`) e validados por use-cases (`*.usecase.ts`), nunca em rotas.

---

## Índice

- [0. Convenções e princípios transversais](#0-convenções-e-princípios-transversais)
- [1. Máquina de estados — Tenant](#1-máquina-de-estados--tenant-control-plane)
- [2. Máquina de estados — Curso e Aula](#2-máquina-de-estados--curso-e-aula)
- [3. Máquina de estados — Vídeo (Bunny)](#3-máquina-de-estados--vídeo-bunny)
- [4. Máquina de estados — Matrícula / Acesso](#4-máquina-de-estados--matrícula--acesso)
- [5. Máquina de estados — Pedido / Pagamento](#5-máquina-de-estados--pedido--pagamento)
- [6. Máquina de estados — Assinatura](#6-máquina-de-estados--assinatura)
- [7. Máquina de estados — Certificado](#7-máquina-de-estados--certificado)
- [8. Máquina de estados — Comissão de afiliado](#8-máquina-de-estados--comissão-de-afiliado)
- [9. Mapa de impacto entre máquinas (orquestração de eventos)](#9-mapa-de-impacto-entre-máquinas-orquestração-de-eventos)
- [10. Edge cases e exceções por área](#10-edge-cases-e-exceções-por-área)
- [11. Regras detalhadas expandindo as 7 regras centrais do PRD](#11-regras-detalhadas-expandindo-as-7-regras-centrais-do-prd)
- [Dependências e pontos para o coordenador](#dependências-e-pontos-para-o-coordenador)

---

## 0. Convenções e princípios transversais

- **Idempotência:** toda transição disparada por evento externo (webhook de pagamento/Bunny) é
  idempotente. Chave de deduplicação: `payment_events.event_id` (unique) e
  `platform.provisioning_jobs.idempotency_key` (unique).
- **Transições válidas apenas:** transições não listadas são **proibidas** e devem lançar
  `DomainError` (`InvalidStateTransitionError`). O domínio é a autoridade da máquina de estado.
- **Escopo de tenant:** toda transição de dados de tenant ocorre dentro de `withTenant(tenantId, fn)`.
- **Auditoria:** transições sensíveis (suspensão de tenant, revogação de certificado, reversão de
  comissão, reembolso, impersonação) gravam em `platform.audit_log`.
- **Monetário:** sempre centavos (`integer`) + `currency`.
- **Notação dos diagramas:** `estadoA --(gatilho / [guarda])--> estadoB`. `[guarda]` = pré-condição.

---

## 1. Máquina de estados — Tenant (control plane)

`platform.tenants.status`: `provisioning | active | suspended | cancelled`

### 1.1 Diagrama

```
        (assinatura SaaS criada / saga onboarding inicia)
   ──────────────────────────────────► provisioning
                                            │
        (saga completa + smoke test OK)     │
                                            ▼
                                          active ◄──────────────────────┐
                                            │  ▲                          │
       (inadimplência SaaS / abuso /        │  │ (pagamento SaaS          │
        suspensão manual super-admin)       │  │  regularizado /          │
                                            ▼  │  reativação manual)      │
                                        suspended ───────────────────────┘
                                            │
        (cancelamento solicitado / fim do período / churn)
                                            ▼
                                        cancelled
                                            │
        (retenção legal expirada → DROP SCHEMA + purga)
                                            ▼
                                       [purgado]
```

### 1.2 Transições, gatilhos e guardas

| De | Para | Gatilho | Guarda / regra |
|----|------|---------|----------------|
| — | `provisioning` | Assinatura SaaS criada (Stripe Billing) | `idempotency_key` único; cria `provisioning_jobs` |
| `provisioning` | `active` | Saga concluída (schema+migrations+seed+Bunny library+roteamento) + smoke test verde | Todos os steps `done`; rollback se falhar |
| `provisioning` | `cancelled` | Saga falha irrecuperável após N retries | Limpa recursos parciais (DROP SCHEMA se criado) |
| `active` | `suspended` | Inadimplência do SaaS (invoice `overdue` após grace), abuso/ToS, suspensão manual super-admin | Grava `audit_log` |
| `suspended` | `active` | Pagamento SaaS regularizado (webhook Stripe) ou reativação manual | Re-habilita roteamento/login |
| `active`/`suspended` | `cancelled` | Cancelamento da assinatura SaaS / fim de período / pedido do dono | Bloqueia novo acesso; inicia retenção |
| `cancelled` | `provisioning` | Reativação dentro da janela de retenção (schema ainda existe) | Apenas se schema não purgado |
| `cancelled` | `[purgado]` | Fim da retenção legal | `DROP SCHEMA tenant_<slug> CASCADE`; anonimiza control plane |

### 1.3 Efeitos por estado

- `provisioning`: login bloqueado; nenhuma rota de tenant servida.
- `active`: operação normal.
- `suspended`: **login e checkout bloqueados** (HTTP 402/403 via Problem Details); páginas de
  vendas e player desativados; dados **preservados**; super-admin ainda acessa para suporte. Jobs
  agendados não-críticos pausados; reconciliação de pagamentos continua para histórico.
- `cancelled`: somente leitura para exportação de dados (LGPD/lock-in) durante a janela de retenção.

---

## 2. Máquina de estados — Curso e Aula

`courses.status` e `lessons.status`: `draft | published | archived` (aula no MVP: `draft | published`;
`archived` aplicável a curso e reservado a aula na F2).

### 2.1 Diagrama (Curso)

```
   draft ──(publicar / [≥1 módulo com ≥1 aula publicável; preço definido se pago])──► published
     ▲                                                                                    │
     │ (despublicar — volta a rascunho)                                                   │
     └────────────────────────────────────────────────────────────────────────────────┐ │
                                                                                        │ │
   published ──(arquivar / [sem novas matrículas; mantém acesso de já matriculados])──► archived
                                                                                          │
                                            (desarquivar / republicar)                    │
   published ◄──────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Transições, gatilhos e guardas

| Entidade | De | Para | Gatilho | Guarda / regra |
|----------|----|------|---------|----------------|
| Curso | `draft` | `published` | Instrutor/Admin publica | ≥1 módulo com ≥1 aula publicável; se `pricing_type≠free` exige `price_cents>0`; landing válida |
| Curso | `published` | `draft` | Despublicar | Remove da vitrine; **mantém** acesso de alunos já matriculados (entitlement preservado) |
| Curso | `published` | `archived` | Arquivar | Bloqueia **novas** matrículas/compras; alunos existentes mantêm acesso conforme entitlement |
| Curso | `archived` | `published` | Desarquivar | Volta à vitrine |
| Aula | `draft` | `published` | Publicar aula | Se `type=video`, exige vídeo em estado `ready` (ver §3); conteúdo válido por tipo |
| Aula | `published` | `draft` | Despublicar | Aula some do player; progresso preservado |

### 2.3 Regras

- **Publicação independe de drip:** publicar a aula define que ela *existe*; `drip_release_at` /
  `drip_days_after_enroll` controlam **quando** o aluno a vê (ver §10.5).
- Editar curso/aula em `draft` **nunca** expõe a alunos (rascunho/publicação do PRD §3.1).
- Despublicar um curso não cancela matrículas nem pedidos pagos; é decisão de catálogo, não de acesso.

---

## 3. Máquina de estados — Vídeo (Bunny)

Estado do vídeo derivado do Bunny e refletido em `lessons` (recomendado campo `video_status`; ver
[Dependências](#dependências-e-pontos-para-o-coordenador)). Estados:
`queued | processing | ready | failed`.

### 3.1 Diagrama

```
   (upload TUS concluído)──► queued ──(encoding inicia)──► processing
                                                              │
                       ┌──────────────────────(webhook ready / HMAC OK)──────► ready
                       │                                                          │
        (encoding falha / formato inválido / timeout)                            │ (re-encode solicitado)
                       ▼                                                          ▼
                    failed ◄──────────────(novo upload / retry)──────────── processing
```

### 3.2 Transições, gatilhos e guardas

| De | Para | Gatilho | Guarda / regra |
|----|------|---------|----------------|
| — | `queued` | Upload TUS concluído | Vídeo criado na library do tenant; `video_guid` salvo |
| `queued` | `processing` | Bunny inicia encoding | — |
| `processing` | `ready` | **Webhook "vídeo pronto"** | HMAC válido (bytes brutos, comparação em tempo constante); `event_id` não processado |
| `processing`/`queued` | `failed` | Falha de encoding / formato inválido / timeout | Notifica instrutor; aula não pode publicar |
| `failed` | `processing` | Novo upload ou retry de encode | Reusa ou recria `video_guid` |
| `ready` | `processing` | Re-encode / substituição de mídia | Aula publicada continua servindo a versão anterior até novo `ready` |

### 3.3 Regras

- **Aula `video` só publica com vídeo `ready`** (guarda em §2.2). Se publicar enquanto `processing`,
  bloquear com `DomainError`.
- Webhook do Bunny é **idempotente** (dedupe por `event_id`/payload) e mapeia `VideoLibraryId → tenant`
  no worker.
- Geração de URL/Embed Token de reprodução exige vídeo `ready` **e** entitlement válido (regra PRD §4.2).

---

## 4. Máquina de estados — Matrícula / Acesso

`enrollments.status`: `active | suspended | refunded | expired`. **Pagamento governa acesso** (PRD §4.3).

### 4.1 Diagrama

```
   (compra aprovada / matrícula manual / bulk / auto-enroll)
   ───────────────────────────────────────────────► active
                                                       │  ▲
        (chargeback / inadimplência de assinatura /     │  │ (pagamento regularizado /
         suspensão manual)                              │  │  reconciliação confirma pago)
                                                        ▼  │
                                                    suspended ──┐
                                                       │        │
        (reembolso confirmado)                         │        │ (reembolso confirmado)
                                                        ▼        ▼
                                                     refunded ◄──┘
                                                       
   active ──(expires_at atingido / fim de assinatura sem renovação)──► expired
   expired ──(renovação/recompra aprovada)──► active
```

### 4.2 Transições, gatilhos e guardas

| De | Para | Gatilho | Guarda / regra |
|----|------|---------|----------------|
| — | `active` | Pedido `paid`, matrícula manual/bulk/auto | Cria `enrollment` (unique `user_id,course_id`); define `expires_at` se aplicável |
| `active` | `suspended` | Chargeback aberto, assinatura `past_due`, suspensão manual | Acesso bloqueado; entitlement inválido para URL de vídeo |
| `suspended` | `active` | Pagamento regularizado / reconciliação confirma `paid` | Restaura acesso |
| `active`/`suspended` | `refunded` | Reembolso confirmado pelo gateway | Acesso revogado de forma definitiva para o ciclo; pode revogar certificado (ver §10.2) |
| `active` | `expired` | `expires_at` atingido ou fim de assinatura sem renovação | Acesso encerra; histórico/progresso preservados |
| `expired` | `active` | Renovação/recompra aprovada | Reabre acesso; mantém progresso |

### 4.3 Regras

- **Acesso = entitlement válido** (PRD §4.2): backend valida `enrollments.status='active'` (e não
  expirado) **antes** de gerar URL assinada de vídeo. Estados `suspended/refunded/expired` ⇒ sem token.
- Matrícula manual/bulk/auto pode ser `active` sem pedido (cortesia/B2B); reembolso não se aplica.
- `refunded` é terminal para o ciclo de pagamento daquele pedido; nova compra cria novo ciclo.

---

## 5. Máquina de estados — Pedido / Pagamento

`orders.status`: `pending | paid | refunded | chargeback`. Provider: `pagarme | asaas`.

### 5.1 Diagrama

```
   (checkout iniciado)──► pending ──(webhook aprovado)──► paid
                            │                               │
        (Pix/boleto expira / cartão recusado /              │
         cancelado / timeout)                               │
                            ▼                               │
                        [expirado/falho]                    │
                                                            │ (reembolso solicitado e confirmado)
                                                            ▼
                                                         refunded
                            paid ──(disputa/chargeback no cartão)──► chargeback
                            chargeback ──(disputa ganha / reconciliação)──► paid
```

### 5.2 Transições, gatilhos e guardas

| De | Para | Gatilho | Guarda / regra |
|----|------|---------|----------------|
| — | `pending` | Checkout iniciado | Cria `order`; aplica cupom/afiliado; gera Pix/boleto/intenção de cartão |
| `pending` | `paid` | Webhook de aprovação (idempotente) | `event_id` único; valida assinatura; dispara matrícula `active` + comissão `pending` |
| `pending` | `[expirado/falho]` | Pix/boleto expira, cartão recusado, cancelamento, timeout | Não cria acesso; `pending` pode ser limpo por job ou marcado expirado (ver §10.1) |
| `paid` | `refunded` | Reembolso confirmado | Suspende/refunda matrícula; reverte comissão (`reversed`) |
| `paid` | `chargeback` | Disputa de cartão aberta | Suspende matrícula; reverte comissão; grava `audit_log` |
| `chargeback` | `paid` | Disputa ganha / reconciliação corrige | Reativa matrícula; comissão pode voltar a `pending` |

### 5.3 Regras

- Toda mudança vem de **webhook verificado** ou **reconciliação periódica**; nunca de input do cliente.
- `paid → refunded` e `paid → chargeback` propagam para Matrícula (§4) e Comissão (§8).
- Pedido só gera **um** entitlement por (user, course); recompra após refund/expired reaproveita o
  registro de matrícula reativando-o.

---

## 6. Máquina de estados — Assinatura

Dois escopos distintos (não confundir):

- **Assinatura do aluno ao conteúdo do tenant** — `subscriptions` (data plane).
- **Assinatura do tenant ao SaaS** — `platform.subscriptions` (control plane, Stripe Billing) → governa
  a máquina do Tenant (§1).

Estados canônicos (alinhados ao vocabulário de gateways): `trialing | active | past_due | canceled |
expired`. **Resolvido pela coordenação:** padronizado no [DATA_MODEL §6.0](../DATA_MODEL.md) (vocabulário
canônico de `status`) e em [ADR-0013](../adr/0013-product-data-model-extensions.md).

### 6.1 Diagrama

```
   (assinatura criada, com trial?)
        │
        ├─ trial ──► trialing ──(fim do trial + cobrança OK)──► active
        └─ sem trial ─────────────────────────────────────────► active
                                                                  │  ▲
                            (cobrança recorrente falha)            │  │ (cobrança recuperada / dunning OK)
                                                                   ▼  │
                                                              past_due ┘
                                                                   │
                            (cancelamento / falha definitiva de dunning)
                                                                   ▼
                                                               canceled
                                  (fim do período pago após cancelamento)
                                                                   ▼
                                                                expired
```

### 6.2 Transições e efeitos sobre acesso

| De | Para | Gatilho | Efeito no acesso (aluno) |
|----|------|---------|--------------------------|
| — | `trialing` | Assinatura com trial criada | Matrícula `active` durante trial |
| `trialing`/`active` | `active` | Cobrança aprovada / renovação | Matrícula `active` |
| `active` | `past_due` | Cobrança recorrente falha | Inicia **dunning**; acesso mantido durante grace; ao fim do grace ⇒ matrícula `suspended` |
| `past_due` | `active` | Cobrança recuperada | Matrícula volta a `active` |
| `past_due`/`active` | `canceled` | Cancelamento ou dunning esgotado | Sem renovação; acesso até fim do período pago |
| `canceled` | `expired` | Fim do período pago | Matrícula `expired` |

### 6.3 Regras

- A assinatura do **aluno** dirige a máquina de Matrícula (§4); a do **tenant** dirige a do Tenant (§1).
- Grace period e número de tentativas de dunning são **parâmetros de configuração** (ver Dependências).

---

## 7. Máquina de estados — Certificado

`certificates.revoked` (bool) ⇒ estados lógicos `issued | revoked`.

### 7.1 Diagrama

```
   (curso 100% concluído + nota mínima atingida quando houver)
   ──────────────────────────────────────────────► issued
                                                      │  ▲
        (reembolso/chargeback do pedido,              │  │ (reemissão / reversão de fraude;
         fraude detectada, revogação manual)          │  │  geralmente NÃO reverte — ver regra)
                                                       ▼  │
                                                    revoked
```

### 7.2 Transições, gatilhos e guardas

| De | Para | Gatilho | Guarda / regra |
|----|------|---------|----------------|
| — | `issued` | Conclusão 100% + nota mínima (se houver quiz/prova com `pass_score`) | Gera `uuid_public`, `hash`, PDF no R2; verificação pública (QR) |
| `issued` | `revoked` | Reembolso/chargeback, fraude, revogação manual admin | Página pública passa a indicar **revogado**; grava `audit_log` |
| `revoked` | `issued` | Reemissão excepcional (correção de erro) | Decisão administrativa; novo `issued_at`/hash; **não** é padrão |

### 7.3 Regras

- **Certificado só emite com curso 100% concluído** (PRD §4.5) e conclusão de aula obedece anti-seek
  (PRD §4.4 — ver §11).
- A verificação pública sempre reflete o estado atual (`issued`/`revoked`) via `uuid_public`+`hash`.
- Reembolso de pedido **pode** revogar certificado (política configurável — ver Dependências).

---

## 8. Máquina de estados — Comissão de afiliado

`affiliate_commissions.status`: `pending | paid | reversed`.

### 8.1 Diagrama

```
   (pedido com affiliate_id aprovado → paid)
   ───────────────────────────────────────► pending
                                               │  ▲
        (reembolso/chargeback do pedido         │  │ (raro: disputa ganha reverte a reversão →
         dentro da janela de garantia)          │  │  volta a pending)
                                                ▼  │
                                            reversed┘
                                               
   pending ──(fim da janela de garantia/clearance + payout executado)──► paid
```

### 8.2 Transições, gatilhos e guardas

| De | Para | Gatilho | Guarda / regra |
|----|------|---------|----------------|
| — | `pending` | Pedido com `affiliate_id` vira `paid` | Calcula `amount_cents` = base × `commission_pct`; respeita regras de `splits` |
| `pending` | `paid` | Janela de garantia/clearance vencida + payout | Só após período antifraude/estorno; via split (Pagar.me) ou repasse |
| `pending`/`paid` | `reversed` | Reembolso/chargeback do pedido associado | Estorna comissão; se já `paid`, gera ajuste/débito futuro; `audit_log` |
| `reversed` | `pending` | Disputa de chargeback ganha (reconciliação) | Excepcional |

### 8.3 Regras

- Comissão **nunca** é `paid` antes do fim da janela de garantia (proteção contra reembolso/chargeback).
- Reembolso/chargeback do pedido (§5) sempre propaga `reversed`.
- Split em tempo de transação (Pagar.me) e comissão em base de cálculo devem ser conciliados na
  reconciliação periódica.

---

## 9. Mapa de impacto entre máquinas (orquestração de eventos)

| Evento de origem | Pedido | Matrícula | Comissão | Certificado | Tenant |
|------------------|--------|-----------|----------|-------------|--------|
| Webhook pagamento aprovado | →`paid` | →`active` | →`pending` | — | — |
| Reembolso confirmado | →`refunded` | →`refunded` | →`reversed` | →`revoked` (se política) | — |
| Chargeback aberto | →`chargeback` | →`suspended` | →`reversed` | →`revoked` (se política) | — |
| Disputa ganha | →`paid` | →`active` | →`pending` | reemissão manual | — |
| Assinatura aluno `past_due` (grace fim) | — | →`suspended` | — | — | — |
| `expires_at` atingido | — | →`expired` | — | — | — |
| Inadimplência SaaS | — | (acesso bloqueado via tenant) | — | — | →`suspended` |
| Curso 100% concluído + nota | — | — | — | →`issued` | — |
| Vídeo `ready` (webhook Bunny) | — | — | — | — | — (libera publicação da aula) |

> Implementação: um event/orchestration layer (ports + use-cases) consome webhooks idempotentes e
> aplica as transições. Jobs via `JobQueue` carregam `tenantId`.

---

## 10. Edge cases e exceções por área

### 10.1 Pagamento recusado / pendente / duplicado / timeout

- **Cartão recusado:** pedido permanece `pending`/falho; sem acesso; oferecer retry. Notificar aluno
  (pagamento recusado).
- **Pix/boleto pendente:** acesso **não** liberado até `paid`. Pix expira em minutos; boleto em dias.
  Job marca `pending` expirados (limpeza) sem criar entitlement.
- **Pagamento duplicado (mesmo intent):** dedupe por `payment_events.event_id` e por
  `provider_order_id`. Segunda aprovação não cria segunda matrícula nem segunda comissão.
- **Webhook fora de ordem:** se `paid` chega antes de `pending` registrado, criar/reconciliar pedido
  pelo `provider_order_id`. Reconciliação periódica corrige divergências.
- **Timeout do gateway no checkout:** não confiar em resposta síncrona; estado só muda por webhook ou
  reconciliação (single source of truth = gateway).

### 10.2 Reembolso / chargeback

- Reembolso parcial: política a definir (ver Dependências); por padrão trata como reembolso total para
  acesso (suspende). Chargeback sempre suspende imediatamente e abre `audit_log`.
- Propaga para Matrícula (`refunded`/`suspended`), Comissão (`reversed`) e, se política, Certificado
  (`revoked`).
- Reembolso após certificado emitido: revogação configurável; verificação pública passa a `revoked`.

### 10.3 Acesso expirado

- Aluno com `enrollment.expired` perde geração de token de vídeo; player retorna 403 com CTA de
  renovação/recompra. Progresso e certificados emitidos **permanecem** no histórico.

### 10.4 Aula bloqueada por drip / pré-requisito

- **Drip por data fixa:** aula visível só após `drip_release_at`.
- **Drip por dias após matrícula:** visível após `enrolled_at + drip_days_after_enroll`.
- **Pré-requisito / bloqueio sequencial (F2):** aula libera só após conclusão da anterior (PRD §3.1).
- Aula bloqueada por drip aparece "agendada" (countdown), não como erro. Backend recusa token de vídeo
  enquanto bloqueada (defense in depth).

### 10.5 Vídeo ainda processando

- Aula `video` com vídeo em `queued/processing` **não publica**. Se já publicada e mídia trocada,
  serve versão anterior até novo `ready`. Aluno vê estado "processando" se não houver versão pronta.
- Falha de encoding (`failed`): notifica instrutor; bloqueia publicação até novo upload/retry.

### 10.6 Cupom inválido / esgotado / expirado

- Validação no checkout: `valid_until` vigente, `uses < max_uses`, código existente. Inválido ⇒
  `DomainError` (cupom inválido) sem aplicar desconto.
- **Corrida em `max_uses`:** incremento de `uses` deve ser atômico (update condicional
  `uses < max_uses`) dentro da transação do pedido, evitando estouro do limite em concorrência.
- Cupom não acumula com regras conflitantes (política de stacking a definir — ver Dependências).

### 10.7 Conflito de e-mail no tenant

- `users.email` é unique **por schema** (PRD §6 auth, DATA_MODEL §2.1). Mesmo e-mail pode existir em
  tenants diferentes (modelo workspace). Compra com e-mail já existente no tenant ⇒ vincula ao usuário
  existente; nunca duplica.
- Convite/auto-enroll para e-mail já cadastrado ⇒ idempotente (não recria usuário).

### 10.8 Tenant suspenso por inadimplência do SaaS

- Tenant `suspended` (§1): login/checkout/player bloqueados; dados preservados; super-admin acessa
  para suporte; alunos veem aviso de indisponibilidade. Reativação ao regularizar (webhook Stripe).
- Distinguir de **matrícula suspensa** (escopo aluno): suspensão de tenant afeta **todos**.

### 10.9 Idempotência de webhooks

- Entrada (Bunny/pagamento): verificar assinatura/HMAC (bytes brutos, comparação tempo constante),
  responder 200, **enfileirar** processamento. Dedupe por `event_id` unique. Reprocessar é seguro
  (no-op se já processado).
- Saída (compra/reembolso/matrícula/conclusão): entregar com retry + backoff; assinar payload;
  expor `event_id` para o consumidor deduplicar.

### 10.10 Reconciliação

- Job agendado compara estado local × gateway (pagamentos) e × Bunny (status de vídeo) e corrige
  divergências (ex.: webhook perdido). Idempotente; gera `audit_log` em correções relevantes.
- Reconciliação de comissões/splits confere base de cálculo e payouts.

### 10.11 Outros edge cases

- **Reordenação concorrente** de módulos/aulas (`position`): resolver por última escrita + reindex.
- **Conclusão de aula com seek/aceleração:** rejeitada se tempo real assistido < duração × 0.8 (§11).
- **Provisionamento parcial:** saga retoma do último step `done` (idempotência); rollback se falha
  irrecuperável (DROP SCHEMA se já criado).
- **Quota estourada** (alunos/storage por plano): bloquear ação que excede com erro de quota; não
  corromper dados existentes.
- **Revogação de afiliado** com comissões `pending`: comissões seguem seu ciclo; novos cliques não
  geram comissão.

---

## 11. Regras detalhadas expandindo as 7 regras centrais do PRD

1. **Isolamento absoluto entre tenants (PRD §4.1).** Todo acesso via `withTenant` +
   `SET LOCAL search_path`. Nenhuma FK cruza schemas. Teste de isolamento cross-tenant é gate de CI.
   `tenant_id` vem da sessão/JWT, nunca de header não autenticado.
2. **Acesso = entitlement válido (PRD §4.2).** URL/Embed Token de vídeo só é gerado se
   `enrollment.status='active'` (não expirado), curso acessível e vídeo `ready`. TTL curto (1–12h).
   Drip/pré-requisito também são guardas para emissão de token.
3. **Pagamento governa acesso (PRD §4.3).** Webhooks idempotentes + reconciliação dirigem as máquinas
   de Pedido→Matrícula→Comissão→Certificado conforme §9. Cliente nunca altera estado diretamente.
4. **Conclusão de aula (PRD §4.4).** `watched_pct ≥ 90%` **e** anti-seek: soma de tempo real assistido
   (heartbeats) ≥ `duration_seconds × 0.8`. Heartbeats throttle 5–15s. Só então
   `lesson_progress.status='completed'`.
5. **Certificado (PRD §4.5).** Emite só com curso 100% concluído (todas as aulas requeridas
   `completed`) e, havendo quiz/prova com `pass_score`, nota mínima atingida. Caso contrário, bloqueia.
6. **Provisionamento idempotente (PRD §4.6).** Saga transacional onde possível; estado em
   `provisioning_jobs` com `idempotency_key`; retoma do último step; smoke test pós-provisionamento.
7. **Super-Admin com auditoria (PRD §4.7).** Opera no control plane; impersonação **sempre** gera
   `audit_log` (ator, tenant, ação, timestamp). Nenhum acesso implícito a dados de tenant sem registro.

---

## Dependências e pontos para o coordenador

1. **Campo de status de vídeo na aula:** o DATA_MODEL não tem coluna explícita de status de vídeo em
   `lessons` (só `video_guid`/`duration_seconds`). Recomendo adicionar `video_status`
   (`queued|processing|ready|failed`) — decisão de modelagem (possível ADR/migration).
2. **Vocabulário de status de assinatura:** `subscriptions.status` e `platform.subscriptions.status`
   estão como `text` genérico. Proponho padronizar `trialing|active|past_due|canceled|expired`
   (alinhar com o coordenador/ADR de pagamentos).
3. **Política de reembolso → certificado:** revogar certificado em reembolso/chargeback é
   configurável. Definir default por tenant ou global.
4. **Reembolso parcial:** comportamento de acesso (suspende total vs proporcional) precisa de decisão
   de produto.
5. **Parâmetros de dunning/grace:** número de tentativas e janela de grace para assinatura `past_due`
   antes de suspender matrícula — definir e onde configurar (plano? global?).
6. **Janela de garantia/clearance de comissão de afiliado:** período antes de `pending→paid` (alinhado
   à janela de reembolso/chargeback). Definir prazo.
7. **Stacking de cupons** e interação cupom × order bump/upsell (F2): regras de acumulação.
8. **`enrollments` sem coluna de motivo:** sugiro registrar motivo de suspensão/expiração (auditoria
   no data plane) — possível campo `status_reason` ou evento de domínio.
9. **Reativação de tenant `cancelled`:** confirmar janela de retenção antes do `DROP SCHEMA`
   (LGPD/contratual).
