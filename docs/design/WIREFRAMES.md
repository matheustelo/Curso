# Wireframes (baixa fidelidade) — Telas Críticas

- **Versão:** 1.0 · **Data:** 2026-06-07
- **Autor:** Product Design (Wireframing & Flows)
- **Status:** Proposta para revisão de produto/engenharia
- **Relacionados (fonte de verdade de termos/estados):**
  [README de produto (glossário)](../product/README.md) ·
  [INFORMATION_ARCHITECTURE](../product/INFORMATION_ARCHITECTURE.md) ·
  [USER_FLOWS](../product/USER_FLOWS.md) ·
  [LEARNING_EXPERIENCE_UX](../product/LEARNING_EXPERIENCE_UX.md) ·
  [AUTHORING_UX](../product/AUTHORING_UX.md) ·
  [LIVE_CLASSES §4](../product/LIVE_CLASSES.md) ·
  [MONETIZATION](../product/MONETIZATION.md) ·
  [RBAC_MATRIX](../product/RBAC_MATRIX.md)

> **Escopo.** Wireframes textuais (blocos ASCII) + anotações de comportamento, estados e responsividade
> das telas críticas. **Não** são pixel-perfect; o objetivo é estrutura clara e implementável.
>
> **Convenções herdadas (não reinventadas):**
> - **Estados sempre presentes** por superfície: **vazio · carregando (skeleton, nunca spinner solto em
>   área grande) · erro (mensagem + ação de recuperação + ID de correlação) · sucesso/conteúdo**
>   (LEARNING_EXPERIENCE_UX §1).
> - **Termos/`status` canônicos** do [glossário](../product/README.md §2): `enrollments`
>   (`active|suspended|refunded|expired`), `orders` (`pending|paid|refunded|chargeback`),
>   `lesson_progress` (`not_started|in_progress|completed`), `video_status`
>   (`none|queued|processing|ready|failed`), `tenants` (`provisioning|active|suspended|cancelled`),
>   `live_sessions.status` (`scheduled|lobby|live|ended|canceled`). **Anti-seek**, **drip**,
>   **entitlement**, **take rate**, **last-click** conforme glossário.
> - **Papéis:** SA · OW · AD · IN · AF · ST (README §2.1). Ações por papel respeitam a
>   [RBAC_MATRIX](../product/RBAC_MATRIX.md).
> - **Branding por tenant** via CSS variables (nunca cor hardcoded). **Mobile-first**, **WCAG 2.1 AA**
>   (teclado, foco visível, `aria-live`, alvos ≥44px).
>
> **Design System (DS).** Ainda não há doc dedicado de DS; as referências de componente usam a base
> **shadcn/ui** (proposta da IA §coordenador). Componentes citados como `‹DS: Button›`, `‹DS: Card›`,
> `‹DS: Tabs›`, `‹DS: Sheet/Drawer›`, `‹DS: Dialog›`, `‹DS: Table (DataTable)›`, `‹DS: Accordion›`,
> `‹DS: Skeleton›`, `‹DS: Badge›`, `‹DS: Toast (sonner)›`, `‹DS: Input/Select/Combobox›`,
> `‹DS: Progress›`, `‹DS: Avatar›`, `‹DS: Tooltip›`, `‹DS: Breadcrumb›`, `‹DS: EmptyState›` (padrão a
> consolidar no DS). **Pendência registrada no fim do doc.**
>
> **Legenda dos blocos:** `[ Botão ]` ação · `(•)`/`( )` rádio · `[x]`/`[ ]` checkbox · `▓▒` barra de
> progresso · `⏳` carregando/processando · `🔒` bloqueado · `✓` concluído · `▾/▸` accordion ·
> `« nota »` anotação de comportamento.

---

## Índice

