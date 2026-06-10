# Análise de Gaps — Revisão Adversarial (Red Team) da Especificação Completa

- **Versão:** 1.0 · **Data:** 2026-06-08
- **Método:** 4 revisores independentes (Técnico/Arquitetura · Produto/Jornadas · Negócio/Financeiro/Legal · Auditoria de Consistência) sobre os ~49 documentos do repositório, com verificação externa de pontos regulatórios/fornecedores.
- **Status:** Achados consolidados e deduplicados. **Nenhum documento-base foi alterado ainda** — este relatório é o insumo da rodada de correção.

> **Veredito honesto:** a especificação está **forte em arquitetura de estados, isolamento multitenant e
> cobertura funcional**, mas o selo "≥95% de assertividade" está **inflado**. Os gaps se concentram em
> (a) **dinheiro de verdade** (unit economics, fiscal, split/chargeback), (b) **costuras entre
> especialidades** que nenhum agente "possuiu" (auth×schema, webhooks×tenant, hosts×cookies) e
> (c) **funcionalidades [MVP] declaradas sem spec operável**. Até as Ondas 0–1 do plano de correção
> serem executadas, a confiança real do conjunto é **~80%**.

---

## Índice
1. [🔴 Críticos — bloqueiam MVP ou decisão de negócio](#1--críticos)
2. [🟠 Importantes — resolver antes da feature correspondente](#2--importantes)
3. [🟡 Registrar — corrigir oportunisticamente](#3--registrar)
4. [🧹 Inconsistências entre documentos (auditoria)](#4--inconsistências-entre-documentos)
5. [Plano de correção em ondas](#5-plano-de-correção-em-ondas)
6. [Impacto no nível de confiança](#6-impacto-no-nível-de-confiança)

---

## 1. 🔴 Críticos

### 💰 Dinheiro / viabilidade do negócio

**G1. A margem inteira depende de um preço de CDN NÃO confirmado (gate de go/no-go do pricing).**
As quotas de banda do MONETIZATION foram dimensionadas no preço da **Bunny Volume Network**
(US$ 0,005/GB), cuja cobertura/qualidade no BR está pendente (OQ #1). Na rede **Standard SA**
(US$ 0,045/GB, preço público), **Pro e Scale ficam com margem negativa no uso pleno da quota**
(ex.: tenant Pro com 500 alunos ativos ≈ 2,7 TB/mês ⇒ ~R$ 670 de custo vs R$ 297 de mensalidade).
Agravante: a North Star ("horas assistidas") maximiza o custo nº 1, e o overage de R$ 0,25/GB mal
cobre o custo Standard. → **Ação:** confirmar a Volume Network com a Bunny **antes** de fixar preços;
plano B numérico assumindo Standard (quotas menores, cap 720p/480p, preços maiores).

**G2. Três obrigações de nota fiscal ignoradas ou adiadas.**
(a) NFS-e da **mensalidade do SaaS** — obrigação desde a 1ª fatura, "fora do MVP" é insustentável;
(b) NFS-e do **take rate** (receita de intermediação por venda, ISS) — sem modelagem em nenhum doc;
(c) NF do **tenant** em F3 exclui tenants PJ sérios (o ICP do Scale/Enterprise) e perde para
Hotmart/Eduzz. → **Ação:** emissor (eNotas/Focus/NFE.io) no MVP para (a)+(b); antecipar (c) para F2
via port, ou ao menos export contábil por venda + guia de integração via webhook no MVP; aceitar
**CNPJ no checkout**.

**G3. Take rate via split: contradição operacional + vazamento via Asaas + enquadramento BACEN.**
(i) O split do Pagar.me liquida na **liquidação do recebível**; a máquina de comissão diz `pending`
por 14 dias — se o dinheiro já saiu, o estado é ficção contábil; segurar fundos transformaria a
plataforma em transitadora (risco de enquadramento como **subcredenciadora** — Circ. BACEN
3.886/2018). (ii) Vendas roteadas ao **Asaas** (Pix/boleto recorrente) **não têm split** ⇒ take rate
e comissão de afiliado não são cobrados nessas vendas; a "regra de roteamento" está aberta.
(iii) Ninguém definiu se afiliado **comissiona renovações** de assinatura. → **Ação:** decidir o
modelo (split na liquidação aceitando estorno como saldo negativo, vs payout agendado); roteamento
Pagar.me×Asaas decidido **pelo impacto no take rate**; regra de comissão em recorrência; **parecer
jurídico-regulatório** do arranjo.

**G4. Chargeback/reembolso modelados como estados, não como fluxo de caixa.**
Chargeback de cartão chega em até ~120 dias (≫ clearance de 14d e garantia de 7–30d); não há matriz
de absorção (`liable` por recipient no Pagar.me), reserva/trava de saque, nem runbook de reembolso
por meio de pagamento (boleto exige dados bancários; Pix exige ação ativa). O **direito de
arrependimento (CDC art. 49, 7 dias)** não aparece em NENHUM doc (nem no COMPLIANCE), e o aluno
**não tem fluxo para pedir reembolso** (sem rota, sem estado `refund_requested`). → **Ação:** matriz
de absorção + `liable`; clearance ≥ garantia; reserva/trava; fluxo self-service de reembolso +
`refund_window_days` (default 7, CDC) + copy no checkout; antifraude de cartão no MVP.

**G5. KYC do recipient Pagar.me não é estado de produto — e colide com a ativação.**
KYC leva horas–dias e pode ser **recusado**; não há estado `payments_pending/kyc_rejected`, UX, nem
e-mails. O trial é de 14 dias e a ativação é "1ª venda em 14 dias" — o KYC consome o trial. Afiliados
**também são recipients** (tier Scale promete 500). → **Ação:** "ativar pagamentos" como sub-estado de
primeira classe (telemetria própria), KYC iniciado na saga de onboarding, funil de afiliado com
`kyc_pending/rejected`.

**G6. LGPD: papel Controlador×Operador pendente bloqueia DPA/Termos/lançamento.**
O próprio COMPLIANCE admite que tudo deriva do enquadramento (OQ #37); prazos de retenção são
placeholders (OQ #24); Encarregado (DPO) sem pessoa indicada; transferência internacional pendente.
É caminho crítico de **semanas** (jurídico externo), não tarefa de F2. → **Ação:** contratar a
validação jurídica agora, em paralelo ao build.

**G7. "Acesso vitalício" vendido pelo tenant × churn do tenant = responsabilidade solidária (CDC).**
Aluno comprou "vitalício"; tenant cancela; schema é dropado após ~60 dias — a plataforma (que hospeda,
processa e retém take rate) responde solidariamente em reclamação de massa. Inverso: manter alunos
ativos durante `suspended` + offboarding = servir egress de graça por ~2,5 meses. → **Ação:** decidir
OQ #14 com jurídico; vedação/limitação contratual de promessas "vitalício" nos Termos B2B; degradar
resolução durante suspensão.

### 🔐 Costuras de arquitetura

**G8. Better-Auth × schema-per-tenant é circular como está escrito — e a auth do Super-Admin não tem desenho.**
As tabelas `users/sessions` vivem no schema do tenant, mas validar a sessão exige escolher o schema
antes — o subdomínio não "apenas sugere", é insumo obrigatório da autenticação (contradiz
ARCHITECTURE §3.2). Instância Better-Auth por tenant (custo/cache) nunca foi especificada; o
super-admin (`platform`) exige uma **segunda stack de auth** não desenhada; fluxos órfãos: login no
host central ("não lembro minha escola"), reset por tenant, "find my school". A **impersonação**
(MVP) não tem mecanismo (SA não tem linha em `users` do tenant). → **Ação:** elevar a PoC a **spike
bloqueante com ADR de resultado**, incluindo impersonação como caso de teste; considerar
`platform.user_directory(email_hash, tenant_id)`; reescrever §3.2. O Plano B (`@fastify/jwt`+argon2)
parece hoje **mais aderente** ao modelo.

**G9. Webhook de pagamento chega sem contexto de tenant — e `payment_events` mora no schema do tenant.**
Conta única do gateway + endpoint único ⇒ o evento não diz o tenant; o mapeamento
(`provider_order_id → tenant`) está **dentro** do schema que ainda não se sabe qual é. Não existe
tabela de roteamento no control plane (a Bunny tem `bunny_library_id`; pagamento não tem nada).
Sem isso **o MVP não fatura**. → **Ação:** `platform.payment_routing(provider, provider_object_id,
tenant_id)` populada na criação do pedido/assinatura (ou metadata validada contra o pedido dentro do
`withTenant`); documentar o fluxo webhook→tenant antes do módulo `payments`.

**G10. O HMAC do webhook da Bunny NÃO existe — 4 docs verificam uma assinatura fictícia.**
O webhook do Bunny Stream é um POST simples sem assinatura; o controle descrito (HMAC, tempo
constante) é inválido para este provedor — qualquer um com a URL marca vídeos como `ready`. E o
webhook é configurado **por Video Library**: a saga de provisionamento (ARCHITECTURE §3.4) **não
configura a WebhookUrl** ⇒ tenant novo = vídeos eternamente `processing`. Asaas também não usa HMAC
(token estático). → **Ação:** tratar webhook Bunny como dica não confiável (URL com segredo por
tenant + **re-fetch do status na API** antes de transicionar); adicionar configuração de webhook à
saga; reescrever o "padrão único de webhook" **por provedor**.

**G11. Assinatura do aluno é [MVP] sem spec operável e órfã no modelo de dados.**
(Confirmado independentemente pelos revisores técnico e de produto.) `subscriptions(plan_ref text)`
não liga a nada: sem plano de conteúdo (catálogo? curso?), sem `enrollments.subscription_id`, a
transição "assinatura `past_due` → matrícula `suspended`" é inexecutável; `expires_at` ×
`current_period_end` sem dono; sem fluxo de checkout recorrente (Pix não tem retry), sem trial
modelado, sem dunning do aluno. → **Ação:** ou especificar ponta a ponta (modelo + telas + fluxos +
ADR) ou **rebaixar honestamente para F2** e lançar o MVP com venda avulsa + gratuito.

**G12. Topologia de hosts/cookies/origem da API indefinida.**
Front na Vercel (`acme.app.com`) e API no Fly: nenhum doc define o host da API; SEC_OPS promete
`SameSite=Lax` E CORS cross-origin com credenciais (incompatíveis); wildcard+SSL na Vercel exige
delegação de NS (não verificado); domínio próprio (F2) quebra cookie `.app.com`; `admin.app.com` é
"proposta" com telas inteiras já especificadas sobre ela. → **Ação:** mini-ADR "Topologia de hosts e
sessão" **antes do scaffolding** (api.app.com + `Domain=.app.com` vs proxy via Next; estratégia F2 de
custom domain).

### 📦 Produto / escopo do MVP

**G13. Anti-seek × velocidade 2x pode impedir conclusão (e certificado).**
"Tempo real ≥ duração × 0.8" nunca define se é tempo de **mídia** ou de **relógio**. A 2x, relógio =
0.5×duração < 0.8 ⇒ aula nunca conclui automaticamente. → **Ação:** definir formalmente tempo de
mídia efetivamente reproduzido (independe de velocidade, com dedupe de intervalos) e atualizar PRD
§4.4 / BUSINESS_RULES / US-4.6.

**G14. Curso editado após a venda sem regras: deleção de aula pode disparar certificados em massa.**
Remover aula/módulo ⇒ alunos a 80% pulam para 100% ⇒ emissão automática em massa; adicionar aula a
curso concluído ⇒ certificado "desatualizado" sem política; `lesson_progress` órfão. → **Ação:**
certificado emitido = **imutável (snapshot)**; recálculo de % via job em mudança estrutural; deleção
com progresso exige confirmação e arquiva; user stories para os casos.

**G15. "Auto-enroll por regra" é feature [MVP] fantasma.**
Zero spec (gatilho, condição, tela, idempotência). → **Ação:** especificar ou cortar do MVP (ajustar
PRD/ROADMAP/`enrollments.source`).

---

## 2. 🟠 Importantes (resolver antes da feature correspondente)

| # | Achado | Onde dói | Ação resumida |
|---|--------|----------|---------------|
| O1 | Pipeline deploya **antes** das migrations; contrato de compatibilidade invertido; sem canary/halt-on-failure | CI/CD | Migrate→deploy (ou expand/contract formal) + canary tenant + runbook de falha no schema N/80 |
| O2 | pg-boss: usa **polling** (ok com PgBouncer, não documentado); schema `pgboss` × least-privilege (role sem CREATE falha no boot); worker deve conectar direto | Worker | Registrar no ADR-0011; `pgboss` criado pela role `migrator`; corrigir "rate-limit store = pg-boss" (errado) |
| O3 | `lesson_progress` sem `unique(enrollment_id, lesson_id)` — heartbeats concorrentes duplicam linhas | Progresso | Constraint + `ON CONFLICT` documentado (idem `quiz_attempts`) |
| O4 | Volume de writes de heartbeat subestimado (300–500 tx/s em pico vs pool 5–10) — agregação só foi resolvida p/ PostHog, não p/ Postgres | Progresso | Batching/flush 30–60s no backend; capacidade-alvo no NFR |
| O5 | Matrícula dupla (manual+compra): linha única não representa 2 entitlements; reembolso revoga acesso de cortesia; transição `refunded→active` (recompra) não existe na máquina | Matrículas | Grants múltiplos ou precedência por `source` + corrigir máquina |
| O6 | Bunny fetch-from-URL: presigned R2 com TTL curto pode expirar no retry de MP4 grande | Live F2 / Import | TTL ≥ 24h p/ ingestão; PoC com arquivo grande |
| O7 | "1 projeto LiveKit por tenant" via API **não tem evidência de existir**; planos limitam projetos | Live F2 | Validar com provedor; plano B: 1 projeto + namespace de sala (atualizar ADR-0015 + threat model) |
| O8 | `splits.recipient_ref text` vago; recipient do **produtor** não tem coluna em lugar nenhum; conflito com política de cifragem | Pagamentos | Tipar (`recipient_type`+`recipient_id`); coluna cifrada do recipient do tenant |
| O9 | Checkout com e-mail já existente: "vincula ao usuário" (BUSINESS_RULES) × "erro+login" (USER_FLOWS) — vínculo sem prova de posse = account takeover | Checkout | E-mail existente ⇒ login/OTP; order paga órfã ⇒ matrícula pendente de reivindicação |
| O10 | Drip "dias após matrícula": notificação prometida é impossível (não há job por aluno) — 3 docs se contradizem | Drip | Job diário por tenant materializando liberações, ou remover a notificação do MVP |
| O11 | `pass_score`/tentativas/tempo de quiz: MVP num doc, F2 em outros; semântica do gate de certificado dupla (`pass_score` × `cert_requires_pass`) | Quiz/Certificado | Decidir fase e unificar semântica |
| O12 | Acesso com validade na compra avulsa: `expires_at` sem fonte (sem `access_duration_days` no curso, sem toggle) | Monetização | Adicionar campo+UI ou declarar avulso = vitalício no MVP |
| O13 | Cupom: escopo por curso prometido sem modelagem; cupom 100% (order R$ 0) sem tratamento de gateway | Checkout | `coupons.course_id`/escopo; order zero ⇒ bypass auditado ou proibir |
| O14 | Aula demo/preview gratuito: prometida nas jornadas, proibida pelo entitlement (sem flag, sem exceção de token) | Landing | `lessons.is_free_preview` + token público limitado, ou remover da jornada |
| O15 | Parcelamento BR sub-especificado: sem `installments` em `orders`, sem máximo/juros/exibição CDC, sem interação com split | Checkout | Especificar; deixar claro que parcelado ≠ recorrência |
| O16 | Instrutor sai do tenant: nenhum fluxo (cursos órfãos, comissões, transferência de ownership pendente) | Equipe | Spec "remover membro": reatribuição obrigatória + autoria histórica |
| O17 | Tenant suspenso: 3 comportamentos diferentes em 4 docs (ver também G7) | Estados | Fixar default único (recomendação OQ #14) e propagar |
| O18 | Home do tenant + editor de landing são MVP sem spec de blocos/edição ("landing de alta conversão" é pilar sem spec) | Marketing | Spec mínima de blocos (hero, currículo, prova social, garantia, FAQ, CTA) |
| O19 | Stripe p/ billing SaaS BR: justificativa do ADR-0010 desatualizada (Stripe BR tem boleto/Pix invite-only); 3 gateways p/ ~100 tenants; sem NFS-e | Billing SaaS | Reavaliar via novo ADR (Asaas/Pagar.me p/ billing doméstico é candidato forte) |
| O20 | SLA/suporte prometidos sem lastro (sem on-call/staffing/créditos; 99,9% dependente da Bunny "fora do nosso controle") | Comercial | SLO-alvo sem multa no MVP; créditos só Enterprise com carve-outs |
| O21 | Concentração de fornecedores sem plano B operacional (Bunny única p/ entrega; Pagar.me único p/ split) | Risco | Runbook de failover de vídeo; split equivalente no Asaas como contingência testada |
| O22 | Plano anual: cancelamento/pro-rata/fidelidade indefinidos; bloqueio de downgrade "até regularizar" = atrito jurídico | Comercial | Política declarada no checkout + Termos |
| O23 | Take rate decrescente zera no Enterprise (receita variável morre onde há GMV; DRM/SSO/SLA caros prometidos no tier) | Pricing | Piso > 0 ou mensalidade indexada; DRM como add-on com repasse |
| O24 | Verificação de certificado: rota divergente entre docs + resolução do tenant em rota pública pendente | Certificados | Unificar rota; definir resolução por `uuid_public` global ou por host |

---

## 3. 🟡 Registrar

Timezone do tenant inexistente (`tenant_settings.timezone` — drip/cupons/dashboards) · §6.0 sem
`courses.status`/`lessons.status` e sem `archived` em lessons (CHECK constraint vai mudar em F2) ·
Índices ausentes (`orders(provider_order_id)` unique, `subscriptions(provider_subscription_id)`,
`affiliate_clicks`, `notifications(user_id, read_at)`, `enrollments(course_id)`,
`certificates(enrollment_id)` unique) · Papel único × aluno-afiliado (OQ #20 conflita com
MONETIZATION §B.7) · Catálogo Postgres com 60+ tabelas × 100 schemas (medir migrations/pg_dump no CI)
· `lesson_embeddings vector(1536)` hardcoded · CSP nonce × SSG · Tenant `cancelled` "somente leitura"
sem desenho de quem autentica · `audit_log` append-only sem mecanismo · Aluno multi-tenant sem
copy/KB ("onde estão meus cursos?") · Captura de lead ausente no MVP (tenants ex-Hotmart esperam) ·
B2B seats ausente (registrar gap consciente) · Curso gratuito sem fluxo de matrícula desenhado ·
Verificação de e-mail × acesso pago já comprado (fricção) · Overage de banda: metering/billing só
esboçado · Cortesia × quota `max_students` · Tributação do afiliado PF (informe de rendimentos) ·
Funil de ativação irrealista (trial 14d × KYC) · Quotas de live sem números antes da F2 · Pixels sem
CMP = exposição ANPD no dia 1 · Cupom × base de cálculo do split na NF · Dashboard de unit economics
sem fonte de custo por tenant.

---

## 4. 🧹 Inconsistências entre documentos

Da auditoria sistemática (49 arquivos; ~400 links verificados, 1 quebrado):

**Comportamento (corrigir primeiro):**
- **A1** Tenant `suspended` — 4 docs, 3 comportamentos (ver G7/O17).
- **A2** Aula ao vivo — 4 docs ainda descrevem "embed Zoom/YouTube" revogado pelo ADR-0015
  (AUTHORING_UX §5.5, USER_FLOWS §7.3, LEARNING_EXPERIENCE_UX §4.6, IA).
- **A3** From dos e-mails — EMAIL_TEMPLATES ("nunca a nossa marca") × ONBOARDING §8 ("From da plataforma no MVP").
- **A4** Gabarito de quiz — dois campos concorrentes (`reveal_answers` × `feedback_policy`) com enums diferentes.
- **A5** Papéis SA — console define 5 papéis; vocabulário canônico tem 4 (`sa_readonly` órfão).
- **A6** Nota mínima no certificado MVP × `pass_score` F2 (ver O11).

**Vocabulário/status (vão direto para enums Zod errados):**
`enrollments` com `cancelled` inexistente (LEARNING §2.10, WIREFRAMES) · `subscriptions` com
`suspended` inexistente (USER_FLOWS §9.3) · `affiliates` com `rejected/suspended` inexistentes
(NOTIFICATIONS, EMAIL_TEMPLATES) · `orders` com `expired` informal virando estado (USER_FLOWS §11; e
máquina omite `chargeback→paid`) · `live_rescheduled` é evento, não estado · `video_status` sem
`none` em BUSINESS_RULES §3 · nomes errados de tabelas (`platform.plans` → `platform_plans` etc. em
3 docs) · glossário §2.3 desatualizado vs §6.0 · `users.status` sem vocabulário definido.

**Specs órfãs / integrações perdidas:**
- **C1 (grave):** os **toggles do AUTHORING_UX §8** (`completion_mode`, `is_optional`,
  `comments_enabled`, `reveal_answers`, `require_view`, `cert_requires_pass`) **nunca foram
  consolidados no DATA_MODEL** — e `LIVE_CLASSES §8` referencia `lessons.is_optional` que não existe.
- **C2/C3:** taxonomia de analytics **bifurcada** — LEARNING_EXPERIENCE_UX usa nomes diferentes do
  tracking plan canônico para os mesmos fatos (`lesson_play` × `video_played`...), e AUTHORING/
  ONBOARDING/suporte/import citam dezenas de eventos fora do catálogo (com colisões:
  `first_sale`×`tenant_first_sale`).
- **C4:** eventos de suporte (`support_ticket_*`) e import/export (`import_ready`...) sem linha na
  NOTIFICATIONS_MATRIX/EMAIL_TEMPLATES (e colidindo com `data_export_ready`/`data_deleted`).
- **C6:** RBAC_MATRIX sem seções de Suporte, Import/Export e papéis `sa_*` granulares.

**Fases contraditórias:** 2FA (4 posições) · quiz avançado (MVP×F2) · preferências de notificação
(resolvido como MVP, mas LEARNING ainda "decidir").

**Higiene:** dependências já resolvidas listadas como abertas em 5 docs · "≥95%" (OPEN_QUESTIONS/PRD/
ARCHITECTURE) × "~89%" (product/README) · ARCHITECTURE ainda "v1.0 Aprovado" após 3 rodadas de edição
(sem changelog) · link quebrado em WIREFRAMES:23 · artefatos de merge (PRD:74 "— áudio.", fences
órfãos em 2 docs) · README §5 cita "#10–#27" (lista vai a #40).

---

## 5. Plano de correção em ondas

### Onda 0 — Decisões de negócio/gates (stakeholder + externos; paralelo ao resto)
| Gate | Decisão | Dono |
|------|---------|------|
| 0.1 | **Bunny Volume Network no BR** (G1) — go/no-go do pricing | Comercial + Bunny |
| 0.2 | **Emissor fiscal** no MVP + posicionamento NF do tenant (G2) | Comercial/Contábil |
| 0.3 | **Parecer jurídico**: split/subcredenciamento (G3) + Controlador×Operador + retenção (G6) + vitalício/OQ#14 (G7) | Jurídico |
| 0.4 | **Escopo do MVP**: assinatura do aluno (G11) e auto-enroll (G15) — especificar ou rebaixar p/ F2 | Stakeholder |
| 0.5 | **Billing do SaaS**: reavaliar Stripe vs Asaas/Pagar.me (O19) | Stakeholder + ADR |

### Onda 1 — Spikes e ADRs estruturais (antes do scaffolding)
1. **ADR-0016 Topologia de hosts e sessão** (G12).
2. **Spike bloqueante: auth multitenant** — Better-Auth vs Plano B, com impersonação e auth do
   control plane como casos de teste (G8) → ADR-0017.
3. **ADR-0018 Roteamento financeiro** — `platform.payment_routing` + fluxo webhook→tenant (G9).
4. **Redesenho do padrão de webhooks por provedor** — Bunny sem HMAC: secret-in-URL + re-fetch;
   configuração de webhook na saga (G10).
5. **ADR-0019 Assinatura do aluno** (modelo de dados + fluxos) OU rebaixamento formal a F2 (G11).

### Onda 2 — Rodada de correção documental (consistência)
Corrigir em lote: A1–A6, vocabulário B1–B9, C1 (toggles → DATA_MODEL §6.19), C2–C4 (taxonomia única
de analytics + notificações de suporte/import), C6 (RBAC), D1–D3 (fases), E (pendências
dessincronizadas), F (versões/changelogs/links/artefatos) + regras novas: anti-seek por tempo de
mídia (G13), certificado snapshot + edição pós-venda (G14), reembolso self-service + CDC (G4 parte
produto), KYC como estado (G5), e os 🟠 documentais (O9–O18, O24).

### Onda 3 — Por módulo, durante o build
O1 (CI/CD), O2 (pg-boss), O3/O4 (progresso), O5 (matrículas), O6/O7 (live F2), O8 (recipients),
O20–O23 (comercial), e o backlog 🟡.

---

## 6. Impacto no nível de confiança

| Dimensão | Antes | Real (pós-auditoria) | Volta a subir quando |
|----------|-------|----------------------|----------------------|
| Arquitetura/isolamento | 90% | **82%** | Onda 1 (spike auth + topologia + webhooks) |
| Monetização/financeiro | 88% | **70%** | Onda 0 (gates 0.1–0.3) + G3/G4 especificados |
| Escopo do MVP | 95% | **80%** | Gate 0.4 (assinatura/auto-enroll) + Onda 2 |
| Consistência documental | 89% | **85%** | Onda 2 (correção em lote) |
| **Global** | ~89–95% | **~80%** | Após Ondas 0–2 ⇒ alvo ≥ 93% |

> A boa notícia: **nenhum achado invalida as decisões estruturais** (schema-per-tenant, Fastify,
> Drizzle, Bunny, Next.js, pg-boss). Os 🔴 são corrigíveis em documentação/decisão antes de qualquer
> linha de código — exatamente o propósito desta revisão.
