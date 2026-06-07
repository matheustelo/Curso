# Suporte, Busca/Descoberta e Inventário de Settings

- **Versão:** 1.0 · **Data:** 2026-06-07
- **Autor:** Product Management (sênior)
- **Status:** Proposta para revisão do coordenador
- **Relacionados:** [INFORMATION_ARCHITECTURE.md](INFORMATION_ARCHITECTURE.md) · [RBAC_MATRIX.md](RBAC_MATRIX.md) · [NOTIFICATIONS_MATRIX.md](NOTIFICATIONS_MATRIX.md) · [DATA_MODEL.md](../DATA_MODEL.md) (§6.9 `notification_preferences`, §6.10 `tenant_settings`) · [PRD.md](../PRD.md) · [README.md](README.md) (glossário/personas) · [CLAUDE.md](../../CLAUDE.md)

> **Personas:** Super-Admin (SA) · Owner (OW) · Admin (AD) · Instrutor (IN) · Afiliado (AF) · Aluno/ST · Visitante (guest).
> **Fases:** **[MVP]** · **[F2]** · **[F3]**.
> Toda decisão respeita a **Regra nº1 (isolamento por tenant)** e SOLID/DRY/Clean Code. Estas superfícies **expandem** os docs-base sem contradizê-los.
> **Princípio transversal:** preferir solução **nativa e leve** (Postgres + `pg_trgm` + Resend + filas pg-boss) antes de introduzir dependência pesada; cada upgrade de stack passa por **ADR** (CLAUDE.md anti-padrões).

---

## Índice

