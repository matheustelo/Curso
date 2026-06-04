# PRD — Plataforma de Cursos Online (SaaS Multitenant)

- **Versão:** 1.0
- **Data:** 2026-06-04
- **Status:** Aprovado para implementação (≥95% de assertividade)
- **Documentos relacionados:** [ARCHITECTURE.md](ARCHITECTURE.md) · [DATA_MODEL.md](DATA_MODEL.md) · [ROADMAP.md](ROADMAP.md) · [ADRs](adr/)

---

## 1. Visão e posicionamento

### 1.1 Resumo do produto
Uma plataforma **SaaS multitenant** onde cada **tenant** (escola, produtor de conteúdo, infoprodutor
ou empresa) tem seu próprio ambiente isolado para **criar, vender e entregar cursos online**, com:
- **LMS profundo** (módulos, aulas, quizzes, certificados, progresso, comunidade) — força dos players
globais (Teachable/LearnWorlds);
- **Checkout brasileiro de alta conversão** (Pix, boleto, cartão, order bump, upsell, afiliados, split)
— força dos players BR (Hotmart/Kiwify/Eduzz);
- **Vídeo protegido** via Bunny.net (token + DRM básico + watermark) — dor real de infoprodutores;
- **White-label real** com isolamento de dados por schema — argumento de confiança e marca própria.

### 1.2 Tese de posicionamento (o "espaço em branco" do mercado)
A pesquisa de mercado identificou que:
- **Players BR** (Hotmart, Kiwify, Eduzz) são fortes em checkout/afiliados, mas **rasos em LMS**
(avaliações, gradebook, gamificação, comunidade).
- **Players globais** (Teachable, Thinkific, Kajabi, LearnWorlds) são fortes em LMS, mas **não
atendem pagamentos BR** (sem Pix/boleto nativos) nem cultura de afiliados.

> **Nosso ângulo:** cruzar **checkout BR de alta conversão + afiliados/split** com **LMS profundo +
> white-label real (isolamento por schema)**. É exatamente a interseção em que ambos os grupos são
> fracos.

### 1.3 Modelo de negócio
- **Receita do SaaS (B2B):** o tenant assina um **plano da plataforma** (mensalidade por tier), com
limites/quotas (nº de alunos, armazenamento, recursos). Billing via Stripe Billing no control plane;
o status da assinatura ativa/suspende o tenant.
- **Receita do tenant (B2C):** o tenant vende cursos para seus alunos via **checkout próprio** com
Pix/boleto/cartão, podendo usar **afiliados e split** de pagamento.

### 1.4 Parâmetros confirmados com o stakeholder
| Parâmetro | Decisão |
|-----------|---------|
| Isolamento multitenant | **Schema-per-tenant** (1 banco PostgreSQL, 1 schema por tenant) |
| Escala esperada (1–3 anos) | **Dezenas de tenants (até ~100)** |
| Checkout / mercado | **BR completo + afiliados/split** |
| Anti-pirataria | **Faseada** (Token+MediaCage Basic → watermark por aluno; DRM Enterprise sob demanda) |
| Streaming | **Bunny.net (Bunny Stream)** |
| Banco | **PostgreSQL** |

---

## 2. Personas

| Persona | Descrição | Necessidades-chave |
|---------|-----------|--------------------|
| **Super-Admin (nós)** | Operador da plataforma SaaS. | Provisionar/suspender tenants, gerir planos e billing do SaaS, observabilidade global, impersonação para suporte. |
| **Admin do Tenant (dono da escola)** | Cliente que assina a plataforma. | Configurar branding/domínio, gerir equipe, cursos, preços, afiliados, relatórios financeiros e de engajamento. |
| **Instrutor** | Cria e gerencia conteúdo. | Criar cursos/módulos/aulas, subir vídeos, montar quizzes, responder a alunos, ver progresso. |
| **Afiliado** | Promove cursos do tenant por comissão. | Links de afiliado, painel de comissões, materiais de divulgação. |
| **Aluno** | Consome os cursos. | Comprar, assistir aulas (retomar, velocidade, legendas), fazer quizzes, obter certificado, interagir na comunidade. |

---

## 3. Funcionalidades por domínio

> Notação de prioridade: **[MVP]** Must-have · **[F2]** Should-have (Fase 2) · **[F3]** Could-have (Fase 3).
> O roadmap consolidado está em [ROADMAP.md](ROADMAP.md).

