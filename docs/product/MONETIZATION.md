# Monetização & Pricing — Plataforma de Cursos SaaS Multitenant

- **Versão:** 1.0
- **Data:** 2026-06-04
- **Status:** Proposta para validação do coordenador de produto
- **Documentos relacionados:** [PRD.md](../PRD.md) · [ROADMAP.md](../ROADMAP.md) · [DATA_MODEL.md](../DATA_MODEL.md) · [ARCHITECTURE.md](../ARCHITECTURE.md) · [ADR-0010 (Pagamentos BR)](../adr/0010-payments-br.md)

> **Princípio central — dois níveis de monetização (não confundir):**
> 1. **SaaS (B2B):** o **tenant paga a NÓS** uma mensalidade/anuidade por tier, com quotas. Cobrança via **Stripe Billing** no control plane (`platform`). O status da assinatura **ativa/suspende** o tenant.
> 2. **Checkout (B2C):** o **aluno paga ao TENANT** pelos cursos. Cobrança via **Pagar.me/Asaas** no schema do tenant. Nós **não** recebemos esse dinheiro diretamente (sem custódia/escrow no MVP) — opcionalmente cobramos um **take rate** que é repassado via split (ver §B.7).

---

## Índice

- [A) Planos do SaaS (tenant → plataforma)](#a-planos-do-saas-tenant--plataforma)
  - [A.1 Tiers concretos e público-alvo](#a1-tiers-concretos-e-público-alvo)
  - [A.2 Matriz de features × quotas](#a2-matriz-de-features--quotas)
  - [A.3 Modelo de cobrança (mensal/anual/trial)](#a3-modelo-de-cobrança-mensalanualtrial)
  - [A.4 Upgrade / downgrade / overage](#a4-upgrade--downgrade--overage)
  - [A.5 Exceder quota e inadimplência (máquina de estados do tenant)](#a5-exceder-quota-e-inadimplência-máquina-de-estados-do-tenant)
- [B) Checkout e monetização do tenant (aluno → tenant)](#b-checkout-e-monetização-do-tenant-aluno--tenant)
  - [B.1 Modalidades de venda](#b1-modalidades-de-venda)
  - [B.2 Cupons](#b2-cupons)
  - [B.3 Order bump (F2)](#b3-order-bump-f2)
  - [B.4 Upsell / downsell one-click (F2)](#b4-upsell--downsell-one-click-f2)
  - [B.5 Bundles / combos (F2)](#b5-bundles--combos-f2)
  - [B.6 Recuperação de carrinho abandonado (F2)](#b6-recuperação-de-carrinho-abandonado-f2)
  - [B.7 Afiliados e split de pagamento](#b7-afiliados-e-split-de-pagamento)
  - [B.8 Meios de pagamento e liberação de acesso](#b8-meios-de-pagamento-e-liberação-de-acesso)
- [Dependências e pontos para o coordenador](#dependências-e-pontos-para-o-coordenador)

---

# A) Planos do SaaS (tenant → plataforma)

## A.1 Tiers concretos e público-alvo

Quatro tiers, alinhados à escala-alvo do PRD (dezenas de tenants, até ~100) e ao faseamento MVP/F2/F3. Preços em **BRL**, sugeridos para o mercado brasileiro de infoprodutos/escolas online (referência: Hotmart/Kiwify cobram take rate de ~5–10% sem mensalidade; Teachable/Kajabi cobram mensalidade USD 39–199). Nosso posicionamento híbrido: **mensalidade enxuta + take rate decrescente por tier** (o tier maior compra a redução da taxa).

| Tier | Preço mensal | Preço anual (≈2 meses grátis) | Take rate sobre vendas | Público-alvo |
|------|-------------|-------------------------------|------------------------|--------------|
| **Starter** | R$ 97/mês | R$ 970/ano | 2,0% + taxas do gateway | Criador iniciante / primeiro infoproduto; valida ideia e faz as primeiras vendas. |
| **Pro** | R$ 297/mês | R$ 2.970/ano | 1,0% + taxas do gateway | Produtor estabelecido / escola pequena com catálogo e afiliados ativos. |
| **Scale** | R$ 697/mês | R$ 6.970/ano | 0,5% + taxas do gateway | Escola/coprodução com volume alto, equipe, domínio próprio e marca. |
| **Enterprise** | Sob consulta (a partir de ~R$ 1.997/mês) | Contrato anual | 0% (mensalidade maior) ou negociado | Empresa/educacional com DRM, SSO, SCORM, SLA e residência de dados. |

> **Take rate:** percentual que a plataforma retém sobre cada venda do checkout do tenant (além das taxas do gateway Pagar.me/Asaas, que são repassadas). Implementado como **recipient da plataforma no split** (ver §B.7). É a alavanca que justifica o modelo "mensalidade baixa": ganhamos junto com o crescimento do tenant. Tiers maiores **reduzem** o take rate como benefício de upgrade.

## A.2 Matriz de features × quotas

Legenda de fase: **[MVP]** disponível no lançamento · **[F2]** Fase 2 · **[F3]** Fase 3. Quando um recurso é F2/F3, a coluna indica **a qual tier ele será liberado** quando entregue (a quota já fica modelada em `platform_plans.limits`).

| Recurso / Quota | Fase | Starter | Pro | Scale | Enterprise |
|-----------------|------|---------|-----|-------|------------|
| **Alunos ativos (matrículas ativas)** | MVP | até 200 | até 2.000 | até 15.000 | ilimitado* |
| **Armazenamento de vídeo (Bunny)** | MVP | 50 GB | 250 GB | 1 TB | negociado |
| **Banda/streaming-mês (egress)** | MVP | 200 GB | 1,5 TB | 8 TB | negociado |
| **Nº de cursos publicados** | MVP | 3 | 25 | ilimitado | ilimitado |
| **Nº de membros de equipe (admin/instrutor)** | MVP | 2 | 10 | 30 | ilimitado |
| **Afiliados ativos** | MVP | 5 | 50 | 500 | ilimitado |
| **Split / co-produção** | MVP (split) / F2 (co-produção) | afiliado simples | + co-produção | + multi-recebedor | completo |
| **Cupons** | MVP | sim | sim | sim | sim |
| **Order bump** | F2 | — | sim | sim | sim |
| **Upsell/downsell one-click** | F2 | — | — | sim | sim |
| **Bundles/combos** | F2 | — | sim | sim | sim |
| **Recuperação de carrinho** | F2 | — | sim | sim | sim |
| **Subdomínio + branding (logo/cores)** | MVP | sim | sim | sim | sim |
| **Domínio próprio + SSL** | F2 | — | sim | sim | sim |
| **White-label (remoção total de marca)** | F2 | — | — | sim | sim |
| **Quiz/certificado** | MVP | sim | sim | sim | sim |
| **Provas avançadas + gradebook** | F2 | — | sim | sim | sim |
| **Gamificação** | F2 | — | sim | sim | sim |
| **Comunidade (comentários)** | MVP | sim | sim | sim | sim |
| **Fórum/feed + lives** | F2 | — | sim | sim | sim |
| **PWA + push** | F2 | — | sim | sim | sim |
| **App mobile branded (lojas)** | F3 | — | — | add-on | sim |
| **IA: transcrição/legenda + geração de quiz** | F2 | — | quota baixa | quota alta | ilimitado* |
| **IA: Tutor IA (RAG)** | F3 | — | — | add-on | sim |
| **Anti-pirataria: Token + MediaCage** | MVP | sim | sim | sim | sim |
| **Watermark dinâmico por aluno** | F2 | — | sim | sim | sim |
| **Limite de sessões/dispositivos** | F2 | — | sim | sim | sim |
| **DRM Enterprise (Widevine/FairPlay)** | F3 | — | — | — | sim |
| **API REST pública + chaves** | F2 | — | — | sim | sim |
| **Webhooks de saída** | MVP | sim | sim | sim | sim |
| **SSO (SAML/OAuth) / SCORM/xAPI / LTI** | F3 | — | — | — | sim |
| **Analytics básico (conclusão/receita)** | MVP | sim | sim | sim | sim |
| **Analytics avançado (coorte/MRR/heatmap)** | F2 | — | sim | sim | sim |
| **Exportação de dados (LGPD/lock-in)** | MVP | sim | sim | sim | sim |
| **Suporte** | — | comunidade/e-mail (48h) | e-mail prioritário (24h) | chat + gerente (8h úteis) | SLA dedicado + Slack |

\* "ilimitado" é soft (fair-use) com alertas de abuso; quotas duras técnicas (armazenamento/banda) sempre existem para conter custo de egress (ver risco de custo de mídia no PRD §7).

> **Onde isso vive no modelo de dados:** `platform_plans.limits jsonb` já comporta `{ max_students, storage_gb, bandwidth_gb, max_courses, max_team, max_affiliates, features[], take_rate_bps }`. Cada quota numérica é um campo; cada feature booleana entra em `features[]`. Use **basis points (bps)** para take rate (ex.: `200` = 2,0%) para evitar float.

## A.3 Modelo de cobrança (mensal/anual/trial)

| Item | Regra |
|------|-------|
| **Ciclos** | Mensal e anual. Anual = preço de ~10 meses (2 meses grátis), cobrado à vista. |
| **Trial** | **14 dias** em Starter/Pro, sem cartão para começar (reduz fricção de ativação — métrica de ativação do PRD: publicar ≥1 curso + ≥1 venda em 14 dias). Cartão exigido para converter. Scale/Enterprise: trial via sales/POC, não self-service. |
| **Gateway** | **Stripe Billing** (control plane), conforme ADR-0010. `platform_subscriptions.stripe_subscription_id` + `status` + `current_period_end`. |
| **Faturamento** | Faturas em `platform_invoices`. Cobrança recorrente automática; e-mail (Resend) com fatura/recibo. |
| **Moeda** | BRL. Valores em **centavos** (`price_cents`), conforme convenção do DATA_MODEL. |
| **Impostos** | Preços anunciados; emissão fiscal do SaaS tratada fora do MVP (nota da plataforma para o tenant). |
| **Take rate** | Cobrado **no momento da venda do aluno** via split (não na fatura SaaS) — ver §B.7. A fatura SaaS é só a mensalidade/anuidade. |

## A.4 Upgrade / downgrade / overage

| Operação | Regra |
|----------|-------|
| **Upgrade (Starter→Pro→Scale)** | Imediato. **Proration** (Stripe) cobra a diferença do período corrente; novas quotas/features liberadas na hora. Take rate menor passa a valer para vendas a partir do upgrade. |
| **Downgrade** | Efetivo **no fim do período corrente** (evita reembolso e perda de valor já pago). Validação prévia: se o tenant **excede a quota do tier de destino** (ex.: 8 cursos publicados mas Starter permite 3), o downgrade é **bloqueado até regularizar** (arquivar cursos, reduzir equipe). UI mostra o "delta a resolver". |
| **Mudança de ciclo (mensal↔anual)** | Anual aplica desconto; mensal→anual com proration a crédito. |
| **Overage (estouro de quota)** | Estratégia híbrida por tipo de quota — ver tabela abaixo. |

**Política de overage por tipo de quota:**

| Quota | Comportamento ao exceder |
|-------|--------------------------|
| **Armazenamento de vídeo** | Soft block: bloqueia **novos uploads** ao atingir 100%; conteúdo existente segue no ar. Alertas em 80%/100%. Add-on de storage avulso (ex.: +100 GB por R$ 49/mês) ou upgrade. |
| **Banda/streaming-mês** | Soft cap: alertas em 80%/100%. Acima de 100% → **cobrança de overage** por GB (ex.: R$ 0,25/GB, repassando custo Bunny + margem) na próxima fatura, OU degradação de resolução máxima (cap configurável) para conter custo. Nunca corta o aluno no meio da aula. |
| **Alunos ativos** | Soft block: permite ultrapassar com **aviso** e cobrança de pacote de alunos extra (ou força upgrade). Nunca suspende alunos já matriculados (impacto reputacional no tenant). |
| **Nº de cursos / equipe / afiliados** | Hard block na **criação** do (N+1)-ésimo recurso → CTA de upgrade. Não afeta recursos existentes. |
| **IA (tokens/minutos)** | Quota mensal; ao esgotar, recurso pausa com opção de comprar pacote adicional. |

> Overages e add-ons geram **itens de fatura** no Stripe Billing. Toda quota é checada por um **serviço de quotas** central (ver §Dependências) que lê `platform_plans.limits` e o uso corrente.

## A.5 Exceder quota e inadimplência (máquina de estados do tenant)

A coluna `tenants.status` já prevê `provisioning | active | suspended | cancelled`. Propomos um estado intermediário lógico de **inadimplência (past_due/grace)** dirigido pelo `platform_subscriptions.status` do Stripe (não precisa migrar `tenants.status`; pode ser derivado):

| Estado | Gatilho | Efeito no tenant | Efeito nos alunos |
|--------|---------|------------------|-------------------|
| **active** | Assinatura paga em dia | Pleno acesso | Acesso normal aos cursos |
| **past_due (grace)** | Falha de cobrança (Stripe `past_due`) | Banner de aviso; admin pode operar; **bloqueia novos uploads e novas vendas**? (decidir — ver dependências). Retries de cobrança (dunning) por 7–14 dias. | **Sem impacto** (alunos continuam estudando — protege a reputação do tenant) |
| **suspended** | Grace esgotado (ex.: 14 dias) ou suspensão manual do Super-Admin | Painel admin em modo leitura; **checkout do tenant desativado** (novas vendas off); conteúdo fica preservado | Acesso ao conteúdo **pausado** com página explicativa (decisão sensível — pode-se optar por manter alunos pagantes ativos; ver dependências) |
| **cancelled** | Cancelamento do tenant ou inadimplência prolongada | Sem acesso ao painel | Sem acesso. Dados retidos por janela de carência antes de `DROP SCHEMA` (LGPD/lock-in: exportação disponível) |

**Regras-chave:**
- Inadimplência do **SaaS** (tenant→nós) é **distinta** da inadimplência do **aluno** (aluno→tenant). Nunca confundir as duas máquinas de estado.
- Suspensão **nunca apaga dados**; `DROP SCHEMA` só após janela de retenção + opção de exportação (PRD §3.9, lock-in PRD §7).
- Toda transição de estado sensível (suspender/cancelar) → `platform.audit_log`.
- Reativação após pagamento: volta a `active` automaticamente via webhook do Stripe (idempotente).

---

# B) Checkout e monetização do tenant (aluno → tenant)

> Todo o checkout vive no **schema do tenant** (`orders`, `subscriptions`, `coupons`, `affiliates`, `affiliate_commissions`, `splits`, `payment_events`). Gateways: **Pagar.me** (principal, split) + **Asaas** (Pix/boleto recorrente), atrás da port `PaymentProvider` (ADR-0010, ARCHITECTURE §8).

## B.1 Modalidades de venda

| Modalidade | Fase | UX / regra | Liberação de acesso |
|------------|------|-----------|---------------------|
| **Venda avulsa (one-time)** | MVP | Aluno escolhe curso → checkout → paga uma vez. `courses.pricing_type='one_time'`, `orders` gerado. | Matrícula `active` ao pagamento aprovado; `expires_at` opcional (acesso vitalício ou com validade). |
| **Assinatura/recorrência** | MVP | Acesso enquanto a assinatura está ativa (mensal/anual). `pricing_type='subscription'` + `subscriptions` do aluno. | Acesso enquanto `subscriptions.status='active'`; expira em `current_period_end` se não renovar. |
| **Trial de assinatura** | MVP | Período inicial (ex.: 7 dias) configurável pelo tenant. | Acesso liberado no trial; suspende se não converter. |
| **Gratuito** | MVP | `pricing_type='free'` (lead magnet). | Matrícula direta sem `order`. |
| **Bundle/combo** | F2 | Vários cursos por preço único (ver §B.5). | Múltiplas matrículas em uma compra. |

**UX de checkout (MVP):** página única, mobile-first, com resumo do pedido, seleção de meio de pagamento (Pix/boleto/cartão), campo de cupom, captura de e-mail/CPF (CPF necessário para nota/antifraude/Pix), e gatilhos de urgência opcionais (timer de oferta). Pixels/UTM (Meta/GA4) já no MVP (PRD §3.8) para atribuição.

## B.2 Cupons

- Modelo: `coupons(code, type, value, valid_until, max_uses, uses)`. `type ∈ {percent, fixed}`.
- Regras: validade, limite de usos global (`max_uses`), aplicável por curso/bundle ou global do tenant. Validação no backend antes de criar a `order` (nunca confiar no front).
- **Interação com afiliado e split:** decidir se o desconto sai da margem do produtor ou rateado (ver dependências). Padrão proposto: desconto **reduz a base** sobre a qual comissão de afiliado e take rate são calculados.

## B.3 Order bump (F2)

- Oferta complementar de baixo atrito **dentro da página de checkout** (checkbox "adicione X por +R$ Y"), antes do pagamento.
- Aceitar o bump adiciona um item à mesma `order` (mesmo pagamento, sem novo cartão). UX: 1 clique, sem sair do checkout.
- Configurável por produto-alvo no admin (qual curso oferecer como bump, preço promocional). Conversão medida no funil de checkout (analytics F2).

## B.4 Upsell / downsell one-click (F2)

- **Pós-compra** (após aprovação do pagamento principal): tela de oferta adicional comprável **em 1 clique reusando o meio de pagamento** já tokenizado (cartão) ou gerando novo Pix.
- **Downsell:** se o aluno recusa o upsell, oferta alternativa mais barata.
- Requisito técnico: tokenização de cartão no Pagar.me para cobrança one-click; para Pix, gera nova cobrança (não é "one-click" puro, mas fluxo encadeado). Cada aceite gera nova `order` vinculada à sessão de compra (campo de correlação a adicionar — ver dependências).

## B.5 Bundles / combos (F2)

- Conjunto de cursos vendido por preço único (com desconto vs. comprar separado). Modelagem: tabela `bundles` + `bundle_items` (a adicionar) ou uma `order` que gera N matrículas.
- Split/afiliado: comissão e take rate calculados sobre o valor total do bundle; rateio interno entre cursos só importa para relatórios de receita por curso (analytics).

## B.6 Recuperação de carrinho abandonado (F2)

- Captura de e-mail no início do checkout → se `order` fica `pending` além de X minutos sem pagamento, dispara sequência (e-mail/push) via event bus + e-mail marketing (PRD §3.8 F2).
- Para Pix/boleto: lembrete antes do vencimento do código. Para cartão recusado: tentativa de recuperação.
- Estado `pending` da `order` é o gatilho; reconciliação evita disparar para pedidos já pagos.

## B.7 Afiliados e split de pagamento

### Cadastro e links
- Afiliado é um `users.role='affiliate'` no tenant + registro em `affiliates(user_id, code, commission_pct, status)`.
- Fluxo: admin habilita programa → afiliado se cadastra/é aprovado (`status ∈ {pending, active, blocked}`) → recebe **código** e **links** (`https://tenant.app.com/c/curso?aff=CODE`).
- Painel do afiliado: cliques, conversões, comissões `pending/paid/reversed`, materiais de divulgação.

### Cookie / atribuição
| Item | Regra proposta |
|------|----------------|
| **Modelo de atribuição** | **Last-click** (padrão de mercado BR/Hotmart). O último `aff` válido antes da compra leva a comissão. |
| **Janela de cookie** | Configurável pelo tenant; padrão **30 dias**. |
| **Persistência** | Cookie first-party no domínio do tenant + fallback de associação ao `user_id` ao logar/comprar. `orders.affiliate_id` registra o atribuído. |
| **Auto-indicação** | Bloqueada (afiliado não comissiona a própria compra). |

### Percentual de comissão
- `affiliates.commission_pct` (padrão por afiliado) com possibilidade de override por curso (via `splits` ou campo por curso — ver dependências). Faixa típica de mercado: **30–60%** em infoprodutos.
- Comissão registrada em `affiliate_commissions(order_id, amount_cents, status)`.

### Split de pagamento (Pagar.me)
O **split nativo do Pagar.me** divide o valor **na liquidação**, direto para os recebedores (cada um com `recipient_id` no Pagar.me) — evita que o tenant receba tudo e tenha que repassar manualmente (reduz risco fiscal e de inadimplência interna). Modelado em `splits(course_id, recipient_ref, percent)`.

**Ordem de cálculo de uma venda (proposta):**

```
valor_bruto (order.amount_cents)
  ├─ (−) taxas do gateway (Pagar.me/Asaas)         → retidas pelo gateway
  ├─ (−) take rate da plataforma (NÓS)             → split p/ recipient da plataforma
  ├─ (−) comissão do afiliado (se houver)          → split p/ recipient do afiliado
  ├─ (−) parte do(s) co-produtor(es) (F2)          → split p/ recipients dos co-produtores
  └─ (=) líquido do produtor (tenant)              → split p/ recipient do produtor
```

| Recebedor | Recipient Pagar.me | Quando entra |
|-----------|--------------------|--------------|
| **Produtor (tenant/dono)** | `recipient` do tenant (provisionado no onboarding ou no opt-in de pagamentos) | Sempre |
| **Plataforma (NÓS)** | `recipient` da plataforma | Conforme `take_rate_bps` do tier (§A.1) |
| **Afiliado** | `recipient` do afiliado | Quando há atribuição válida |
| **Co-produtor(es)** | `recipients` adicionais | F2 (co-produção) |

> O **take rate do SaaS** é, portanto, **operacionalizado aqui no split** do checkout do aluno — não na fatura Stripe. A fatura Stripe é só a mensalidade. Isso exige que o cálculo de split leia o `take_rate_bps` do plano do tenant (control plane) no momento da venda (ver dependências — cruzamento platform × tenant).

### Liberação, reembolso e estorno de comissão
| Evento | Regra |
|--------|-------|
| **Liberação da comissão** | Comissão `pending` até o fim do **prazo de garantia/reembolso** do produto (ex.: 7–30 dias, configurável). Após isso → `paid` (liquidação). Protege contra reembolso retroativo. |
| **Reembolso / chargeback** | `order.status='refunded'/'chargeback'` → comissão correspondente vira `reversed`; se já tiver sido paga ao afiliado, gera saldo negativo/compensação futura. Split do Pagar.me deve refletir o estorno proporcional entre recebedores. |
| **Idempotência** | Reembolso/estorno disparado por webhook idempotente (`payment_events.event_id`), nunca duplica reversão. |
| **Take rate em reembolso** | Reembolso integral → plataforma estorna seu take rate proporcional (decidir política de retenção de taxa em reembolso parcial — ver dependências). |

## B.8 Meios de pagamento e liberação de acesso

| Meio | Gateway | Tempo até aprovação | Liberação de acesso |
|------|---------|---------------------|---------------------|
| **Pix** | Pagar.me / Asaas | Segundos a minutos | Imediata ao webhook `paid` (idempotente). Melhor conversão BR. |
| **Cartão de crédito** | Pagar.me | Imediato (autorização) | Imediata ao aprovar. **Parcelamento** com ou sem juros (juros configuráveis pelo tenant; quem absorve a taxa de parcelamento é decisão do tenant). |
| **Boleto** | Pagar.me / Asaas | 1–3 dias úteis (compensação) | **Só após compensação** (webhook `paid`). Até lá `order='pending'` e acesso **não** liberado. Lembrete de vencimento + recuperação (F2). |

**Regra-mãe (PRD §4.3, ARCHITECTURE §8):** *pagamento governa acesso*. Webhook aprovado → matrícula `active` + acesso liberado; reembolso/chargeback/cancelamento → suspende. Tudo idempotente (`payment_events`) + **reconciliação periódica** com o gateway para corrigir divergências. O backend **sempre** revalida entitlement antes de gerar URL assinada de vídeo (TTL curto).

**Estados de `orders` e seu efeito:**

| `orders.status` | `enrollments.status` | Acesso |
|-----------------|----------------------|--------|
| `pending` | (não criada ainda ou `suspended`) | Bloqueado |
| `paid` | `active` | Liberado |
| `refunded` | `refunded` | Suspenso + comissão `reversed` |
| `chargeback` | `suspended` | Suspenso + comissão `reversed` |

---

## Dependências e pontos para o coordenador

Pontos que **cruzam** com outros domínios e precisam de decisão/alinhamento antes da implementação.

### 1. RBAC (cruza com ARCHITECTURE §6 — papéis `owner|admin|instructor|affiliate|student`)
- **Quem gerencia o quê:** propor que só `owner`/`admin` configurem planos de venda, cupons, programa de afiliados, splits e meios de pagamento. `instructor` cria conteúdo mas não mexe em pricing. `affiliate` vê só o próprio painel. `student` só compra/consome. Confirmar matriz de permissões por feature de monetização.
- **Super-Admin (platform):** quem define o `take_rate_bps` por tier e pode conceder exceções (ex.: take rate negociado Enterprise). Toda alteração → `audit_log`.

### 2. Estados de pagamento (cruza com PRD §3.6/§4.3, DATA_MODEL §2.5)
- **Duas máquinas de estado distintas** que NÃO podem ser confundidas: (a) inadimplência do **SaaS** (tenant→nós, via Stripe → `tenants.status`/`platform_subscriptions.status`); (b) inadimplência do **aluno** (aluno→tenant, via Pagar.me → `orders`/`enrollments`).
- **Decisão pendente (sensível):** quando o **tenant** fica inadimplente conosco (`suspended`), os **alunos pagantes** dele perdem acesso? Trade-off: cortar pressiona o tenant a pagar, mas pune alunos inocentes e gera dano reputacional/jurídico. Recomendação: na fase `grace` **não** cortar alunos; em `suspended` avaliar manter alunos com matrícula ativa acessando, bloqueando só o painel/novas vendas do tenant.
- **Modelagem a confirmar:** estado intermediário `past_due/grace` é derivado do Stripe ou vira coluna explícita em `tenants.status`? (Proposta: derivado, sem migração.)
- **Take rate em reembolso parcial:** política de retenção/estorno da taxa da plataforma.

### 3. Analytics (cruza com PRD §3.13, ROADMAP F2)
- Métricas que a monetização exige: **funil de checkout** (visita→checkout→pagamento, conversão por meio de pagamento), **MRR/churn do SaaS**, **GMV/take rate** por tenant, **conversão de order bump/upsell**, **performance de afiliados** (cliques→vendas), **overage** por quota.
- North Star do PRD (horas assistidas) é de engajamento; monetização adiciona métricas de receita. Garantir que o event bus capture eventos de compra/comissão/split para os dashboards.

### 4. Quotas técnicas (cruza com PRD §3.12, DATA_MODEL `platform_plans.limits`, risco de egress PRD §7)
- **Serviço de quotas central** (provável em `packages/` ou módulo `identity`/control plane) que lê `platform_plans.limits` e o **uso corrente** (alunos ativos, GB de storage/banda Bunny, nº de cursos/equipe/afiliados) para autorizar ou bloquear ações. Precisa de fonte de uso confiável (consultar Bunny para storage/banda; contar no schema do tenant para o resto).
- **Custo de egress de vídeo** é o maior risco de margem (PRD §7): as quotas de banda e a política de overage por GB precisam estar calibradas contra o custo real Bunny (Volume vs Standard) — depende da resolução de OPEN_QUESTIONS.
- **Cruzamento control plane × tenant:** o cálculo de split no checkout (schema do tenant) precisa ler o `take_rate_bps` do plano (schema `platform`). Definir como esse dado chega ao use-case de pagamento sem violar o isolamento (ex.: injetar via contexto resolvido no `onRequest`, junto com `request.tenant`).

### 5. Modelagem de dados a estender (novas tabelas/campos sugeridos)
- `platform_plans.limits`: padronizar chaves (`max_students`, `storage_gb`, `bandwidth_gb`, `max_courses`, `max_team`, `max_affiliates`, `take_rate_bps`, `features[]`).
- Checkout F2: `bundles`/`bundle_items`; campo de correlação de **sessão de compra** para encadear upsell/downsell à compra original; `order_items` (para order bump/bundle, hoje `orders` tem só 1 `course_id`).
- Afiliados: confirmar override de `commission_pct` por curso (via `splits` ou novo campo) e relação afiliado↔co-produtor (F2).
- Recipients Pagar.me: onde guardar `recipient_id` de produtor/afiliado/co-produtor/plataforma (control plane para a plataforma; schema do tenant para produtor/afiliado) — chaves sensíveis cifradas (pgcrypto/KMS), como já feito com keys da Bunny.

### 6. Gateways e provisionamento
- **Onboarding de pagamentos do tenant:** o tenant precisa criar/conectar um `recipient` no Pagar.me (KYC) antes de vender. Definir se isso entra na saga de provisionamento (PRD §5.1) ou é um passo de "ativar pagamentos" pós-onboarding. Sem recipient válido, o checkout do tenant fica indisponível.
- **Pagar.me vs Asaas por feature:** split → Pagar.me; Pix/boleto recorrente de baixo custo → Asaas. A seleção por estratégia/feature flag (ADR-0010) precisa de regra clara de roteamento por tipo de cobrança.
