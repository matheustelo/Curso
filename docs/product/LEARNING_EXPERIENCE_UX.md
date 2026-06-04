# Learning Experience & Engagement — Especificação de UX

- **Versão:** 1.0
- **Data:** 2026-06-04
- **Autor:** Product Design (Learning Experience)
- **Status:** Proposta para revisão do coordenador de produto
- **Documentos relacionados:** [PRD.md](../PRD.md) · [ROADMAP.md](../ROADMAP.md) · [DATA_MODEL.md](../DATA_MODEL.md) · [ARCHITECTURE.md](../ARCHITECTURE.md) · [CLAUDE.md](../../CLAUDE.md)

> **Escopo deste documento.** Detalha a UX das superfícies centrais de **aprendizado** e **engajamento**:
> player/sala de aula, progresso/avanços, comentários/comunidade, gamificação, certificados e avaliações.
> Para cada tema: comportamento, regras de negócio, estados (vazio/carregando/erro/sucesso),
> microinterações, wireframe textual e **eventos de analytics** disparados.
>
> **Notação de prioridade** (herdada do PRD/ROADMAP): **[MVP]** · **[F2]** · **[F3]**.
>
> **Convenções de eventos.** Nomes em `snake_case`, prefixo de domínio (`lesson_`, `course_`, `quiz_`,
> `comment_`, `cert_`, `gam_`). Todo evento carrega implicitamente `tenant_id`, `user_id`, `session_id`,
> `ts` (timestamp UTC) e `device` — não repetidos nas tabelas abaixo. Persistência respeita
> schema-per-tenant (`withTenant`); eventos de produto vão para o pipeline de analytics (PostHog) com
> `tenant_id` como propriedade obrigatória.

---

## Índice