### 3.1 Gestão de cursos e conteúdo
- **[MVP]** Hierarquia **Curso → Módulo → Aula**, reordenável (drag-and-drop).
- **[MVP]** Tipos de conteúdo: **vídeo (Bunny)**, **texto rico**, **PDF/anexos para download**.
- **[MVP]** **Rascunho/publicação** de cursos e aulas (editar sem expor a alunos).
- **[MVP]** **Drip / liberação programada** — por **data fixa** e por **dias após a matrícula**.
- **[F2]** Tipos adicionais: **quiz como aula**, **aula ao vivo** (embed Zoom/YouTube Live), **áudio**.
- **[F2]** **Pré-requisitos / bloqueio sequencial** (liberar aula só após concluir a anterior).
- **[F2]** **Biblioteca de mídia reutilizável** entre cursos.
- **[F3]** Importação **SCORM/xAPI** (abre mercado corporativo/educacional).
- **[F3]** Versionamento de conteúdo.

### 3.2 Player de vídeo e experiência de aprendizado
- **[MVP]** Player Bunny (iframe) com **HLS adaptativo**, **controle de velocidade (0.5x–2x)**,
**retomar de onde parou**, **legendas**.
- **[MVP]** **Tracking de progresso por heartbeats** (posição, % assistido) — base para conclusão,
gamificação e certificado.
- **[F2]** **Player próprio (Vidstack)** com **watermark dinâmico por aluno** (e-mail/CPF).
- **[F2]** **Anotações do aluno** e **marcadores com timestamp**.
- **[F2]** **Transcrição automática** (Bunny Transcribe AI / Whisper) pesquisável.
- **[F3]** **Vídeo interativo** (perguntas/CTAs sobre o vídeo).
- **[F3]** **Download offline seguro** (app mobile).

### 3.3 Avaliações (quizzes, provas, certificados)
- **[MVP]** **Quiz builder** — múltipla escolha e verdadeiro/falso; correção automática.
- **[MVP]** **Certificado de conclusão** automático (PDF) com template básico + **verificação pública**
(UUID + QR).
- **[F2]** Tipos de questão adicionais (dissertativa, lacuna, ordenar); **tentativas limitadas**, **nota
mínima**, **tempo**, **embaralhamento**.
- **[F2]** **Gradebook / boletim** por aluno e turma.
- **[F3]** **Tarefas com upload** e **correção manual / por pares (peer review)**.
- **[F3]** Templates de certificado customizáveis pelo admin.

### 3.4 Gamificação
- **[F2]** **Pontos/XP** por ação (concluir aula/quiz, login diário).
- **[F2]** **Badges/conquistas** e marcos.
- **[F2]** **Ranking/leaderboard** por turma/período.
- **[F3]** **Níveis** com desbloqueio de conteúdo, **streaks**, **trilhas de aprendizagem**.

### 3.5 Comunidade e engajamento
- **[MVP]** **Comentários por aula** (dúvidas no contexto).
- **[F2]** **Fórum/feed** por curso ou por tópico; **grupos**.
- **[F2]** **Lives nativas** (integração Zoom/embed) + calendário de eventos.
- **[F3]** Chat/DMs, segmentação de comunidade, resumo de feed por IA.

### 3.6 Gestão de alunos
- **[MVP]** **Matrícula** manual, por compra e em massa (CSV); **auto-enroll** por regra.
- **[MVP]** **Progresso por aluno** e **dashboard** do aluno e do instrutor.
- **[MVP]** **Máquina de estados de acesso** atrelada ao status de pagamento (ativo, suspenso por
inadimplência, reembolsado, expirado) — reconciliação periódica com o gateway.
- **[F2]** **Cohorts/turmas** com datas e cronograma compartilhado.
- **[F2]** **Relatórios de conclusão** e detecção de **drop-off**.

### 3.7 Monetização (checkout dos alunos)
- **[MVP]** **Venda avulsa** (one-time) e **assinatura/recorrência** (mensal/anual/trial).
- **[MVP]** **Checkout próprio** com **Pix, boleto e cartão** (parcelamento).
- **[MVP]** **Cupons de desconto**.
- **[MVP]** **Afiliados** (links, painel de comissões) e **split de pagamento** (Pagar.me).
- **[MVP]** **Webhooks de pagamento** ↔ liberação/suspensão de acesso (idempotentes).
- **[F2]** **Order bump** (no checkout) e **upsell/downsell one-click** (pós-compra).
- **[F2]** **Bundles/combos** de cursos; **recuperação de carrinho abandonado**.
- **[F2]** **Co-produção** (split entre produtores).
- **[F3]** **Marketplace de afiliados** (descoberta cross-tenant) — só se virar estratégia.
- **[F3]** Emissão de **nota fiscal** / integração fiscal.

