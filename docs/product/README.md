# Especificação de Produto — Índice, Terminologia e Relatório de Consistência

- **Versão:** 1.0 · **Data:** 2026-06-04
- **Mantido por:** Coordenação de Produto (Head of Product + Tech Lead)
- **Status:** Reconciliado — pronto para implementação do MVP (com decisões de stakeholder rastreadas)
- **Relacionados (base):** [PRD.md](../PRD.md) · [ARCHITECTURE.md](../ARCHITECTURE.md) · [DATA_MODEL.md](../DATA_MODEL.md) · [ROADMAP.md](../ROADMAP.md) · [OPEN_QUESTIONS.md](../OPEN_QUESTIONS.md) · [ADRs](../adr/) · [CLAUDE.md](../../CLAUDE.md)

Esta pasta contém o **detalhamento de produto** elaborado por 8 especialistas e **reconciliado** pela
coordenação. O PRD/ARCHITECTURE/DATA_MODEL/ADRs continuam sendo a **fonte de verdade** de escopo e técnica;
estes documentos os **expandem** sem contradizê-los. Conflitos e lacunas cruzadas foram resolvidos e estão
registrados no **Relatório de Consistência** (§4) e, quando exigiam o stakeholder, em
[OPEN_QUESTIONS §2-bis](../OPEN_QUESTIONS.md).

Filtro de qualidade de toda decisão: **SOLID/DRY/Clean Code** e a **Regra nº1 — isolamento por tenant**.

---

## 1. Índice dos documentos

| # | Documento | Conteúdo | Status |
|---|-----------|----------|--------|
| 1 | [USER_JOURNEYS.md](USER_JOURNEYS.md) | Jornadas das 5 personas + Super-Admin (B2B e B2C), JTBD, fases, emoções. | Reconciliado |
| 2 | [USER_FLOWS.md](USER_FLOWS.md) | Fluxos passo a passo (cadastro, compra→acesso, upload, certificado, impersonação...). | Reconciliado |
| 3 | [INFORMATION_ARCHITECTURE.md](INFORMATION_ARCHITECTURE.md) | Mapa de hosts/áreas/rotas, inventário de telas, navegação por papel. | Reconciliado (hosts = proposta) |
| 4 | [LEARNING_EXPERIENCE_UX.md](LEARNING_EXPERIENCE_UX.md) | UX da sala de aula: player, progresso, quiz, conclusão, comunidade, gamificação. | Reconciliado |
| 5 | [USER_STORIES.md](USER_STORIES.md) | Épicos e user stories (Gherkin) com critérios de aceite e prioridade MoSCoW. | Reconciliado |
| 6 | [BUSINESS_RULES_AND_STATES.md](BUSINESS_RULES_AND_STATES.md) | Regras de negócio + máquinas de estado (tenant, curso, vídeo, matrícula, pedido, assinatura, certificado, comissão). | Reconciliado |
| 7 | [RBAC_MATRIX.md](RBAC_MATRIX.md) | Matriz papel × ação por domínio (SA/OW/AD/IN/AF/ST) + condições. | Reconciliado |
| 8 | [NOTIFICATIONS_MATRIX.md](NOTIFICATIONS_MATRIX.md) | Eventos × canais (in-app/e-mail/push) × prioridade × idempotência. | Reconciliado |
| 9 | [MONETIZATION.md](MONETIZATION.md) | Pricing do SaaS (Stripe), checkout do aluno (Pagar.me/Asaas), cupons, afiliados, split, take rate. | Reconciliado |
| 10 | [ANALYTICS_AND_DASHBOARDS.md](ANALYTICS_AND_DASHBOARDS.md) | Tracking plan (PostHog), catálogo de eventos/KPIs, dashboards por papel. | Reconciliado |
| 11 | [NON_FUNCTIONAL_REQUIREMENTS.md](NON_FUNCTIONAL_REQUIREMENTS.md) | NFRs: performance, segurança, LGPD, a11y (WCAG AA), i18n, confiabilidade. | Reconciliado |
| 12 | [AUTHORING_UX.md](AUTHORING_UX.md) | UX do Estúdio (autoria): cursos/aulas, editor por tipo, drip, quiz, publicação e **toggles de autoria**. | Reconciliado |

