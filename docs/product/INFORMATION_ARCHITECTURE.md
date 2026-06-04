# Arquitetura de Informação — Plataforma de Cursos SaaS Multitenant

- **Versão:** 1.0 · **Data:** 2026-06-04
- **Autor:** Information Architecture / Interaction Design
- **Status:** Proposta para revisão de produto/engenharia
- **Relacionados:** [PRD.md](../PRD.md) · [ROADMAP.md](../ROADMAP.md) · [ARCHITECTURE.md](../ARCHITECTURE.md) · [DATA_MODEL.md](../DATA_MODEL.md) · [USER_FLOWS.md](USER_FLOWS.md)

> **Personas:** Super-Admin · Admin do Tenant (`owner`/`admin`) · Instrutor (`instructor`) · Afiliado (`affiliate`) · Aluno (`student`).
> **Fases:** **[MVP]** · **[F2]** · **[F3]**.
> **Modelo de roteamento:** subdomínio por tenant (`tenant.app.com`); domínio próprio na F2 (`escola.com.br`); site da plataforma em domínio raiz (`app.com` / `www.app.com`); painel Super-Admin em host dedicado (`admin.app.com`).

---

## Índice

1. [Áreas de informação (visão macro)](#1-áreas-de-informação-visão-macro)
2. [Sitemap por área](#2-sitemap-por-área)
   - 2.1 Site da plataforma (B2B / SaaS) · 2.2 Site público do tenant · 2.3 Área do aluno · 2.4 Painel admin/instrutor · 2.5 Painel do afiliado · 2.6 Painel Super-Admin
3. [Inventário de telas](#3-inventário-de-telas)
4. [Navegação por papel](#4-navegação-por-papel)
5. [Estrutura de URLs](#5-estrutura-de-urls)
6. [Dependências e pontos para o coordenador](#dependências-e-pontos-para-o-coordenador)

---

## 1. Áreas de informação (visão macro)

| # | Área | Host/escopo | Público | Auth |
|---|------|-------------|---------|------|
| A | **Site da plataforma (SaaS)** | `app.com` (raiz) | Prospects B2B (futuros admins de tenant) | Pública + checkout Stripe |
| B | **Site público do tenant** | `tenant.app.com` (ou domínio próprio F2) | Visitantes/leads do tenant | Pública |
| C | **Área do aluno** | `tenant.app.com/app/*` | Alunos matriculados | Sessão (student) |
| D | **Painel admin/instrutor** | `tenant.app.com/manage/*` | Equipe do tenant | Sessão (owner/admin/instructor) |
| E | **Painel do afiliado** | `tenant.app.com/affiliate/*` | Afiliados do tenant | Sessão (affiliate) |
| F | **Painel Super-Admin** | `admin.app.com` (control plane) | Staff interno (nós) | Sessão global + MFA |

> As áreas B–E compartilham o subdomínio do tenant e o branding por tenant (CSS variables). A área F é o control plane (schema `platform`), totalmente separada.

---

## 2. Sitemap por área

### 2.1 Site da plataforma (B2B / SaaS) — `app.com`

```
app.com (raiz, marketing do SaaS)
├─ / .................................. Home (proposta de valor)
├─ /recursos .......................... Funcionalidades (LMS + checkout BR + white-label)
├─ /precos ............................ Planos do SaaS (tiers + quotas) [MVP]
├─ /precos/checkout ................... Checkout Stripe Billing (assinar plano) [MVP]
├─ /precos/checkout/sucesso ........... Confirmação + escolha de slug/onboarding [MVP]
├─ /casos / /clientes ................. Social proof [F2]
├─ /blog .............................. Conteúdo/SEO [F3]
├─ /contato / /demo ................... Lead/vendas
├─ /login-plataforma .................. (redireciona admins ao seu subdomínio)
└─ /legal/{termos|privacidade} ........ Jurídico/LGPD
```

### 2.2 Site público do tenant — `tenant.app.com`

```
tenant.app.com (vitrine do tenant, branding próprio)
├─ / .................................. Home/landing do tenant
├─ /cursos ............................ Catálogo de cursos (published) [MVP]
├─ /curso/{slug} ...................... Landing de vendas do curso (SEO, pixels, UTM) [MVP]
├─ /curso/{slug}?ref={code} ........... Mesma landing com atribuição de afiliado [MVP]
├─ /checkout/{slug} ................... Checkout do aluno (Pix/boleto/cartão + cupom) [MVP]
│   ├─ /checkout/aguardando-pix
│   ├─ /checkout/boleto
│   ├─ /checkout/recusado
│   └─ /checkout/sucesso .............. Compra confirmada (+ upsell F2)
├─ /certificados/verificar/{uuid} ..... Verificação pública de certificado (QR) [MVP]
├─ /seja-afiliado ..................... Adesão ao programa de afiliados [MVP]
├─ /entrar (login) .................... Login do tenant [MVP]
├─ /cadastro .......................... Cadastro de aluno [MVP]
├─ /recuperar-senha ................... Solicitar/definir nova senha [MVP]
├─ /aceitar-convite/{token} ........... Aceite de convite (equipe/afiliado) [MVP]
└─ /legal/{termos|privacidade} ........ Jurídico do tenant
```

### 2.3 Área do aluno — `tenant.app.com/app/*` (auth: student)

```
/app
├─ /app (Meus cursos) ................. Dashboard do aluno (cursos + progresso) [MVP]
├─ /app/curso/{slug} .................. Player do curso (índice + aula ativa) [MVP]
│   ├─ aba Conteúdo (vídeo/texto/pdf)
│   ├─ aba Comentários ................ Comentários por aula [MVP]
│   ├─ aba Materiais .................. Anexos/PDFs (download) [MVP]
│   ├─ aba Quiz ....................... Responder avaliação [MVP]
│   └─ aba Anotações .................. Notas/timestamps [F2]
├─ /app/certificados .................. Meus certificados (baixar) [MVP]
├─ /app/quizzes ....................... Histórico de tentativas [MVP]
├─ /app/comunidade .................... Fórum/feed/grupos [F2]
│   ├─ /app/comunidade/grupo/{id}
│   └─ /app/comunidade/eventos ........ Calendário/lives [F2]
├─ /app/conquistas .................... Gamificação (pontos/badges/ranking) [F2]
├─ /app/conta
│   ├─ /app/conta/perfil .............. Dados, senha, idioma [MVP]
│   ├─ /app/conta/assinatura .......... Minha assinatura ao conteúdo (cancelar) [MVP]
│   ├─ /app/conta/compras ............. Histórico de pedidos/recibos [MVP]
│   ├─ /app/conta/notificacoes ........ Preferências [F2]
│   └─ /app/conta/privacidade ......... Exportar/excluir meus dados (LGPD) [MVP]
└─ /app/seja-afiliado ................. Vira afiliado (se programa aberto) [MVP]
```

### 2.4 Painel admin/instrutor — `tenant.app.com/manage/*` (auth: owner/admin/instructor)

```
/manage
├─ /manage (Dashboard) ................ KPIs do tenant (receita, conclusão, alunos) [MVP]
├─ /manage/onboarding ................. Wizard inicial pós-provisionamento [MVP]
├─ /manage/cursos ..................... Lista de cursos (draft/published/archived) [MVP]
│   ├─ /manage/cursos/novo ............ Criar curso [MVP]
│   └─ /manage/cursos/{id}
│       ├─ /editor .................... Estrutura: módulos/aulas (drag-and-drop) [MVP]
│       │   └─ /aula/{id} ............. Editor da aula (vídeo TUS / texto / pdf / drip) [MVP]
│       ├─ /avaliacoes ................ Quiz builder [MVP]
│       ├─ /precos ................... Preço, cupons aplicáveis, pricing_type [MVP]
│       ├─ /pagina-de-vendas ......... Landing (SEO, pixels, UTM) [MVP]
│       ├─ /alunos ................... Matriculados + progresso do curso [MVP]
│       ├─ /comentarios .............. Moderação de comentários [MVP]
│       ├─ /comunidade ............... Espaço de comunidade do curso [F2]
│       └─ /split .................... Regras de split/co-produção [MVP/F2]
├─ /manage/biblioteca-de-midia ........ Mídia reutilizável [F2]
├─ /manage/alunos ..................... Todos os alunos do tenant [MVP]
│   ├─ /manage/alunos/{id} ........... Detalhe do aluno (matrículas, progresso) [MVP]
│   ├─ /manage/alunos/matricular ..... Matrícula manual [MVP]
│   └─ /manage/alunos/importar ....... Import CSV (matrícula em massa) [MVP]
├─ /manage/turmas ..................... Cohorts/turmas [F2]
├─ /manage/financeiro
│   ├─ /manage/financeiro/pedidos ..... Pedidos (status, reembolso) [MVP]
│   ├─ /manage/financeiro/assinaturas . Assinaturas dos alunos [MVP]
│   ├─ /manage/financeiro/cupons ...... Gestão de cupons [MVP]
│   ├─ /manage/financeiro/reembolsos .. Solicitações/ações de reembolso [MVP]
│   └─ /manage/financeiro/relatorios .. Receita por curso, exportação CSV [MVP/F2]
├─ /manage/afiliados .................. Programa de afiliados [MVP]
│   ├─ /manage/afiliados/{id} ........ Detalhe/aprovação do afiliado [MVP]
│   └─ /manage/afiliados/comissoes ... Comissões (pending/paid/reversed) [MVP]
├─ /manage/marketing
│   ├─ /manage/marketing/pixels ....... Meta/GA4 + UTM [MVP]
│   └─ /manage/marketing/email ........ E-mail/automação + integrações [F2]
├─ /manage/analytics .................. Engajamento, conclusão, watch-time [MVP/F2]
├─ /manage/equipe ..................... Membros + convites + papéis [MVP]
├─ /manage/integracoes
│   ├─ /manage/integracoes/webhooks ... Webhooks de saída [MVP]
│   └─ /manage/integracoes/api ........ Chaves de API por tenant [F2]
├─ /manage/configuracoes
│   ├─ /manage/configuracoes/marca .... Logo, cores, favicon [MVP]
│   ├─ /manage/configuracoes/dominio .. Subdomínio [MVP] / domínio próprio [F2]
│   ├─ /manage/configuracoes/pagamentos Conta gateway, split padrão [MVP]
│   ├─ /manage/configuracoes/idioma ... i18n da interface [F2]
│   └─ /manage/configuracoes/certificado Template de certificado [MVP/F3]
└─ /manage/plano-saas ................. Plano/quota do tenant + billing (owner) [MVP]
```

### 2.5 Painel do afiliado — `tenant.app.com/affiliate/*` (auth: affiliate)

```
/affiliate
├─ /affiliate (Dashboard) ............. Resumo: cliques, vendas, comissão [MVP]
├─ /affiliate/links ................... Gerar/copiar links por curso (?ref=code) [MVP]
├─ /affiliate/materiais ............... Materiais de divulgação [MVP]
├─ /affiliate/comissoes ............... Extrato de comissões (pending/paid/reversed) [MVP]
└─ /affiliate/conta ................... Perfil + dados de recebimento (split) [MVP]
```

### 2.6 Painel Super-Admin — `admin.app.com` (auth: super-admin + MFA)

```
admin.app.com
├─ /login ............................. Login da plataforma (MFA) [MVP]
├─ / (Dashboard) ...................... Visão global (tenants, MRR, saúde) [MVP]
├─ /tenants ........................... Lista de tenants (status, plano) [MVP]
│   ├─ /tenants/novo ................. Criar tenant (dispara saga) [MVP]
│   └─ /tenants/{id}
│       ├─ (visão geral) ............. Status, plano, uso/quota [MVP]
│       ├─ /provisionamento .......... Progresso/retry da saga [MVP]
│       ├─ /usuarios ................. Usuários do tenant (impersonar) [MVP]
│       ├─ /billing .................. Assinatura/faturas do SaaS (Stripe) [MVP]
│       ├─ /bunny .................... Library/keys do tenant [MVP]
│       └─ /acoes .................... Suspender/ativar/cancelar [MVP]
├─ /planos ............................ Planos do SaaS + quotas [MVP]
├─ /faturamento ....................... Faturas/assinaturas do SaaS (global) [MVP]
├─ /auditoria ......................... Audit log (impersonação, suspensões) [MVP]
├─ /provisionamento ................... Fila de jobs de provisionamento [MVP]
├─ /observabilidade ................... Saúde/erros (links Sentry/OTel) [MVP]
└─ /super-admins ...................... Gestão da equipe interna + MFA [MVP]
```

---

## 3. Inventário de telas

> Legenda de tipo: **P** página · **M** modal/drawer · **W** wizard. Auth: papel exigido.

### 3.1 Site da plataforma (A)

| Tela | Tipo | Propósito | Fase | Auth |
|------|------|-----------|------|------|
| Home plataforma | P | Proposta de valor do SaaS | MVP | público |
| Recursos | P | Detalhar LMS + checkout BR + white-label | MVP | público |
| Planos do SaaS | P | Tiers, quotas, comparação | MVP | público |
| Checkout Stripe Billing | P | Assinar plano + escolher slug | MVP | público |
| Confirmação/onboarding inicial | P/W | Pós-assinatura, primeiro acesso ao tenant | MVP | público→admin |
| Contato/Demo | P | Captura de lead B2B | MVP | público |
| Legal (termos/privacidade) | P | Jurídico/LGPD do SaaS | MVP | público |

### 3.2 Site público do tenant (B)

| Tela | Tipo | Propósito | Fase | Auth |
|------|------|-----------|------|------|
| Home do tenant | P | Vitrine com branding do tenant | MVP | público |
| Catálogo de cursos | P | Listar cursos publicados | MVP | público |
| Landing de venda do curso | P | Conversão (SEO, pixels, UTM, preço) | MVP | público |
| Checkout do aluno | P | Pix/boleto/cartão + cupom + identificação | MVP | público/student |
| Checkout: aguardando Pix | P | QR/copia-e-cola + polling | MVP | público/student |
| Checkout: boleto gerado | P | Linha digitável + PDF | MVP | público/student |
| Checkout: pagamento recusado | P | Erro de cartão + retry | MVP | público/student |
| Checkout: compra confirmada | P | Sucesso + CTA acessar (+ upsell F2) | MVP | student |
| Upsell/Downsell one-click | P/M | Oferta pós-compra | F2 | student |
| Verificação de certificado | P | Validar autenticidade (QR/uuid) | MVP | público |
| Seja afiliado | P | Adesão ao programa | MVP | público/student |
| Login do tenant | P | Autenticação (todos os papéis do tenant) | MVP | público |
| Cadastro de aluno | P | Criar conta + consentimento LGPD | MVP | público |
| Recuperar senha (solicitar) | P | Disparar reset (anti-enumeração) | MVP | público |
| Definir nova senha | P | Concluir reset via token | MVP | público |
| Aceitar convite | P | Aceite de equipe/afiliado | MVP | público |
| Tenant indisponível/suspenso | P | Estado quando tenant suspenso | MVP | qualquer |

### 3.3 Área do aluno (C)

| Tela | Tipo | Propósito | Fase | Auth |
|------|------|-----------|------|------|
| Meus cursos (dashboard) | P | Cursos matriculados + progresso | MVP | student |
| Player do curso | P | Assistir aula + índice + abas | MVP | student |
| Aba Comentários | (aba) | Dúvidas no contexto da aula | MVP | student |
| Aba Materiais | (aba) | Download de anexos (R2) | MVP | student |
| Aba Quiz | (aba) | Responder avaliação | MVP | student |
| Aba Anotações | (aba) | Notas/marcadores com timestamp | F2 | student |
| Meus certificados | P | Listar + baixar PDFs | MVP | student |
| Histórico de quizzes | P | Tentativas e notas | MVP | student |
| Comunidade (feed/fórum) | P | Interação, grupos | F2 | student |
| Eventos/lives | P | Calendário + embed Zoom/YouTube | F2 | student |
| Conquistas | P | Pontos, badges, ranking | F2 | student |
| Perfil | P | Dados, senha, idioma | MVP | student |
| Minha assinatura | P | Status + cancelar assinatura ao conteúdo | MVP | student |
| Minhas compras | P | Pedidos/recibos | MVP | student |
| Privacidade (LGPD) | P | Exportar/excluir meus dados | MVP | student |
| Notificações | P | Preferências | F2 | student |

### 3.4 Painel admin/instrutor (D)

| Tela | Tipo | Propósito | Fase | Auth |
|------|------|-----------|------|------|
| Dashboard do tenant | P | KPIs (receita, conclusão, alunos ativos) | MVP | admin/owner/instr |
| Wizard de onboarding | W | Configurar marca, 1º curso, 1ª venda | MVP | admin/owner |
| Lista de cursos | P | Gerir cursos por status | MVP | instr+ |
| Criar curso | P/M | Novo curso (título, preço, capa) | MVP | instr+ |
| Editor do curso (estrutura) | P | Módulos/aulas drag-and-drop | MVP | instr+ |
| Editor da aula | P | Vídeo (TUS)/texto/pdf + drip + publicar | MVP | instr+ |
| Quiz builder | P | Criar/editar quizzes e questões | MVP | instr+ |
| Preços do curso | P | pricing_type, price_cents, cupons | MVP | admin/owner |
| Página de vendas (editor) | P | Landing do curso (SEO/pixels) | MVP | admin/owner |
| Alunos do curso | P | Matriculados + progresso | MVP | instr+ |
| Moderação de comentários | P | Gerir comentários | MVP | instr+ |
| Biblioteca de mídia | P | Reuso de mídia entre cursos | F2 | instr+ |
| Lista de alunos (tenant) | P | Todos os alunos | MVP | admin/owner |
| Detalhe do aluno | P | Matrículas, progresso, ações | MVP | admin/owner |
| Matrícula manual | P/M | Matricular avulso | MVP | admin/owner |
| Importar CSV | P/W | Matrícula em massa | MVP | admin/owner |
| Turmas/cohorts | P | Gerir turmas | F2 | admin/owner |
| Pedidos | P | Listar/filtrar pedidos | MVP | admin/owner |
| Assinaturas (alunos) | P | Gerir assinaturas ao conteúdo | MVP | admin/owner |
| Cupons | P | CRUD de cupons | MVP | admin/owner |
| Reembolsos | P | Aprovar/processar reembolsos | MVP | admin/owner |
| Relatórios financeiros | P | Receita por curso, export CSV | MVP/F2 | admin/owner |
| Programa de afiliados | P | Listar/aprovar afiliados | MVP | admin/owner |
| Detalhe do afiliado | P | Aprovar/recusar, comissão | MVP | admin/owner |
| Comissões | P | Acompanhar comissões | MVP | admin/owner |
| Split/co-produção | P | Regras de divisão | MVP/F2 | admin/owner |
| Pixels & tracking | P | Meta/GA4, UTM | MVP | admin/owner |
| E-mail marketing | P | Sequências/integrações | F2 | admin/owner |
| Analytics | P | Engajamento, conclusão, watch-time | MVP/F2 | admin/owner |
| Equipe | P | Membros, convites, papéis | MVP | admin/owner |
| Webhooks de saída | P | Configurar webhooks | MVP | admin/owner |
| API keys | P | Chaves por tenant | F2 | admin/owner |
| Config — Marca | P | Logo, cores, favicon | MVP | admin/owner |
| Config — Domínio | P | Subdomínio/domínio próprio + SSL | MVP/F2 | admin/owner |
| Config — Pagamentos | P | Conta gateway, split padrão | MVP | owner |
| Config — Idioma | P | i18n | F2 | admin/owner |
| Config — Certificado | P | Template do certificado | MVP/F3 | admin/owner |
| Plano/quota do tenant | P | Uso vs. limites, billing do SaaS | MVP | owner |

### 3.5 Painel do afiliado (E)

| Tela | Tipo | Propósito | Fase | Auth |
|------|------|-----------|------|------|
| Dashboard do afiliado | P | Cliques, vendas, comissão | MVP | affiliate |
| Gerar links | P | Links por curso (?ref) | MVP | affiliate |
| Materiais de divulgação | P | Banners/criativos | MVP | affiliate |
| Extrato de comissões | P | pending/paid/reversed | MVP | affiliate |
| Conta/recebimento | P | Perfil + dados de split | MVP | affiliate |

### 3.6 Painel Super-Admin (F)

| Tela | Tipo | Propósito | Fase | Auth |
|------|------|-----------|------|------|
| Login plataforma (MFA) | P | Autenticação global | MVP | super-admin |
| Dashboard global | P | Tenants, MRR, saúde | MVP | super-admin |
| Lista de tenants | P | Gerir tenants (status/plano) | MVP | super-admin |
| Criar tenant | P/W | Dispara saga de provisionamento | MVP | super-admin |
| Tenant — visão geral | P | Status, plano, uso/quota | MVP | super-admin |
| Tenant — provisionamento | P | Progresso/retry da saga | MVP | super-admin |
| Tenant — usuários (impersonar) | P | Suporte via impersonação | MVP | super-admin |
| Tenant — billing SaaS | P | Assinatura/faturas Stripe | MVP | super-admin |
| Tenant — Bunny | P | Library/keys cifradas | MVP | super-admin |
| Tenant — ações | P/M | Suspender/ativar/cancelar | MVP | super-admin |
| Planos do SaaS | P | CRUD de planos + quotas | MVP | super-admin |
| Faturamento global | P | Faturas/assinaturas SaaS | MVP | super-admin |
| Auditoria | P | Audit log (ações sensíveis) | MVP | super-admin |
| Fila de provisionamento | P | Monitorar jobs/saga | MVP | super-admin |
| Observabilidade | P | Links Sentry/OTel, saúde | MVP | super-admin |
| Gestão de super-admins | P | Staff interno + MFA | MVP | super-admin |

---

## 4. Navegação por papel

> O que cada papel vê no menu principal (itens [F2]/[F3] aparecem conforme habilitados por fase/quota).

### 4.1 Aluno (`student`) — área `/app`
```
Meus cursos · Comunidade [F2] · Conquistas [F2] · Meus certificados
└ menu da conta: Perfil · Minha assinatura · Minhas compras · Privacidade (LGPD) · Notificações [F2] · Sair
```

### 4.2 Instrutor (`instructor`) — área `/manage` (escopo de conteúdo)
```
Dashboard · Cursos · Biblioteca de mídia [F2] · Alunos (dos seus cursos) · Analytics
(NÃO vê: Financeiro completo, Afiliados, Equipe, Configurações de tenant, Plano SaaS)
```

### 4.3 Admin do Tenant (`admin`) — área `/manage` (gestão ampla)
```
Dashboard · Cursos · Alunos · Turmas [F2] · Financeiro (Pedidos/Assinaturas/Cupons/Reembolsos/Relatórios)
· Afiliados · Marketing · Analytics · Equipe · Integrações · Configurações
(NÃO vê: Plano/billing SaaS — restrito ao owner; gestão de super-admins)
```

### 4.4 Owner (`owner`) — área `/manage` (admin + billing do SaaS)
```
Tudo do Admin + Plano/quota do tenant (billing SaaS) + Config — Pagamentos
(é o responsável pela assinatura do tenant na plataforma)
```

### 4.5 Afiliado (`affiliate`) — área `/affiliate`
```
Dashboard · Gerar links · Materiais · Comissões · Conta/recebimento · Sair
(escopo isolado; não acessa /manage nem /app de gestão)
```

### 4.6 Super-Admin — `admin.app.com`
```
Dashboard global · Tenants · Planos do SaaS · Faturamento · Provisionamento · Auditoria · Observabilidade · Super-admins
(control plane; acesso a dados de tenant só via impersonação auditada)
```

### 4.7 Visitante (não autenticado) — site público do tenant
```
Home · Cursos · [landing do curso] · Entrar · Cadastrar · Seja afiliado
(rota pública adicional: Verificação de certificado)
```

---

## 5. Estrutura de URLs

### 5.1 Hosts (roteamento por tenant)

| Host | Resolve para | Observação |
|------|--------------|------------|
| `app.com`, `www.app.com` | Site da plataforma (A) | Marketing/checkout do SaaS |
| `admin.app.com` | Painel Super-Admin (F) | Control plane (`platform`), MFA |
| `{slug}.app.com` | Tenant (B/C/D/E) | `slug` resolvido em `platform.tenants` |
| `{custom_domain}` [F2] | Tenant | CNAME + SSL automático; mapeado em `tenants.custom_domain` |

> **Resolução:** o subdomínio/domínio **sugere** o tenant; o **claim do JWT/sessão (`tenant_id`)** é a autoridade final (impede troca de tenant por header). Hook `onRequest` valida e aplica `search_path` via `withTenant`.

### 5.2 Prefixos de área dentro do tenant

| Prefixo | Área | Auth |
|---------|------|------|
| `/` (raiz e rotas públicas) | Site público (B) | público |
| `/app/*` | Área do aluno (C) | student |
| `/manage/*` | Painel admin/instrutor (D) | owner/admin/instructor |
| `/affiliate/*` | Painel do afiliado (E) | affiliate |

### 5.3 Padrões de URL (exemplos)

```
Público do tenant
  https://acme.app.com/
  https://acme.app.com/cursos
  https://acme.app.com/curso/typescript-pro
  https://acme.app.com/curso/typescript-pro?ref=JOAO10&utm_source=ig
  https://acme.app.com/checkout/typescript-pro
  https://acme.app.com/certificados/verificar/9f3c-...-uuid
  https://acme.app.com/entrar
  https://acme.app.com/aceitar-convite/{token}

Aluno
  https://acme.app.com/app
  https://acme.app.com/app/curso/typescript-pro
  https://acme.app.com/app/certificados
  https://acme.app.com/app/conta/assinatura

Admin/Instrutor
  https://acme.app.com/manage
  https://acme.app.com/manage/cursos/{courseId}/editor
  https://acme.app.com/manage/cursos/{courseId}/aula/{lessonId}
  https://acme.app.com/manage/financeiro/pedidos
  https://acme.app.com/manage/afiliados/comissoes
  https://acme.app.com/manage/configuracoes/marca

Afiliado
  https://acme.app.com/affiliate
  https://acme.app.com/affiliate/links
  https://acme.app.com/affiliate/comissoes

Plataforma / Super-Admin
  https://app.com/precos
  https://app.com/precos/checkout
  https://admin.app.com/tenants
  https://admin.app.com/tenants/{tenantId}/provisionamento
  https://admin.app.com/auditoria
```

### 5.4 Convenções de URL

- **Recursos por `slug`** quando voltados ao público/SEO (`/curso/{slug}`); por **`id` (uuid)** em rotas de gestão (`/manage/cursos/{id}`).
- **kebab-case** em segmentos legíveis; sem extensões.
- **Idempotência de SEO:** landings públicas com canonical; UTMs preservados; `?ref=` apenas para atribuição (não altera canonical).
- **i18n [F2]:** estratégia via next-intl (locale por cookie/header; sem prefixo de path no MVP, PT-BR único).
- **API** (consumida pelo front, não navegável): versionada sob `/api/*` no backend Fastify, contratos Zod/OpenAPI — fora deste sitemap de UI.

---

## Dependências e pontos para o coordenador

1. **Host do Super-Admin (`admin.app.com`):** o ARCHITECTURE.md cita subdomínios de tenant mas não fixa o host do control plane. Proposta: host dedicado. **Coordenação: marcado como PROPOSTA A VALIDAR COM ENGENHARIA** (DNS, cookies/sessão, isolamento) — ver [OPEN_QUESTIONS #23](../OPEN_QUESTIONS.md).
2. **Prefixos `/app`, `/manage`, `/affiliate`:** convenção proposta aqui (não definida nos docs). **PROPOSTA A VALIDAR COM ENGENHARIA** (middleware Next.js de roteamento por área + RBAC) — [OPEN_QUESTIONS #23](../OPEN_QUESTIONS.md).
3. **Separação Instrutor × Admin no menu (§4.2/4.3):** o RBAC tem os papéis, mas a fronteira exata de telas visíveis ao instrutor (ex.: vê financeiro? só dos próprios cursos?) precisa de decisão de produto.
4. **Owner × Admin:** apenas o `owner` vê billing do SaaS e pagamentos — confirmar se há mais de um owner e regras de transferência de propriedade.
5. **Domínio próprio [F2] e cookies:** sessão/Better-Auth precisa funcionar tanto em `{slug}.app.com` quanto em `custom_domain` — definir estratégia de cookie/domínio.
6. **Telas de "oferta" para order bump/upsell [F2]:** dependem de modelagem ainda inexistente no DATA_MODEL (entidade de ofertas/bumps). Inventário inclui as telas, mas dados pendentes.
7. **Template de certificado:** MVP tem template básico; customização é [F3]. Confirmar onde fica a edição (Config — Certificado) e o que é editável no MVP.
8. **Página "Tenant indisponível":** comportamento quando `status='suspended'/'cancelled'` (owner vê billing; demais veem bloqueio) — confirmar copy e fluxo.
9. **Catálogo público:** confirmar se todo tenant tem vitrine pública (`/cursos`) ou se alguns operam só por landings diretas/links de afiliado (configurável).
10. **Navegabilidade de API/OpenAPI:** definir se haverá portal de documentação da API pública [F2] sob host/área própria (ex.: `developers.app.com` ou `/manage/integracoes/api/docs`).

> Próximos artefatos sugeridos: mapa de componentes shadcn por tela crítica, especificação de breadcrumbs/estado vazio por área, e a malha de permissões (matriz papel × tela × ação) para alimentar os guards RBAC do Fastify.