### 3.8 Marketing
- **[MVP]** **Página de vendas (landing) por curso** com SEO básico e **pixels/tracking** (Meta, GA4) +
UTM.
- **[F2]** **E-mail marketing/automação básica** (sequências de boas-vindas, abandono) e integração com
RD Station/ActiveCampaign via event bus.
- **[F3]** **Construtor de landing/funil** nativo, blog.

### 3.9 Certificados e compliance
- **[MVP]** Emissão de certificado + **página pública de verificação** (UUID/QR/hash).
- **[MVP]** **LGPD essencial:** consentimento, **exportação** e **deleção** de dados do aluno; deleção
de tenant via `DROP SCHEMA`.
- **[F2]** **Trilha de auditoria** (logs de acesso por tenant).
- **[F3]** Compliance training (validade/recertificação) para B2B.

### 3.10 Mobile
- **[F2]** **PWA** instalável com **push** (Web Push/OneSignal-Novu).
- **[F3]** **App white-label** nas lojas (modelo de upsell premium tipo Kajabi).

### 3.11 White-label / branding / i18n
- **[MVP]** Branding por tenant: **logo, cores, subdomínio** (`tenant.app.com`).
- **[F2]** **Domínio próprio** com SSL automático; remoção total de marca.
- **[F2]** **i18n da interface** (PT-BR no lançamento; arquitetura pronta para ES/EN).

### 3.12 Administração SaaS / Multitenant (control plane)
- **[MVP]** **Provisionamento de tenant** automatizado (criar schema + migrations + seed +
Bunny Video Library) — idempotente.
- **[MVP]** **Painel Super-Admin:** listar/gerir tenants, planos do SaaS, status; **suspender/ativar**
tenant; **impersonar** para suporte.
- **[MVP]** **Planos da plataforma + quotas** (nº de alunos, armazenamento, recursos).
- **[MVP]** **Billing do SaaS** (Stripe Billing) — status da assinatura controla o tenant.

### 3.13 Analytics e relatórios
- **[MVP]** Métricas básicas: aulas assistidas, progresso, **conclusão por curso**, receita por curso.
- **[F2]** **Coorte/retenção**, **MRR/churn**, funil de checkout, **heatmap de vídeo**, watch-time.
- **[F2]** Relatórios **exportáveis** (CSV).

### 3.14 Acessibilidade
- **[MVP]** Baseline **WCAG 2.1 AA** nos componentes (navegação por teclado, contraste, labels,
legendas nos vídeos) — tratado desde o início, não como retrofit.

### 3.15 Integrações e API
- **[MVP]** **Webhooks de saída** (compra aprovada, reembolso, matrícula, conclusão) para integração
com CRM/automação do tenant.
- **[MVP]** **Webhooks de entrada** do gateway de pagamento e do **Bunny** (vídeo pronto), com
verificação de assinatura e idempotência.
- **[F2]** **API REST pública** documentada (OpenAPI gerado do Zod) + chaves por tenant.
- **[F3]** Zapier/Make, SSO (SAML/OAuth), LTI.

### 3.16 Anti-pirataria / proteção de conteúdo
- **[MVP]** **Token Authentication** (URLs assinadas, expiração curta) + **MediaCage Basic** (grátis) +
**MP4 progressivo desabilitado** + restrição de **referer**.
- **[F2]** **Watermark dinâmico por aluno** (overlay com e-mail/CPF) via player Vidstack.
- **[F2]** **Limite de sessões/dispositivos simultâneos** e detecção de compartilhamento de conta.
- **[F3]** **DRM Enterprise** (Widevine/FairPlay) sob demanda de cliente premium.

### 3.17 IA na plataforma
- **[F2]** **Transcrição/legenda automática** (Bunny Transcribe AI) — acessibilidade + busca + SEO.
- **[F2]** **Geração de quizzes** a partir da transcrição da aula (revisão humana opcional).
- **[F3]** **Tutor IA** (RAG sobre transcrições + pgvector + Anthropic/OpenAI).
- **[F3]** **Recomendação** de cursos/aulas.

---

## 4. Regras de negócio centrais

