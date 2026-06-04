# Questões em Aberto / Itens a Validar

- **Versão:** 1.0 · **Data:** 2026-06-04

As decisões de produto e arquitetura estão fechadas em ≥95% de assertividade. Os itens abaixo **não
bloqueiam** o início da implementação, mas precisam ser validados/decididos durante a execução do MVP.

---

## 1. Validações operacionais (não bloqueiam design)

| # | Item | Por que importa | Ação |
|---|------|-----------------|------|
| 1 | **Custo de egress de vídeo no Brasil** | A rede Standard da Bunny cobra ~US$0.045/GB na América do Sul (~9x a Volume Network a US$0.005/GB). É o **maior risco financeiro** do projeto. | Contato comercial Bunny: confirmar cobertura/qualidade da **Volume Network** no BR. Definir cap de resolução (ex.: 720p padrão) + alertas de custo. |
| 2 | **PoC Better-Auth em schema-per-tenant** | Os exemplos da lib assumem DB único; precisamos validar auth + resolução de tenant + `search_path` ponta a ponta. | PoC no início da implementação (passo 2 do roadmap). Plano B: auth próprio Fastify (`@fastify/jwt` + `@fastify/cookie` + argon2). |
| 3 | **Estratégia de pooling PgBouncer** | `SET LOCAL search_path` em transaction pooling é o padrão de ouro; validar limites de pool × tenants ativos. | Definir `max` por pool e testar isolamento sob carga. |

---

## 2. Decisões de produto a refinar com dados (pós-MVP)

| # | Item | Default adotado | Quando revisar |
|---|------|-----------------|----------------|
| 4 | **Anti-pirataria** (você respondeu "não sei") | Faseado: MVP = Token + MediaCage Basic; F2 = watermark por aluno; DRM Enterprise sob demanda. Ver [ADR-0009](adr/0009-anti-piracy.md). | Se surgir cliente premium exigindo DRM "Hollywood-grade". |
| 5 | **Escopo de IA no MVP** | IA fica na **Fase 2** (transcrição + geração de quiz); MVP sem IA para focar no loop de receita. | Reavaliar se IA virar prioridade de marketing. |
| 6 | **Mobile** | **PWA primeiro** (F2); app nativo white-label só na F3. | Demanda de tenants por app branded nas lojas. |
| 7 | **Billing do SaaS** (como o tenant paga você) | Stripe Billing por plano (mensalidade fixa). | Se quiser modelo % sobre vendas (estilo Hotmart) no futuro. |
| 8 | **E-mail marketing** | Integrar terceiros (RD Station/ActiveCampaign) via event bus; não construir. | Depende da operação de marketing. |
| 9 | **Idiomas** | Lançar em **PT-BR**; arquitetura i18n pronta. | Expansão LATAM (ES/EN). |

---

## 2-bis. Decisões de produto pendentes do stakeholder (da coordenação dos docs de produto)

> Levantadas na reconciliação dos 8 docs de `docs/product/` (ver [README de produto](product/README.md)).
> A coordenação adotou **defaults** sensatos (registrados em ADR-0013/0014 e nas tabelas de `tenant_settings`/
> `platform_plans.limits`) para não bloquear a implementação; os itens abaixo precisam de confirmação comercial/jurídica.