1. [Landing / página de vendas do curso (público)](#1-landing--página-de-vendas-do-curso-público)
2. [Catálogo & "Meus cursos" do aluno](#2-catálogo--meus-cursos-do-aluno)
3. [Sala de aula / Player](#3-sala-de-aula--player)
4. [Sala AO VIVO (WebRTC)](#4-sala-ao-vivo-webrtc)
5. [Checkout (Pix/boleto/cartão + cupom + order bump)](#5-checkout-pixboletocartão--cupom--order-bump)
6. [Editor de curso / aula (Estúdio)](#6-editor-de-curso--aula-estúdio)
7. [Dashboard do aluno](#7-dashboard-do-aluno)
8. [Dashboard do Admin do tenant](#8-dashboard-do-admin-do-tenant)
9. [Console Super-Admin (lista de tenants)](#9-console-super-admin-lista-de-tenants)
10. [Padrões transversais (estados reutilizáveis)](#10-padrões-transversais-estados-reutilizáveis)
11. [Dependências e pontos para o coordenador](#dependências-e-pontos-para-o-coordenador)

---

## 1. Landing / página de vendas do curso (público)

- **Rota:** `tenant.app.com/curso/{slug}` (e `?ref={code}&utm_*`). **Auth:** público (guest/ST).
- **Objetivo:** conversão (SEO, pixels Meta/GA4, UTM, preço). Atribuição de afiliado por `?ref` →
  cookie last-click 30d (MONETIZATION §B.7); **`?ref` não altera canonical** (IA §5.4).

### Desktop

```
┌──────────────────────────────────────────────────────────────────────────┐
│ [logo tenant]                       Cursos · Entrar · [ Criar conta ]      │ ‹DS: Navbar (branding)›
├──────────────────────────────────────────────────────────────────────────┤
│  HERO                                                                      │
│  ┌──────────────────────────────┐   ┌───────────────────────────────────┐ │
│  │ ▶ vídeo de vendas (Bunny)     │   │  TypeScript Pro                   │ │ ‹card de oferta sticky›
│  │   ou imagem de capa           │   │  ⭐ 4,8 (320) · 12h · 48 aulas    │ │
│  │                               │   │  ───────────────────────────────  │ │
│  └──────────────────────────────┘   │  De R$ 497  por  R$ 297           │ │ « preço vem do backend »
│  Headline de transformação           │  ou 12× R$ 29,70                  │ │
│  Subhead / promessa                  │  [ Comprar agora ]   ‹DS: Button› │ │ → /checkout/{slug}
│                                      │  ⏱ Oferta termina em 02:14:50     │ │ « timer opcional (config) »
│                                      │  🔒 Compra segura · 7 dias garantia│ │
│                                      └───────────────────────────────────┘ │
├──────────────────────────────────────────────────────────────────────────┤
│  O que você vai aprender   [✓ tópico] [✓ tópico] [✓ tópico] [✓ tópico]     │
│  Conteúdo do curso (currículo)                                             │
│    ▾ Módulo 1 · 5 aulas · 1h12     ▸ Módulo 2 · 8 aulas (prévia 🔓)        │ ‹DS: Accordion›
│  Para quem é · Instrutor (bio) · Depoimentos · FAQ                          │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │  CTA final:  R$ 297   [ Quero me inscrever ]   garantia · meios de pgto│ │ « repete oferta »
│  └──────────────────────────────────────────────────────────────────────┘ │
│  Rodapé: termos · privacidade (LGPD) · CNPJ do tenant                      │
└──────────────────────────────────────────────────────────────────────────┘
```

- **Componentes-chave:** Navbar com branding; card de oferta **sticky** (desktop) com preço/parcelas/
  CTA; `‹DS: Accordion›` do currículo com aulas de **prévia** (`🔓`) reproduzíveis sem login; selos de
  garantia/segurança; bloco de instrutor/depoimentos/FAQ.
- **Ações por papel:** guest/ST → **Comprar** (checkout), **Entrar/Criar conta**, **ver prévia**. Se ST
  **já matriculado** ativo → CTA muda para **“Ir para o curso”** (`/app/curso/{slug}`). « Detecção via
  sessão; não bloquear página pública ».
- **Estados:**
  - **Carregando:** skeleton do hero + card de oferta + 3 linhas de currículo.
  - **Erro (curso não encontrado / despublicado):** 404 amigável com link ao catálogo; sem vazar dados
    de outro tenant.
  - **Vazio (sem depoimentos/FAQ):** seções se ocultam (não renderiza bloco vazio).
  - **Gratuito (`pricing_type='free'`):** CTA vira **“Acessar grátis”** → cadastro + matrícula direta
    (sem checkout).
  - **Esgotado/encerrado (turma fechada, F2):** CTA desabilitado + “Inscrições encerradas” + captura de
    lista de espera.
- **Mobile:** coluna única; o card de oferta vira **barra fixa inferior** (preço + `[ Comprar ]`);
  currículo em accordion; vídeo full-width 16:9.

---

## 2. Catálogo & "Meus cursos" do aluno

Duas superfícies próximas: **catálogo público** (`/cursos`, guest) e **“Meus cursos”** (`/app`, ST,
matriculados). Mesma linguagem de card; contexto diferente.

### 2.1 Catálogo público — `/cursos`

```
┌──────────────────────────────────────────────────────────────────────────┐
│ [logo]                              Cursos · Entrar · [ Criar conta ]      │
├──────────────────────────────────────────────────────────────────────────┤
│ [ 🔎 Buscar cursos…]   Categoria ▾   Ordenar ▾                            │ ‹DS: Input + Select›
│ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐                        │
│ │ capa     │ │ capa     │ │ capa     │ │ capa     │  ‹DS: Card›            │
│ │ Curso A  │ │ Curso B  │ │ Curso C  │ │ Curso D  │                        │
│ │ R$ 297   │ │ Grátis   │ │ R$ 497   │ │ Assinat. │                        │
│ │ [ Ver ]  │ │ [ Ver ]  │ │ [ Ver ]  │ │ [ Ver ]  │  → /curso/{slug}      │
│ └──────────┘ └──────────┘ └──────────┘ └──────────┘                        │
│ [ Carregar mais ]  (paginação por cursor)                                  │
└──────────────────────────────────────────────────────────────────────────┘
```

- **Estados:** vazio (“Este catálogo ainda não tem cursos publicados.”); carregando (grid de skeletons);
  erro (retry). « Só `courses.status='published'` aparecem ». Catálogo público pode ser desativado por
  tenant (IA coordenador #9).

### 2.2 "Meus cursos" — `/app` (ST autenticado)

```
┌──────────────────────────────────────────────────────────────────────────┐
│ [logo]   Meus cursos · Certificados        🔔  [avatar ▾ conta]           │ ‹DS: Navbar (app)›
├──────────────────────────────────────────────────────────────────────────┤
│  Meus cursos             [ Em andamento | Concluídos | Todos ]  ‹DS: Tabs› │ « filtro »
│  ┌────────────────────┐ ┌────────────────────┐ ┌────────────────────┐      │
│  │ capa  Curso A      │ │ capa  Curso B      │ │ capa  Curso C      │       │
│  │ Aula 6/15          │ │ 100% ✓             │ │ 0%                 │       │
│  │ 42% ▓▓▓▒▒          │ │ ▓▓▓▓▓▓             │ │ ▒▒▒▒▒▒             │       │
│  │ [ Retomar ]        │ │ [ Ver certificado ]│ │ [ Começar ]        │       │
│  └────────────────────┘ └────────────────────┘ └────────────────────┘      │
└──────────────────────────────────────────────────────────────────────────┘
```

- **Card de curso (LEARNING_EXPERIENCE_UX §3.2):** capa, título, contagem (“Aula 6 de 15”), barra
  `watched_pct`, **CTA contextual**: `0%`→**Começar**, em progresso→**Retomar**, `100%`+elegível→
  **Ver certificado**, `100%` sem certificado→**Revisar**.
- **Ordenação padrão:** em progresso (por `updated_at` desc) → não iniciados → concluídos.
- **Ações por papel:** ST abre player, baixa certificado. « Acesso = entitlement; card de curso com
  `enrollment` `suspended/expired` mostra selo de estado + CTA de regularização/renovação ».
- **Estados:**
  - **Vazio (sem matrículas):** ilustração + “Você ainda não está matriculado em nenhum curso.” + CTA
    catálogo (`‹DS: EmptyState›`).
  - **Carregando:** grid de skeletons.
  - **Erro:** banner + retry com `requestId`.
- **Mobile:** coluna única, cards full-width, filtro vira `‹DS: Segmented/Select›`. Barra de navegação
  inferior (Meus cursos · Certificados · Conta).

---

## 3. Sala de aula / Player

- **Rota:** `/app/curso/{slug}` (resolve aula de retomada) ou `/app/curso/{slug}/aula/{lessonId}`.
  **Auth:** ST com `enrollment.active`. Espelha LEARNING_EXPERIENCE_UX §2.

### Desktop (2 zonas)

```
┌──────────────────────────────────────────────────────────────────────────┐
│ ‹ Voltar ao curso │ TypeScript Pro │ Progresso 42% ▓▓▓▒▒ │ ⚙ preferências │ ‹topbar›
├───────────────────────────────────────────┬──────────────────────────────┤
│  PLAYER 16:9 (Bunny iframe)                │ CONTEÚDO        [🔎 Buscar…]  │
│  ┌─────────────────────────────────────┐  │ ▾ Módulo 1 · 3/5 ▓▓▓▒▒        │ ‹DS: Accordion›
│  │            vídeo HLS                  │  │   ✓ Aula 1 · 04:12           │
│  │   ◀◀ ▶ ▶▶  0.5–2x  CC  ⛶            │  │   ▶ Aula 2 · 12:30 (atual)   │ « ▶ = em reprodução »
│  └─────────────────────────────────────┘  │   ○ Aula 3 · 08:00 · 30% ▓▒  │ « ○ + barra parcial »
│  TÍTULO DA AULA · Módulo 1                 │   🔒 Aula 4 · libera em 3d    │ « drip »
│  [ ✓ Marcar como concluída ]               │   🔒 Aula 5 · conclua a A.4   │ « pré-req (F2) »
│  ───────────────────────────────────────  │   ⏳ Aula 6 · processando     │ « video_status »
│  [ Descrição │ Materiais (2) │ Comentários(14) │ Anotações(F2) ] ‹DS: Tabs›│
│  Conteúdo da aba selecionada…              │ ▸ Módulo 2 · 0/4 (bloq.)      │
│                                            │ ───────────────────────────  │
│                                            │ [ Próxima aula → ]            │ ‹CTA fixo rodapé sidebar›
└───────────────────────────────────────────┴──────────────────────────────┘
```

- **Topbar:** breadcrumb de volta; **barra de progresso global** clicável; ⚙ (velocidade/legenda/
  qualidade padrão, persistidas por usuário).
- **Player:** iframe Bunny (HLS adaptativo). **Continuar de onde parou**: inicia em
  `position_seconds − 5s`; toast “Retomando de 12:30 · [Recomeçar do início]”. **Heartbeats** com
  throttle 5–15s → `/progress`; conclusão **automática** (`watched_pct ≥ 90% E anti-seek`) é
  server-side — a UI só reflete e anima o `✓` com `aria-live`. **Overlay de próxima aula** ao `ended`
  (countdown 5s, **Reproduzir agora / Cancelar autoplay**); se a próxima estiver bloqueada, mostra o
  motivo e desabilita.
- **Conclusão manual:** botão sob o player aparece só para aulas **não-vídeo** (texto/PDF) ou quando o
  toggle de autoria `completion_mode='manual'` (AUTHORING_UX §8). Em vídeo com gate, botão desabilitado
  com tooltip “Assista ao menos 90% para concluir”.
- **Sidebar (lista):** accordion por módulo com progresso; ícones de estado (§2.2 do doc de UX): `✓`,
  `▶`, `○` (+% parcial), `🔒` (drip / pré-req com motivo), `⏳` (processando). Busca de aula. CTA
  “Próxima aula” fixo.
- **Abas:**
  - **Descrição:** texto rico (`lessons.content`).
  - **Materiais:** lista `lesson_assets` (nome/tipo/tamanho) com “Baixar” (URL assinada R2). Aba
    **oculta quando 0 anexos** (UX §2.7). Erro de URL → “Não foi possível preparar o download.”
  - **Comentários (14):** thread 1 nível; composer (Ctrl/Cmd+Enter publica); badge **🎓 Instrutor**;
    “Responder”, ♥ (otimista), ⋯ (editar/excluir/denunciar), **Resolvido ✓**; ordenar
    (Recentes/Curtidos/Não respondidos); “Carregar mais”. **Desativado** por toggle do instrutor →
    “Os comentários estão desativados nesta aula.”
  - **Anotações (F2).**
- **Estados de substituição do player (UX §2.8–2.11):**
  - **Bloqueado por drip/pré-req:** painel “🔒 Conteúdo bloqueado · Esta aula libera em 3 dias
    (DD/MM)” **ou** “Conclua a aula X” + [ Ir para a próxima disponível → ].
  - **Vídeo processando:** “⏳ Vídeo em processamento…”, troca automática ao ficar `ready` (polling/SSE).
  - **Acesso suspenso/expirado (UX §2.10):** `suspended`→“Seu acesso está suspenso. Regularize o
    pagamento.” + link; `refunded/cancelled`→“Este acesso foi encerrado.”; `expired`→“Seu acesso
    expirou em DD/MM.” + renovar.
  - **Carregando:** skeleton 16:9 + skeleton da lista.
  - **Erro de player/token:** “Não foi possível carregar o vídeo.” + [ Tentar novamente ] (regenera
    Embed Token); token expirado em sessão → renovação silenciosa, senão overlay “recarregando…”.
- **Ações por papel:** ST consome/comenta/baixa. IN/AD em **“Ver como aluno”** (preview do Estúdio)
  navegam o mesmo player respeitando drip/bloqueios.
- **Mobile:** empilhado — player full-width no topo; abas abaixo; **lista de aulas vira
  `‹DS: Sheet/Drawer›`** acionado por “Conteúdo do curso” (ou accordion abaixo das abas). Controles de
  toque ≥44px; CTA “Próxima aula” fixo no rodapé.

---

## 4. Sala AO VIVO (WebRTC)

- **Rota:** sala da aula `type='live'` (acesso via player/currículo). **Auth:** token de sala emitido
  **só pelo backend** após entitlement + quota (LIVE_CLASSES §6). **Fase: F2.** Espelha LIVE_CLASSES §4.

### Desktop (palco + painel lateral)

```
┌──────────────────────────────────────────────────────────────────────────┐
│ [● AO VIVO 12:34]  Título da aula            [ Participantes 23 ]  [⚙]     │ « cronômetro + nº »
├───────────────────────────────────────────┬──────────────────────────────┤
│                                            │ [ Chat │ Pessoas │ Q&A │ Mãos]│ ‹DS: Tabs› (Mãos=host)
│        PALCO (speaker ativo / share)        │ ┌──────────────────────────┐ │
│                                            │ │ Maria: e quando usar…?    │ │
│   ┌────┐ ┌────┐ ┌────┐ ┌────┐  thumbs       │ │ Instrutor 🎓: ótimo…      │ │ ‹aria-live nas msgs›
│   │vid │ │vid │ │vid │ │vid │              │ │ …                         │ │
│   └────┘ └────┘ └────┘ └────┘              │ └──────────────────────────┘ │
│                                            │ [ Escreva uma mensagem…  ↵ ] │
├───────────────────────────────────────────┴──────────────────────────────┤
│ [🎤 mudo] [📷 câmera] [🖥 compartilhar] [✋ mão] [⛶]  …  HOST:[⏺ rec][⏹ encerrar]│ ‹barra controles›
└──────────────────────────────────────────────────────────────────────────┘
        « indicador ⏺ "gravando" visível a TODOS (transparência/LGPD) »
```

- **Controles:** mic, câmera, screen share (aluno conforme modo), levantar mão, tela cheia. **Host**
  (IN autor/AD/OW): ⏺ gravar, ⏹ encerrar, e **moderação** na aba Pessoas/Mãos (mutar, desligar câmera,
  remover, banir, travar chat, baixar mão). Grants vêm do token (RBAC C8-live).
- **Painel lateral (abas):** **Chat** (data channel do provedor; persistência p/ histórico/moderação),
  **Pessoas** (lista + status falando/mudo), **Q&A** (perguntas → host marca **respondida**/destaca),
  **Mãos** (fila, só host; “convidar ao palco”).
- **Estados de tela (LIVE_CLASSES §4.2):**
  - **Agendada (countdown):** “Começa em HH:MM” + [ Adicionar ao calendário ] (.ics/Google). Sem sala.
  - **Lobby / sala de espera:** “Aguardando o instrutor iniciar.” + device check disponível.
  - **Device check:** permissão câmera/mic, seleção de dispositivo, preview; aviso se bloqueado pelo
    navegador.
  - **Conectando:** skeleton da grade + spinner (sem CLS).
  - **Reconexão:** banner “Reconectando…”, preserva UI (ICE restart, NFR-LIVE-05).
  - **Erro de mídia:** “Habilite a câmera nas configurações do navegador.” (acionável).
  - **Sala cheia (quota):** “A sala atingiu o limite de participantes.” (hard block por plano).
  - **Sem entitlement:** bloqueio com CTA de compra/regularização (Regra nº1).
  - **Encerrada:** “A aula foi encerrada. A gravação estará disponível em breve.”
  - **Replay em processamento:** card “Processando gravação…” (espelha `video_status`).
  - **Replay disponível:** vira **player VOD Bunny normal** (Tela 3) — mesma aula amadurece em VOD.
  - **Sem gravação / falha:** “Esta aula não foi gravada.” / aviso ao host (`live_recording_failed`).
- **Ações por papel:** ST entra/publica (conforme modo), chat, mão, Q&A. Host inicia/encerra/grava/
  modera só nos **seus** cursos (C6). AF não acessa.
- **Mobile:** palco full-width; painel (Chat/Pessoas/Q&A) em **`‹DS: Sheet/Drawer›`** inferior; barra de
  controles compacta fixa; “gravando” e “AO VIVO” sempre visíveis.

---

## 5. Checkout (Pix/boleto/cartão + cupom + order bump)

- **Rota:** `/checkout/{slug}` (+ telas de resultado). **Auth:** público/ST. Página única mobile-first
  (MONETIZATION §B). **Pagamento governa acesso** (idempotência por `payment_events`).

### Desktop (2 colunas: dados | resumo)

```
┌──────────────────────────────────────────────────────────────────────────┐
│ [logo tenant]                                   🔒 Ambiente seguro         │
├────────────────────────────────────────────┬─────────────────────────────┤
│ 1) IDENTIFICAÇÃO                             │ RESUMO DO PEDIDO            │ ‹sticky›
│   ( ) Já tenho conta  [ Entrar ]             │ TypeScript Pro              │
│   (•) Criar conta                            │ Subtotal        R$ 297,00   │
│   Nome [__________]  E-mail [____________]   │ Cupom (PROMO10) −R$ 29,70   │ « valida no backend »
│   CPF  [__________]  (Pix/boleto/nota)       │ Order bump      +R$ 47,00   │ « se aceito »
│   [x] Aceito Termos e Política (LGPD) *      │ ─────────────────────────   │
│                                              │ Total           R$ 314,30   │
│ 2) ORDER BUMP (F2)                           │ ou 12× R$ 30,20             │
│   [x] Adicione "Pacote de exercícios" +R$47  │ [ Pagar agora ]  ‹DS:Button›│
│                                              │ ⏱ Oferta termina em 09:58   │
│ 3) PAGAMENTO   [ Pix ][ Boleto ][ Cartão ]   │ 7 dias de garantia          │
│   ── Cartão ──                               └─────────────────────────────┘
│   Número [________]  Validade[__] CVV[__]                                   
│   Parcelas [ 12× R$ 30,20 ▾ ]                  « juros conforme tenant »   
│   [ Cupom: __________ ] [ Aplicar ]                                         
└──────────────────────────────────────────────────────────────────────────┘
```

- **Identificação:** login OU **cadastro embutido** (cria conta na mesma jornada + consentimento LGPD
  obrigatório com timestamp). CPF p/ Pix/boleto/nota/antifraude.
- **Cupom:** valida no backend (existe, vigente, `uses<max_uses`, aplicável). Inválido → erro inline
  “cupom inválido/expirado”. Desconto **reduz a base** de comissão/take rate.
- **Order bump (F2):** checkbox 1-clique, soma ao resumo, mesma `order`/pagamento.
- **Métodos (tabs):**
  - **Pix** → após “Pagar”: **tela aguardando Pix** (QR + copia-e-cola + countdown de expiração;
    polling/SSE até `paid`).
  - **Boleto** → **tela boleto gerado** (linha digitável + PDF; compensa em 1–3 dias; acesso só ao
    `paid`).
  - **Cartão** → autorização imediata; aprovado→sucesso; recusado→**tela pagamento recusado** + CTA
    tentar outro método.
- **Telas de resultado:**

```
 AGUARDANDO PIX            BOLETO GERADO            RECUSADO               SUCESSO
 ┌───────────────┐         ┌───────────────┐        ┌──────────────┐       ┌──────────────┐
 │  [ QR code ]  │         │ Linha digit.: │        │ ✖ Pagamento  │       │ ✓ Compra     │
 │  [Copiar c&c] │         │ 34191.7...    │        │   recusado   │       │   confirmada │
 │ ⏳ Expira 09:5│         │ [Copiar][PDF] │        │ Tente outro  │       │ [ Acessar    │
 │ Aguardando…   │         │ Compensa 1-3d │        │ método       │       │   curso → ]  │
 │ (polling)     │         │               │        │ [ Voltar ]   │       │ (+upsell F2) │
 └───────────────┘         └───────────────┘        └──────────────┘       └──────────────┘
```

- **Estados:**
  - **Carregando:** skeleton do resumo + desabilita “Pagar” durante submit (evita duplo clique;
    idempotência server-side).
  - **Erro de gateway:** mensagem amigável + retry; nunca perde dados digitados.
  - **Pix/boleto expirado:** `pending → expired`; CTA “Gerar novo”. Recuperação de carrinho = F2.
  - **Já matriculado ativo:** aviso “Você já tem acesso a este curso” + [ Ir para o curso ].
  - **Checkout do tenant indisponível** (sem recipient Pagar.me / tenant `suspended`): bloqueio com
    aviso.
- **Ações por papel:** guest/ST compra. « `affiliate_id` anexado se houver cookie `ref` (last-click) ».
- **Mobile:** coluna única — **resumo colapsável no topo** (“Ver resumo ▾”) + CTA **“Pagar” fixo no
  rodapé**; métodos em tabs full-width; QR Pix grande e botão “Copiar” proeminente.

---

## 6. Editor de curso / aula (Estúdio)

- **Rota:** `/manage/cursos/{id}/editor` (currículo) e `/manage/cursos/{id}/aula/{lessonId}` (aula).
  **Auth:** IN (seus cursos)/AD/OW. **Autosave** com estado `salvando…/salvo/erro` (AUTHORING_UX §1).

### 6.1 Editor de currículo (módulos/aulas)

```
┌──────────────────────────────────────────────────────────────────────────┐
│ ‹ Cursos / TypeScript Pro                Status: Rascunho ▾  [ Publicar ]  │ ‹breadcrumb + ação›
│                                          ⟳ Salvo há 2s   [ Ver como aluno ]│ « autosave state »
├───────────────────────────────────────────┬──────────────────────────────┤
│ CURRÍCULO (drag-and-drop)                  │ INSPETOR (aula selecionada)   │
│ ▾ Módulo 1                          [ ⋮ ]  │ Aula: "O que é TypeScript"    │
│   ⠿ ✓ Aula 1 · vídeo · ready        [ ⋮ ]  │ Tipo: ( vídeo ▾ )             │
│   ⠿ ⏳ Aula 2 · vídeo · processing   [ ⋮ ] │ ── Vídeo (Bunny/TUS) ──       │
│   ⠿ ○ Aula 3 · texto                [ ⋮ ]  │ [████████░░] 80% enviando…    │ ‹DS: Progress›
│   ⠿ 🔒 Aula 4 · drip D+7  · opcional [ ⋮ ] │ (retoma após falha · TUS)     │
│   [ + Adicionar aula ]                     │ Thumb · Duração · Legendas VTT│
│ ▸ Módulo 2                          [ ⋮ ]  │ [x] Materiais (PDF) [+]       │
│ [ + Adicionar módulo ]                     │ Liberação: (sem|data|D+N)     │ « drip »
│                                            │ [ ] Conclusão manual          │ « completion_mode »
│                                            │ [ ] Aula opcional             │ « is_optional »
│                                            │ Status: ( Rascunho | Publicado )│
└───────────────────────────────────────────┴──────────────────────────────┘
```

- **Árvore Módulo→Aula** com **drag-and-drop** (⠿ handle) → persiste `position`; reordenar **não**
  republica nem quebra progresso. Indicadores por aula: tipo, `video_status` (none/queued/processing/
  ready/failed), drip, **opcional** (badge), quiz anexado.
- **Inspetor por tipo de aula:**
  - **Vídeo:** upload **TUS resumable** (barra, retomada após falha), cartão de estado
    `queued→processing→ready`; pós-`ready`: thumb, duração, legendas VTT (auto = F2), materiais.
  - **Texto:** editor WYSIWYG (`lessons.content jsonb`).
  - **PDF/Download:** upload `lesson_assets` (R2) + “exigir visualização para concluir” (`require_view`).
  - **Quiz:** abre builder (§6.2).
  - **Live (F2):** data/hora, modo, gravar auto, chat/Q&A, lobby, presença conclui (`live_sessions`).
- **Toggles de autoria (cascata tenant→curso→aula, AUTHORING_UX §8):** `completion_mode`, `is_optional`,
  `comments_enabled`, `require_view` na aula; `reveal_answers`/`max_attempts`/`pass_score` no quiz;
  `cert_requires_pass` no curso.
- **Publicação:** **checklist bloqueante** (≥1 aula publicável, vídeos `ready`, preço se pago, capa+
  descrição). Falha → lista de pendências; sucesso → `draft→published`. **“Ver como aluno”** abre o
  player em preview respeitando drip/bloqueios.
- **Estados:** vazio (“Nenhum módulo ainda — comece criando o Módulo 1”); salvando/salvo/erro de
  autosave; upload em progresso; vídeo processando; erro de publicação (checklist).

### 6.2 Quiz builder (resumo)

```
┌──────────────────────────────────────────────┐
│ Quiz: "Avaliação do Módulo 1"   ⟳ Salvo       │
│ Política: nota mín [70]%  Tentativas [3]      │
│ Gabarito: (nunca|após enviar|se aprovado)     │ « reveal_answers »
│ ── Questão 1 (múltipla escolha) ── [ ⋮ ]      │
│   Enunciado [____________________]            │
│   (•) Alt A (correta)  ( ) Alt B  ( ) Alt C   │
│ [ + Questão ]   [ V/F ]                        │
└──────────────────────────────────────────────┘
```

- **Validações:** ≥1 questão; toda questão com resposta correta; `pass_score` 0–100. Quiz publicado sem
  questões é barrado (UX §7.4).
- **Mobile:** Estúdio é desktop-first (uso de produção); em telas estreitas, currículo e inspetor
  **empilham** (inspetor vira `‹DS: Sheet›` ao tocar a aula). Drag-and-drop com handles maiores.
- **Ações por papel:** IN edita conteúdo dos **seus** cursos (sem financeiro global); AD/OW tudo do
  curso. Preço/oferta/afiliados ficam em telas de AD/OW (RBAC #16).

---

## 7. Dashboard do aluno — `/app` (visão início)

> Variante “início” de Meus cursos (§2.2), com **continuar assistindo** e marcos. Espelha
> LEARNING_EXPERIENCE_UX §3.2.

```
┌──────────────────────────────────────────────────────────────────────────┐
│ Olá, {nome}! 👋                                              🔔 [avatar ▾] │
├──────────────────────────────────────────────────────────────────────────┤
│ ── Continuar assistindo ─────────────────────────────────────────────────│
│ [ ▶ Curso A · Aula 2/15 · 42% ▓▓▓▒▒   "Retomar 12:30"  → ]                 │ ‹card destaque›
│ ── Próximas lives (F2) ──────────────────────────────────────────────────│
│ [ ● Curso A · "Tira-dúvidas" · hoje 19:00 · [Adicionar ao calendário] ]    │
│ ── Meus cursos ──────────────────────────────────────────────────────────│
│ ┌──────────┐ ┌──────────┐ ┌──────────┐                                    │
│ │ Curso A  │ │ Curso B  │ │ Curso C  │   (cards §2.2 — CTA contextual)     │
│ │ 42% ▓▓▒▒ │ │ 100% ✓   │ │ 0% ▒▒▒▒  │                                    │
│ └──────────┘ └──────────┘ └──────────┘                                    │
│ ── Conquistas (F2) ──────────────────────────────────────────────────────│
│ 🏅 Maratonista · Nível 4 ▓▓▓▒ 320/500 XP · ⏳ Faltam 2 aulas p/ Módulo 1   │
└──────────────────────────────────────────────────────────────────────────┘
```

- **Conta (menu avatar):** Perfil · **Minha assinatura** (cancelar — assinatura **ao conteúdo**, NÃO
  confundir com SaaS) · Minhas compras (recibos) · Privacidade (LGPD: exportar/excluir) ·
  Notificações (F2) · Sair.
- **Estados:**
  - **Vazio:** sem matrículas → bloco “continuar” oculto; EmptyState + CTA catálogo.
  - **Carregando:** skeleton do card de retomada + grid.
  - **Erro:** banner por seção (uma seção que falha não derruba a página).
  - **Múltiplos cursos em progresso:** decisão de 1 vs N cards de retomada (UX §9 — coordenador).
- **Ações por papel:** ST. (IN/AD/OW têm seu próprio dashboard em `/manage`, §8.)
- **Mobile:** seções empilhadas; card de retomada full-width; navegação inferior.

---

## 8. Dashboard do Admin do tenant — `/manage`

- **Rota:** `/manage`. **Auth:** OW/AD (IN vê versão reduzida — só conteúdo/engajamento). KPIs do tenant
  (IA 3.4). **Branding do tenant** no chrome.

### Desktop (sidebar + KPIs)

```
┌───────────────┬────────────────────────────────────────────────────────────┐
│ [logo tenant] │ Dashboard                              Período: [ 30d ▾ ]   │
│ ───────────── │ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐         │
│ ▸ Dashboard   │ │ Receita  │ │ Alunos   │ │ Conclusão│ │ Pedidos  │ ‹DS:Card│
│ ▸ Cursos      │ │ R$ 48,2k │ │ ativos   │ │ 37%      │ │ pend. 12 │   stat› │
│ ▸ Alunos      │ │ ▲ 12%    │ │ 1.240 ▲  │ │ ▼ 3%     │ │          │         │
│ ▸ Financeiro  │ └──────────┘ └──────────┘ └──────────┘ └──────────┘         │
│ ▸ Afiliados   │ ── Receita no período ─────────────────────────────────────│
│ ▸ Marketing   │ [ ▁▂▃▅▇▆▅▇ gráfico de linha/barras ]                        │
│ ▸ Analytics   │ ── Pendências ─────────────────────────────────────────────│
│ ▸ Equipe      │ • 3 vídeos processando · 2 reembolsos a revisar · 1 afiliado│
│ ▸ Integrações │   aguardando aprovação                                      │
│ ▸ Config      │ ── Cursos recentes ───────────────────────────────────────│
│ ───────────── │ [ tabela: curso · status · alunos · conclusão · receita ]   │ ‹DS: DataTable›
│ ⚠ Plano: 80%  │                                                            │
│   storage     │ « banner de quota 80%/100% + CTA upgrade (owner) »          │
└───────────────┴────────────────────────────────────────────────────────────┘
```

- **KPIs (`‹DS: Card stat›`):** receita (período), alunos ativos, taxa de conclusão, pedidos pendentes;
  com deltas vs período anterior. Filtro de período.
- **Pendências acionáveis:** vídeos processando, reembolsos a revisar, afiliados pendentes, lives a
  iniciar (F2) — cada item linka à tela de ação.
- **Banner de quota:** alerta 80%/100% (storage/banda/alunos) com CTA upgrade — **só OW** vê billing/
  plano SaaS (MONETIZATION §A); AD não vê billing.
- **Navegação por papel (IA §4):**
  - **IN:** Dashboard · Cursos (seus) · Alunos (dos seus cursos, leitura) · Analytics. **Não** vê
    Financeiro/Afiliados/Equipe/Config/Plano SaaS.
  - **AD:** + Alunos (todos) · Financeiro · Afiliados · Marketing · Equipe · Integrações · Config.
  - **OW:** tudo do AD + **Plano/quota do tenant (billing SaaS)** + Config—Pagamentos.
- **Estados:** vazio (tenant novo → “Crie seu primeiro curso” / wizard de onboarding); carregando
  (skeletons de cards + tabela); erro por widget (cada card recupera independente).
- **Mobile:** sidebar vira **`‹DS: Sheet›`** (menu hambúrguer); KPIs em coluna; tabela vira lista de
  cards. Ações primárias no topo.

---

## 9. Console Super-Admin (lista de tenants) — `admin.app.com/tenants`

- **Host:** `admin.app.com` (control plane, **proposta a validar** — IA coordenador #1). **Auth:** SA +
  **MFA**. Sem acesso silencioso a dados de tenant (só via impersonação **auditada**).

### Desktop

```
┌──────────────────────────────────────────────────────────────────────────┐
│ [Plataforma · admin]   Tenants  Planos  Faturamento  Provisionamento  …    │ ‹nav SA›
├──────────────────────────────────────────────────────────────────────────┤
│ Tenants                          [ 🔎 buscar slug/nome ]  [ + Novo tenant ]│
│ Filtros: Status [ todos ▾ ]  Plano [ todos ▾ ]                             │
│ ┌────────────────────────────────────────────────────────────────────────┐│
│ │ Slug        Nome          Plano    Status        Uso        Criado  Ações││ ‹DS: DataTable›
│ │ acme        Acme Escola   Pro      ● active       62% stor  12/03   [ ⋮ ]││
│ │ beta        Beta Cursos   Starter  ◐ provisioning ▓ saga 3/5 02/06  [ ⋮ ]││ « link → saga »
│ │ gamma       Gamma EAD     Scale    ⚠ past_due*    88% bw    20/01   [ ⋮ ]││ « *derivado Stripe »
│ │ delta       Delta Edu     Pro      ■ suspended    —         05/12   [ ⋮ ]││
│ └────────────────────────────────────────────────────────────────────────┘│
│  Linhas: 1–20 de 87        [ ‹ ][ › ]                                       │
└──────────────────────────────────────────────────────────────────────────┘
```

- **Colunas:** slug, nome, plano, **status canônico** (`provisioning|active|suspended|cancelled`;
  `past_due/grace` **derivado** do Stripe, exibido como aviso), uso de quota (storage/banda), criado em,
  ações.
- **Ações (⋮ por linha):** Ver detalhe · **Provisionamento** (progresso/retry da saga) · Usuários
  (**Impersonar** — confirma + **motivo obrigatório** → `audit_log`, banner “modo suporte”) · Billing
  SaaS (Stripe) · Bunny (library/keys cifradas) · **Suspender/Ativar/Cancelar** (`‹DS: Dialog›` de
  confirmação — impacto: todos os usuários perdem acesso).
- **Criar tenant (`‹DS: Dialog/Wizard›`):** slug (checa disponibilidade) + plano + e-mail do admin →
  dispara **saga de provisionamento** (idempotente).
- **Estados:**
  - **Carregando:** skeleton de linhas da tabela.
  - **Vazio:** “Nenhum tenant ainda.” + [ + Novo tenant ].
  - **Erro:** retry; toda ação sensível registra `audit_log`.
  - **Provisioning na linha:** mini progresso da saga (3/5) com link para retry de passo.
- **Ações por papel:** **só SA**. « Impersonação respeita `withTenant`; jamais cruza tenant ».
- **Mobile:** uso pontual; tabela vira **cards** (slug + status + plano + ⋮). Filtros em `‹DS: Sheet›`.

---

## 10. Padrões transversais (estados reutilizáveis — DRY)

Para não repetir por tela; aplicar onde indicado acima.

- **Carregando:** `‹DS: Skeleton›` com a forma do conteúdo (cards, linhas de tabela, player 16:9). Nunca
  spinner solto em área grande.
- **Vazio:** `‹DS: EmptyState›` = ilustração/ícone + frase curta + **1 CTA primário** (ex.: “Você ainda
  não está matriculado… → Ver catálogo”).
- **Erro:** banner/inline com mensagem amigável (RFC 9457 → texto humano) + **ação de recuperação**
  ([ Tentar novamente ]) + **ID de correlação** (`requestId`) para suporte.
- **Sucesso/feedback:** `‹DS: Toast (sonner)›` discreto; ações otimistas (curtir, marcar concluído
  manual, postar comentário) com **rollback** se falhar; ações críticas (submeter quiz, pagar, emitir
  certificado) **aguardam** confirmação do servidor.
- **Bloqueio de acesso (entitlement):** painel dedicado (não “erro”) com motivo e CTA: drip / pré-req /
  suspenso / expirado / vídeo processando (Tela 3).
- **Quota (admin):** banner 80%/100% + CTA upgrade (visível conforme papel; billing só OW).
- **Acessibilidade:** foco visível, navegação por teclado, `aria-live` em mudanças de estado (“Aula
  concluída”, novas mensagens de chat ao vivo), legendas de vídeo, alvos ≥44px.
- **Responsividade (regra geral):** desktop = multi-coluna; mobile = coluna única, listas laterais →
  `‹DS: Sheet/Drawer›`, CTAs primários **fixos no rodapé**, resumos/painéis **colapsáveis**.

---

## Dependências e pontos para o coordenador

1. **Design System inexistente.** Estes wireframes referenciam componentes **shadcn/ui** como base
   (`‹DS: …›`), mas não há doc de DS. **Recomendação:** produzir um **mapa de componentes/tokens**
   (cores via CSS variables por tenant, tipografia, espaçamento, `EmptyState`/`StatCard` próprios) antes
   da implementação — cruza com a “pendência” da [IA §coordenador](../product/INFORMATION_ARCHITECTURE.md).
2. **Hosts/prefixos ainda são proposta** (`admin.app.com`, `/app`, `/manage`, `/affiliate`) — Tela 9 e
   navegação dependem da validação de engenharia (DNS/cookies/middleware) — [README §4 #22].
3. **Order bump/upsell (Tela 5) e lives (Tela 4) são F2.** Wireframes incluídos para continuidade, mas
   dependem de modelagem (`order_items`/`offers`/`purchase_session_id`; tabelas `live_*`) e de **valores
   de quota de live** pendentes do stakeholder ([README §5 #10]).
4. **Timer/urgência no checkout e na landing:** confirmar se é configurável por tenant/oferta (afeta
   componente e dados) — hoje desenhado como opcional.
5. **“Minha assinatura” (aluno) × “Plano SaaS” (owner):** as duas superfícies usam a palavra
   “assinatura”. Mantida a separação visual (Tela 7 vs Tela 8/9), mas **confirmar copy** para evitar
   confusão ([README §3]).
6. **Catálogo público opcional por tenant** (Tela 2.1): confirmar se todo tenant expõe `/cursos` ou
   alguns operam só por landings/links ([IA §coordenador #9]).
7. **Conclusão de Player — N cards de “continuar assistindo”** (Tela 3/7): decidir 1 vs N quando há
   múltiplos cursos em progresso ([LEARNING_EXPERIENCE_UX §9]).
8. **Estados de acesso (textos/CTAs por estado de `enrollments`)** dos painéis de bloqueio (Tela 3 §
   suspenso/expirado): alinhar copy de regularização com o time de pagamentos
   ([LEARNING_EXPERIENCE_UX §9]).
9. **Mobile do Estúdio (Tela 6):** assumido desktop-first; confirmar nível de suporte mobile esperado
   para autoria (drag-and-drop em toque).
10. **Próximos artefatos sugeridos:** protótipos navegáveis (Figma) das Telas 3, 4 e 5; spec de
    breadcrumbs/estado-vazio por área; e o **mapa de componentes** (item 1) para alimentar o DS.
```