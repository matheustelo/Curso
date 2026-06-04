# Jornadas de Usuário — Plataforma de Cursos Online (SaaS Multitenant)

- **Versão:** 1.0
- **Data:** 2026-06-04
- **Status:** Para revisão do coordenador de produto
- **Autor:** UX Research / Journey Strategy
- **Documentos de referência:** [PRD.md](../PRD.md) · [ROADMAP.md](../ROADMAP.md) · [DATA_MODEL.md](../DATA_MODEL.md) · [ARCHITECTURE.md](../ARCHITECTURE.md) · [CLAUDE.md](../../CLAUDE.md)

> Este documento descreve as **jornadas de ponta a ponta** das 5 personas do produto. É a fonte de
> verdade de **experiência/UX** e alimenta a definição de flows, telas, notificações, RBAC, pricing e
> analytics. As fases e features citadas seguem a priorização do roadmap: **[MVP]**, **[F2]**, **[F3]**.
> Quando uma oportunidade depende de feature ainda não priorizada, ela é marcada explicitamente.

---

## Índice

1. [Conceitos e convenções](#1-conceitos-e-convenções)
2. [Mapa de superfícies (telas/produtos)](#2-mapa-de-superfícies-telasprodutos)
3. [Persona 1 — Super-Admin (plataforma)](#3-persona-1--super-admin-plataforma)
4. [Persona 2 — Admin do Tenant (dono da escola)](#4-persona-2--admin-do-tenant-dono-da-escola)
5. [Persona 3 — Instrutor](#5-persona-3--instrutor)
6. [Persona 4 — Afiliado](#6-persona-4--afiliado)
7. [Persona 5 — Aluno](#7-persona-5--aluno)
8. [Matriz de momentos críticos por persona](#8-matriz-de-momentos-críticos-por-persona)
9. [Loops de engajamento e gatilhos cruzados](#9-loops-de-engajamento-e-gatilhos-cruzados)
10. [Dependências e pontos para o coordenador](#dependências-e-pontos-para-o-coordenador)

---

## 1. Conceitos e convenções

- **Tenant:** ambiente isolado (schema-per-tenant) de uma escola/produtor. Acessado por subdomínio
  (`acme.app.com`) e, na F2, por domínio próprio.
- **Control plane (`platform`):** onde o Super-Admin opera (gestão de tenants, planos do SaaS, billing
  Stripe). Não tem acesso implícito a dados de tenant — impersonação é auditada.
- **Data plane (`tenant_<slug>`):** onde vivem Admin do Tenant, Instrutor, Afiliado e Aluno daquele
  tenant.
- **Duas economias distintas:**
  - **B2B (SaaS):** o tenant paga a plataforma (Stripe Billing). Persona: Admin do Tenant ↔ Super-Admin.
  - **B2C (checkout do tenant):** o aluno paga o tenant (Pagar.me/Asaas, Pix/boleto/cartão, afiliados,
    split). Personas: Aluno ↔ Afiliado ↔ Admin/Instrutor.
- **Touchpoints (pontos de contato):** `e-mail` (Resend), `in-app` (banner/toast/modal/empty-state/
  checklist), `push` (Web Push/PWA — F2), `público` (landing/catálogo/verificação de certificado).
- **"Aha moment":** o instante em que a persona percebe o valor central pela primeira vez. Marcado em
  cada jornada.

### Convenção das tabelas de journey map

Cada persona tem um **journey map** com as colunas: **Fase | Ações | Touchpoints | Dores | Oportunidades
| Métrica**. As métricas referem-se às definições do PRD §6 (North Star = horas assistidas por aluno
ativo/mês) e a métricas de suporte específicas de cada persona.

---

## 2. Mapa de superfícies (telas/produtos)

Para ancorar os passos das jornadas, estas são as superfícies que aparecem ao longo do documento
(detalhamento fino fica para o doc de telas/flows do coordenador):

| Superfície | Quem usa | Exemplos de telas |
|---|---|---|
| **Console Super-Admin** (`platform`) | Super-Admin | Lista de tenants, detalhe do tenant, planos/quotas, faturas do SaaS, impersonação, audit log, status de provisionamento, observabilidade global |
| **Admin do Tenant** (área logada do tenant) | Owner/Admin | Dashboard, branding, domínio, equipe/RBAC, cursos, preços/cupons, afiliados, financeiro, relatórios, configurações, LGPD |
| **Studio de conteúdo** | Instrutor/Admin | Editor de curso (módulos/aulas, drag-and-drop), uploader de vídeo (TUS), quiz builder, drip, publicação, comentários/dúvidas |
| **Painel do Afiliado** | Afiliado | Links/UTM, materiais de divulgação, comissões, extrato/saque |
| **Landing/Catálogo público** | Aluno/Afiliado/visitante | Página de vendas por curso, catálogo, checkout |
| **Área do Aluno (LMS)** | Aluno | Meus cursos, player, progresso, quiz, certificado, comentários, perfil |
| **Página pública de verificação** | Qualquer um | Validação de certificado (UUID/QR/hash) |
| **E-mails transacionais** (Resend) | Todas | Boas-vindas, confirmação de compra, vídeo pronto, conclusão, cobrança, comissão |

---

## 3. Persona 1 — Super-Admin (plataforma)

**Quem é:** operador interno da plataforma SaaS (nós). Vive no schema `platform`. Objetivo macro:
**manter tenants saudáveis, provisionados, pagando e sem incidentes de isolamento.**

**Job principal:** "Quero que cada escola entre, seja provisionada sem fricção, pague em dia e, se algo
quebrar, eu consiga diagnosticar e agir (inclusive impersonar) com rastro de auditoria."

### Fase A — Aquisição/Provisionamento (entra um novo tenant)

- **JTBD:** transformar uma assinatura aprovada (Stripe) em um ambiente funcional e isolado, de forma
  idempotente e observável.
- **Passos concretos:**
  1. Recebe sinal de **assinatura criada** no Stripe Billing → `platform.platform_subscriptions`.
  2. A saga de provisionamento dispara (worker pg-boss): `INSERT tenants(status=provisioning)` →
     `CREATE SCHEMA` → migrations → seed (papéis + admin + curso exemplo) → cria **Bunny Video Library**
     → registra roteamento → `status=active` → smoke test.
  3. No **Console Super-Admin**, acompanha a tela de **status de provisionamento** (`provisioning_jobs`:
     step, attempts, last_error).
  4. Em caso de falha, faz **retry idempotente** de um step específico ou aciona runbook.
- **Touchpoints:** in-app (timeline da saga, badges de step); e-mail/alerta interno em falha; Sentry/
  OTel para erros.
- **Emoções/atritos:** ansiedade se a saga travar em um step (ex.: Bunny library); alívio quando o
  smoke test passa. **Atrito:** falta de visibilidade de "onde travou".
- **Oportunidade de encantamento:** timeline visual da saga com **retry por step** e cópia do
  `idempotency_key`; botão "rodar smoke test de novo".
- **Métrica:** **tempo de provisionamento** (assinatura → `active`); taxa de provisionamento sem
  intervenção manual; nº de retries por tenant.
- **Momento crítico:** **smoke test verde** = tenant pronto para o admin entrar.

### Fase B — Onboarding operacional (configura planos/quotas)

- **JTBD:** garantir que existam planos do SaaS coerentes e quotas aplicadas por tier.
- **Passos:** cria/edita `platform_plans` (price_cents, limits jsonb: `max_students`, `storage_gb`,
  `features[]`); associa tenant ao plano; valida que as quotas aparecem no Admin do Tenant.
- **Touchpoints:** in-app (CRUD de planos), e-mail só em mudança de tier de um tenant.
- **Dores:** risco de editar limits e impactar tenants ativos sem aviso.
- **Oportunidade:** **preview de impacto** ("X tenants neste plano, Y acima do novo limite").
- **Métrica:** distribuição de tenants por plano; nº de tenants acima de quota.

### Fase C — Uso recorrente (operação e suporte)

- **JTBD:** monitorar saúde, dar suporte sem vazar dados, agir sobre inadimplência.
- **Passos:**
  1. Dashboard global: tenants ativos/suspensos, MRR do SaaS, faturas em atraso, alertas de custo de
     mídia (egress Bunny), incidentes.
  2. **Impersonação auditada:** ao receber um ticket, abre o tenant e impersona um usuário para
     reproduzir o problema — toda ação grava em `audit_log` (`actor_type`, `action`, `metadata`).
  3. **Reconciliação:** acompanha jobs de reconciliação pagamento↔acesso (divergências).
- **Touchpoints:** in-app (dashboard, audit log), alerta (Sentry/Slack interno), e-mail de cobrança
  automática a tenants inadimplentes.
- **Dores:** medo de **vazamento cross-tenant** (o bug mais grave); precisar de dados sem rastro.
- **Oportunidade:** **banner de impersonação sempre visível** ("Você está agindo como X no tenant Y —
  sair") + tudo logado; meta de incidentes de vazamento = **0**.
- **Métrica:** nº de impersonações; tempo de resolução de ticket; incidentes de isolamento (meta 0);
  faturas em atraso.

### Fase D — Crescimento (expansão de tenants)

- **JTBD:** ajudar tenants a subir de tier conforme crescem (mais alunos/storage).
- **Passos:** detecta tenants encostando na quota → sinaliza upsell (manual no MVP; automação na F2);
  acompanha conversão de upgrade.
- **Touchpoints:** in-app (flag "próximo do limite"); e-mail ao admin do tenant (oferta de upgrade).
- **Oportunidade:** **alerta proativo de quota** com CTA de upgrade self-service.
- **Métrica:** taxa de upgrade de tier; expansion MRR.

### Fase E — Retenção/Churn (suspensão, cancelamento, offboarding)

- **JTBD:** lidar com inadimplência e saída de tenant respeitando LGPD e auditoria.
- **Passos:**
  1. Inadimplência → após política de dunning, `status=suspended` (suspende o tenant; alunos perdem
     acesso conforme regra).
  2. Cancelamento confirmado → offboarding: **exportação de dados** disponibilizada ao tenant →
     `DROP SCHEMA tenant_<slug> CASCADE` (com janela de retenção).
  3. Registro completo em `audit_log`.
- **Touchpoints:** e-mail (avisos de suspensão/cancelamento), in-app (banner de conta suspensa).
- **Dores:** equilibrar cobrança e relação; risco de deletar dados cedo demais.
- **Oportunidade:** **win-back** antes do DROP; checklist de offboarding com export garantido (anti
  lock-in, conforme PRD §7).
- **Métrica:** churn de tenants; recuperação de inadimplentes (dunning recovery); tempo entre
  cancelamento e DROP.

### Journey map — Super-Admin

| Fase | Ações | Touchpoints | Dores | Oportunidades | Métrica |
|---|---|---|---|---|---|
| Provisionamento | Acompanha saga, retry por step, smoke test | In-app (timeline), alerta em falha, Sentry | Travar sem saber onde | Timeline visual + retry idempotente | Tempo p/ `active`; % sem intervenção |
| Onboarding op. | CRUD planos/quotas, associa tier | In-app | Editar limits e quebrar tenants | Preview de impacto da mudança | Tenants por plano; acima de quota |
| Uso recorrente | Dashboard global, impersonação auditada, reconciliação | In-app, alerta, e-mail de cobrança | Vazamento cross-tenant; agir sem rastro | Banner de impersonação + audit log total | Incidentes de isolamento (0); MTTR ticket |
| Crescimento | Detecta limite, sinaliza upsell | In-app, e-mail ao admin | Upsell manual e tardio | Alerta proativo de quota + upgrade self-service | Taxa de upgrade; expansion MRR |
| Retenção/Churn | Dunning, suspende, offboarding, DROP | E-mail, in-app | Cobrar sem perder relação; deletar cedo | Win-back + export garantido | Churn de tenants; dunning recovery |

---

## 4. Persona 2 — Admin do Tenant (dono da escola)

**Quem é:** cliente que assina a plataforma para montar sua escola/infoproduto. Persona central da
ativação B2B. Vive no schema do tenant com papel `owner`/`admin`.

**Job principal:** "Quero colocar minha escola no ar com a minha marca, publicar meu primeiro curso e
fazer a primeira venda o mais rápido possível — sem depender de TI."

> **Foco especial (solicitado):** assinar SaaS → configurar marca → criar 1º curso → 1ª venda.

### Fase A — Descoberta e decisão (pré-tenant)

- **JTBD:** avaliar se a plataforma resolve "LMS profundo + checkout BR + white-label" melhor que
  Hotmart/Kiwify/Teachable.
- **Passos:** chega via site institucional/indicação → compara features (Pix/boleto, afiliados/split,
  marca própria, vídeo protegido) → escolhe um **plano do SaaS** → assina via Stripe Billing → recebe
  e-mail de boas-vindas com link do subdomínio.
- **Touchpoints:** público (site de marketing — fora do escopo deste produto), e-mail de boas-vindas
  pós-assinatura.
- **Emoções/atritos:** ceticismo ("é mais um genérico?"), esperança de marca própria de verdade.
- **Oportunidade:** posicionamento claro do "espaço em branco" (PRD §1.2) já na assinatura; trial
  guiado.
- **Métrica:** conversão de trial → pago do SaaS (PRD §6 — Aquisição).
- **Momento crítico:** **assinatura aprovada** → dispara provisionamento (ver Super-Admin Fase A).

### Fase B — Onboarding/Ativação (configurar marca + 1º curso + 1ª venda)

- **JTBD:** sair do zero a "loja vendendo" no menor tempo possível. **Esta é a fase mais importante da
  retenção B2B.**
- **Passos concretos (checklist de ativação in-app):**
  1. **Primeiro login** no subdomínio (`acme.app.com/admin`) — credenciais semeadas no provisionamento.
  2. **Configurar marca** [MVP]: upload de logo, cores (CSS variables por tenant), confirmar subdomínio.
     (Domínio próprio com SSL é **[F2]**.)
  3. **Criar 1º curso** [MVP]: título/descrição/capa → módulos/aulas (vídeo Bunny / texto / PDF) →
     definir **preço** (one_time/subscription/free) e moeda → publicar. Há um **curso de exemplo**
     semeado no provisionamento para servir de modelo.
  4. **Configurar checkout** [MVP]: conectar conta Pagar.me/Asaas (split/afiliados), criar cupom de
     lançamento.
  5. **Convidar equipe** [MVP]: adicionar instrutor(es) e (opcional) afiliados, via RBAC.
  6. **Publicar landing** do curso [MVP]: página de vendas com SEO básico + pixels (Meta/GA4) + UTM.
  7. **Primeira venda:** divulga → aluno compra → webhook aprova → matrícula automática.
- **Touchpoints:** in-app (checklist de onboarding com progresso, empty-states que ensinam, tooltips);
  e-mail (boas-vindas, "seu curso foi publicado", "primeira venda!"); push (F2).
- **Emoções/atritos:** sobrecarga inicial ("muita coisa pra configurar"); fricção ao conectar gateway
  de pagamento (KYC); insegurança com preço.
- **Oportunidade de encantamento:** **checklist gamificado de ativação** ("3 de 5 para vender"); preview
  ao vivo da marca; e-mail celebrando a **1ª venda** com confete in-app.
- **Métrica:** **ativação** = % de tenants que publicam ≥1 curso **e** realizam ≥1 venda nos primeiros
  14 dias (PRD §6); tempo até 1º curso publicado; tempo até 1ª venda.
- **Momentos críticos:**
  - **Aha #1:** ver a **escola com a própria marca** no ar (white-label real).
  - **Aha #2 / ativação plena:** **primeira venda** caindo + aluno matriculado automaticamente.

### Fase C — Uso recorrente (operar a escola)

- **JTBD:** manter conteúdo, acompanhar vendas e engajamento, atender alunos, gerir afiliados.
- **Passos:** acompanha **dashboard** (vendas, receita por curso, conclusão); gerencia **cupons** e
  campanhas; aprova/gerencia **afiliados** e comissões; revisa **financeiro** (orders, split,
  reembolsos); acompanha **máquina de estados de acesso** (ativo/suspenso/reembolsado).
- **Touchpoints:** in-app (dashboard, financeiro, afiliados); e-mail (resumo de vendas, alertas de
  reembolso/chargeback); webhooks de saída para CRM próprio do tenant.
- **Dores:** conciliar pagamento↔acesso; entender por que um aluno perdeu acesso; relatórios rasos.
- **Oportunidade:** **central de financeiro clara** (status de cada pedido + split + comissão);
  explicação humana de cada estado de acesso.
- **Métrica:** GMV do tenant; receita por curso; taxa de reembolso/chargeback; nº de afiliados ativos.

### Fase D — Crescimento (escalar a operação)

- **JTBD:** vender mais por aluno e por funil, profissionalizar a operação.
- **Passos:** liga **order bump/upsell/downsell** e **bundles** [F2]; ativa **recuperação de carrinho**
  [F2]; expande **programa de afiliados**; usa **co-produção/split entre produtores** [F2]; abre
  **API/domínio próprio** [F2]; aproveita **gamificação/comunidade** [F2] para engajar.
- **Touchpoints:** in-app (novas features sinalizadas), e-mail (release notes/oportunidades).
- **Dores:** complexidade de funil; medo de canibalizar margem com comissões.
- **Oportunidade:** **simulador de margem** (preço − gateway − split − comissão); templates de funil.
- **Métrica:** ticket médio; conversão de checkout; receita por funil; expansion (upgrade de plano SaaS).

### Fase E — Retenção/Churn (renovação e risco de saída)

- **JTBD:** ver ROI contínuo da plataforma; decidir renovar o plano do SaaS.
- **Passos:** acompanha métricas de negócio (MRR/churn dos seus alunos [F2]); gerencia LGPD
  (export/deleção de alunos); em risco, considera migrar — **exportação nativa** reduz lock-in.
- **Touchpoints:** e-mail (lembrete de renovação, resumo de valor entregue), in-app (relatório de
  impacto), Super-Admin aciona win-back se inadimplente.
- **Dores:** sensação de não usar tudo que paga; medo de aprisionamento.
- **Oportunidade:** **"relatório de valor"** trimestral (alunos ativos, horas assistidas, GMV gerado);
  export sempre disponível como sinal de confiança.
- **Métrica:** churn de tenants; renovação de plano; NPS do admin.

### Journey map — Admin do Tenant

| Fase | Ações | Touchpoints | Dores | Oportunidades | Métrica |
|---|---|---|---|---|---|
| Descoberta/Decisão | Compara, escolhe plano, assina (Stripe) | Site, e-mail boas-vindas | "É mais um genérico?" | Posicionar o espaço em branco; trial guiado | Conversão trial→pago SaaS |
| Onboarding/Ativação | Marca → 1º curso → checkout → 1ª venda | In-app checklist, e-mail, push (F2) | Sobrecarga; KYC gateway; medo de precificar | Checklist gamificado; preview de marca; confete na 1ª venda | % ativados em 14d; tempo até 1ª venda |
| Uso recorrente | Dashboard, cupons, afiliados, financeiro | In-app, e-mail, webhooks saída | Conciliar pagamento↔acesso; relatórios rasos | Central financeira clara; estados de acesso explicados | GMV; receita/curso; reembolso |
| Crescimento | Order bump/upsell/bundles, co-produção, API | In-app, e-mail | Complexidade de funil; margem | Simulador de margem; templates de funil | Ticket médio; conversão; expansion |
| Retenção/Churn | Renova, LGPD, avalia ROI | E-mail, in-app, win-back | Subutilização; lock-in | Relatório de valor; export sempre disponível | Churn; renovação; NPS |

---

## 5. Persona 3 — Instrutor

**Quem é:** quem cria e mantém o conteúdo. Papel `instructor` no tenant. Pode ser o próprio dono
(escola individual) ou contratado.

**Job principal:** "Quero transformar meu conhecimento em aulas publicadas, com vídeo que sobe sem dor,
e acompanhar se os alunos estão aprendendo."

### Fase A — Onboarding (primeiro acesso ao Studio)

- **JTBD:** entender rapidamente como montar um curso.
- **Passos:** recebe convite do admin → primeiro login → abre o **Studio de conteúdo** → vê o curso de
  exemplo semeado → cria a estrutura Curso → Módulo → Aula (drag-and-drop).
- **Touchpoints:** e-mail (convite), in-app (tour do editor, empty-states).
- **Emoções/atritos:** insegurança com ferramenta nova; medo de "vídeo não subir".
- **Oportunidade:** template de curso e estrutura sugerida; tour curto contextual.
- **Métrica:** tempo até 1ª aula criada.

### Fase B — Ativação (publicar a 1ª aula em vídeo)

- **JTBD:** publicar conteúdo de qualidade sem fricção técnica.
- **Passos concretos:**
  1. Cria aula tipo **vídeo** → inicia upload → backend cria o vídeo na **library do tenant** e gera
     assinatura → **upload direto ao Bunny via TUS (resumable)**.
  2. Encoding automático → **webhook "vídeo pronto"** (HMAC) → status atualizado.
  3. Adiciona **texto rico/PDF**, configura **drip** (data fixa ou dias após matrícula), monta **quiz**
     (múltipla escolha / V-F).
  4. **Publica** a aula (sai de draft).
- **Touchpoints:** in-app (barra de progresso de upload, status "processando"→"pronto"); e-mail/in-app
  ("seu vídeo está pronto para publicar").
- **Emoções/atritos:** ansiedade durante encoding ("travou?"); frustração se upload cai (mitigado por
  TUS resumable).
- **Oportunidade de encantamento:** **upload resiliente** com retomada; notificação "vídeo pronto";
  preview do player do aluno antes de publicar.
- **Métrica:** taxa de sucesso de upload; tempo upload→publicado; nº de aulas publicadas.
- **Momento crítico (Aha):** **vídeo pronto e publicado**, visível como o aluno verá.

### Fase C — Uso recorrente (manter e ensinar)

- **JTBD:** manter o curso vivo e atender dúvidas.
- **Passos:** adiciona/edita aulas; responde **comentários por aula** [MVP]; acompanha **progresso dos
  alunos** e **conclusão**; ajusta drip; (F2) usa **transcrição/legenda automática** e **geração de
  quiz por IA**, fórum/lives, gradebook.
- **Touchpoints:** in-app (inbox de dúvidas, dashboard de progresso); e-mail (resumo de novas dúvidas).
- **Dores:** dúvidas espalhadas; não saber onde alunos travam (drop-off).
- **Oportunidade:** **inbox unificado de dúvidas** por aula; sinal de **drop-off** por aula [F2].
- **Métrica:** tempo de resposta a dúvidas; engajamento por aula; North Star (horas assistidas).

### Fase D — Crescimento (melhorar e expandir conteúdo)

- **JTBD:** elevar qualidade e quantidade, reaproveitar mídia.
- **Passos:** usa **biblioteca de mídia reutilizável** [F2]; cria **provas** com tentativas/nota/tempo e
  **gradebook** [F2]; adota **gamificação** [F2]; estrutura **pré-requisitos/bloqueio sequencial** [F2];
  versiona conteúdo [F3].
- **Touchpoints:** in-app, e-mail.
- **Oportunidade:** insights de quais aulas geram conclusão x abandono; IA sugerindo quiz da transcrição.
- **Métrica:** taxa de conclusão por curso; nota média; reuso de mídia.

### Fase E — Retenção (instrutor engajado)

- **JTBD:** sentir que o conteúdo gera resultado (alunos concluindo, vendas).
- **Passos:** acompanha conclusões/certificados emitidos; vê impacto nas vendas (se for co-produtor com
  split [F2]).
- **Touchpoints:** in-app (mural de conquistas dos alunos), e-mail (resumo mensal).
- **Oportunidade:** **resumo de impacto** ("X alunos concluíram, Y certificados emitidos este mês").
- **Métrica:** certificados emitidos; recorrência de criação de conteúdo.

### Journey map — Instrutor

| Fase | Ações | Touchpoints | Dores | Oportunidades | Métrica |
|---|---|---|---|---|---|
| Onboarding | Convite, tour do Studio, estrutura curso | E-mail convite, in-app tour | Ferramenta nova; medo do upload | Template + tour contextual | Tempo até 1ª aula |
| Ativação | Upload TUS, encoding, drip, quiz, publicar | In-app progresso/status, e-mail "vídeo pronto" | Ansiedade no encoding; queda de upload | Upload resumable; preview do player | Sucesso de upload; upload→publicado |
| Uso recorrente | Editar aulas, responder dúvidas, ver progresso | In-app inbox/dashboard, e-mail | Dúvidas dispersas; não ver drop-off | Inbox unificado; sinal de drop-off (F2) | Tempo de resposta; horas assistidas |
| Crescimento | Provas/gradebook, gamificação, pré-requisitos, IA | In-app, e-mail | Esforço de melhorar conteúdo | Insights de conclusão; quiz por IA | Conclusão/curso; reuso de mídia |
| Retenção | Acompanha conclusões/certificados/impacto | In-app, e-mail | Não enxergar resultado | Resumo de impacto mensal | Certificados; recorrência de criação |

---

## 6. Persona 4 — Afiliado

**Quem é:** promove cursos do tenant por comissão. Papel `affiliate` no tenant. No MVP, afiliação é
**dentro de um tenant** (marketplace cross-tenant é [F3]).

**Job principal:** "Quero pegar um link, divulgar para minha audiência e ver minha comissão entrando de
forma transparente e confiável."

### Fase A — Descoberta/Adesão

- **JTBD:** entrar no programa de afiliados de um tenant.
- **Passos:** recebe convite/aprovação do admin → cria conta `affiliate` no tenant → acessa o **Painel
  do Afiliado**.
- **Touchpoints:** e-mail (convite/aprovação), in-app (boas-vindas ao programa).
- **Emoções/atritos:** dúvida sobre comissão (% / regras / quando paga); desconfiança de tracking.
- **Oportunidade:** termos de comissão claros no onboarding (commission_pct, prazo de pagamento).
- **Métrica:** nº de afiliados ativos; tempo até 1º link gerado.

### Fase B — Ativação (1ª divulgação → 1ª comissão)

- **JTBD:** gerar o primeiro link e a primeira venda atribuída.
- **Passos concretos:**
  1. Gera **link de afiliado** (code único) com **UTM** para um curso.
  2. Baixa **materiais de divulgação** (criativos/copys) — biblioteca do tenant.
  3. Divulga → aluno compra com atribuição → `orders.affiliate_id` → gera
     `affiliate_commissions(status=pending)`.
  4. Acompanha no painel a comissão pendente.
- **Touchpoints:** in-app (gerador de link, materiais, painel de comissões); e-mail/push (F2) "você fez
  uma venda!".
- **Emoções/atritos:** ansiedade de atribuição ("vai contar como minha?"); empolgação na 1ª comissão.
- **Oportunidade de encantamento:** **notificação imediata de venda atribuída**; explicação clara do
  status `pending → paid → reversed`.
- **Métrica:** cliques→conversão por link; tempo até 1ª comissão; nº de links ativos.
- **Momento crítico (Aha):** **primeira comissão registrada** com atribuição correta.

### Fase C — Uso recorrente (vender mais)

- **JTBD:** otimizar divulgação e acompanhar ganhos.
- **Passos:** vê desempenho por link/curso; pega novos materiais; acompanha extrato; recebe pagamentos
  (split via Pagar.me).
- **Touchpoints:** in-app (dashboard de comissões/extrato), e-mail (resumo, "comissão paga").
- **Dores:** falta de dados de funil (só vê venda, não o caminho); atraso/opacidade de pagamento.
- **Oportunidade:** **transparência total** do extrato (pendente/pago/revertido + datas); ranking de
  afiliados [F2/gamificação].
- **Métrica:** comissão total; taxa de reversão (refund/chargeback); recorrência de divulgação.

### Fase D — Crescimento (afiliado de alto volume)

- **JTBD:** escalar com mais cursos/tenants e melhores condições.
- **Passos:** promove **bundles/upsell** [F2]; negocia comissões; (F3) descobre cursos via
  **marketplace cross-tenant**.
- **Touchpoints:** in-app, e-mail.
- **Oportunidade:** tiers de comissão por performance; deep-links para funis com order bump.
- **Métrica:** GMV atribuído; nº de cursos promovidos.

### Fase E — Retenção/Churn

- **JTBD:** continuar valendo a pena divulgar este tenant.
- **Passos:** avalia consistência de pagamento e conversão; pode parar se reversões altas ou pagamento
  opaco.
- **Touchpoints:** e-mail (resumo de ganhos), in-app.
- **Dores:** reversões inesperadas; sensação de injustiça na atribuição.
- **Oportunidade:** política clara de reversão; **alerta proativo** quando uma comissão é revertida e
  por quê.
- **Métrica:** retenção de afiliados; churn de afiliados; LTV do afiliado para o tenant.

### Journey map — Afiliado

| Fase | Ações | Touchpoints | Dores | Oportunidades | Métrica |
|---|---|---|---|---|---|
| Descoberta/Adesão | Convite, cria conta, entra no painel | E-mail, in-app | Regras de comissão obscuras | Termos claros no onboarding | Afiliados ativos; tempo até 1º link |
| Ativação | Gera link+UTM, pega materiais, 1ª venda | In-app, e-mail/push (F2) | Medo de atribuição; opacidade | Notificação de venda; status explicado | Conversão por link; tempo até 1ª comissão |
| Uso recorrente | Acompanha extrato, novos materiais, recebe | In-app, e-mail | Falta de funil; atraso de pagamento | Extrato transparente; ranking (F2) | Comissão total; taxa de reversão |
| Crescimento | Promove bundles/upsell, negocia, marketplace (F3) | In-app, e-mail | Limite de cursos/condições | Tiers por performance; deep-links | GMV atribuído; cursos promovidos |
| Retenção/Churn | Avalia consistência, decide continuar | E-mail, in-app | Reversões; injustiça percebida | Política clara + alerta de reversão | Retenção/churn de afiliados |

---

## 7. Persona 5 — Aluno

**Quem é:** consumidor final do conteúdo. Papel `student` no tenant. **Persona de maior volume e
diretamente ligada à North Star** (horas assistidas por aluno ativo/mês).

**Job principal:** "Quero encontrar um curso que resolve meu problema, comprar de forma simples e
brasileira (Pix/cartão/boleto), assistir sem travar, progredir, concluir e ter meu certificado."

> **Foco especial (solicitado):** descobrir curso → comprar → assistir → progredir → concluir →
> certificado → recompra.

### Fase A — Descoberta

- **JTBD:** descobrir um curso confiável que resolva uma dor específica.
- **Passos:** chega via **landing do curso** (anúncio/UTM/afiliado/orgânico) → lê benefícios, currículo,
  prova social → decide.
- **Touchpoints:** público (landing com SEO + pixels Meta/GA4), e-mail (se já é lead — F2).
- **Emoções/atritos:** desconfiança ("vale o preço?"); excesso de informação; dúvida sobre acesso.
- **Oportunidade:** landing de alta conversão com prova social, preview de aula gratuita, garantia.
- **Métrica:** visitantes → checkout (topo do funil); conversão da landing.
- **Momento crítico:** clicar em **"Comprar/Matricular"**.

### Fase B — Compra (checkout BR)

- **JTBD:** pagar com o mínimo de fricção, no método que prefere.
- **Passos concretos:**
  1. **Checkout próprio** [MVP]: escolhe **Pix / boleto / cartão** (parcelamento); aplica **cupom**.
  2. (F2) **order bump** no checkout e **upsell/downsell one-click** pós-compra.
  3. Paga → gateway processa → **webhook de pagamento aprovado** (idempotente) → **matrícula criada** →
     acesso liberado (máquina de estados: `active`).
  4. Pix: confirmação quase imediata; boleto: aguarda compensação (estado `pending`).
- **Touchpoints:** público (checkout); e-mail (confirmação de compra + acesso); push (F2).
- **Emoções/atritos:** ansiedade no boleto ("já liberou?"); medo de erro de cartão; abandono de carrinho.
- **Oportunidade de encantamento:** **liberação instantânea no Pix/cartão** com e-mail imediato; status
  claro para boleto pendente; **recuperação de carrinho** [F2].
- **Métrica:** conversão de checkout; abandono; share por método de pagamento; tempo pagamento→acesso.
- **Momento crítico (ativação de compra):** **acesso liberado** + e-mail de boas-vindas com link da 1ª
  aula.

### Fase C — Onboarding/1ª aula (ativação de aprendizado)

- **JTBD:** começar a assistir e sentir que o investimento valeu.
- **Passos:** entra na **Área do Aluno** → "Meus cursos" → abre o curso → **assiste a 1ª aula** no
  **player Bunny** (HLS adaptativo, velocidade 0.5x–2x, legendas, **retomar de onde parou**) →
  progresso rastreado por **heartbeats**.
- **Touchpoints:** in-app (player, barra de progresso), e-mail ("comece por aqui").
- **Emoções/atritos:** travamento de vídeo (egress/qualidade); não saber por onde começar.
- **Oportunidade de encantamento:** **"continue de onde parou"**; primeira aula curta de boas-vindas;
  qualidade adaptativa sem buffering.
- **Métrica:** **ativação de aprendizado** = % que concluem a 1ª aula; horas assistidas (North Star).
- **Momento crítico (Aha):** **primeira aula concluída** (≥90% assistido com checagem anti-seek).

### Fase D — Uso recorrente/Progresso

- **JTBD:** avançar consistentemente até concluir.
- **Passos:** assiste aulas liberadas (respeitando **drip**); faz **quizzes** (correção automática);
  comenta dúvidas por aula; vê **% de progresso**; (F2) ganha pontos/badges, anota com timestamp,
  participa de fórum/lives, usa transcrição pesquisável.
- **Touchpoints:** in-app (dashboard de progresso, quiz, comentários); e-mail (lembrete de retomada,
  "nova aula liberada" via drip); push (F2 — lembrete de estudo).
- **Emoções/atritos:** queda de ritmo/procrastinação; frustração se travar num quiz ou aula bloqueada
  por pré-requisito.
- **Oportunidade de encantamento:** **lembretes inteligentes de retomada**; gamificação [F2];
  celebração de marcos de progresso.
- **Métrica:** retenção por coorte; taxa de progresso; conclusão de quizzes; North Star.

### Fase E — Conclusão e Certificado

- **JTBD:** finalizar e ter prova do esforço.
- **Passos:** conclui 100% (e nota mínima quando houver) → **certificado** gerado automaticamente (PDF,
  worker/Puppeteer) com `uuid_public` + **QR de verificação pública** → baixa/compartilha.
- **Touchpoints:** in-app (tela de conclusão + download), e-mail ("parabéns, seu certificado"); público
  (página de verificação).
- **Emoções/atritos:** orgulho/realização; frustração se a regra de conclusão (≥90% + anti-seek) não
  estiver clara.
- **Oportunidade de encantamento:** **momento de celebração** na conclusão; compartilhamento fácil
  (LinkedIn) com link de verificação — vira marketing orgânico para o tenant.
- **Métrica:** **taxa de conclusão de curso** (PRD §6 — Retenção); certificados emitidos;
  compartilhamentos.
- **Momento crítico (Aha de valor):** **certificado emitido e verificável.**

### Fase F — Retenção/Recompra (expansão do aluno)

- **JTBD:** continuar aprendendo / comprar o próximo curso.
- **Passos:** recebe recomendação de próximo curso/bundle; recompra (checkout já conhecido); renova
  assinatura (se modelo `subscription`); (F3) recomendação por IA, trilhas de aprendizagem.
- **Touchpoints:** in-app (catálogo, "próximo passo"), e-mail (oferta pós-conclusão, renovação), push
  (F2).
- **Emoções/atritos:** churn de assinatura por inatividade; falta de próximo passo claro.
- **Oportunidade de encantamento:** **oferta contextual pós-certificado** ("nível 2 com desconto");
  trilha sugerida; win-back de assinante inativo.
- **Métrica:** recompra/LTV do aluno; renovação de assinatura; churn de alunos por coorte.
- **Momento crítico (1ª venda recorrente do aluno):** **segunda compra** = aluno vira recorrente.

### Journey map — Aluno (persona prioritária)

| Fase | Ações | Touchpoints | Dores | Oportunidades | Métrica |
|---|---|---|---|---|---|
| Descoberta | Chega na landing, avalia, decide | Público (landing/SEO/pixels), e-mail (F2) | "Vale o preço?"; excesso de info | Prova social + preview grátis + garantia | Visitantes→checkout; conversão landing |
| Compra | Pix/boleto/cartão, cupom, paga | Checkout, e-mail confirmação, push (F2) | Boleto "já liberou?"; abandono | Liberação instantânea; recuperação de carrinho (F2) | Conversão checkout; abandono; pagamento→acesso |
| Onboarding/1ª aula | Abre curso, assiste 1ª aula, retoma | In-app player, e-mail "comece aqui" | Buffering; não saber por onde começar | Continue de onde parou; aula de boas-vindas | % conclui 1ª aula; horas assistidas |
| Progresso | Aulas (drip), quizzes, dúvidas, % progresso | In-app, e-mail (drip/retomada), push (F2) | Procrastinação; bloqueio por pré-requisito | Lembretes inteligentes; gamificação (F2) | Retenção por coorte; conclusão de quiz; North Star |
| Conclusão/Certificado | Conclui 100%, gera/baixa certificado | In-app, e-mail, página de verificação | Regra de conclusão pouco clara | Celebração + compartilhamento (marketing orgânico) | Taxa de conclusão; certificados; shares |
| Retenção/Recompra | Recomendação, recompra, renova | In-app, e-mail, push (F2) | Churn por inatividade; sem próximo passo | Oferta pós-certificado; trilha; win-back | Recompra/LTV; renovação; churn de alunos |

---

## 8. Matriz de momentos críticos por persona

| Persona | Aha moment | Ativação | "Primeira vez" decisiva | Risco de churn |
|---|---|---|---|---|
| Super-Admin | Smoke test verde (tenant pronto) | Tenant `active` sem intervenção manual | 1º provisionamento idempotente bem-sucedido | Incidente de isolamento; provisionamento frágil |
| Admin do Tenant | Escola no ar com a própria marca | ≥1 curso publicado + ≥1 venda em 14d | **Primeira venda** caindo | Subutilização; ROI não percebido |
| Instrutor | Vídeo pronto e publicado (visão do aluno) | 1ª aula em vídeo publicada | 1º upload TUS bem-sucedido | Não enxergar resultado/engajamento |
| Afiliado | Primeira comissão atribuída corretamente | 1º link gerado + 1ª venda atribuída | **Primeira comissão** | Reversões/atribuição opaca |
| Aluno | Certificado emitido e verificável | 1ª aula concluída | **Primeira compra** liberada na hora | Inatividade; sem próximo passo |

---

## 9. Loops de engajamento e gatilhos cruzados

As personas se conectam em loops que se reforçam — relevantes para o desenho de notificações e
analytics:

- **Loop de receita do tenant (PRD §7):** Admin/Instrutor **cria** → Afiliado/Admin **vende** →
  plataforma **entrega** (vídeo protegido) → Admin **acompanha**. Quebrar qualquer elo trava a ativação.
- **Gatilho compra → acesso (cross-persona):** compra do **Aluno** → webhook → matrícula → notifica
  **Aluno** (acesso) + **Afiliado** (comissão) + **Admin** (venda). Um evento, três touchpoints.
- **Gatilho conclusão → marketing:** **Aluno** conclui → certificado verificável compartilhado →
  descoberta orgânica → novos **Alunos** (e prova social para o **Admin**).
- **Gatilho quota → expansão:** crescimento de **Aluno**/storage do **Instrutor** → quota → upsell de
  plano (**Admin** ↔ **Super-Admin**).
- **Gatilho inadimplência → suspensão:** falha de billing SaaS → **Super-Admin** suspende tenant →
  impacta **Alunos** (acesso). Reforça necessidade de dunning e comunicação clara.

---

## Dependências e pontos para o coordenador

Esta seção lista o que estas jornadas **cruzam** com outras áreas, para o coordenador reconciliar com os
demais documentos (flows, telas, notificações, RBAC, pricing, analytics).

### Flows (fluxos detalhados a especificar)
- **Provisionamento de tenant (saga):** estados/retry por step e tela de status — alinhar com
  ARCHITECTURE §3.4 e `provisioning_jobs`.
- **Compra → webhook → matrícula → acesso:** máquina de estados de acesso (active/suspended/refunded/
  expired) e idempotência — alinhar PRD §3.6/§5.3 e `payment_events`.
- **Upload de vídeo (TUS) → encoding → webhook "vídeo pronto" → publicação** — ARCHITECTURE §7.
- **Conclusão → emissão de certificado** (regra ≥90% + anti-seek; nota mínima) — PRD §4 itens 4–5.
- **Atribuição de afiliado → comissão (pending/paid/reversed) + split** — DATA_MODEL §2.6.
- **Recuperação de carrinho, order bump, upsell** [F2] — confirmar quando entram no funil do Aluno.
- **Offboarding de tenant** (export → janela de retenção → DROP SCHEMA) — ARCHITECTURE §3.6.

### Telas/superfícies (a detalhar no doc de telas)
- Console Super-Admin (timeline da saga, impersonação com banner, audit log, planos/quotas).
- Checklist de ativação do Admin do Tenant (gamificado) — peça-chave da ativação B2B.
- Studio (uploader resiliente, status de vídeo, drip, quiz builder).
- Painel do Afiliado (gerador de link/UTM, materiais, extrato transparente).
- Área do Aluno (player com "continuar de onde parou", quiz, conclusão/celebração, verificação pública).
- Landing/checkout de alta conversão (Pix/boleto/cartão, cupom, order bump F2).

### Notificações (matriz e-mail/in-app/push a consolidar)
- **E-mail (Resend):** boas-vindas (tenant/aluno/instrutor/afiliado), vídeo pronto, confirmação de
  compra/acesso, conclusão/certificado, venda/comissão, cobrança/dunning SaaS, renovação, win-back.
- **In-app:** checklist de ativação, status de provisionamento/vídeo, banner de impersonação, banner de
  conta suspensa, celebrações (1ª venda, conclusão), estados de acesso explicados.
- **Push (PWA) [F2]:** lembretes de retomada do aluno, venda atribuída ao afiliado, novas dúvidas ao
  instrutor.
- **Definir:** triggers exatos, idempotência (evitar duplicidade com webhooks), opt-in/LGPD.

### RBAC
- Papéis citados: `owner`, `admin`, `instructor`, `affiliate`, `student` (tenant) + Super-Admin
  (`platform`). Reconciliar permissões por superfície (ex.: quem publica curso, quem aprova afiliado,
  quem vê financeiro, quem impersona) com ARCHITECTURE §6.
- **Impersonação** sempre auditada (`audit_log`); Super-Admin sem acesso implícito a dados de tenant.

### Pricing
- **Dois níveis de pricing a não confundir:** plano do SaaS (tenant→plataforma, Stripe, quotas
  `max_students/storage_gb/features`) **vs** preço do curso (aluno→tenant: one_time/subscription/free +
  cupons + afiliados/split). Definir copy e telas distintas.
- Quotas por tier impactam gatilhos de upsell (Super-Admin/Admin) — alinhar com `platform_plans.limits`.
- **Simulador de margem** (preço − gateway − split − comissão) sugerido para o Admin — confirmar
  viabilidade.

### Analytics
- **North Star:** horas assistidas por aluno ativo/mês — instrumentar heartbeats/`lesson_progress`.
- **Ativação B2B:** % tenants com ≥1 curso + ≥1 venda em 14d; tempo até 1ª venda.
- **Ativação aluno:** % que conclui 1ª aula; conversão de checkout por método; abandono de carrinho.
- **Receita:** MRR SaaS, GMV do tenant, take rate, expansion; comissões/reversões de afiliado.
- **Retenção:** churn de tenants/alunos/afiliados; taxa de conclusão de curso; coorte/retenção [F2].
- **Qualidade:** incidentes de isolamento cross-tenant (meta 0); custo de mídia por aluno ativo.
- Confirmar ferramenta (PostHog citado em ARCHITECTURE) e eventos a emitir por jornada/fase.

### Lacunas/decisões em aberto (para o coordenador decidir)
- Quando exatamente order bump/upsell/recuperação de carrinho entram (impacta jornada de compra do
  Aluno e crescimento do Admin) — hoje marcados [F2].
- Política de retenção de dados entre cancelamento e `DROP SCHEMA` (janela win-back).
- Regras finas de reversão de comissão e comunicação ao afiliado.
- Política de dunning do SaaS (nº de tentativas, prazos) antes da suspensão do tenant.
