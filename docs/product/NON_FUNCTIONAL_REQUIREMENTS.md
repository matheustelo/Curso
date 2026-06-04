# Requisitos Não-Funcionais (NFR) — Plataforma de Cursos SaaS Multitenant

- **Versão:** 1.0
- **Data:** 2026-06-04
- **Status:** Proposto para revisão
- **Documentos relacionados:** [PRD.md](../PRD.md) · [ARCHITECTURE.md](../ARCHITECTURE.md) · [ROADMAP.md](../ROADMAP.md) · [DATA_MODEL.md](../DATA_MODEL.md) · [CLAUDE.md](../../CLAUDE.md)

> **Escopo deste documento.** Define os requisitos de **qualidade** (não-funcionais) da plataforma sob a
> perspectiva do **produto e do usuário final**, com **metas mensuráveis** e **checklists verificáveis**.
> Cada requisito recebe um ID estável (`NFR-<área>-NN`) para rastreio em backlog, testes e CI.
> Notação de fase do [ROADMAP](../ROADMAP.md): **[MVP]** · **[F2]** · **[F3]**.

---

## Índice

1. [Personas e telas de referência](#1-personas-e-telas-de-referência)
2. [Performance e Core Web Vitals](#2-performance-e-core-web-vitals)
3. [Acessibilidade (WCAG 2.1 AA)](#3-acessibilidade-wcag-21-aa)
4. [Internacionalização (i18n) e localização](#4-internacionalização-i18n-e-localização)
5. [Privacidade / LGPD na perspectiva do usuário](#5-privacidade--lgpd-na-perspectiva-do-usuário)
6. [Confiabilidade e disponibilidade](#6-confiabilidade-e-disponibilidade)
7. [Segurança aplicada à UX](#7-segurança-aplicada-à-ux)
8. [Compatibilidade, responsividade e PWA](#8-compatibilidade-responsividade-e-pwa)
9. [Observabilidade do usuário](#9-observabilidade-do-usuário)
10. [Matriz consolidada de metas (resumo executivo)](#10-matriz-consolidada-de-metas-resumo-executivo)
11. [Como medimos e validamos (gates de CI e monitoração)](#11-como-medimos-e-validamos-gates-de-ci-e-monitoração)
12. [Dependências e pontos para o coordenador](#12-dependências-e-pontos-para-o-coordenador)

---

## 1. Personas e telas de referência

Os NFRs são aplicados por **tipo de tela**, pois as exigências mudam conforme o contexto de uso e a
persona predominante.

| # | Tela / superfície | Personas | Renderização (ARCHITECTURE §5) | Criticidade NFR |
|---|-------------------|----------|-------------------------------|-----------------|
| T1 | **Landing / página de vendas** (pública) | Visitante, Aluno | SSG/ISR | Performance + SEO + Acessibilidade |
| T2 | **Catálogo do tenant** (público) | Visitante, Aluno | SSG/ISR | Performance + paginação |
| T3 | **Checkout** | Aluno | SSR + Client | Performance + Confiabilidade + Segurança |
| T4 | **Player / sala de aula** | Aluno | RSC + Client (player) | Performance (TTF-Play) + Acessibilidade + Proteção de conteúdo |
| T5 | **Dashboard do aluno** | Aluno | RSC/SSR | Performance + Acessibilidade |
| T6 | **Estúdio do instrutor** (criação de curso, upload) | Instrutor | SSR + Client | Confiabilidade (upload) + listas grandes |
| T7 | **Admin do tenant** (cursos, alunos, financeiro, afiliados) | Admin, Afiliado | SSR + Client | Listas grandes + Acessibilidade |
| T8 | **Painel Super-Admin** (control plane) | Super-Admin | SSR + Client | Segurança (impersonação auditada) + Observabilidade |

> **Regra transversal de tenant (CLAUDE.md §1):** nenhum NFR pode justificar bypass de `withTenant`. Cache,
> CDN, edge e pré-render **DEVEM** respeitar o isolamento por tenant — chave de cache sempre inclui
> `tenantId`. Vazamento cross-tenant é P0 e tem meta **zero** (PRD §6).

---

## 2. Performance e Core Web Vitals

**Objetivo:** experiência fluida em rede 4G mediana de aluno brasileiro (device mid-tier Android), sem
sacrificar a percepção de "produto premium" do white-label.

### 2.1 Budgets de Core Web Vitals (p75, campo real — RUM)

Metas medidas no **percentil 75** de usuários reais (RUM via PostHog/Sentry), por tipo de tela. Alinhadas
ao "good" do Google e endurecidas onde o negócio exige.

| ID | Métrica | T1 Landing | T3 Checkout | T4 Player | T5/T7 Dashboards | Limite "falha" (CI) |
|----|---------|-----------|-------------|-----------|------------------|---------------------|
| NFR-PERF-01 | **LCP** (carregamento) | ≤ 2,0 s | ≤ 2,5 s | ≤ 2,5 s | ≤ 2,5 s | > 4,0 s |
| NFR-PERF-02 | **INP** (interatividade) | ≤ 200 ms | ≤ 200 ms | ≤ 200 ms | ≤ 200 ms | > 500 ms |
| NFR-PERF-03 | **CLS** (estabilidade) | ≤ 0,1 | ≤ 0,1 | ≤ 0,1 | ≤ 0,1 | > 0,25 |
| NFR-PERF-04 | **TTFB** (servidor + edge) | ≤ 0,6 s | ≤ 0,8 s | ≤ 0,8 s | ≤ 0,8 s | > 1,8 s |
| NFR-PERF-05 | **FCP** | ≤ 1,5 s | ≤ 1,8 s | ≤ 1,8 s | ≤ 1,8 s | > 3,0 s |

### 2.2 Budgets de transferência (laboratório — Lighthouse CI, mobile)

| ID | Recurso | Meta (T1 Landing) | Meta (área logada) |
|----|---------|-------------------|--------------------|
| NFR-PERF-06 | JS inicial (transferido, gzip/br) | ≤ 170 KB | ≤ 250 KB |
| NFR-PERF-07 | CSS inicial | ≤ 60 KB | ≤ 80 KB |
| NFR-PERF-08 | Peso total da rota (inicial) | ≤ 1,0 MB | ≤ 1,6 MB |
| NFR-PERF-09 | Fontes web | ≤ 100 KB, `font-display: swap`, no máx. 2 famílias | idem |
| NFR-PERF-10 | Score Lighthouse (Performance, mobile) | ≥ 90 | ≥ 80 |

### 2.3 Início do player de vídeo (métrica de produto crítica)

O player é o coração da experiência (PRD §3.2). Definimos **TTF-Play** (Time To First Play): do clique em
"assistir" até o primeiro frame renderizado.

| ID | Métrica | Meta (p75) | Limite |
|----|---------|-----------|--------|
| NFR-PERF-11 | **TTF-Play** (clique → 1º frame), inclui validação de entitlement + assinatura de token | ≤ 2,0 s | > 4,0 s |
| NFR-PERF-12 | Latência da API de **assinatura de URL/token de vídeo** (backend, p95) | ≤ 300 ms | > 800 ms |
| NFR-PERF-13 | Buffering ratio durante a reprodução (tempo em rebuffer / watch-time) | < 1,0% | > 3,0% |
| NFR-PERF-14 | Troca de qualidade (ABR/HLS) sem parada visível | adaptativo automático | — |
| NFR-PERF-15 | "Retomar de onde parou" disponível ao abrir a aula | ≤ 500 ms após load | — |

> O encoding/entrega pesada fica com a Bunny (ARCHITECTURE §7); nossas metas cobrem o que controlamos:
> validação de acesso, assinatura de token (TTL curto 1–12h) e renderização do iframe/player.

### 2.4 Latência de API (backend Fastify)

| ID | Classe de endpoint | p95 | p99 | Observação |
|----|--------------------|-----|-----|-----------|
| NFR-PERF-16 | Leituras simples (GET curso, aula, perfil) | ≤ 200 ms | ≤ 400 ms | inclui `SET LOCAL search_path` por transação |
| NFR-PERF-17 | Listas paginadas (alunos, pedidos, progresso) | ≤ 400 ms | ≤ 800 ms | índices obrigatórios; ver §2.5 |
| NFR-PERF-18 | Escritas (matrícula, progresso/heartbeat, comentário) | ≤ 300 ms | ≤ 600 ms | heartbeat com throttle 5–15s (ARCHITECTURE §7) |
| NFR-PERF-19 | Checkout: criação de pedido / intenção de pagamento | ≤ 800 ms | ≤ 1,5 s | excl. tempo do gateway externo |
| NFR-PERF-20 | Webhooks de entrada (responder 200, enfileirar) | ≤ 200 ms | ≤ 500 ms | processamento pesado vai p/ worker (assíncrono) |

### 2.5 Paginação e listas grandes

Telas T6/T7 podem ter milhares de alunos/pedidos/aulas. Regras obrigatórias:

- [ ] **NFR-PERF-21** — Toda lista usa **paginação por cursor (keyset)**, não `OFFSET` grande, para
  manter latência estável independentemente da página.
- [ ] **NFR-PERF-22** — Tamanho de página padrão **20–50 itens**; máximo aceito por request: **100**.
- [ ] **NFR-PERF-23** — Listas com possível volume alto usam **virtualização** no client (render só do
  visível) e **skeletons** durante o carregamento (evita CLS).
- [ ] **NFR-PERF-24** — Busca/filtro em listas grandes é **server-side** com `debounce` ≥ 300 ms; nunca
  carregar a coleção inteira no client.
- [ ] **NFR-PERF-25** — Exportações grandes (CSV de alunos/relatórios) são **assíncronas via job**
  (pg-boss) com notificação de "pronto para download" — nunca bloqueiam a request HTTP.
- [ ] **NFR-PERF-26** — Toda coluna usada em ordenação/filtro de lista tem **índice** no schema do tenant.

### 2.6 Caching e otimização de entrega

- [ ] **NFR-PERF-27** — Páginas públicas (T1/T2) servidas via **ISR/SSG** com revalidação; chave de cache
  inclui `tenantId` + `locale`.
- [ ] **NFR-PERF-28** — Imagens via `next/image` (formatos modernos AVIF/WebP, `srcset`, lazy-loading
  abaixo da dobra; eager + `priority` no LCP).
- [ ] **NFR-PERF-29** — Code-splitting por rota; componentes pesados (player, editor de quiz, gráficos)
  carregados com `dynamic import`.
- [ ] **NFR-PERF-30** — Assets estáticos com cache imutável (`Cache-Control: immutable`, hash no nome).

---

## 3. Acessibilidade (WCAG 2.1 AA)

**Meta global:** conformidade **WCAG 2.1 nível AA** desde o MVP (PRD §3.14, tratada como baseline e não
retrofit). A11y é responsabilidade de **componente** (shadcn/ui + Radix dá base acessível) e de **tela**.

### 3.1 Checklist transversal (todos os componentes)

- [ ] **NFR-A11Y-01** — **Navegação 100% por teclado**: todo elemento interativo é alcançável por `Tab`,
  ativável por `Enter`/`Space`, ordem de foco lógica (WCAG 2.1.1, 2.1.2 sem armadilha de foco).
- [ ] **NFR-A11Y-02** — **Foco visível** sempre (anel de foco com contraste ≥ 3:1; nunca `outline:none`
  sem substituto) (WCAG 2.4.7).
- [ ] **NFR-A11Y-03** — **Contraste de cor** ≥ **4,5:1** para texto normal e ≥ **3:1** para texto grande
  e ícones/bordas funcionais (WCAG 1.4.3, 1.4.11). Vale também para temas white-label do tenant — ver §3.4.
- [ ] **NFR-A11Y-04** — **Labels/ARIA**: todo `input` tem `<label>` associado; botões só com ícone têm
  `aria-label`; estados (`aria-expanded`, `aria-selected`, `aria-invalid`) corretos (WCAG 1.3.1, 4.1.2).
- [ ] **NFR-A11Y-05** — **Texto alternativo** em imagens informativas; imagens decorativas com `alt=""`
  (WCAG 1.1.1).
- [ ] **NFR-A11Y-06** — **Estrutura semântica**: landmarks (`header`/`nav`/`main`/`footer`), hierarquia de
  headings sem pular níveis, **skip-to-content** link (WCAG 1.3.1, 2.4.1).
- [ ] **NFR-A11Y-07** — **Compatível com leitores de tela** (NVDA + Windows, VoiceOver + iOS/macOS como
  alvos de teste); mudanças dinâmicas anunciadas via `aria-live` (toasts, validações, estado de upload).
- [ ] **NFR-A11Y-08** — **Sem dependência só de cor** para transmitir informação (ex.: status de pedido
  tem ícone/texto além da cor) (WCAG 1.4.1).
- [ ] **NFR-A11Y-09** — **Zoom até 200%** sem perda de conteúdo/funcionalidade; layout responsivo
  (reflow a 320 px de largura) (WCAG 1.4.4, 1.4.10).
- [ ] **NFR-A11Y-10** — **Alvos de toque** ≥ 44×44 px em mobile.
- [ ] **NFR-A11Y-11** — `prefers-reduced-motion` respeitado (animações reduzidas/desligadas) (WCAG 2.3.3).
- [ ] **NFR-A11Y-12** — Mensagens de erro de formulário **textuais, específicas e associadas** ao campo
  (`aria-describedby`), não apenas borda vermelha (WCAG 3.3.1, 3.3.3).
- [ ] **NFR-A11Y-13** — `lang` correto no `<html>` (sincronizado com o locale — ver §4).

### 3.2 Player de vídeo / sala de aula (T4) — exigências específicas

- [ ] **NFR-A11Y-14** — **Legendas/closed captions** disponíveis e ativáveis no player (PRD §3.2). No
  MVP: legendas enviadas/manuais; na F2: **transcrição automática** (Bunny Transcribe AI) gera legenda.
- [ ] **NFR-A11Y-15** — Controles do player (play/pause, volume, velocidade, legenda, fullscreen) são
  **focáveis e operáveis por teclado**, com `aria-label` e estado anunciado.
- [ ] **NFR-A11Y-16** — Atalhos de teclado padrão (espaço = play/pause; setas = seek/volume) sem
  conflitar com leitor de tela.
- [ ] **NFR-A11Y-17** — **Transcrição textual** pesquisável da aula (F2) — beneficia surdos e SEO.
- [ ] **NFR-A11Y-18** — Player não dispara autoplay com áudio sem ação do usuário (WCAG 1.4.2).
- [ ] **NFR-A11Y-19** — **Watermark** de proteção (F2) **não pode** reduzir contraste/legibilidade do
  conteúdo nem cobrir legendas (cruza §7).

### 3.3 Por tipo de tela (foco adicional)

| Tela | Pontos de atenção a11y |
|------|------------------------|
| T1 Landing | Headings semânticos, CTA com texto descritivo, contraste em cima de imagens de fundo. |
| T3 Checkout | Campos com `autocomplete` (cc-number, etc.), erros associados, foco gerenciado entre etapas, não usar só placeholder como label. |
| T4 Player | Ver §3.2. |
| T6 Estúdio | Drag-and-drop de reordenação (PRD §3.1) com **alternativa por teclado** (mover ↑/↓) — WCAG 2.1.1. |
| T7/T8 Painéis | Tabelas com `<th scope>`, paginação anunciada, gráficos com tabela/resumo textual alternativo. |

### 3.4 Acessibilidade do white-label (tema do tenant)

- [ ] **NFR-A11Y-20** — O seletor de cores do branding do tenant (PRD §3.11) **valida contraste em tempo
  real** e **bloqueia/avisa** combinações que reprovem em AA (4,5:1). Garante que a a11y não dependa do
  bom gosto do admin.
- [ ] **NFR-A11Y-21** — Fornecemos paletas-padrão pré-aprovadas em contraste como ponto de partida.

### 3.5 Metas de auditoria

- [ ] **NFR-A11Y-22** — **0 violações críticas** no `axe-core` (automatizado em CI) nas telas-chave
  (T1, T3, T4, T5, T7).
- [ ] **NFR-A11Y-23** — Auditoria manual com leitor de tela a cada release de marco (checklist guiado).
- [ ] **NFR-A11Y-24** — Score Lighthouse Accessibility ≥ **95** nas telas-chave.

---

## 4. Internacionalização (i18n) e localização

**Princípio (PRD §3.11, ARCHITECTURE §5):** lançar em **PT-BR**, com arquitetura **pronta para ES/EN**
(F2), sem retrabalho. Lib: **next-intl**.

### 4.1 Arquitetura de i18n (já no MVP)

- [ ] **NFR-I18N-01** — **Zero strings hardcoded** na UI: todo texto vem de catálogos de mensagens
  (`messages/pt-BR.json`, futuro `es`, `en`). Gate de lint procura strings literais em JSX.
- [ ] **NFR-I18N-02** — Catálogo organizado por namespace/feature (DRY, alinhado a vertical slices).
- [ ] **NFR-I18N-03** — Suporte a **pluralização e interpolação** via ICU MessageFormat (não concatenar
  strings).
- [ ] **NFR-I18N-04** — `locale` resolvido por preferência do usuário > config do tenant > `Accept-Language`
  > default `pt-BR`; persiste na sessão/perfil.
- [ ] **NFR-I18N-05** — `<html lang>` e atributos `dir` derivados do locale ativo (sincroniza com
  NFR-A11Y-13). Estrutura preparada para RTL no futuro (não usar margens "left/right" hardcoded; preferir
  propriedades lógicas CSS).
- [ ] **NFR-I18N-06** — Locale na **URL** (ex.: `/pt-BR/...`) ou estratégia equivalente que permita SEO
  multi-idioma com `hreflang` (F2).

### 4.2 Formatação e localização

- [ ] **NFR-I18N-07** — **Moeda**: usar `Intl.NumberFormat` (BRL no MVP: `R$ 1.234,56`). Nunca formatar
  manualmente. Valor monetário trafega como **inteiro em centavos** + código de moeda (evita float).
- [ ] **NFR-I18N-08** — **Datas/horas**: `Intl.DateTimeFormat`; armazenar em **UTC**, exibir no fuso do
  usuário (default `America/Sao_Paulo`). Formato BR: `dd/mm/aaaa`.
- [ ] **NFR-I18N-09** — Números, percentuais (progresso, notas) e contagens via `Intl`.
- [ ] **NFR-I18N-10** — E-mails transacionais (Resend) e certificados (PDF) também localizados pelo locale
  do destinatário.

### 4.3 Conteúdo multi-idioma do tenant (futuro — F2/F3)

- [ ] **NFR-I18N-11** — O **modelo de dados** prevê conteúdo do tenant (título/descrição de curso, aula)
  em múltiplos idiomas no futuro (campos localizáveis), mesmo que o MVP só exponha PT-BR. Evita migração
  destrutiva. _(Coordenar com DATA_MODEL — ver §12.)_
- [ ] **NFR-I18N-12** — Distinção clara entre **idioma da plataforma/UI** (i18n nosso) e **idioma do
  conteúdo do curso** (definido pelo tenant/instrutor).

---

## 5. Privacidade / LGPD na perspectiva do usuário

**Base (PRD §3.9, §4, ARCHITECTURE §3.6):** consentimento, exportação, deleção/anonimização já no MVP.
Aqui detalhamos **como aparece para o usuário** e os direitos do titular sob a LGPD.

### 5.1 Consentimento e transparência

- [ ] **NFR-LGPD-01** — **Banner de cookies/consentimento** no primeiro acesso (por tenant), com opções
  granulares: **Necessários** (sempre ativos), **Analytics**, **Marketing/Pixels**. Aceitar/Rejeitar/
  Personalizar com **igual proeminência** (rejeitar não pode ser escondido).
- [ ] **NFR-LGPD-02** — Scripts de **analytics/pixels (Meta, GA4 — PRD §3.8) só carregam após
  consentimento** da categoria correspondente (consent gating real, não cosmético).
- [ ] **NFR-LGPD-03** — Registro do consentimento (timestamp, versão da política, escopo) auditável; o
  usuário pode **revisar e alterar** sua escolha a qualquer momento (link no rodapé/perfil).
- [ ] **NFR-LGPD-04** — **Política de Privacidade** e **Termos de Uso** acessíveis em todas as páginas
  (rodapé) e linkados no checkout e no cadastro, em linguagem clara.
- [ ] **NFR-LGPD-05** — No cadastro/checkout, coleta de consentimento explícito para tratamento de dados;
  comunicações de marketing exigem **opt-in separado** (não pré-marcado).

### 5.2 Direitos do titular (autoatendimento na perspectiva do usuário)

- [ ] **NFR-LGPD-06** — **Exportação de dados** (portabilidade): o aluno solicita e recebe seus dados
  pessoais + progresso/certificados em formato legível por máquina (JSON/CSV), via job assíncrono com
  link seguro de download e expiração. Meta: disponibilizado em **≤ 72h** (idealmente self-service imediato).
- [ ] **NFR-LGPD-07** — **Exclusão / direito ao esquecimento**: o usuário solicita exclusão da conta;
  fluxo com **confirmação clara** das consequências (perda de acesso a cursos/certificados). Execução por
  **deleção ou anonimização** dentro do schema do tenant (ARCHITECTURE §3.6).
- [ ] **NFR-LGPD-08** — Quando a exclusão total conflita com retenção legal (ex.: registros fiscais/
  financeiros de compras), aplica-se **anonimização** dos dados pessoais preservando o registro
  transacional mínimo exigido por lei — explicado ao usuário.
- [ ] **NFR-LGPD-09** — Confirmação por e-mail e prazo de atendimento comunicado; **SLA ≤ 15 dias**
  (alinhado a expectativa LGPD) com meta interna de **≤ 72h**.
- [ ] **NFR-LGPD-10** — **Edição de dados do perfil** (retificação) self-service.

### 5.3 Retenção e minimização

- [ ] **NFR-LGPD-11** — Política de **retenção** documentada e visível: por quanto tempo cada categoria de
  dado é mantida (sessões, logs, dados de progresso, dados financeiros).
- [ ] **NFR-LGPD-12** — **Minimização**: coletar apenas o necessário. CPF só quando exigido (checkout/
  nota fiscal/watermark); não solicitar dados sensíveis sem finalidade.
- [ ] **NFR-LGPD-13** — Logs e observabilidade (§9) **não** persistem dados pessoais além do necessário;
  retenção de logs limitada e configurável.

### 5.4 Watermark e IA — visibilidade ao usuário

- [ ] **NFR-LGPD-14** — O **watermark dinâmico** (F2, e-mail/CPF sobre o vídeo — PRD §3.16) é uma medida
  anti-pirataria que expõe dado pessoal do próprio aluno **para ele mesmo**; deve ser **informado nos
  Termos** ("seu identificador pode ser sobreposto ao vídeo para proteção de conteúdo").
- [ ] **NFR-LGPD-15** — Recursos de **IA** (F2/F3: transcrição, geração de quiz, tutor IA — PRD §3.17)
  têm aviso de uso de IA; dados enviados a provedores de IA respeitam a finalidade e o consentimento; o
  aluno é informado quando interage com IA.
- [ ] **NFR-LGPD-16** — **Impersonação** do Super-Admin (PRD §4.7) exibe **banner visível** ao operar como
  outro usuário e é **auditada**; do ponto de vista de privacidade, é acesso registrado e justificável.

### 5.5 Isolamento percebido (confiança white-label)

- [ ] **NFR-LGPD-17** — Para o usuário, cada tenant é um ambiente isolado: dados, e-mails e branding nunca
  se misturam. Mensagens/telas nunca expõem existência de outros tenants. (Reforça PRD §4.1, CLAUDE.md §1.)

---

## 6. Confiabilidade e disponibilidade

### 6.1 Metas de disponibilidade (SLO)

| ID | Serviço | SLO de uptime mensal | Erro budget |
|----|---------|----------------------|-------------|
| NFR-REL-01 | App web + API (área logada, player, checkout) | **99,9%** | ~43 min/mês |
| NFR-REL-02 | Páginas públicas (landing/catálogo, ISR) | **99,95%** | ~22 min/mês |
| NFR-REL-03 | Processamento assíncrono (worker/pg-boss: e-mails, certificados, webhooks) | **99,5%** (com retry) | — |
| NFR-REL-04 | Entrega de vídeo | herda SLA da Bunny (CDN) — fora do nosso controle direto |

> O **checkout** e o **webhook de pagamento** são caminhos críticos de receita: priorizados em
> resiliência (idempotência + reconciliação — PRD §4.3).

### 6.2 Comportamento em falhas (graceful degradation)

- [ ] **NFR-REL-05** — **Vídeo indisponível** (Bunny fora/encoding pendente): a sala de aula mostra estado
  amigável ("Este vídeo está sendo processado / temporariamente indisponível, tente novamente em
  instantes"), **não** tela branca ou stack trace; demais conteúdos da aula (texto/anexos) seguem
  acessíveis.
- [ ] **NFR-REL-06** — **Gateway de pagamento fora** (Pagar.me/Asaas): checkout informa indisponibilidade,
  oferece **método alternativo** quando possível e preserva o carrinho; nunca cobra em duplicidade
  (idempotência de criação de pedido).
- [ ] **NFR-REL-07** — **Falha de rede no progresso/heartbeat**: o player **não interrompe** a aula;
  buffer local e reenvio quando a rede voltar (perda de progresso tolerável, nunca perda da sessão).
- [ ] **NFR-REL-08** — **Upload de vídeo do instrutor** (TUS, ARCHITECTURE §7) é **resumível**: queda de
  rede retoma de onde parou, sem reiniciar.
- [ ] **NFR-REL-09** — **Páginas de erro amigáveis** (404, 403, 500) com branding do tenant, linguagem
  humana, ação de saída ("voltar ao início", "falar com suporte") — **sem** detalhes técnicos/stack.
- [ ] **NFR-REL-10** — Estados de **loading** (skeletons), **vazio** (empty states com call-to-action) e
  **erro** previstos em **toda** lista/tela de dados.

### 6.3 Retry, idempotência e consistência

- [ ] **NFR-REL-11** — Chamadas a serviços externos (gateway, Bunny, Resend) usam **retry com backoff
  exponencial + jitter** e **timeout** definido; falha definitiva vira job de retry/dead-letter.
- [ ] **NFR-REL-12** — **Webhooks de entrada idempotentes** (chave do evento) — reprocessar não duplica
  matrícula/cobrança (PRD §4.3, ARCHITECTURE §8).
- [ ] **NFR-REL-13** — **Reconciliação periódica** pagamento↔acesso corrige divergências (job agendado) —
  o usuário nunca fica "pago mas sem acesso" por mais que o intervalo de reconciliação.
- [ ] **NFR-REL-14** — Ações de mutação no front (criar pedido, publicar) protegidas contra
  **duplo-clique** (botão desabilita + idempotency key).
- [ ] **NFR-REL-15** — **RPO ≤ 24h / RTO ≤ 4h**: backups diários do PostgreSQL com PITR; restauração
  testada periodicamente (cruza com infraestrutura — ARCHITECTURE §12).

### 6.4 Comunicação de incidentes

- [ ] **NFR-REL-16** — Mensagens de erro nunca culpam o usuário, oferecem próxima ação e **ID de
  correlação** (requestId) para suporte (cruza §9).

---

## 7. Segurança aplicada à UX

> Foco aqui é a **experiência de segurança** (o que o usuário vê/sente). Mecanismos profundos (HMAC,
> cifragem de keys, search_path) estão em ARCHITECTURE §3/§6/§7/§8.

### 7.1 Sessão e autenticação

- [ ] **NFR-SEC-01** — Sessões via **cookies HttpOnly + Secure + SameSite** (Better-Auth, ARCHITECTURE §6);
  token nunca exposto a JS no front.
- [ ] **NFR-SEC-02** — **Expiração de sessão** com renovação transparente (sliding) e **logout** explícito
  que invalida a sessão; aviso amigável ao expirar ("sua sessão expirou, entre novamente") preservando o
  destino para retorno pós-login.
- [ ] **NFR-SEC-03** — **Logout de todos os dispositivos** disponível no perfil; lista de sessões ativas
  (F2, cruza limite de dispositivos).
- [ ] **NFR-SEC-04** — Mudança de senha/e-mail exige **reautenticação**; notifica o usuário por e-mail
  após alteração de credenciais (detecção de invasão).

### 7.2 Proteção contra abuso

- [ ] **NFR-SEC-05** — **Rate limiting / limite de tentativas** no login, recuperação de senha e checkout;
  após N tentativas, **bloqueio temporário progressivo** e/ou desafio (CAPTCHA) — mensagem clara, sem
  revelar se o e-mail existe (anti-enumeração).
- [ ] **NFR-SEC-06** — Recuperação de senha com **token de uso único e expiração curta**; resposta
  genérica ("se o e-mail existir, enviaremos instruções").
- [ ] **NFR-SEC-07** — Política de **senha forte** com feedback de força em tempo real; bloqueio de senhas
  vazadas conhecidas (lista comum) quando viável.

### 7.3 2FA / MFA

- [ ] **NFR-SEC-08** — **2FA (TOTP)** disponível para papéis de alto privilégio — **Super-Admin
  obrigatório**, **Admin/Owner do tenant recomendado/obrigatório por política** (MVP para Super-Admin;
  expansão F2). Aluno: opcional. Fluxo de recuperação com códigos de backup.
- [ ] **NFR-SEC-09** — **Impersonação** do Super-Admin sempre com 2FA + banner visível + auditoria
  (cruza NFR-LGPD-16).

### 7.4 Proteção de conteúdo (perspectiva do usuário)

- [ ] **NFR-SEC-10** — **URLs de vídeo assinadas com expiração curta** (TTL 1–12h, ARCHITECTURE §7): o
  usuário não percebe, mas links copiados **expiram** e não funcionam fora da sessão autorizada.
- [ ] **NFR-SEC-11** — Acesso ao vídeo **sempre revalida entitlement** (matrícula/pagamento ativo) antes
  de assinar — aluno sem acesso vê mensagem de "compre/matricule-se", não o vídeo.
- [ ] **NFR-SEC-12** — **MP4 progressivo desabilitado**, restrição de **referer**, **MediaCage Basic**
  (MVP); **watermark por aluno** (F2) — comunicados como proteção, ver NFR-LGPD-14.
- [ ] **NFR-SEC-13** — **Limite de sessões/dispositivos simultâneos** (F2, PRD §3.16): ao exceder, o
  usuário recebe aviso claro ("limite de dispositivos atingido; desconecte outro para continuar"),
  nunca bloqueio silencioso. Limite por tenant/plano (coordenar quota — ver §12).

### 7.5 Higiene web e dados

- [ ] **NFR-SEC-14** — **HTTPS obrigatório** em tudo (HSTS); domínio próprio do tenant (F2) com SSL
  automático (PRD §3.11).
- [ ] **NFR-SEC-15** — Cabeçalhos de segurança: **CSP**, `X-Content-Type-Options`, `Referrer-Policy`,
  `Permissions-Policy`; CSP compatível com iframe do player Bunny e pixels consentidos.
- [ ] **NFR-SEC-16** — Proteção **CSRF** em mutações; **sanitização** de conteúdo rico (texto de aula,
  comentários) contra XSS.
- [ ] **NFR-SEC-17** — **Chaves Bunny/pagamento nunca chegam ao frontend** (CLAUDE.md anti-padrões;
  ARCHITECTURE §7) — uploads via pre-signed/TUS, tokens gerados no backend.
- [ ] **NFR-SEC-18** — Formulários de pagamento usam **tokenização/iframe do gateway** (dados de cartão
  não trafegam/armazenam em nossos servidores — escopo PCI reduzido).

---

## 8. Compatibilidade, responsividade e PWA

### 8.1 Navegadores e dispositivos suportados

- [ ] **NFR-COMP-01** — Suporte às **2 últimas versões** de Chrome, Edge, Firefox, Safari (desktop e
  mobile) e **Samsung Internet**. Sem suporte a IE11.
- [ ] **NFR-COMP-02** — **Mobile-first**: design e teste primários em viewport mobile (≥ 320 px);
  breakpoints para tablet e desktop. Tráfego de alunos é majoritariamente mobile.
- [ ] **NFR-COMP-03** — **Responsividade** sem scroll horizontal em qualquer largura ≥ 320 px (cruza
  NFR-A11Y-09 reflow).
- [ ] **NFR-COMP-04** — Player de vídeo funcional em **iOS Safari** (HLS nativo) e Android Chrome,
  incluindo fullscreen e legendas.
- [ ] **NFR-COMP-05** — Funcionalidade essencial **sem JS** apenas para conteúdo público crítico de SEO
  (landing renderiza conteúdo via SSG); área logada pode exigir JS (degradação aceitável).
- [ ] **NFR-COMP-06** — **Graceful degradation** de recursos modernos (AVIF→WebP→JPEG; etc.).

### 8.2 PWA (F2)

- [ ] **NFR-COMP-07** — **PWA instalável** (manifest + ícones por tenant/branding + service worker),
  conforme PRD §3.10.
- [ ] **NFR-COMP-08** — **Web Push** (F2) com permissão solicitada em momento contextual (não no primeiro
  segundo), respeitando consentimento.
- [ ] **NFR-COMP-09** — Service worker faz **cache de shell/estáticos** para reabertura rápida; **não**
  cacheia conteúdo de vídeo protegido nem dados sensíveis de outro tenant.
- [ ] **NFR-COMP-10** — Estado **offline** mínimo: tela informativa amigável quando sem conexão (download
  offline seguro de aulas é F3, PRD §3.2).

---

## 9. Observabilidade do usuário

**Base:** Sentry (front+back), Pino, OpenTelemetry, PostHog (ARCHITECTURE §10).

- [ ] **NFR-OBS-01** — **Sentry** captura erros de front e back com **`tenantId` e `requestId`** como tags
  (correlação ponta-a-ponta) — mas **sem PII** no corpo do evento.
- [ ] **NFR-OBS-02** — **Scrubbing/PII redaction** obrigatório antes do envio: e-mail, CPF, tokens,
  cookies, dados de cartão e Authorization são removidos/mascarados (cruza NFR-LGPD-13).
- [ ] **NFR-OBS-03** — **Session Replay** (se adotado) com **masking** total de inputs e texto sensível
  por padrão; só ativado com consentimento de analytics (NFR-LGPD-02).
- [ ] **NFR-OBS-04** — **RUM (Real User Monitoring)** coleta Core Web Vitals (§2.1) por tipo de tela e por
  tenant, alimentando os SLOs de performance — respeitando consentimento.
- [ ] **NFR-OBS-05** — Mensagem de erro ao usuário inclui um **ID de correlação** copiável ("informe o
  código XYZ ao suporte") que mapeia para o trace/log no backend, sem expor detalhes internos
  (cruza NFR-REL-16).
- [ ] **NFR-OBS-06** — **Alertas** sobre orçamentos: queda de SLO (§6.1), regressão de Web Vitals (§2.1),
  pico de erros 5xx, falha de webhook de pagamento. Alerta de **custo de egress de vídeo** (risco PRD §7).
- [ ] **NFR-OBS-07** — Logs estruturados (Pino) **nunca** registram segredos nem dados sensíveis; níveis
  de log configuráveis; retenção limitada.
- [ ] **NFR-OBS-08** — Telemetria de produto (PostHog) usa **identificadores pseudonimizados** quando
  possível e respeita o opt-out de analytics.

---

## 10. Matriz consolidada de metas (resumo executivo)

| Área | Métrica-chave | Meta | Gate |
|------|---------------|------|------|
| Performance | LCP p75 (landing) | ≤ 2,0 s | RUM + Lighthouse CI |
| Performance | INP p75 | ≤ 200 ms | RUM |
| Performance | TTF-Play (player) | ≤ 2,0 s p75 | RUM custom |
| Performance | API leitura p95 | ≤ 200 ms | k6/teste de carga |
| Performance | Listas grandes | keyset, ≤ 100/página, virtualização | revisão + perf test |
| Acessibilidade | Conformidade | WCAG 2.1 AA | axe-core CI (0 críticas) + auditoria manual |
| Acessibilidade | Lighthouse a11y | ≥ 95 | Lighthouse CI |
| i18n | Strings externalizadas | 100% (0 hardcoded) | lint gate |
| LGPD | Direitos do titular | exportação + exclusão self-service | teste e2e + revisão jurídica |
| LGPD | Consent gating | analytics/pixels só pós-consentimento | teste e2e |
| Confiabilidade | Uptime app/API | 99,9% | monitor de SLO |
| Confiabilidade | RPO/RTO | ≤ 24h / ≤ 4h | teste de restore |
| Segurança | Sessão | HttpOnly+Secure+SameSite, rate limit, 2FA super-admin | security review |
| Segurança | Conteúdo | URL assinada TTL curto + entitlement | teste de acesso |
| Compatibilidade | Suporte | 2 últimas versões dos principais browsers, mobile-first | matriz de teste |
| Observabilidade | Erros | Sentry com tenantId/requestId, sem PII | revisão de config |

---

## 11. Como medimos e validamos (gates de CI e monitoração)

- **Performance (lab):** **Lighthouse CI** em PR para T1/T3/T4/T5 com budgets (§2.2) — falha o build se
  estourar.
- **Performance (campo):** **RUM** (PostHog/Sentry Performance) com dashboards de Web Vitals por tela e
  tenant; alertas de regressão (NFR-OBS-06).
- **Carga/latência de API:** **k6** (ou similar) com cenários de player (assinatura de token), checkout e
  listas grandes; valida §2.4/§2.5.
- **Acessibilidade:** **axe-core** automatizado (Playwright + `@axe-core/playwright`) como **gate de CI**
  nas telas-chave; auditoria manual com leitor de tela por marco; Lighthouse a11y.
- **i18n:** lint que detecta strings literais em JSX; verificação de chaves de tradução faltantes (catálogo
  PT-BR completo; placeholders para ES/EN).
- **LGPD:** testes e2e dos fluxos de consentimento, exportação e exclusão; checklist de revisão por
  release; **isolamento cross-tenant** já é gate (CLAUDE.md/ARCHITECTURE §11).
- **Confiabilidade:** monitor de SLO + alertas; teste periódico de restore (RPO/RTO); testes de
  idempotência de webhook (já em ARCHITECTURE §11).
- **Segurança:** security review de PR; teste de rate limit; teste de expiração de token de vídeo; verificar
  que keys não vazam ao front (gate de revisão).

---

## 12. Dependências e pontos para o coordenador

Pontos que **cruzam fronteiras** de outros documentos/áreas e exigem decisão ou alinhamento:

1. **DATA_MODEL — campos localizáveis (NFR-I18N-11/12):** decidir já no MVP se o conteúdo do tenant
   (título/descrição/aula) terá colunas/estrutura preparadas para multi-idioma (F2/F3), para evitar
   migração destrutiva depois. Pequeno custo agora, grande economia futura.

2. **Quotas por plano e limite de dispositivos (NFR-SEC-13):** o "limite de sessões/dispositivos
   simultâneos" (F2) precisa ser um **parâmetro de plano/quota** no control plane (PRD §3.12). Coordenar
   com o doc de planos/billing.

3. **TTF-Play e custo de egress (NFR-PERF-11/13 vs PRD §7):** metas de performance de vídeo podem
   conflitar com o **cap de resolução** usado para conter custo de egress da Bunny. Coordenar a política
   de qualidade adaptativa com a decisão de custo (OPEN_QUESTIONS).

4. **Política de retenção concreta (NFR-LGPD-11):** os **prazos** de retenção por categoria de dado
   (logs, progresso, financeiro) precisam de validação **jurídica/LGPD** e devem virar tabela oficial
   (sugere-se documento `docs/legal/` ou seção dedicada). Aqui ficou o requisito; falta o número.

5. **Conflito watermark × acessibilidade (NFR-A11Y-19 vs NFR-SEC-12):** o watermark dinâmico (F2) não pode
   degradar legibilidade/legendas; precisa de spec visual (opacidade, posição, contraste) revisada por UX
   e a11y juntos.

6. **2FA obrigatório para Admin/Owner do tenant (NFR-SEC-08):** definir se é **obrigatório** ou
   **recomendado** — decisão de produto/segurança com impacto em onboarding/fricção. Proposta: obrigatório
   para Super-Admin (MVP) e Owner do tenant; recomendado para Admin; opcional para aluno.

7. **SLOs e infraestrutura (NFR-REL-01..04, RPO/RTO):** as metas de uptime/backup dependem da escolha final
   de hospedagem (Fly.io/Railway/Neon/Supabase — ARCHITECTURE §12). Confirmar se o provedor gerenciado
   atende PITR e a janela RTO ≤ 4h.

8. **Stack de consentimento (NFR-LGPD-01/02):** decidir entre CMP próprio (controle total, mais trabalho)
   vs CMP de terceiro. Impacta consent gating de PostHog/Meta/GA4 e a integração com Sentry Replay.

9. **Definição de "tela-chave" para gates de CI:** confirmar a lista T1/T3/T4/T5/T7 como conjunto mínimo
   auditado em a11y/perf automaticamente, para dimensionar o esforço de CI.