1. **Isolamento absoluto entre tenants.** Nenhum dado de um tenant pode ser acessível por outro.
Garantido por schema-per-tenant + `SET LOCAL search_path` por transação + testes automatizados de
vazamento cross-tenant. Ver [ARCHITECTURE.md §4](ARCHITECTURE.md).
2. **Acesso = entitlement válido.** O backend SEMPRE valida matrícula/pagamento ativo antes de gerar a
URL assinada de vídeo. Token de vídeo com TTL curto (1–12h).
3. **Pagamento governa acesso.** Compra aprovada → libera; reembolso/chargeback/cancelamento → suspende.
Webhooks idempotentes + reconciliação periódica.
4. **Conclusão de aula** exige ≥90% assistido **com** checagem anti-seek (tempo real ≥ duração × 0.8).
5. **Certificado** só é emitido com o curso 100% concluído e (quando houver) nota mínima atingida.
6. **Provisionamento de tenant** é idempotente e transacional onde possível; estado registrado no
control plane; smoke test pós-provisionamento.
7. **Super-Admin** opera no control plane e pode impersonar com auditoria; nunca há acesso implícito a
dados de tenant sem registro.

---

## 5. Fluxos de usuário (resumo)

### 5.1 Onboarding de tenant
1. Assinatura criada (Stripe Billing) → 2. `INSERT` tenant (`status=provisioning`) → 3. `CREATE SCHEMA`
+ migrations + seed → 4. cria **Bunny Video Library** do tenant → 5. registra roteamento + `status=active`
→ 6. e-mail de boas-vindas ao admin do tenant.

### 5.2 Criação e publicação de aula em vídeo
1. Instrutor inicia upload → 2. backend cria vídeo na library do tenant + gera assinatura → 3. upload
**direto ao Bunny via TUS** → 4. encoding automático → 5. **webhook "vídeo pronto"** (HMAC) → worker
atualiza status → 6. instrutor publica a aula.

### 5.3 Compra e acesso do aluno
1. Aluno na landing → checkout (Pix/boleto/cartão) → 2. **webhook de pagamento** aprova → 3. matrícula +
acesso liberado → 4. aluno assiste (URL assinada, TTL curto) com tracking de progresso → 5. ao concluir,
**certificado** emitido.

> Fluxos técnicos detalhados (assinatura de URL, heartbeats, idempotência de webhook) em
> [ARCHITECTURE.md §8](ARCHITECTURE.md).

---

## 6. Métricas de sucesso (North Star e suporte)

- **North Star:** **horas de conteúdo efetivamente assistidas por aluno ativo/mês** (engajamento real).
- **Aquisição:** nº de tenants ativos, conversão de trial → pago do SaaS.
- **Ativação:** % de tenants que publicam ≥1 curso e realizam ≥1 venda nos primeiros 14 dias.
- **Receita:** MRR do SaaS; GMV transacionado pelos tenants; take rate.
- **Retenção:** churn de tenants; retenção de alunos por coorte; taxa de conclusão de curso.
- **Qualidade:** custo de mídia por aluno ativo; incidentes de vazamento cross-tenant (**meta: 0**).

---

## 7. Riscos de produto e mitigação

| Risco | Mitigação |
|-------|-----------|
| "Tudo em um" cedo demais → nunca lançar | Travar no loop MVP (criar → vender → entregar → acompanhar). |
| Pirataria de vídeo | Token + MediaCage Basic no MVP; watermark por aluno na F2; `VideoProvider` abstraído p/ DRM futuro. |
| Custo de egress de vídeo no Brasil | Validar **Bunny Volume Network** vs Standard; cap de resolução; alertas de custo. Ver [OPEN_QUESTIONS](OPEN_QUESTIONS.md). |
| Sincronia pagamento ↔ acesso | Máquina de estados + webhooks idempotentes + reconciliação. |
| LGPD/acessibilidade tratadas "depois" | Baseline WCAG AA e fluxos de consentimento/exportação/deleção já no MVP. |
| i18n adicionado tarde | Arquitetura i18n desde o início, mesmo lançando só PT-BR. |
| Lock-in (saída do tenant) | Exportação de dados nativa. |

---

## 8. Fora de escopo (por ora)

- Produção de vídeo in-house (modelo Domestika).
- Certificação acadêmica credenciada/diplomas (modelo Coursera).
- Suite de e-mail marketing nível Mailchimp (integrar terceiros).
- CMS/website builder genérico completo.

---

_Este PRD é a fonte de verdade de produto. Mudanças de escopo devem ser refletidas aqui e, quando
alterarem decisões técnicas, registradas como novos ADRs._