1. [Princípios transversais de UX](#1-princípios-transversais-de-ux)
2. [Player e sala de aula](#2-player-e-sala-de-aula)
3. [Progresso e avanços](#3-progresso-e-avanços)
4. [Comentários e comunidade](#4-comentários-e-comunidade)
5. [Gamificação (F2)](#5-gamificação-f2)
6. [Certificados](#6-certificados)
7. [Avaliações (quiz)](#7-avaliações-quiz)
8. [Mapa consolidado de eventos de analytics](#8-mapa-consolidado-de-eventos-de-analytics)
9. [Dependências e pontos para o coordenador](#9-dependências-e-pontos-para-o-coordenador)

---

## 1. Princípios transversais de UX

Aplicam-se a todas as superfícies abaixo e evitam repetição (DRY de design):

- **Acesso = entitlement válido.** Toda superfície de aprendizado parte do pressuposto de matrícula
  ativa (`enrollments.status = active`). Se o acesso estiver `suspended`/`refunded`/`expired`, a sala de
  aula entra em **estado bloqueado por acesso** (ver §2.9) — distinto do bloqueio por drip/pré-requisito.
- **Quatro estados sempre presentes.** Toda lista/painel tem desenho explícito para **vazio**,
  **carregando** (skeleton, nunca spinner solto em áreas grandes), **erro** (mensagem + ação de
  recuperação) e **sucesso/conteúdo**. Erros seguem o handler único (RFC 9457) → mensagem amigável + ID
  de correlação para suporte.
- **Otimismo com reconciliação.** Ações de baixa criticidade (curtir, marcar concluído manualmente,
  postar comentário) usam **UI otimista** com rollback em caso de falha. Ações críticas (submeter quiz,
  emitir certificado) aguardam confirmação do servidor.
- **Acessibilidade (WCAG 2.1 AA, MVP).** Navegação por teclado completa, foco visível, contraste,
  `aria-live` para mudanças de estado (ex.: "aula concluída"), legendas em vídeo, alvos de toque ≥44px.
- **Mobile-first responsivo.** Layouts degradam para coluna única; player ocupa largura total; a lista
  de aulas vira drawer/accordion abaixo do player.
- **Branding por tenant.** Cores/acentos vêm de CSS variables do tenant; os componentes nunca
  hardcodam cor de marca.

---

## 2. Player e sala de aula

### 2.1 Objetivo e layout

A **sala de aula** (`/cursos/[slug]/aulas/[lessonId]`) é a superfície de maior valor (North Star =
horas efetivamente assistidas). Layout em duas zonas em desktop, empilhado em mobile.

```
┌───────────────────────────────────────────────────────────────────────┐
│ TOPBAR: ‹ Voltar ao curso | Título do curso | Progresso 42% ▓▓▓▒▒ | ⚙ │
├──────────────────────────────────────────┬────────────────────────────┤
│  PLAYER (16:9, foco)                      │  SIDEBAR — CONTEÚDO         │
│  ┌──────────────────────────────────┐    │  [Buscar aula…]             │
│  │                                  │    │  ▾ Módulo 1 · 3/5 ▓▓▓▒▒     │
│  │        vídeo (Bunny iframe)      │    │    ✓ Aula 1 · 04:12          │
│  │                                  │    │    ▶ Aula 2 · 12:30  (atual) │
│  │  ◀◀  ▶  ▶▶   0.5–2x  CC  ⛶       │    │    ○ Aula 3 · 08:00          │
│  └──────────────────────────────────┘    │    🔒 Aula 4 · libera em 3d  │
│  TÍTULO DA AULA · Módulo 1               │    🔒 Aula 5 · conclua A.4   │
│  [✓ Marcar como concluída]               │  ▸ Módulo 2 · 0/4 (bloq.)    │
│  ─────────────────────────────────────── │  …                          │
│  [ Descrição | Materiais (2) | Comentários (14) | Anotações(F2) ]       │
│  Conteúdo da aba selecionada…             │  [ Próxima aula → ]         │
└──────────────────────────────────────────┴────────────────────────────┘
```

- **Topbar:** breadcrumb de volta ao curso, barra de progresso global do curso (clicável → §3),
  engrenagem de preferências (velocidade/legenda padrão, qualidade).
- **Player:** iframe Bunny no MVP (HLS adaptativo); na F2, player próprio Vidstack com watermark.
- **Abas abaixo do player:** Descrição (texto rico), Materiais (anexos), Comentários (§4), Anotações
  (F2). A contagem de comentários/materiais aparece no rótulo da aba.
- **Sidebar (lista de aulas/módulos):** accordion por módulo com progresso por módulo; ícones de estado
  por aula; CTA "Próxima aula" fixo no rodapé.

### 2.2 Estados de uma aula na lista (ícones)

| Ícone | Estado | Significado |
|-------|--------|-------------|
| `✓` (preenchido) | `completed` | Aula concluída (regra §3.4). |
| `▶` (destaque) | atual | Aula em reprodução. |
| `○` (vazio) | `in_progress` / `not_started` | Disponível, ainda não concluída (mostra % parcial se >0). |
| `🔒` + "libera em Nd / em DD/MM" | bloqueada por **drip** | `drip_release_at` futuro ou `drip_days_after_enroll` não atingido. |
| `🔒` + "conclua a Aula X" | bloqueada por **pré-requisito** (F2) | Aula sequencial anterior não concluída. |
| `⏳` "processando" | vídeo ainda em encoding | `video_guid` existe mas Bunny ainda não emitiu "vídeo pronto". |

### 2.3 "Continuar de onde parou"

- **Origem do dado:** `lesson_progress.position_seconds` da última aula com `status = in_progress`
  (mais recente por `updated_at`).
- **Entrada na sala de aula sem `lessonId`** → resolve a aula de retomada e abre nela; o player
  posiciona em `position_seconds - 5s` (margem de contexto), nunca antes de 0.
- **Microinteração:** toast discreto "Retomando de 12:30" com ação "Recomeçar do início" (zera a
  posição visual, não apaga progresso).
- **Card "Continuar assistindo"** no dashboard (§3.2) leva direto ao ponto.

**Eventos:**

| Evento | Quando | Propriedades |
|--------|--------|--------------|
| `lesson_resume_shown` | toast de retomada exibido | `lesson_id`, `resume_position_seconds` |
| `lesson_resume_from_start` | usuário escolhe recomeçar | `lesson_id` |

### 2.4 Reprodução: velocidade, legendas, qualidade, próxima aula

- **Velocidade:** 0.5x, 0.75x, 1x, 1.25x, 1.5x, 1.75x, 2x. Preferência **persistida por usuário**
  (localStorage + sincronização opcional em preferências de conta) e reaplicada na próxima aula.
- **Legendas (CC):** lista de faixas disponíveis (idiomas); estado **"sem legenda"** quando a aula não
  tem `lesson_transcripts`/track. Preferência de legenda persistida. (F2: legendas auto via Bunny
  Transcribe.)
- **Qualidade:** automática (adaptativo) por padrão; seleção manual disponível; cap de resolução pode
  vir de política do tenant (custo de egress — ver risco no PRD).
- **Próxima aula:** ao terminar o vídeo (`ended`), overlay "Próxima aula em 5s ▸ [Título]" com
  contagem regressiva, botões **Reproduzir agora** e **Cancelar autoplay**. Respeita bloqueios: se a
  próxima estiver bloqueada (drip/pré-requisito), o overlay mostra o motivo e o botão fica desabilitado,
  sugerindo a primeira aula disponível.
- **Autoplay** é **opt-in por preferência** (default ligado, mas cancelável; preferência lembrada).

**Eventos:**

| Evento | Quando | Propriedades |
|--------|--------|--------------|
| `lesson_play` | play inicial/retomada | `lesson_id`, `position_seconds` |
| `lesson_pause` | pause | `lesson_id`, `position_seconds` |
| `lesson_speed_change` | troca velocidade | `lesson_id`, `from`, `to` |
| `lesson_caption_change` | liga/desliga/troca legenda | `lesson_id`, `track_lang` ou `off` |
| `lesson_quality_change` | troca qualidade manual | `lesson_id`, `quality` |
| `lesson_seek` | usuário arrasta a timeline | `lesson_id`, `from_seconds`, `to_seconds` |
| `lesson_ended` | vídeo chega ao fim | `lesson_id`, `watched_pct` |
| `lesson_next_autoplay` | autoplay dispara próxima | `from_lesson_id`, `to_lesson_id` |
| `lesson_next_cancelled` | usuário cancela autoplay | `from_lesson_id` |

### 2.5 Tracking de progresso (heartbeats)

- **Mecânica (do ARCHITECTURE §7):** o player emite `timeupdate` com **throttle de 5–15s** →
  `POST /progress` → atualiza `lesson_progress.position_seconds` e `watched_pct`. Heartbeats também
  disparam em `pause`, `seek` e `beforeunload` (sendBeacon) para não perder a última posição.
- **Anti-seek (regra de negócio do PRD nº 4):** o backend acumula **tempo real assistido** (soma de
  intervalos efetivamente reproduzidos), separado de `position_seconds`. Pular para o fim **não**
  conta como assistido. A UI nunca calcula conclusão — apenas reflete `lesson_progress.status` retornado.
- **Microinteração:** indicador sutil de "salvando progresso" só aparece se um heartbeat falhar e
  precisar reenfileirar (não polui a UI no caminho feliz).

**Eventos:** `lesson_heartbeat` (`lesson_id`, `position_seconds`, `watched_pct`, `real_watched_seconds`,
`playback_rate`) — alta frequência; pode ir para pipeline de watch-time/heatmap (F2). Marcado como
evento de **alto volume** (amostragem/agregação a definir com analytics).

### 2.6 Conclusão de aula: automática e manual

- **Automática (regra nº 4 do PRD):** quando `watched_pct ≥ 90%` **E** anti-seek satisfeito
  (tempo real ≥ duração × 0.8), o backend marca `status = completed` e retorna na resposta do
  heartbeat. A UI então:
  - anima o ícone da aula para `✓` (check com micro-animação + `aria-live` "Aula concluída");
  - recalcula a barra de progresso do módulo/curso (§3);
  - se houver próxima aula disponível, prepara o overlay de "próxima aula".
- **Manual:** botão **"Marcar como concluída"** sob o player para aulas **não-vídeo** (texto/PDF) ou
  como override quando a aula de vídeo já passou do gate. Para vídeo, a marcação manual **antes** de
  atingir o gate é desabilitada por padrão (tooltip "Assista ao menos 90% para concluir") — política
  configurável pelo instrutor (F2: "permitir conclusão manual"). Aulas de texto/PDF concluem por botão
  manual ou por scroll-to-end (configurável).
- **Reabrir/desfazer:** aula concluída pode ser revisitada livremente; "Marcar como não concluída"
  disponível em aulas de conclusão manual (não em vídeo com gate automático).

**Eventos:**

| Evento | Quando | Propriedades |
|--------|--------|--------------|
| `lesson_completed_auto` | gate ≥90% + anti-seek | `lesson_id`, `watched_pct`, `real_watched_seconds` |
| `lesson_completed_manual` | botão manual | `lesson_id`, `lesson_type` |
| `lesson_uncompleted_manual` | desfazer | `lesson_id` |

### 2.7 Anexos / materiais

- Aba **Materiais** lista `lesson_assets` (PDF, slides, arquivos) com nome, tipo e tamanho. Download via
  URL assinada de curta duração (R2). Botão "Baixar" por item; "Baixar todos" (zip) quando >1 (F2).
- **Estado vazio:** "Esta aula não tem materiais para download." (aba só aparece se houver ≥1 anexo, ou
  fica visível com contador 0 — **recomendação: ocultar a aba quando 0** para reduzir ruído).
- **Estado erro:** falha ao gerar URL → "Não foi possível preparar o download. Tente novamente."

**Eventos:** `lesson_asset_download` (`lesson_id`, `asset_id`, `filename`, `size_bytes`).

### 2.8 Estados de bloqueio: drip e pré-requisito

Quando o aluno abre (ou tenta abrir) uma aula bloqueada, o **player é substituído por um painel de
bloqueio** (não tela de erro):

```
┌──────────────────────────────────────────┐
│            🔒  Conteúdo bloqueado          │
│  Esta aula libera em 3 dias (12/06/2026).  │   ← drip por dias após matrícula
│  — ou —                                    │
│  Conclua a aula "Fundamentos" para liberar.│   ← pré-requisito (F2)
│  [ Ir para a próxima aula disponível → ]   │
└──────────────────────────────────────────┘
```

- **Regras (do DATA_MODEL):** drip por `drip_release_at` (data fixa) **ou** `drip_days_after_enroll`
  (relativo a `enrollments.enrolled_at`). Pré-requisito sequencial é F2.
- O cálculo de liberação é **server-side**; a UI nunca decide se libera — apenas renderiza o estado e o
  motivo retornados. A lista lateral mostra a contagem regressiva/data.
- **Microinteração:** countdown que atualiza diariamente; quando a aula libera, badge "Novo" e
  notificação (push F2 / e-mail) "Nova aula liberada".

**Eventos:** `lesson_blocked_view` (`lesson_id`, `reason: drip_date|drip_days|prerequisite`,
`unlock_at` ou `prerequisite_lesson_id`); `lesson_unlocked` (quando passa a disponível).

### 2.9 Estado: vídeo ainda processando

- Aula publicada cujo vídeo está em encoding (Bunny ainda não emitiu "vídeo pronto") mostra, no lugar
  do player:

```
┌──────────────────────────────────────────┐
│   ⏳  Vídeo em processamento                │
│   Estamos preparando este vídeo. Isso pode  │
│   levar alguns minutos.                     │
│   [ Avisar quando estiver pronto ]  (F2)    │
└──────────────────────────────────────────┘
```

- **Comportamento:** polling leve (ou realtime/SSE na F2) do status da aula; quando o webhook "vídeo
  pronto" atualiza o status, a UI troca automaticamente para o player (sem reload).
- **Para o instrutor (visão de autoria):** o mesmo estado aparece no editor com label "Processando —
  você poderá publicar/visualizar quando concluir".

**Eventos:** `lesson_video_processing_view` (`lesson_id`); `lesson_video_ready_shown` (`lesson_id`,
`wait_seconds`).

### 2.10 Estado: acesso suspenso/expirado

Distinto do bloqueio de conteúdo. Se `enrollments.status ≠ active`, a sala de aula mostra painel de
acesso com CTA apropriado:

- `suspended` (inadimplência): "Seu acesso está suspenso. Regularize o pagamento para continuar." +
  link de regularização.
- `refunded`/`cancelled`: "Este acesso foi encerrado." 
- `expired`: "Seu acesso expirou em DD/MM." + CTA de renovação (se aplicável).

**Eventos:** `lesson_access_denied_view` (`enrollment_status`, `course_id`).

### 2.11 Estados de carregamento e erro do player

- **Carregando:** skeleton do player (retângulo 16:9 com shimmer) + skeleton da lista lateral.
- **Erro de player/token:** "Não foi possível carregar o vídeo." + botão "Tentar novamente" (regenera
  o Embed Token). Logar com `requestId` para suporte.
- **Token expirado durante a sessão:** renovação silenciosa de token; se falhar, pausa + overlay
  "Sessão de vídeo expirada, recarregando…".

---

## 3. Progresso e avanços

### 3.1 Modelo de progresso (origem dos números)

- **Aula:** `lesson_progress.status` ∈ {`not_started`, `in_progress`, `completed`} + `watched_pct` para
  barra parcial.
- **Módulo:** `aulas concluídas / aulas publicadas do módulo` (apenas aulas que contam para conclusão;
  aulas opcionais — se houver flag F2 — não entram no denominador).
- **Curso:** `aulas concluídas / total de aulas publicadas que contam`. **Recálculo server-side** a cada
  conclusão; a UI recebe os percentuais prontos (não recalcula localmente, para evitar divergência).
- **Quiz como requisito:** quando o curso tem quiz obrigatório/nota mínima, o curso só chega a 100% com
  o quiz aprovado (ver §6 e §7).

### 3.2 "Meus cursos" e dashboard do aluno

Rota `/meus-cursos` (lista) e `/inicio` (dashboard). Wireframe do dashboard:

```
┌───────────────────────────────────────────────────────────────┐
│  Olá, {nome}! 👋                                                │
│  ── Continuar assistindo ───────────────────────────────────── │
│  [▶ Curso A · Aula 2/15 · 42% ▓▓▓▒▒  "Retomar 12:30"]          │
│  ── Meus cursos ─────────────────────────────────────────────  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                        │
│  │ capa     │ │ capa     │ │ capa     │                         │
│  │ Curso A  │ │ Curso B  │ │ Curso C  │                         │
│  │ 42% ▓▓▒▒ │ │ 100% ✓   │ │ 0% ▒▒▒▒  │                         │
│  │ Retomar  │ │ Certif.  │ │ Começar  │                         │
│  └──────────┘ └──────────┘ └──────────┘                         │
│  ── Conquistas (F2) · Próximos marcos ──────────────────────── │
│  🏅 Maratonista  ·  ⏳ Faltam 2 aulas para "Módulo 1 completo"  │
└───────────────────────────────────────────────────────────────┘
```

- **Card de curso:** capa, título, barra de progresso, **contagem** ("Aula 6 de 15" + "% concluído"),
  CTA contextual (**Começar** se 0%, **Retomar** se em progresso, **Ver certificado** se 100% e elegível,
  **Revisar** se 100% sem certificado).
- **Ordenação padrão:** cursos em progresso primeiro (por `updated_at` desc), depois não iniciados,
  depois concluídos. Filtros: Em andamento / Concluídos / Todos.
- **Estado vazio (sem matrículas):** ilustração + "Você ainda não está matriculado em nenhum curso." +
  CTA para o catálogo do tenant.

**Eventos:**

| Evento | Quando | Propriedades |
|--------|--------|--------------|
| `dashboard_view` | abre dashboard | `enrolled_courses_count` |
| `my_courses_view` | abre lista | `filter`, `count` |
| `course_card_continue_click` | clica retomar/começar | `course_id`, `progress_pct` |
| `course_card_certificate_click` | clica ver certificado | `course_id` |

### 3.3 Barras e indicadores de progresso

- **Barra de aula (parcial):** dentro da lista, aulas em progresso mostram fina barra com `watched_pct`.
- **Barra de módulo:** no cabeçalho do accordion ("3/5 ▓▓▓▒▒").
- **Barra de curso:** topbar da sala de aula + card do dashboard + página de detalhe do curso.
- **Microinteração de avanço:** ao concluir uma aula, a barra **anima** do valor antigo ao novo
  (transição suave) com `aria-live` anunciando "Progresso do curso: 50%".

### 3.4 Regra de conclusão (consolidada)

> **Conclusão de aula de vídeo** (PRD nº 4): `watched_pct ≥ 90%` **E** tempo real assistido ≥
> `duration × 0.8` (anti-seek). Conclusão de aula de texto/PDF: ação manual ou scroll-to-end (config).
> **Conclusão de quiz-aula:** atingir nota mínima (se definida) ou simplesmente submeter (se sem nota).
>
> **Conclusão de curso:** 100% das aulas que contam concluídas **+** quiz(zes) obrigatório(s) aprovados.
> Dispara elegibilidade de **certificado** (§6).

Toda a avaliação é **server-side**; a UI reflete. Mudanças de regra (ex.: o instrutor altera quais
aulas contam) **recalculam** o progresso de todas as matrículas do curso via job (`withTenant`), com a
UI atualizando no próximo carregamento.

### 3.5 Marcos de conclusão (milestones)

- **Marcos de curso:** 25%, 50%, 75%, 100% → toast/celebração leve ("Metade do caminho! 🎉") e, na F2,
  XP/badge (§5). Marco de **módulo concluído** também celebra.
- **100% do curso:** modal de conquista (ver §6.1) com CTA para certificado/quiz final.
- **Anti-spam:** cada marco celebra **uma vez** por matrícula (idempotente; estado em
  `lesson_progress`/agregado de matrícula).

**Eventos:** `course_milestone_reached` (`course_id`, `milestone: 25|50|75|100`);
`module_completed` (`module_id`, `course_id`); `course_completed` (`course_id`, `total_lessons`,
`days_to_complete`).

### 3.6 Recálculo e consistência

- O cliente **nunca** recalcula percentuais de verdade — só anima entre valores que o servidor fornece.
- Em caso de divergência (ex.: heartbeat perdido), a próxima entrada na sala de aula reconcilia
  (servidor recomputa a partir de `lesson_progress`). Sem números "fantasmas" persistentes na UI.

---

## 4. Comentários e comunidade

### 4.1 Comentários por aula [MVP]

Aba **Comentários** sob o player. Modelo de dados: `lesson_comments(lesson_id, user_id, body, parent_id)`
— suporta **threads de 1 nível** (comentário raiz + respostas). MVP mantém 1 nível de aninhamento
(respostas a respostas viram respostas ao raiz, citando o autor) para simplicidade de UI.

```
┌──────────────────────────────────────────────────────────────┐
│ Comentários (14)                 [ Mais recentes ▾ ]           │
│ ┌──────────────────────────────────────────────────────────┐ │
│ │ [avatar] Escreva uma dúvida ou comentário…   [ Publicar ] │ │
│ └──────────────────────────────────────────────────────────┘ │
│ ── Thread ───────────────────────────────────────────────────│
│ [av] Maria · há 2h                                             │
│      No minuto 5:30 fiquei confusa sobre…   [↳ 3 respostas]    │
│      ♥ 4   Responder   ⋯(denunciar/editar/excluir)            │
│      └ [av] Instrutor 🎓 · há 1h  ✓ Resposta do instrutor      │
│             Ótima pergunta! O que acontece é…  ♥ 8  Responder   │
│             [ Marcar como resolvido ✓ ]  (autor/instrutor)     │
│ ── …                                                           │
│ [ Carregar mais comentários ]                                  │
└──────────────────────────────────────────────────────────────┘
```

### 4.2 Comportamento e regras

- **Postar:** campo de texto (suporta quebras de linha, links auto-detectados, sem HTML arbitrário —
  sanitização server-side). Enter publica? Não — **Ctrl/Cmd+Enter** publica (Enter quebra linha) para
  evitar envios acidentais; botão "Publicar" sempre visível.
- **Editar/Excluir:** autor pode editar (janela curta com marca "editado") e excluir o próprio
  comentário. Instrutor/admin podem **ocultar** (moderação) ou excluir qualquer comentário.
- **Respostas:** "Responder" abre composer inline sob a thread; resposta entra como `parent_id`.
- **Respostas do instrutor:** comentários de `role ∈ {instructor, owner, admin}` recebem **badge 🎓
  "Instrutor"** e destaque visual. Filtro "Ver só respostas do instrutor".
- **Curtidas (♥):** toggle otimista; contagem agregada. (Tabela de likes a definir — ver dependências.)
- **Resolvido:** o **autor da dúvida** ou um **instrutor** pode marcar uma thread como **"Resolvido ✓"**;
  threads resolvidas podem ser recolhidas/filtradas ("Ocultar resolvidos"). (Campo de status de
  resolução a adicionar no modelo — ver dependências.)
- **Ordenação:** Mais recentes (default) | Mais curtidos | Não respondidos (útil para o instrutor).
- **Paginação:** cursor/infinite scroll ("Carregar mais"); contagem total no cabeçalho.
- **Tempo:** timestamps relativos ("há 2h"), absolutos no hover.

### 4.3 Moderação (ocultar/denunciar)

- **Denunciar (aluno):** menu ⋯ → "Denunciar" → motivo (spam, ofensivo, off-topic, outro). Gera item
  na fila de moderação do instrutor/admin. UI otimista: "Denúncia enviada, obrigado."
- **Ocultar (instrutor/admin):** remove da visão dos alunos (soft, reversível); autor vê "Comentário
  removido por um moderador". Painel de moderação (no admin/instrutor) lista denúncias pendentes.
- **Antiabuso:** rate limit de postagem (server-side); detecção básica de spam/links (F2).

### 4.4 Notificações de resposta

- **Quando notificar:** (a) alguém responde ao meu comentário; (b) instrutor responde à minha dúvida;
  (c) minha thread foi marcada como resolvida; (d) menção `@usuário` (F2).
- **Canais:** in-app (sino) no MVP; e-mail (Resend) configurável; push (PWA) na F2.
- **Preferências:** o aluno controla por tipo (respostas, instrutor, comunidade) em Configurações.

### 4.5 Estados

- **Vazio:** "Seja o primeiro a comentar nesta aula." + composer em destaque.
- **Carregando:** skeletons de 3 comentários.
- **Erro ao postar:** comentário fica com estado "Falha ao enviar — Tentar novamente" (não some o
  texto digitado).
- **Comentários desativados** (config do instrutor por curso/aula): "Os comentários estão desativados
  nesta aula."

**Eventos (comentários):**

| Evento | Quando | Propriedades |
|--------|--------|--------------|
| `comment_create` | publica comentário/resposta | `lesson_id`, `comment_id`, `is_reply`, `parent_id?` |
| `comment_edit` | edita | `comment_id` |
| `comment_delete` | exclui | `comment_id`, `by_role` |
| `comment_like` / `comment_unlike` | curte/descurte | `comment_id` |
| `comment_reply_instructor` | instrutor responde | `lesson_id`, `comment_id` |
| `comment_mark_resolved` | thread resolvida | `lesson_id`, `thread_id`, `by_role` |
| `comment_report` | denúncia | `comment_id`, `reason` |
| `comment_hidden` | moderação oculta | `comment_id`, `moderator_role` |
| `comment_notification_click` | clica notificação | `comment_id`, `channel` |

### 4.6 Fórum / feed / grupos e lives [F2]

> Não detalhado em wireframe completo (fora do MVP), mas a UX de comentários é desenhada para **escalar**
> para fórum/feed: mesmo componente de thread, mesma moderação, mesmas notificações.

- **Fórum por curso/tópico:** lista de tópicos (título, autor, nº respostas, última atividade, tags),
  criação de tópico, busca. Estados padrão (vazio/erro/carregando).
- **Feed:** timeline por curso/comunidade (posts + reações), pin de avisos do instrutor.
- **Grupos:** subcomunidades; entrada por curso/cohort.
- **Lives:** card de live agendada (embed Zoom/YouTube), countdown, "Adicionar ao calendário",
  estado **ao vivo agora** (badge pulsante) → **gravação disponível** após. Chat da live (F2/F3).

**Eventos (F2):** `forum_topic_create`, `forum_post_create`, `feed_post_create`, `feed_reaction`,
`group_join`, `live_reminder_set`, `live_join`, `live_replay_view`.

---

## 5. Gamificação (F2)

> **Prioridade F2.** Objetivo: reforçar o North Star (horas assistidas) e a conclusão, sem incentivar
> "farming" de pontos vazio.

### 5.1 Mecânicas e regras de pontuação

| Ação | XP (sugerido, configurável pelo tenant) | Antifarming |
|------|------------------------------------------|-------------|
| Concluir uma aula | +10 | Conta **uma vez** por aula (idempotente); só com conclusão válida (§3.4). |
| Concluir um módulo | +30 (bônus) | Uma vez por módulo. |
| Concluir um curso | +100 (bônus) | Uma vez por curso. |
| Passar num quiz | +20 (+ bônus por nota alta) | Por **primeira aprovação**; reprovações não pontuam; tentativas extras não acumulam. |
| Login diário / streak | +5/dia | Máx. 1×/dia; streak quebra se pular dia. |
| Comentar/ajudar (resposta marcada resolvida) | +5 | Cap diário; não pontua auto-respostas; moderação reverte XP de spam. |

- **Princípio antifarming:** XP é derivado de **eventos canônicos já validados server-side**
  (`lesson_completed_*`, `quiz_passed`, etc.), nunca de cliques de UI. Reverter ação (descurtir,
  comentário removido por moderação, reembolso) **reverte** o XP correspondente. Caps diários por
  categoria. Velocidade de conclusão suspeita (anti-seek já barra vídeo) é auditável.

### 5.2 Exibição na UI

- **Badge de XP/nível** no header da área logada e no dashboard (barra "Nível 4 · 320/500 XP para o
  Nível 5").
- **Conquistas (badges):** grade de badges conquistadas + bloqueadas (com dica de como obter). Ex.:
  "Maratonista" (5 aulas num dia), "Concluinte" (1 curso), "Curioso" (10 comentários úteis).
- **Ranking/leaderboard:** por turma/curso/período (semana/mês). Mostra top N + posição do usuário.
  **Opt-out de aparecer no ranking** (privacidade) e opção do tenant de **desativar ranking**.
- **Microinteração:** ao ganhar XP, animação `+10 XP` flutuante; ao subir de nível, modal de
  celebração; ao desbloquear badge, toast + entrada na grade.

### 5.3 Estados

- **Vazio:** "Comece a aprender para ganhar seus primeiros pontos." 
- **Gamificação desativada pelo tenant:** superfícies de XP/ranking não renderizam (feature flag).

**Eventos:** `gam_xp_awarded` (`source_event`, `amount`, `total_xp`), `gam_level_up` (`new_level`),
`gam_badge_unlocked` (`badge_id`), `gam_leaderboard_view` (`scope`, `period`, `user_rank`),
`gam_xp_reverted` (`source_event`, `amount`, `reason`), `gam_optout_toggle` (`opted_out`).

> **Nota de modelo de dados:** gamificação **não existe** no DATA_MODEL atual (F2). Ver dependências —
> precisa de tabelas `gamification_*` no schema de tenant.

---

## 6. Certificados

### 6.1 Tela de conquista (emissão)

Ao concluir o curso (100% + nota mínima, regra nº 5 do PRD), o backend enfileira a geração do PDF
(worker → R2) e cria `certificates`. Enquanto gera, a UI mostra estado intermediário.

```
┌──────────────────────────────────────────────────────────┐
│                 🎉  Parabéns, {nome}!                       │
│        Você concluiu o curso "{título}".                    │
│                                                            │
│   [ ⏳ Gerando seu certificado… ]   →   [ ⬇ Baixar PDF ]    │
│   [ 🔗 Link de verificação ]  [ Compartilhar ]             │
│   Emitido em DD/MM/AAAA · ID: {uuid_public}                 │
└──────────────────────────────────────────────────────────┘
```

- **Estados:**
  - **Elegível, gerando:** "Gerando seu certificado…" (polling/realtime); CTA de download desabilitado.
  - **Pronto:** botão "Baixar PDF" (URL assinada R2), "Copiar link de verificação", "Compartilhar"
    (LinkedIn/redes — adiciona ao perfil), pré-visualização da arte.
  - **Não elegível ainda:** se o aluno chega à tela sem cumprir requisitos → "Faltam: quiz final
    (nota mínima 70%) e 2 aulas" (lista clara do que falta).
  - **Revogado:** se `certificates.revoked = true` (ex.: reembolso/fraude) → "Este certificado foi
    revogado." (e a página pública também reflete).
  - **Erro de geração:** "Não foi possível gerar o certificado. Já estamos reprocessando." + retry
    automático (worker) e botão manual.
- **Acesso posterior:** o certificado fica acessível em `/meus-certificados` e no card do curso
  (CTA "Ver certificado").

### 6.2 Download

- Download via URL assinada de curta duração. Nome de arquivo amigável: `certificado-{curso}-{nome}.pdf`.
- Mobile: abre o PDF no viewer nativo / opção compartilhar.

### 6.3 Página pública de verificação

Rota pública (sem login): `/verificar/{uuid_public}` — UX para terceiros (empregadores) validarem.

```
┌──────────────────────────────────────────────────────────┐
│  ✅ Certificado válido                                      │
│  Aluno: {nome}                                             │
│  Curso: {título}                                           │
│  Emitido por: {tenant/escola}  ·  em DD/MM/AAAA            │
│  ID: {uuid_public}  ·  Verificação por hash ✓              │
│  [ QR code ]                                               │
└──────────────────────────────────────────────────────────┘
```

- **Estados:**
  - **Válido:** dados do certificado + selo verde + (opcional) carga horária.
  - **Não encontrado:** "Certificado não encontrado. Verifique o código." (404 amigável, sem vazar
    informação de outros tenants).
  - **Revogado:** selo vermelho "Certificado revogado" + data de revogação.
- **Privacidade:** expõe apenas o necessário (nome, curso, emissor, data, ID); sem dados sensíveis.
  Multitenant: a página é servida no domínio/branding do tenant emissor.
- **QR:** o QR no PDF aponta para esta página (`uuid_public`).

**Eventos:** `cert_issued` (`course_id`, `certificate_id`), `cert_generation_failed`,
`cert_download` (`certificate_id`), `cert_share_click` (`channel`),
`cert_verify_view` (`certificate_id`, `result: valid|not_found|revoked`) — este último é **público**
(sem `user_id` do verificador, só `tenant_id` e resultado).

---

## 7. Avaliações (quiz)

### 7.1 Tipos e contexto

- **MVP:** múltipla escolha e verdadeiro/falso, **correção automática**. Quiz pode ser **aula do tipo
  `quiz`** ou **prova de curso** (`quizzes.course_id` com `lesson_id` nulo).
- **F2:** dissertativa/lacuna/ordenar, **tentativas limitadas** (`max_attempts`), **nota mínima**
  (`pass_score`), **tempo** (`time_limit_sec`), **embaralhamento**.

### 7.2 Fazer o quiz (fluxo)

```
─ Tela de início ────────────────────────────────────────────
  Quiz: "Avaliação do Módulo 1"
  • 10 questões · Nota mínima 70% · Tentativas: 2 de 3 restantes
  • Tempo limite: 15:00 (se houver)
  [ Iniciar quiz ]
─ Em andamento ──────────────────────────────────────────────
  Questão 3 de 10            ⏱ 12:34          ▓▓▓▒▒▒▒▒▒▒
  Enunciado…
  ( ) Alternativa A
  (•) Alternativa B
  ( ) Alternativa C
  [ ‹ Anterior ]                       [ Próxima › ]
  ……
  [ Revisar respostas ]                [ Enviar quiz ]
─ Resultado ─────────────────────────────────────────────────
  ✅ Você passou!  Nota: 80% (8/10)
  [ Ver gabarito/feedback ]   [ Refazer (1 restante) ]  [ Continuar curso → ]
```

- **Navegação:** uma questão por tela (ou lista, configurável); barra de progresso de questões; salvar
  respostas parciais (autosave) para não perder em refresh.
- **Timer (F2):** countdown visível; ao zerar, **auto-submit** com aviso prévio (1 min restante).
- **Embaralhamento (F2):** ordem de questões/alternativas randômica por tentativa.
- **Confirmação de envio:** se houver questões sem resposta → modal "Você deixou 2 questões em branco.
  Enviar mesmo assim?".

### 7.3 Feedback, resultado e nota

- **Correção automática** server-side (UI nunca conhece o gabarito antes do envio — segurança).
- **Resultado:** nota (% e fração), aprovado/reprovado vs `pass_score`, tempo gasto.
- **Feedback por questão (configurável):** mostrar/não mostrar gabarito; explicação por questão se o
  instrutor cadastrou. Opções do instrutor: "mostrar respostas certas após envio" / "só a nota".
- **Tentativas:** mostra tentativas restantes; bloqueia "Refazer" quando esgotar (`max_attempts`).
  Política de nota: **maior nota** ou **última** (configurável; default maior).
- **Impacto no progresso/certificado:** quiz aprovado conta para conclusão do curso (§3.4/§6); a barra
  de progresso e a elegibilidade do certificado atualizam.

### 7.4 Estados

- **Vazio (sem questões):** visão do aluno não deve ocorrer (quiz publicado sem questões é barrado na
  autoria); fallback: "Este quiz ainda não está disponível."
- **Carregando:** skeleton de questão.
- **Erro ao enviar:** mantém respostas; "Falha ao enviar suas respostas. Tentar novamente." (não
  consome tentativa se o envio não foi registrado server-side — idempotência por `attempt_id`).
- **Tentativas esgotadas:** "Você usou todas as tentativas. Sua melhor nota: X%." 
- **Tempo esgotado:** auto-submit + aviso.

**Eventos:**

| Evento | Quando | Propriedades |
|--------|--------|--------------|
| `quiz_start` | inicia tentativa | `quiz_id`, `attempt_number` |
| `quiz_question_answer` | responde questão | `quiz_id`, `question_id` (sem revelar correção) |
| `quiz_autosave` | autosave de progresso | `quiz_id`, `answered_count` |
| `quiz_submit` | envia | `quiz_id`, `attempt_id`, `answered_count`, `time_spent_sec` |
| `quiz_result` | resultado calculado | `quiz_id`, `attempt_id`, `score`, `passed`, `attempt_number` |
| `quiz_retry` | refaz | `quiz_id`, `attempts_remaining` |
| `quiz_time_expired` | auto-submit por tempo | `quiz_id`, `attempt_id` |
| `quiz_review_feedback` | abre gabarito/feedback | `quiz_id`, `attempt_id` |

---

## 8. Mapa consolidado de eventos de analytics

Resumo para o time de analytics (todos com `tenant_id`, `user_id`, `session_id`, `ts`, `device`):

- **Player/sala de aula:** `lesson_resume_shown`, `lesson_resume_from_start`, `lesson_play`,
  `lesson_pause`, `lesson_speed_change`, `lesson_caption_change`, `lesson_quality_change`,
  `lesson_seek`, `lesson_ended`, `lesson_next_autoplay`, `lesson_next_cancelled`, **`lesson_heartbeat`
  (alto volume)**, `lesson_completed_auto`, `lesson_completed_manual`, `lesson_uncompleted_manual`,
  `lesson_asset_download`, `lesson_blocked_view`, `lesson_unlocked`, `lesson_video_processing_view`,
  `lesson_video_ready_shown`, `lesson_access_denied_view`.
- **Progresso:** `dashboard_view`, `my_courses_view`, `course_card_continue_click`,
  `course_card_certificate_click`, `course_milestone_reached`, `module_completed`, `course_completed`.
- **Comentários/comunidade:** `comment_create`, `comment_edit`, `comment_delete`, `comment_like`,
  `comment_unlike`, `comment_reply_instructor`, `comment_mark_resolved`, `comment_report`,
  `comment_hidden`, `comment_notification_click`; (F2) `forum_*`, `feed_*`, `group_join`, `live_*`.
- **Gamificação (F2):** `gam_xp_awarded`, `gam_level_up`, `gam_badge_unlocked`, `gam_leaderboard_view`,
  `gam_xp_reverted`, `gam_optout_toggle`.
- **Certificados:** `cert_issued`, `cert_generation_failed`, `cert_download`, `cert_share_click`,
  `cert_verify_view` (público).
- **Quiz:** `quiz_start`, `quiz_question_answer`, `quiz_autosave`, `quiz_submit`, `quiz_result`,
  `quiz_retry`, `quiz_time_expired`, `quiz_review_feedback`.

**Métricas-chave que estes eventos alimentam:** North Star (horas assistidas — de `lesson_heartbeat`),
taxa de conclusão por curso (`course_completed`/matrículas), drop-off por aula (funil de
`lesson_play`→`lesson_completed_*`), watch-time/heatmap (F2, de heartbeats+seek), engajamento de
comunidade (`comment_*`), eficácia de avaliações (`quiz_result`).

---

## 9. Dependências e pontos para o coordenador

Pontos que **cruzam** com outros documentos/superfícies e precisam de decisão ou alinhamento:

### Flows (jornadas)
- **Onboarding da sala de aula:** definir o primeiro acesso pós-compra (tour rápido? destaque de
  "continuar"?). Cruza com o flow de compra→acesso (PRD §5.3).
- **Resolução de "continuar de onde parou"** quando há múltiplos cursos em progresso: definir se o
  dashboard mostra 1 ou N cards de retomada.

### Estados (máquina de acesso)
- **Sala de aula depende da máquina de estados de acesso** (`enrollments.status`: active/suspended/
  refunded/expired — PRD §3.6). Precisamos do contrato exato de cada estado para desenhar os painéis de
  §2.10 (textos/CTAs por estado, especialmente regularização de inadimplência).
- **Distinção clara** entre bloqueio de conteúdo (drip/pré-req) e bloqueio de acesso (pagamento) já
  está desenhada, mas precisa de alinhamento com o time de pagamentos sobre a mensagem de regularização.

### Notificações
- **Canais e preferências:** MVP define in-app + e-mail (Resend); push é F2 (PWA). Precisa de uma
  **central de preferências de notificação** (por tipo: respostas, instrutor, nova aula liberada,
  certificado pronto). Decidir se entra no MVP (recomendado, leve) ou F2.
- **Eventos que geram notificação:** resposta a comentário, instrutor respondeu, thread resolvida, nova
  aula liberada (drip), vídeo pronto (instrutor), certificado emitido, marco de progresso. Cruza com
  webhooks de saída (PRD §3.15) e com o worker (pg-boss).

### Analytics
- **`lesson_heartbeat` é alto volume** — definir com analytics a estratégia de
  amostragem/agregação/retEnção e se vai para PostHog cru ou para uma tabela agregada de watch-time.
- **Eventos públicos** (`cert_verify_view`) não têm `user_id` do verificador — confirmar política de
  privacidade/coleta.
- **Padronização de nomes/propriedades** deste documento precisa virar um **tracking plan** versionado
  (idealmente schema Zod em `packages/contracts` para os payloads de eventos — DRY back/front).

### Modelo de dados (lacunas a resolver — cruza com DATA_MODEL.md)
- **Comentários:** o modelo atual (`lesson_comments`) **não tem** campos para: status de **resolvido**,
  flag de **oculto/moderação**, **denúncias**, **curtidas**. Sugiro adicionar:
  `lesson_comments.resolved_at`, `hidden_at`, `hidden_by`; tabelas `comment_likes(comment_id, user_id)`
  e `comment_reports(comment_id, reporter_id, reason, status)`. **Requer ADR/atualização do modelo.**
- **Gamificação (F2):** inexistente no modelo. Precisa de `gamification_xp_ledger` (idempotente, com
  `source_event_id` para antifarming/reversão), `badges`, `user_badges`, agregado de XP/nível,
  `leaderboard_optout`.
- **Progresso/anti-seek:** `lesson_progress` precisa de campo para **tempo real assistido** acumulado
  (ex.: `real_watched_seconds`) separado de `watched_pct`, para a regra anti-seek. Confirmar no modelo.
- **Quiz (F2):** `quiz_attempts` precisa de `attempt_number`/`status` e idempotência por `attempt_id`;
  política de nota (maior vs última) e config de feedback por quiz.
- **Marcos/milestones:** decidir onde guardar idempotência de celebração de marcos (agregado por
  matrícula).
- **Certificado:** carga horária no certificado (campo de duração total do curso) — confirmar fonte.

### Gamificação (priorização)
- Confirmar valores de XP, badges iniciais e se ranking entra com **opt-out** (privacidade) — e se é
  configurável por tenant (recomendado: sim, feature flag).

### Autoria (instrutor) — fora deste escopo, mas dependente
- Várias UX aqui assumem **configurações de autoria** ainda não especificadas: permitir conclusão
  manual de vídeo, ligar/desligar comentários, mostrar gabarito do quiz, definir aulas opcionais,
  política de tentativas/nota. Recomendo um documento de **UX de Autoria/Instrutor** que defina esses
  toggles (este doc lista os que consumimos).