| # | Item | Default adotado pela coordenação | Quem decide |
|---|------|----------------------------------|-------------|
| 10 | **Tabela de planos/quotas/preços do SaaS** (tiers, limites, features, `take_rate_bps`) | Estrutura definida (Starter/Pro/Scale/Enterprise em MONETIZATION); valores numéricos pendentes | Comercial |
| 11 | **Fee transacional sobre GMV** além da mensalidade? | Take rate operacionalizado via split (`take_rate_bps`); sem fee extra além disso | Comercial |
| 12 | **Grace period de inadimplência** (aluno e SaaS) | Aluno: `student_dunning_grace_days=7`; SaaS: dunning 7–14 dias antes de `suspended` | Comercial |
| 13 | **Reembolso parcial** (acesso e take rate) | MVP: não suspende proporcional; take rate estornado proporcional; reembolso parcial fora do MVP | Produto/Financeiro |
| 14 | **Tenant inadimplente × acesso dos alunos** | `grace`: alunos seguem; `suspended`: avaliar manter alunos pagantes ativos (recomendado), bloquear só painel/novas vendas | Produto/Jurídico |
| 15 | **Revogação de certificado em reembolso** | `tenant_settings.refund_revokes_certificate=true` (configurável por tenant) | Produto |
| 16 | **Clearance de comissão de afiliado** | `affiliate_clearance_days=14` (alinhar à janela de garantia/chargeback) | Produto |
| 17 | **Janela de cookie / atribuição de afiliado** | Last-click, `cookie_window_days=30`, aprovação `manual`, auto-indicação bloqueada | Produto |
| 18 | **Verificação de e-mail obrigatória no cadastro** | Recomendado obrigatória; conteúdo gratuito acessível antes da verificação | Produto |
| 19 | **Escopo de 2FA** | Obrigatório Super-Admin (MVP) e Owner do tenant; recomendado Admin; opcional aluno; equipe/aluno full = F2 | Produto/Segurança |
| 20 | **Papéis múltiplos** (`users.role` único) | MVP: papel único por usuário no tenant; papéis compostos fora do MVP | Produto |
| 21 | **Delegação configurável de permissões** (owner→admin/instrutor) | MVP: papéis fixos; flags de permissão por tenant = pós-MVP | Produto |
| 22 | **Visibilidade financeira ao instrutor** | Engajamento sim; financeiro global não; por curso a confirmar | Produto |
| 23 | **Hosts/prefixos** (`admin.app.com`, `/app`, `/manage`, `/affiliate`) | **Proposta a validar com engenharia** (DNS/cookies/middleware) | Engenharia |
| 24 | **Política de retenção LGPD** (prazos por categoria + janela win-back antes do `DROP SCHEMA`) | Requisito definido; **números pendentes de validação jurídica** | Jurídico |
| 25 | **Stack de consentimento (CMP)** | CMP próprio vs terceiro — pendente; gating de PostHog/Meta/GA4 depende disso | Produto/Eng |
| 26 | **UX de Autoria/Instrutor** (toggles: conclusão manual, ligar/desligar comentários, gabarito, aulas opcionais, política de tentativas) | ✅ Resolvido — [docs/product/AUTHORING_UX.md](product/AUTHORING_UX.md) (toggles §8; campos a consolidar no DATA_MODEL §6) | Produto/UX |
| 27 | **Central de preferências de notificação no MVP** | Adotado no MVP (in-app + e-mail; push F2) — modelado em `notification_preferences` | Produto |
| 28 | **Aulas ao vivo: fase e valores de quota** ([LIVE_CLASSES.md](product/LIVE_CLASSES.md), [ADR-0015](adr/0015-live-classes-interactive.md)) | Confirmado **F2**; estrutura de quota em `platform_plans.limits` (`live_concurrent_participants`, `live_hours_month`, feature `live_classes`); **valores numéricos pendentes** — alinhar com #10 | Comercial |
| 29 | **LGPD da gravação de live** (consentimento explícito de captura de áudio/vídeo do aluno + base legal; banner "esta aula está sendo gravada") + **deleção** da gravação (R2/Bunny + linhas `live_*`) no direito ao esquecimento | Requisito definido (banner de gravação visível a todos); base legal/prazos pendentes — alinhar com #24 | Jurídico |
| 30 | **Region pinning / residência de dados BR do LiveKit** | A validar com o provedor (PoP/região no BR; self-host regional é o plano de contingência — ADR-0015 Riscos) | Engenharia/Provedor |
| 31 | **Calibração de custo de live** (participante-minuto + recording-minuto + egress bandwidth do provedor vs margem) e **política de overage** | Quotas duras de concorrência + soft cap de horas/mês + alertas; calibrar contra custo real do provedor (mesmo risco do egress Bunny, #1) | Comercial/FinOps |

---

## 3. Itens explicitamente fora de escopo (decididos)

SCORM/xAPI, SSO/SAML, LTI, marketplace cross-tenant, multi-região, produção de vídeo in-house e
certificação acadêmica credenciada estão em **Fase 3 ou Won't** (ver [ROADMAP.md](ROADMAP.md)).

---

> Atualize este arquivo conforme os itens forem resolvidos, promovendo decisões relevantes a ADRs.