1. [Suporte / Help Center](#1-suporte--help-center)
   - 1.1 Os dois níveis de suporte · 1.2 Superfícies do aluno · 1.3 Superfícies da equipe do tenant · 1.4 Base de conhecimento (FAQ) · 1.5 Tickets/contato · 1.6 Build vs. buy (Crisp/Intercom vs. nativo) · 1.7 Estados · 1.8 RBAC · 1.9 Notificações · 1.10 Modelo de dados proposto · 1.11 Fases
2. [Busca e descoberta](#2-busca-e-descoberta)
   - 2.1 Mapa das 4 superfícies de busca · 2.2 Catálogo público · 2.3 "Meus cursos" (aluno) · 2.4 Estúdio (admin) · 2.5 Busca dentro do curso · 2.6 Tecnologia (`pg_trgm` e quando escalar) · 2.7 RBAC e isolamento · 2.8 Fases
3. [Inventário de configurações (Settings)](#3-inventário-de-configurações-settings)
   - 3.1 Conta do usuário · 3.2 Settings do tenant · 3.3 Mapa campo→`tenant_settings`/DATA_MODEL · 3.4 RBAC consolidado · 3.5 Fases
4. [Dependências e pontos para o coordenador](#dependências-e-pontos-para-o-coordenador)

---

## 1. Suporte / Help Center

### 1.1 Os dois níveis de suporte (não confundir — espelha os dois domínios de pagamento)

| | **Nível 1 — Suporte do tenant ao aluno (B2C)** | **Nível 2 — Suporte da plataforma ao tenant (B2B)** |
|---|---|---|
| Quem ajuda quem | Equipe do tenant (OW/AD/IN) → Aluno | Nós (SA) → Owner/Admin do tenant |
| Onde vive | Schema do tenant (`/app`, `/manage`) | Control plane (`platform`) / canal externo |
| Escopo de dados | Restrito ao tenant (`withTenant`) | Global; sobre dados do tenant **só via impersonação auditada** (RBAC C3) |
| Canal MVP | FAQ/KB nativa + formulário de contato → e-mail (`support_email` do tenant) + central in-app | E-mail/`support_email` da plataforma + painel SA + impersonação |
| Governa | Dúvida do aluno sobre curso/acesso/pagamento | Provisionamento, billing SaaS, quota, incidentes do tenant |

> **Regra de roteamento de copy:** o aluno **nunca** é direcionado ao suporte da plataforma; ele fala com o **tenant**. O tenant fala com a **plataforma**. O `support_email` exibido ao aluno é o do tenant (placeholder `{support_email}` já existe na NOTIFICATIONS_MATRIX §8); o da plataforma é fixo.

### 1.2 Superfícies do aluno (Nível 1)

| Superfície | Onde (IA) | Descrição | Fase |
|---|---|---|---|
| Central de Ajuda / FAQ do tenant | `/app/ajuda` (novo) e link no rodapé público | Base de conhecimento navegável + busca (ver §2) | MVP |
| "Preciso de ajuda" contextual | Botão no player (`/app/curso/{slug}`) e em `/app/conta/compras` | Abre formulário pré-preenchido com contexto (curso, pedido) | MVP |
| Formulário de contato → ticket | `/app/ajuda/contato` | Cria ticket no tenant; e-mail para `support_email`; eco in-app ao aluno | MVP |
| Meus chamados | `/app/ajuda/meus-chamados` | Histórico/estado dos próprios tickets | MVP |
| Dúvida na aula (já existe) | aba Comentários do player | Canal de dúvida pedagógica (≠ suporte operacional) — **reusar**, não duplicar | MVP |
| Widget de chat ao vivo | embed global | Opcional por tenant (ver §1.6); F2 | F2 |

> **DRY:** a "dúvida pedagógica" usa `lesson_comments` (já modelado, com `new_question` em NOTIFICATIONS §2). Suporte **operacional** (acesso/pagamento/conta) é o novo fluxo de tickets. Não misturar os dois.

### 1.3 Superfícies da equipe do tenant

| Superfície | Onde | Descrição | Fase |
|---|---|---|---|
| Caixa de chamados (Nível 1) | `/manage/suporte` (novo) | Fila de tickets dos alunos: filtrar/atribuir/responder/fechar | MVP |
| Editor da base de conhecimento | `/manage/suporte/ajuda` | CRUD de artigos/FAQ públicos do tenant | MVP |
| Configurar suporte | `/manage/configuracoes/suporte` | Define `support_email`, canal (nativo vs widget), horários | MVP |
| Falar com a plataforma (Nível 2) | `/manage/suporte/plataforma` ou link em `/manage/plano-saas` | Abre chamado B2B com a plataforma SaaS | MVP/F2 |

### 1.4 Base de conhecimento (FAQ)

- Artigos com `slug`, categoria, corpo (markdown/rich), `status` (`draft | published`), busca via `pg_trgm` (§2).
- Públicos no site do tenant (`/ajuda`, SEO) e dentro de `/app/ajuda`.
- Versão da plataforma (sobre o SaaS) vive separada em `app.com/ajuda` (control plane) — fora do schema do tenant.

### 1.5 Tickets/contato

- Canal transacional via **Resend** (DRY com NOTIFICATIONS_MATRIX); jobs via `JobQueue` com `tenantId` no payload.
- Idempotência no envio (sem duplicar e-mail em reprocessamento) — mesma regra das notificações.
- Anti-spam/rate-limit no endpoint público de contato.

### 1.6 Build vs. buy — recomendação

**Recomendação: nativo leve no MVP**, com porta para integração opcional na F2.

- **MVP (nativo):** tabela de tickets + KB no schema do tenant; e-mail via Resend; central in-app já existente. Custo baixo, isolamento garantido por `withTenant`, zero dependência pesada.
- **F2 (opcional, por tenant):** widget externo (**Crisp** preferido a Intercom por custo/peso) embutido via `tenant_settings`. **Não** substitui o nativo; é um canal adicional configurável. Abstrair atrás de uma port `SupportProvider` (DIP) caso se decida integrar — evita acoplar Crisp ao domínio.
- **Evitar no MVP:** Zendesk/Intercom (peso, custo, e dados de aluno saindo do tenant → fricção LGPD). Decisão de integração externa exige **ADR**.

### 1.7 Estados

- **Ticket:** `open | pending | resolved | closed` (alinhar enum a `comment_reports.status` para consistência; **canônico a registrar no DATA_MODEL §6.0**).
- **Artigo KB:** `draft | published` (reusa vocabulário de publicação de `courses`/`lessons`).
- **Tenant suspenso (BUSINESS_RULES §1):** abertura de novos tickets do aluno pausada com aviso de indisponibilidade; canal owner↔plataforma continua (espelha regra de NOTIFICATIONS §0).

### 1.8 RBAC (estende RBAC_MATRIX §3)

| Ação / Recurso | SA | OW | AD | IN | AF | ST | Obs |
|---|:--:|:--:|:--:|:--:|:--:|:--:|---|
| Abrir ticket de aluno (Nível 1) | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | Aluno autenticado |
| Ver/responder fila de tickets do tenant | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C3 (SA) / IN só de seus cursos |
| Atribuir/fechar ticket | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C3 / escopo IN |
| CRUD artigos da KB do tenant | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C3 / IN se delegado |
| Configurar `support_email`/canal | 🔶 | ✅ | ✅ | ❌ | ❌ | ❌ | C3 |
| Abrir chamado B2B com a plataforma (Nível 2) | n/a | ✅ | ✅ | ❌ | ❌ | ❌ | — |
| Atender chamado B2B / impersonar para suporte | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | Impersonação sempre auditada (C3) |
| Ler artigo KB público | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | Público |

> Reuso de condições da RBAC_MATRIX: **C3** (impersonação auditada do SA), **C6** (escopo do instrutor aos próprios cursos).

### 1.9 Notificações (estende NOTIFICATIONS_MATRIX)

Novos eventos sugeridos (consolidar no enum único de `packages/contracts`, NOTIFICATIONS dep. #6):

| Evento | E-mail | In-app | Destinatário | Prio | Template (sugerido) |
|---|:--:|:--:|---|:--:|---|
| Ticket recebido (eco) | ✅ | ✅ | ST (autor) | P1 | `support_ticket_received` |
| Nova resposta no ticket | ✅ | ✅ | ST ou equipe | P1 | `support_ticket_reply` |
| Ticket resolvido/fechado | ✅ | ✅ | ST | P2 | `support_ticket_resolved` |
| Novo ticket na fila | — | ✅ | OW/AD (e IN do curso) | P2 | `support_ticket_assigned` |

> Honram `notification_preferences` (categoria nova `support`) só em P2; P1 não suprimível (NOTIFICATIONS §0).

### 1.10 Modelo de dados proposto (a confirmar no DATA_MODEL)

Tudo no **schema do tenant**, acessado via `withTenant`; cada tabela com dado de tenant exige teste de isolamento cross-tenant (gate de CI). **Nenhuma FK cruza schemas.**

```sql
support_tickets(
  id uuid pk, requester_id uuid fk -> users,
  subject text, status text,            -- open | pending | resolved | closed
  priority text default 'normal',       -- low | normal | high
  context jsonb null,                    -- { course_id?, order_id?, lesson_id? } (preenchido pelo botão contextual)
  assignee_id uuid null fk -> users,
  created_at timestamptz, updated_at timestamptz, resolved_at timestamptz null
)
support_messages(
  id uuid pk, ticket_id uuid fk, author_id uuid fk -> users,
  body text, is_staff boolean,           -- distingue resposta da equipe vs aluno
  created_at timestamptz
)
kb_articles(
  id uuid pk, slug text unique, category text,
  title text, body text, status text,    -- draft | published
  created_at timestamptz, updated_at timestamptz
  -- índice GIN/pg_trgm sobre title+body para busca (§2)
)
-- Suporte B2B (Nível 2) vive no control plane (platform), NÃO aqui:
-- platform.support_tickets(tenant_id, owner_user_id, ...) — sem FK para schemas de tenant.
```

### 1.11 Fases

- **MVP:** KB nativa + tickets nativos (Nível 1) + e-mail Resend + central in-app; `support_email` em settings; chamado B2B básico via e-mail/painel SA + impersonação auditada (já existe).
- **F2:** widget Crisp opcional (`SupportProvider`); SLA/horários; macros/respostas-padrão; digest de fila.
- **F3:** sugestão de artigos por IA (RAG sobre KB + transcrições, reusa pgvector já previsto), deflection de tickets.

---

## 2. Busca e descoberta

### 2.1 Mapa das 4 superfícies de busca

| # | Superfície | Host/área | Ator | Escopo dos dados | Fase |
|---|---|---|---|---|---|
| S1 | **Catálogo público** | site do tenant `/cursos` | Visitante/Aluno | cursos `published` do tenant | MVP |
| S2 | **"Meus cursos"** | `/app` (aluno) | Aluno | matrículas `active` do próprio aluno | MVP |
| S3 | **Estúdio (admin)** | `/manage/*` | OW/AD/IN | cursos/alunos/pedidos do tenant (IN escopado) | MVP |
| S4 | **Dentro do curso** | player `/app/curso/{slug}` | Aluno | aulas/materiais (e transcrição F2) do curso | MVP base / F2 transcrição |

> **Todas** as buscas são restritas ao tenant via `withTenant` + `search_path`. Nenhum índice/consulta cruza schemas (Regra nº1).

### 2.2 Catálogo público (S1)

- **Entrada:** caixa de busca + filtros na vitrine; só vê cursos `published` e (se `tenant_settings.public_catalog_enabled=true`) com catálogo aberto.
- **Busca:** título + descrição via `pg_trgm` (similaridade), tolerante a erro de digitação.
- **Filtros:** categoria/tag, faixa de preço, gratuito/pago, nível, instrutor.
- **Ordenação:** relevância (default), mais recentes, preço asc/desc, mais populares (usa contagem de matrículas — analytics).
- **Estado vazio:** sem resultado → sugerir limpar filtros + cursos em destaque.
- **i18n:** busca considera colunas-base PT-BR; overrides `courses.i18n` na F2 (DATA_MODEL §6.1).

### 2.3 "Meus cursos" — busca interna do aluno (S2)

- **Escopo:** apenas matrículas do próprio aluno (`enrollment` ativo/suspenso/expirado), nunca o catálogo todo.
- **Busca:** por título do curso; filtro por **progresso** (não iniciado/em andamento/concluído — reusa `lesson_progress.status`) e por status de acesso.
- **Ordenação:** continuar de onde parou (último acesso), A–Z, progresso.
- **Volume baixo por aluno** → `pg_trgm`/ILIKE é suficiente; sem necessidade de motor externo.

### 2.4 Estúdio (admin) — S3

Busca por entidade, cada uma escopada ao tenant e ao papel:

| Busca em | Campos | Filtros | Escopo de papel |
|---|---|---|---|
| Cursos | título, slug | status (`draft/published/archived`), instrutor | IN só os seus (C6) |
| Alunos | nome, e-mail | curso matriculado, status de matrícula | AD/OW; IN vê alunos dos seus cursos |
| Pedidos | id, comprador, e-mail | status (`pending/paid/refunded/chargeback`), período, método | AD/OW; IN financeiro só se habilitado (C13) |
| Afiliados | nome, e-mail | status (`pending/active/blocked`) | AD/OW |
| Comentários (moderação) | trecho do texto | status (oculto/denunciado/resolvido) | IN nos seus cursos |

> Reuso de condições RBAC: **C6** (instrutor), **C13** (visibilidade financeira do instrutor).

### 2.5 Busca dentro do curso (S4)

- **MVP:** índice de aulas/módulos (título) + busca em materiais (nome do anexo). Pular para a aula.
- **F2 — busca em transcrição:** quando `lessons.video_status='ready'` e transcrição disponível (PRD §3.2 F2, Bunny Transcribe/Whisper). Resultado retorna **timestamp** → deep-link no player (`?t=`). Reusa a infra de anotações/marcadores com timestamp (IA §2.3, aba Anotações F2).
- **F3 — Tutor IA (RAG):** busca semântica sobre transcrições com **pgvector** (já reservado em `public`, DATA_MODEL §6 / PRD §3.6 F3). Distinto da busca lexical; não substitui `pg_trgm` para listagens.

### 2.6 Tecnologia — `pg_trgm` e quando escalar

**Recomendação: `pg_trgm` (já previsto em `public`, DATA_MODEL linha 23/229) para S1–S4 lexical no MVP/F2.**

- Índices **GIN/GiST** com `gin_trgm_ops` sobre as colunas de texto relevantes, **por schema de tenant** (criados nas migrations que rodam em todos os `tenant_*`).
- Cobre similaridade, prefixo e tolerância a typo — suficiente para o volume por tenant (cursos/aulas/alunos na casa de centenas–milhares).
- **`tsvector`/FTS nativo do Postgres** como passo intermediário se precisar de ranking por relevância textual mais rico (stemming PT-BR) antes de cogitar motor externo — **ainda sem dependência nova**.

**Quando justificaria algo maior (Meilisearch/Typesearch/OpenSearch) — exige ADR:**
- Catálogo cross-tenant agregado (não existe hoje; violaria Regra nº1 se ingênuo).
- Busca facetada de altíssimo volume com latência <50ms em milhões de itens por tenant.
- Necessidade de typo-tolerance + sinônimos + ranking configurável que o Postgres não entrega bem.
- Busca semântica em escala → primeiro **pgvector** (já no stack) antes de motor dedicado.

> **Isolamento ao escalar:** um motor externo precisaria de índice por tenant (namespace/index isolado) e o `tenantId` da sessão como filtro **obrigatório** — caso contrário é vazamento cross-tenant (o bug mais grave). Por isso o default conservador é manter no Postgres.

### 2.7 RBAC e isolamento

- Toda query de busca herda `search_path` do `withTenant`; o `tenantId` vem da sessão/JWT (nunca de header).
- S3 aplica filtros de escopo de papel **no use-case** (defense in depth), não só na UI.
- Resultados nunca incluem entidades de outro tenant — coberto por **teste de isolamento cross-tenant** obrigatório.

### 2.8 Fases

- **MVP:** S1 (catálogo `pg_trgm` + filtros/ordenação), S2 (meus cursos), S3 (estúdio por entidade), S4 (índice de aulas/materiais).
- **F2:** busca em transcrição (S4) com deep-link por timestamp; KB com `pg_trgm` (§1).
- **F3:** Tutor IA / busca semântica (pgvector).

---

## 3. Inventário de configurações (Settings)

> Mapa **completo** de telas/abas de settings por papel. Coluna "Campo/Fonte" liga ao DATA_MODEL (`tenant_settings` §6.10, `notification_preferences` §6.9, `affiliate_program_settings` §6.8, control plane §6.11). Fase entre colchetes.

### 3.1 Conta do usuário (todos os papéis, no próprio) — área `/app/conta` (aluno) e equivalente em `/manage` (equipe)

| Aba/Tela | Campos | Campo/Fonte (DATA_MODEL) | RBAC | Fase |
|---|---|---|---|---|
| **Perfil** | nome, e-mail, avatar, idioma da interface | `users` (nome/e-mail); locale = `tenant_settings.default_locale` como default | Todos no próprio (RBAC §3.3 "Editar próprio perfil") | MVP |
| **Senha** | senha atual, nova senha | `users` (hash via Better-Auth) | Todos no próprio | MVP |
| **Verificação de e-mail** | reenviar verificação | `users.email_verified` | Todos no próprio | MVP (recomendado obrigatório — consistência #20) |
| **2FA / Autenticação** | ativar/desativar TOTP, recovery codes | Better-Auth (a confirmar coluna/tabela no DATA_MODEL) | Todos; **obrigatório SA+Owner**, recomendado Admin, opcional aluno (consistência #21) | MVP (owner) / F2 (geral) |
| **Sessões/dispositivos** | listar e revogar sessões | Better-Auth sessions | Todos no próprio | F2 (alinha "novo login/dispositivo" NOTIFICATIONS §4) |
| **Notificações** | opt-out por canal × categoria | `notification_preferences(user_id, channel, category, enabled)` §6.9 | Todos no próprio | MVP (in-app+e-mail) / push F2 |
| **Privacidade (LGPD)** | exportar meus dados, excluir/anonimizar conta | LGPD (RBAC §3.12); job de export/delete | Todos no próprio (owner não se autodeleta sem transferir — **C14**) | MVP |
| **Recebimento** (afiliado/instrutor) | dados de recipient/split | `affiliates.pagarme_recipient_id_encrypted` §6.8 (cifrado) | AF/IN no próprio | MVP |

> **DRY:** a aba Notificações é a **mesma** superfície para todos os papéis, lendo `notification_preferences`; só muda o conjunto de categorias visíveis (aluno vê `replies/new_lesson/certificate`; equipe vê `sale/support`, etc.).

### 3.2 Settings do tenant — área `/manage/configuracoes` e adjacências

| Grupo (tela) | IA (rota) | Campos | Fase |
|---|---|---|---|
| **Geral** | `/manage/configuracoes/geral` (novo) | nome da escola, locale base, locales suportados, catálogo público on/off | MVP |
| **Marca** | `/manage/configuracoes/marca` | logo, favicon, cor primária/secundária | MVP |
| **Domínio** | `/manage/configuracoes/dominio` | subdomínio (MVP) / domínio próprio + SSL (F2) | MVP/F2 |
| **Pagamentos** | `/manage/configuracoes/pagamentos` | conta gateway (recipient Pagar.me), split padrão, ativar pagamentos/KYC | MVP |
| **Equipe** | `/manage/equipe` | membros, convites, papéis | MVP |
| **Afiliados (programa)** | `/manage/afiliados` (+ settings) | habilitar, comissão default, janela de cookie, política de aprovação, self-referral | MVP |
| **Integrações / Webhooks** | `/manage/integracoes/webhooks` | endpoints de saída, segredo HMAC, eventos | MVP |
| **API keys** | `/manage/integracoes/api` | gerar/rotacionar chaves do tenant | F2 |
| **Pixels & tracking** | `/manage/marketing/pixels` | Meta/GA4, UTM | MVP |
| **E-mail marketing** | `/manage/marketing/email` | integrações/automação | F2 |
| **Certificado** | `/manage/configuracoes/certificado` | template (básico MVP, customização F3) | MVP/F3 |
| **Idioma/i18n** | `/manage/configuracoes/idioma` | i18n da interface | F2 |
| **Suporte** | `/manage/configuracoes/suporte` (novo, §1) | `support_email`, canal (nativo/widget), horários | MVP |
| **LGPD/Privacidade** | `/manage/configuracoes/privacidade` (novo) | retenção, deleção de aluno, política/consentimento | MVP (deleção) / prazos ⏳ jurídico (#24) |
| **Políticas financeiras** | `/manage/configuracoes/politicas` (novo) | revogar certificado em reembolso, carência dunning aluno, garantia de comissão | MVP |
| **Plano/quota SaaS** | `/manage/plano-saas` | uso vs. limites, billing do SaaS (portal Stripe) | MVP (só **owner**) |

### 3.3 Mapa campo → `tenant_settings` / DATA_MODEL

| Campo na UI | Coluna / Tabela | Default | Fase |
|---|---|---|---|
| Logo | `tenant_settings.logo_url` | null | MVP |
| Cor primária | `tenant_settings.primary_color` | null | MVP |
| Cor secundária | `tenant_settings.secondary_color` | null | MVP |
| Idioma base | `tenant_settings.default_locale` | `pt-BR` | MVP |
| Idiomas suportados | `tenant_settings.supported_locales` | `{pt-BR}` | MVP (ativação i18n F2) |
| Catálogo público on/off | `tenant_settings.public_catalog_enabled` | `true` | MVP |
| Revogar certificado em reembolso | `tenant_settings.refund_revokes_certificate` | `true` | MVP (confirmar #17) |
| Carência inadimplência do aluno (dias) | `tenant_settings.student_dunning_grace_days` | `7` | MVP (valor ⏳ #18) |
| Garantia antes de comissão paga (dias) | `tenant_settings.affiliate_clearance_days` | `14` | MVP |
| `support_email` (suporte ao aluno) | **novo** `tenant_settings.support_email` (proposto) | null → fallback owner | MVP |
| Canal de suporte (nativo/widget) | **novo** `tenant_settings.support_channel` + `support_widget_config jsonb` (proposto) | `native` | MVP/F2 |
| Preferências de notificação | `notification_preferences(user_id, channel, category, enabled)` | `enabled=true` | MVP |
| Programa de afiliados (habilitado, %, janela, aprovação, self-referral) | `affiliate_program_settings` (§6.8) | enabled=false / 30d / manual / false | MVP |
| Recipient de pagamento do produtor/afiliado | `affiliates.pagarme_recipient_id_encrypted` (cifrado, §6.8) | null | MVP |
| Quotas/limites/take rate | control plane `platform_plans.limits` (§6.11) — **read-only** no tenant, injetado no `onRequest` | por plano | MVP |
| Domínio próprio | `tenants.custom_domain` (control plane) | null | F2 |
| Pixels/UTM | tabela de marketing do tenant (a confirmar) | — | MVP |

> **Campos novos propostos em `tenant_settings`:** `support_email`, `support_channel`, `support_widget_config` (§1) e um `default_certificate_template_id` quando a customização entrar (F3). **A confirmar com a coordenação no DATA_MODEL §6.10.**

### 3.4 RBAC consolidado de Settings (estende RBAC_MATRIX §3.2/§3.3)

| Grupo | SA | OW | AD | IN | AF | ST | Obs |
|---|:--:|:--:|:--:|:--:|:--:|:--:|---|
| Conta própria (perfil/senha/2FA/notif./privacidade) | 🔶 | ✅ | ✅ | ✅ | ✅ | ✅ | Todos no próprio; SA via impersonação (C3) |
| Marca / Geral / Idioma | 🔶 | ✅ | ✅ | ❌ | ❌ | ❌ | C3 |
| Domínio próprio | 🔶 | ✅ | 🔶 | ❌ | ❌ | ❌ | C4 (admin se delegado) |
| Pagamentos (gateway/credenciais) | 🔶 | ✅ | 🔶 | ❌ | ❌ | ❌ | C4 + C9 (segredos nunca no front) |
| Equipe / papéis | 🔶 | ✅ | 🔶 | ❌ | ❌ | ❌ | C5 (admin não cria owner) |
| Afiliados (programa) | 🔶 | ✅ | ✅ | ❌ | ❌ | ❌ | C3 |
| Webhooks / API keys | 🔶 | ✅ | 🔶 | ❌ | ❌ | ❌ | C4 + C9 |
| Suporte (config) | 🔶 | ✅ | ✅ | ❌ | ❌ | ❌ | C3 |
| Políticas financeiras / LGPD do tenant | 🔶 | ✅ | ✅ | ❌ | ❌ | ❌ | C3; deleção de aluno + `audit_log` |
| Plano/quota SaaS (billing) | 🔶 | ✅ | ❌ | ❌ | ❌ | ❌ | **Exclusivo do owner** (C2) |

### 3.5 Fases

- **MVP:** perfil, senha, verificação de e-mail, 2FA (owner), notificações (in-app+e-mail), privacidade/LGPD, recebimento; settings do tenant: geral, marca, domínio (subdomínio), pagamentos, equipe, afiliados, webhooks, pixels, suporte, políticas financeiras, plano SaaS.
- **F2:** 2FA geral, sessões/dispositivos, push (notificações), domínio próprio, API keys, e-mail marketing, idioma/i18n, widget de suporte.
- **F3:** template de certificado customizável.

---

## Dependências e pontos para o coordenador

1. **Novas tabelas de Suporte (Nível 1):** `support_tickets`, `support_messages`, `kb_articles` no schema do tenant — confirmar inclusão no DATA_MODEL §6 e exigir **teste de isolamento cross-tenant** (gate CI). Suporte B2B (Nível 2) em `platform.*` sem FK cross-schema.
2. **Enum canônico de `status` de ticket** (`open|pending|resolved|closed`) — registrar no DATA_MODEL §6.0 e no enum único de `packages/contracts` (DRY com `comment_reports`).
3. **Novos campos em `tenant_settings`:** `support_email`, `support_channel`, `support_widget_config jsonb` — validar nomes/defaults. Hoje o placeholder `{support_email}` (NOTIFICATIONS §8) não tem coluna de origem definida.
4. **Novos eventos de notificação de suporte** (`support_ticket_*`) e categoria `support` em `notification_preferences` — adicionar ao catálogo canônico (NOTIFICATIONS dep. #6).
5. **Build vs. buy de suporte:** confirmar nativo no MVP e adiar Crisp/Intercom para F2 atrás de port `SupportProvider`; integração externa exige **ADR** (dados de aluno saindo do tenant → revisão LGPD).
6. **Busca — índices `pg_trgm` por schema:** confirmar que as migrations criam índices GIN/GiST em todos os `tenant_*` (e ao provisionar novos tenants). Decidir se FTS `tsvector` PT-BR entra já no MVP para ranking.
7. **Gatilho para motor de busca externo:** acordar critérios objetivos (volume/latência/facetas) que justifiquem ADR para Meilisearch/OpenSearch; reforçar exigência de índice isolado por tenant para não violar a Regra nº1.
8. **Busca em transcrição (F2):** depende de transcrição (Bunny/Whisper, PRD §3.2 F2) e de deep-link por timestamp no player; alinhar com aba Anotações (IA §2.3) e Tutor IA F3 (pgvector).
9. **Novas rotas de IA propostas:** `/app/ajuda*`, `/manage/suporte*`, `/manage/configuracoes/{geral,suporte,privacidade,politicas}` — não existem na INFORMATION_ARCHITECTURE; validar e incorporar ao sitemap/inventário de telas.
10. **2FA / sessões:** confirmar tabela/colunas (Better-Auth) no DATA_MODEL para suportar a aba de 2FA (MVP owner) e sessões/dispositivos (F2), alinhado a consistência #21 e ao evento `new_login` (NOTIFICATIONS §4).
11. **Escopo do instrutor em Suporte/Busca:** o instrutor vê fila de tickets e busca de alunos **só dos próprios cursos** (C6/C13) — confirmar a fronteira (mesma pendência da IA §4.2 e RBAC dep. #3).
12. **Settings de marketing/pixels:** a tabela-fonte de pixels/UTM não está explícita no DATA_MODEL — confirmar onde os valores persistem.