### Ordem de leitura recomendada
1. **Visão** → PRD → USER_JOURNEYS → USER_FLOWS.
2. **Estrutura** → INFORMATION_ARCHITECTURE → RBAC_MATRIX.
3. **Comportamento** → BUSINESS_RULES_AND_STATES → LEARNING_EXPERIENCE_UX → NOTIFICATIONS_MATRIX.
4. **Negócio/dados** → MONETIZATION → ANALYTICS_AND_DASHBOARDS → [DATA_MODEL §6](../DATA_MODEL.md).
5. **Qualidade/execução** → NON_FUNCTIONAL_REQUIREMENTS → USER_STORIES → ROADMAP.
6. **Decisões** → [ADRs](../adr/) (em especial 0013, 0014) → OPEN_QUESTIONS.

---

## 2. Terminologia canônica (glossário)

> Use **exatamente** estes termos e valores de `status` em código, contratos (Zod) e docs. Padronizado em
> [DATA_MODEL §6.0](../DATA_MODEL.md).

### 2.1 Personas e papéis
| Termo | Sigla | Schema | Definição |
|-------|-------|--------|-----------|
| Super-Admin | SA | `platform` | Operador do SaaS (nós). Global; impersonação **sempre auditada**; sem acesso silencioso a dados de tenant. |
| Owner | OW | `tenant_<slug>` | Dono da escola; superusuário do tenant; único que vê/paga billing do SaaS. |
| Admin | AD | `tenant_<slug>` | Gestor delegado pelo owner; tudo no tenant exceto atos exclusivos do owner. |
| Instrutor | IN | `tenant_<slug>` | Cria/gerencia conteúdo dos seus cursos; engajamento sim, financeiro global não. |
| Afiliado | AF | `tenant_<slug>` | Promove cursos por comissão; vê só seus links/comissões. |
| Aluno (student) | ST | `tenant_<slug>` | Consome cursos; vê suas matrículas/dados. |
| Visitante (guest) | — | público | Não autenticado: landing/catálogo/checkout/verificação de certificado. |

> **MVP:** `users.role` é **único** por usuário no tenant (papéis compostos e delegação configurável são pós-MVP — OPEN_QUESTIONS #20/#21).

### 2.2 Tenant e isolamento
- **Tenant** = escola/produtor; 1 schema `tenant_<slug>`. **Control plane** = schema `platform`.
- **`withTenant(tenantId, fn)`** = único caminho de acesso a dados de tenant (`SET LOCAL search_path`).
- **Regra nº1** = isolamento absoluto entre tenants (inclui analytics).

### 2.3 Estados canônicos (`status`)
| Entidade | Valores |
|----------|---------|
| `tenants` (platform) | `provisioning \| active \| suspended \| cancelled` (`past_due/grace` é **derivado** do Stripe, sem coluna) |
| `platform_subscriptions` / `subscriptions` (aluno) | `trialing \| active \| past_due \| canceled \| expired` |
| `enrollments` | `active \| suspended \| refunded \| expired` |
| `orders` | `pending \| paid \| refunded \| chargeback` |
| `lesson_progress` | `not_started \| in_progress \| completed` |
| `lessons.video_status` | `none \| queued \| processing \| ready \| failed` |
| `affiliates` | `pending \| active \| blocked` |
| `affiliate_commissions` | `pending \| paid \| reversed` |
| `courses` / `lessons` (publicação) | `draft \| published \| archived` (lessons: `draft \| published`) |
| `quiz_attempts` | `in_progress \| submitted \| graded` |

### 2.4 Termos de negócio
- **Take rate** = comissão da plataforma sobre a venda do aluno, em **bps** (`take_rate_bps`), aplicada
  **no split** do checkout (não na fatura Stripe).
- **Split** = divisão da venda na liquidação (Pagar.me) entre produtor, plataforma, afiliado, co-produtor (F2).
- **Entitlement** = direito de acesso = matrícula ativa + pagamento válido (governa URL assinada de vídeo).
- **Anti-seek** = conclusão exige `watched_pct ≥ 90` **e** `real_watched_seconds ≥ duration×0.8`.
- **Drip** = liberação programada (data fixa **ou** dias após matrícula).
- **Last-click** = modelo de atribuição de afiliado (janela padrão 30 dias).
- **Dunning/grace** = janela de retentativa de cobrança antes de suspender (distinto SaaS vs aluno).

---

## 3. Dois domínios de pagamento (esclarecimento mestre)

Conforme PRD §1.3 e ADR-0010 — **nunca confundir**:

| | **Billing do SaaS (B2B)** | **Checkout do aluno (B2C)** |
|---|---|---|
| Quem paga quem | Tenant → plataforma (nós) | Aluno → tenant |
| Gateway | **Stripe Billing** | **Pagar.me** (split) + **Asaas** (Pix/boleto recorrente) |
| Onde vive | Control plane (`platform`) | Schema do tenant |
| Governa | `tenants.status` (ativa/suspende tenant) | `orders`/`enrollments` (acesso do aluno) |
| Tabelas | `platform_subscriptions`, `platform_invoices` | `orders`, `subscriptions`, `coupons`, `affiliate_commissions`, `splits`, `payment_events` |

Consistência verificada em **todos** os docs (USER_JOURNEYS, USER_FLOWS, IA, MONETIZATION, ANALYTICS,
RBAC, USER_STORIES, BUSINESS_RULES) — **sem divergência**.

---

## 4. Relatório de consistência (conflito → decisão → status)

| # | Conflito / lacuna cruzada | Decisão da coordenação | Status |
|---|---------------------------|------------------------|--------|
| 1 | Stripe (SaaS) × Pagar.me/Asaas (aluno) consistente em todos os docs? | Sim; esclarecimento mestre em §3. Take rate via split, não na fatura Stripe. | ✅ Resolvido |
| 2 | Vocabulário de `status` genérico (`text`) | Padronizado em [DATA_MODEL §6.0](../DATA_MODEL.md) + ADR-0013. | ✅ Resolvido |
| 3 | `lessons` sem status de vídeo | `video_status (none\|queued\|processing\|ready\|failed)` (§6.1). | ✅ Resolvido |
| 4 | Comentários sem resolvido/oculto/denúncia/curtidas | Colunas + `comment_likes`/`comment_reports` (§6.3). | ✅ Resolvido |
| 5 | Anti-seek sem campo de tempo real | `lesson_progress.real_watched_seconds` (§6.2). | ✅ Resolvido |
| 6 | Gamificação ausente do modelo | `gamification_xp_ledger` idempotente + agregado + badges (§6.5, F2). | ✅ Resolvido |
| 7 | Quiz sem `attempt_number`/idempotência | `attempt_number`, `status`, `idempotency_key`, políticas (§6.4). | ✅ Resolvido |
| 8 | Order bump/upsell/bundle sem modelagem | `order_items`, `offers`, `bundles`/`bundle_items`, `purchase_session_id` (§6.7, F2). | ✅ Resolvido |
| 9 | Atribuição de afiliado (cookie/janela/aprovação) | `affiliate_program_settings` (30d, last-click, manual), `affiliate_clicks`, override por curso (§6.8). | ✅ Resolvido |
| 10 | Campos localizáveis (i18n) | Colunas `i18n jsonb` em `courses`/`lessons` já no MVP; ativação F2 (§6.1). | ✅ Resolvido |
| 11 | Limite de dispositivos | Quota de plano (`platform_plans.limits`), não coluna por usuário; F2. | ✅ Resolvido |
| 12 | Pricing → `platform_plans.limits` + `take_rate_bps` | Chaves padronizadas (§6.11); valores numéricos = stakeholder (#10). | ✅ Estrutura / ⏳ valores |
| 13 | Serviço de quotas + take rate sem violar isolamento | Resolvido no `onRequest` (injeção de valor); use-case não consulta `platform` (§6.11/6.12, ADR-0013). | ✅ Resolvido |
| 14 | Tracking plan + port de analytics | ADR-0014: port `AnalyticsProvider` + schemas Zod em contracts; heartbeats agregados. | ✅ Resolvido |
| 15 | Preferências de notificação (escopo MVP) | `notification_preferences` + `notifications` no MVP (in-app+e-mail); push F2 (§6.9). | ✅ Resolvido |
| 16 | RBAC: fronteira IN × AD × OW | OW exclusivo (billing/transferência); AD tudo menos atos do owner; IN só seus cursos, sem financeiro global. Delegação fina = pós-MVP. | ✅ Resolvido (detalhe em #21/#22) |
| 17 | Revogação de certificado em reembolso | `tenant_settings.refund_revokes_certificate=true` (configurável). | ✅ Default / ⏳ confirmar |
| 18 | Grace de inadimplência (aluno e SaaS) | Aluno 7d (`tenant_settings`); SaaS dunning 7–14d. | ✅ Default / ⏳ valores |
| 19 | Chargeback × reembolso | Ambos: `enrollment=suspended/refunded` + comissão `reversed`; fluxos de disputa distintos no operacional. MVP sem tratamento divergente além disso. | ✅ Resolvido |
| 20 | Verificação de e-mail | Recomendado obrigatória; gratuito acessível antes (#18 OQ). | ✅ Default / ⏳ confirmar |
| 21 | Escopo do 2FA | Obrigatório SA (MVP) + Owner; recomendado Admin; opcional aluno. | ✅ Default / ⏳ confirmar |
| 22 | Hosts/prefixos (`admin.app.com`, `/app`, `/manage`, `/affiliate`) | **Marcado como proposta a validar com engenharia** (DNS/cookies/middleware). | ⏳ Validar (eng) |
| 23 | Doc de UX de Autoria/Instrutor (toggles) | Produzido: [AUTHORING_UX.md](AUTHORING_UX.md) (toggles em §8; campos a confirmar no DATA_MODEL §6). | ✅ Resolvido |
| 24 | Política de retenção LGPD (prazos) | Requisito definido; números pendentes de validação jurídica. | ⏳ Jurídico |
| 25 | Recipient da plataforma vs tenant (Pagar.me) | Plataforma em `platform.platform_payment_recipients`; produtor/afiliado cifrados no data plane (§6.8/6.11). | ✅ Resolvido |
| 26 | Onboarding de pagamentos do tenant (recipient/KYC) | Passo de "ativar pagamentos" pós-onboarding (não bloqueia provisionamento); sem recipient válido → checkout indisponível. | ✅ Resolvido |

**Legenda:** ✅ resolvido pela coordenação · ⏳ aguarda stakeholder/engenharia/jurídico (não bloqueia MVP).

---

## 5. Decisões pendentes do stakeholder

Lista completa e com defaults adotados em [OPEN_QUESTIONS §2-bis](../OPEN_QUESTIONS.md) (#10–#27).
Resumo dos **bloqueadores comerciais** (precisam de número para enforcement):
- **#10** Tabela final de planos/quotas/preços do SaaS (valores).
- **#11** Existe fee transacional sobre GMV além da mensalidade?
- **#12/#13/#14** Grace de inadimplência, reembolso parcial, e se alunos perdem acesso quando o tenant fica inadimplente.
- **#24** Prazos de retenção LGPD (validação jurídica).

Itens com **default seguro** já aplicado (podem seguir no MVP e ser revisados): #15–#23, #25–#27.

---

## 6. Lacunas de documentação a produzir (pós-coordenação)
- ✅ **UX de Autoria/Instrutor** (Studio): produzido em [AUTHORING_UX.md](AUTHORING_UX.md) — toggles de
  conclusão manual de vídeo, ligar/desligar comentários, exibir gabarito, aulas opcionais, política de
  tentativas/nota (campos a consolidar no DATA_MODEL §6 com a coordenação).
- **Wireframes** de telas críticas (Checkout, Player, Editor de curso, Console Super-Admin).
- **Schemas Zod em `packages/contracts`** para os fluxos/eventos (tracking plan, DTOs) — fonte única.
- **Tabela legal** de retenção/LGPD (`docs/legal/`).

---

## 7. Nível de confiança global

**~90%** para o conjunto reconciliado.

| Dimensão | Confiança | Observação |
|----------|-----------|------------|
| Isolamento/multitenancy (Regra nº1) | 90% | Nenhuma decisão de produto a violou; quotas e take rate resolvidos sem FK cross-schema. |
| Modelo de dados (com §6) | 87% | Extensões fechadas; restam valores comerciais (quotas/preços). |
| Pagamentos (SaaS × aluno) | 88% | Consistente; pendências são políticas (grace/reembolso parcial), não estruturais. |
| RBAC | 85% | Papéis fixos no MVP; delegação fina e papéis múltiplos adiados. |
| Analytics/tracking plan | 84% | Port + Zod definidos; fee sobre GMV em aberto. |
| Notificações | 88% | MVP in-app+e-mail; push F2. |
| IA/Hosts/Autoria | 80% | Hosts = proposta a validar; doc de autoria a produzir. |

Os ~10% residuais concentram-se em **decisões comerciais/jurídicas** (valores de plano, prazos LGPD,
fee sobre GMV) e **validações de engenharia** (hosts/cookies), nenhuma das quais bloqueia o início da
implementação do MVP.
