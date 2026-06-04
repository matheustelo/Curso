# Aulas ao Vivo Interativas (Sala WebRTC) — Spec de Produto + Técnica

- **Versão:** 1.0 · **Data:** 2026-06-04
- **Status:** Proposta para validação do coordenador (feature nova — pós-MVP)
- **Fase recomendada:** **Fase 2 (F2)** — ver §1.3 e §16.
- **Autor:** Arquiteto de streaming ao vivo interativo (WebRTC / salas em tempo real / pipelines de gravação)
- **Relacionados:** [PRD.md](../PRD.md) (§3.1, §3.5, §3.16) · [ARCHITECTURE.md](../ARCHITECTURE.md) (§7, §9) · [DATA_MODEL.md](../DATA_MODEL.md) (§6) · [ROADMAP.md](../ROADMAP.md) (F2) · [ADR-0008 (Bunny)](../adr/0008-streaming-bunny.md) · [ADR-0011 (pg-boss)](../adr/0011-queue-pgboss.md) · [ADR-0012 (apoio)](../adr/0012-supporting-platform.md) · **[ADR-0015 (esta feature)](../adr/0015-live-classes-interactive.md)** · [RBAC_MATRIX](RBAC_MATRIX.md) · [NOTIFICATIONS_MATRIX](NOTIFICATIONS_MATRIX.md) · [MONETIZATION](MONETIZATION.md) · [ANALYTICS_AND_DASHBOARDS](ANALYTICS_AND_DASHBOARDS.md) · [NON_FUNCTIONAL_REQUIREMENTS](NON_FUNCTIONAL_REQUIREMENTS.md) · [BUSINESS_RULES_AND_STATES](BUSINESS_RULES_AND_STATES.md) · [AUTHORING_UX](AUTHORING_UX.md)

> **Filtros de qualidade não-negociáveis (CLAUDE.md):** **SOLID/DRY/Clean Code**, stack **leve**, e a
> **Regra nº1 — isolamento absoluto por tenant**. Toda decisão abaixo respeita esses filtros. O provedor de
> tempo real é isolado atrás de uma **port `LiveProvider`** (DIP), e **a gravação reentra no pipeline VOD da
> Bunny** (ADR-0008), virando uma aula gravada normal — sem duplicar o domínio de vídeo.

---

## Índice

1. [Objetivo, escopo e modos](#1-objetivo-escopo-e-modos)
2. [Decisão de provedor (resumo) e por que F2](#2-decisão-de-provedor-resumo-e-por-que-f2)
3. [Jornadas e flows](#3-jornadas-e-flows)
4. [UX da sala (estados)](#4-ux-da-sala-estados)
5. [Agendamento e calendário](#5-agendamento-e-calendário)
6. [Controle de acesso por matrícula (entitlement + token de sala)](#6-controle-de-acesso-por-matrícula-entitlement--token-de-sala)
7. [Gravação → VOD (pipeline passo a passo)](#7-gravação--vod-pipeline-passo-a-passo)
8. [Presença / attendance (e relação com conclusão/certificado)](#8-presença--attendance-e-relação-com-conclusãocertificado)
9. [Moderação](#9-moderação)
10. [RBAC](#10-rbac)
11. [Notificações](#11-notificações)
12. [Monetização / quota por plano](#12-monetização--quota-por-plano)
13. [Analytics (eventos com tenant_id)](#13-analytics-eventos-com-tenant_id)
14. [NFR (latência, concorrência, uptime, reconexão)](#14-nfr-latência-concorrência-uptime-reconexão)
15. [Modelo de dados PROPOSTO](#15-modelo-de-dados-proposto)
16. [Port `LiveProvider` (SOLID)](#16-port-liveprovider-solid)
17. [Tempo real / infra: chat e presença (revisita ADR-0011 — Redis?)](#17-tempo-real--infra-chat-e-presença-revisita-adr-0011--redis)
18. [Faseamento recomendado](#18-faseamento-recomendado)
19. [Dependências e pontos para o coordenador](#dependências-e-pontos-para-o-coordenador)

---

## 1. Objetivo, escopo e modos

### 1.1 Objetivo

Permitir que um **Instrutor (IN)** — ou Admin/Owner — conduza uma **aula ao vivo interativa** dentro de um
curso, na qual **alunos matriculados (ST)** participam com **câmera/áudio**, **chat em tempo real**,
**levantar a mão** e **Q&A moderado**. A sessão é **gravada** e, ao encerrar, a gravação **reentra no
pipeline VOD da Bunny (ADR-0008)**, transformando-se em uma **aula gravada normal** (replay), com a mesma
proteção, player, tracking de progresso e regras de entitlement já existentes.

Esta feature substitui a antiga linha do PRD §3.1/§3.5 de "aula ao vivo (embed Zoom/YouTube Live)" por uma
**sala nativa interativa** — uma evolução de escopo (registrada no ADR-0015). O embed simples (link externo)
permanece como degradação/alternativa de baixo custo (ver §19).

### 1.2 Escopo (in/out)

**Dentro do escopo (F2):**

- Sala WebRTC **interativa** com vídeo+áudio de múltiplos participantes, screen share, chat, raise hand, Q&A.
- **Gravação na nuvem** (composite) e **reingestão automática no Bunny** como aula VOD.
- **Agendamento** de sessões vinculadas a uma aula (`lessons.type='live'`) e calendário do aluno.
- **Controle de acesso por matrícula** (entitlement) e **token de sala** com escopo de tenant (Regra nº1).
- **Presença/attendance** e ligação opcional com conclusão da aula.
- **Moderação** (mutar, remover, banir, travar chat).
- **Quota por plano** (horas de live/mês, participantes simultâneos) em `platform_plans.limits`.

**Fora do escopo (F3+ ou won't):**

- **Breakout rooms** (salas paralelas) — F3.
- **Webinar massivo > limite de sala interativa** (centenas/milhares assistindo): tratar via **modo broadcast
  (HLS/RTMP) 
  unidirecional** futuramente (F3), não como sala WebRTC interativa (ver §1.4).
- Whiteboard colaborativo, enquetes ricas, legendas ao vivo (live captions) — F3.
- Streaming simultâneo para YouTube/Twitch (RTMP out) — F3 (a port deixa o gancho previsto).
- DRM/anti-screen-grab na sala ao vivo — fora de escopo (a proteção forte vive no VOD pós-gravação).

### 1.3 Modos e limites

| Modo | Descrição | Limite-alvo (config. por plano) | Fase |
|------|-----------|----------------------------------|------|
| **Sala interativa** | Todos podem publicar câmera/áudio (sujeito a moderação). É o foco desta spec. | **até ~50 publicadores simultâneos** por sala (recomendado para qualidade); teto duro **100 conexões** alinhado ao provedor | F2 |
| **Sala "palco + plateia" (host-mostly)** | Instrutor + poucos convidados publicam; demais entram mutados e "levantam a mão" para subir ao palco | até ~50 no palco; plateia limitada pela quota de concorrência do plano | F2 |
| **Broadcast unidirecional (HLS/RTMP)** | 1→muitos, não interativo (apenas chat). Para audiências grandes. | centenas/milhares (não é WebRTC bidirecional) | **F3** |

> **Por que limitar a sala interativa a ~50 publicadores:** acima disso, a qualidade de uma SFU WebRTC com
> todos publicando degrada e o custo de participante-minuto cresce linearmente. Audiências grandes pedem o
> **modo broadcast** (HLS), que é F3. No F2 entregamos a sala interativa (caso de uso de turma/cohort).

### 1.4 Relação com o catálogo existente

- A aula ao vivo é uma `lessons` com **`type='live'`** (valor já previsto no DATA_MODEL §2.2).
- A **gravação** vira **a mesma aula** (preenchendo `video_guid` + `video_status`) ou, opcionalmente, uma
  **nova aula `type='video'`** de replay (configurável — ver §7.6). Decisão default: **mesma aula** (a aula
  ao vivo "amadurece" em VOD), preservando progresso/posição no currículo.

---

## 2. Decisão de provedor (resumo) e por que F2

**Recomendado:** **LiveKit** — usar **LiveKit Cloud** no F2 (operação leve) com **caminho de migração para
self-hosted** (open source, Apache-2.0) quando o volume justificar; a **port `LiveProvider`** torna a troca
local. **Plano B:** **Daily** (managed puro, DX excelente). Detalhes, prós/contras, custos e nível de
confiança no **[ADR-0015](../adr/0015-live-classes-interactive.md)**.

Razões resumidas (a análise completa está no ADR-0015):

- **Gravação → VOD elegante:** o **Egress** do LiveKit grava **composite MP4** direto para **storage
  S3-compatível** — e nós já usamos **Cloudflare R2** (ADR-0012, egress zero). O Bunny então **busca o
  arquivo por URL** (fetch-from-URL) e o transforma em VOD. Zero download/reupload no nosso backend.
- **Webhooks** nativos (`room_started`, `room_finished`, `egress_ended`) encaixam no padrão de webhook +
  `JobQueue` (pg-boss) já estabelecido (ARCHITECTURE §9, ADR-0011).
- **Multitenancy por design:** acesso por **JWT de sala** com grants por participante (publish/subscribe,
  nome da sala) — casa com nosso modelo de token + entitlement (ADR-0008, Regra nº1).
- **Stack leve + sem lock-in:** open source permite self-host futuro; a port impede acoplamento.
- **Custo previsível** por participante-minuto + recording-minuto, calibrável por quota de plano.

**Por que F2 (não MVP):** o MVP fecha o loop de receita (criar → vender → entregar VOD → acompanhar). Live
interativo adiciona um **fornecedor de tempo real**, **infra de gravação** e **novas máquinas de estado** —
valor de **retenção/diferenciação**, não de fundação. O ROADMAP já posiciona "lives" em F2. Entregar no MVP
atrasaria o loop de valor e violaria o princípio "travar no loop MVP" (PRD §7).

---

## 3. Jornadas e flows

> Notação: `[guarda]` = pré-condição. Todo acesso a dados de tenant ocorre em `withTenant(tenantId, fn)`.

### 3.1 Agendar uma aula ao vivo (Instrutor — no Estúdio `/manage`)

1. IN abre o **Editor de aula** (AUTHORING_UX §5) e cria/edita uma aula com **tipo `live`**.
2. Define **título, módulo/posição, data/hora de início, duração estimada, fuso** e modo (interativa /
   host-mostly).
3. (Opcional) Configura **gravar automaticamente** (default **ligado**), **chat habilitado**, **Q&A
   habilitado**, **lobby/sala de espera**, e se a **presença conta para conclusão** (§8).
4. Backend cria `live_sessions(status='scheduled')` **no schema do tenant** e agenda os **lembretes**
   (NOTIFICATIONS) via `JobQueue`.
5. A aula aparece no **currículo** (bloqueada/agendada) e no **calendário** do aluno matriculado (§5).

### 3.2 Entrar na sala como Instrutor (host)

1. No horário (janela de abertura T-15min, configurável), IN clica **"Iniciar aula ao vivo"**.
2. Backend valida **RBAC** (host = IN autor/atribuído, ou AD/OW) + **entitlement de host** + **quota de plano**
   (horas de live/concorrência — §12) → chama `LiveProvider.createRoom()` (idempotente por `live_session_id`)
   → emite **token de sala** com grants de host (`canPublish`, `roomAdmin`/moderação).
3. `live_sessions.status: scheduled → live` (gatilho confirmado pelo webhook `room_started`).
4. IN entra na **UI da sala** (host controls). Se "gravar automaticamente" estiver ligado, o backend dispara
   `LiveProvider.startRecording()` ao iniciar (ou ao 1º participante).

### 3.3 Entrar na sala como Aluno

1. ST clica **"Entrar na aula ao vivo"** (no player/currículo ou no e-mail/in-app de lembrete "ao vivo
   agora").
2. Backend valida **entitlement** (`enrollments.status='active'` no curso da aula) **[guarda]** + sessão
   `live` (ou em `lobby`) + quota de concorrência → emite **token de sala** com grants de aluno (publish
   conforme modo; subscribe sempre).
3. ST passa pelo **device check** (permissão de câmera/mic, seleção de dispositivo, preview) → entra.
4. Registro de presença começa (`live_attendance` join — §8).

### 3.4 Durante a aula (interações)

- **Vídeo/áudio:** publicar/parar câmera e mic; ver grade de participantes (ativos/falando).
- **Screen share:** host sempre; aluno conforme permissão do modo (default: só host/convidados).
- **Chat:** mensagens em tempo real (persistidas em `live_chat_messages` — §15, §17).
- **Levantar a mão:** sinaliza ao host (fila de mãos levantadas); host pode "convidar ao palco".
- **Q&A:** perguntas enviadas → host marca **respondida**/destaca; (F3: upvote de perguntas).
- **Moderação (host):** mutar, remover, banir, travar chat, baixar mão (§9).

### 3.5 Encerrar a aula

1. Host clica **"Encerrar aula"** → backend `LiveProvider.stopRecording()` (se ativa) +
   `LiveProvider.endRoom()`.
2. `live_sessions.status: live → ended`. Webhook `room_finished` confirma; `live_attendance` fecha sessões
   abertas.
3. Webhook `egress_ended` chega com a **URL do arquivo no R2** → enfileira job de **reingestão VOD** (§7).
4. `live_sessions.recording_status: recording → processing`; aluno vê "Gravação em processamento".

### 3.6 Replay (aula gravada disponível)

1. Job de reingestão sobe o arquivo ao **Bunny** (fetch-from-URL) → `lessons.video_status` segue a máquina
   normal (`queued → processing → ready`) (BUSINESS_RULES §3).
2. Ao `ready`: `live_sessions.recording_status: processing → ready`; a aula passa a ter **replay VOD**
   (player Bunny, token, anti-seek, progresso) idêntico a qualquer aula de vídeo.
3. Notificação "replay disponível" (§11). A aula ao vivo encerrada **vira VOD normal** — objetivo central.

---

## 4. UX da sala (estados)

Tela **T-LIVE** (nova superfície, parente de T4 Player — NFR §1). Layout responsivo, mobile-first, WCAG AA.

### 4.1 Layout

```
┌──────────────────────────────────────────────────────────────┐
│ [● AO VIVO 12:34]  Título da aula        [Participantes 23] [⚙]│
├───────────────────────────────────────┬──────────────────────┤
│                                        │  Aba: Chat | Pessoas  │
│         PALCO (speaker ativo /          │  | Q&A | Mãos (host) │
│         screen share / grade)           │ ┌──────────────────┐ │
│                                        │ │ chat em tempo real │ │
│   ┌────┐ ┌────┐ ┌────┐ ┌────┐          │ │ ...                │ │
│   │vid │ │vid │ │vid │ │vid │  thumbs   │ │                    │ │
│   └────┘ └────┘ └────┘ └────┘          │ └──────────────────┘ │
├───────────────────────────────────────┴──────────────────────┤
│ [🎤 mute] [📷 cam] [🖥 share] [✋ mão] [⛶] ... [host: ⏺ rec ⏹ end]│
└──────────────────────────────────────────────────────────────┘
```

- **Badge "AO VIVO"** com cronômetro; indicador de **gravando** (⏺) visível a todos (transparência/LGPD).
- **Painel lateral** com abas: **Chat**, **Pessoas** (lista + status), **Q&A**, **Mãos levantadas** (host).
- **Barra de controles**: mic, câmera, screen share, levantar mão, tela cheia; **controles de host** (rec,
  encerrar, moderação) só para quem tem grant.

### 4.2 Estados de tela (obrigatórios)

| Estado | Quando | UX |
|--------|--------|----|
| **Agendada (countdown)** | `scheduled`, antes da abertura | "Começa em HH:MM" + botão "Adicionar ao calendário"; sem sala. |
| **Lobby / sala de espera** | sessão aberta, aluno aguardando host | "Aguardando o instrutor iniciar"; device check disponível. |
| **Device check** | antes de entrar | Permissão de câmera/mic, seleção e preview; aviso se bloqueado pelo navegador. |
| **Carregando (conectando)** | join em progresso | Skeleton da grade + spinner; sem CLS. |
| **Ao vivo (conectado)** | `live` | Layout §4.1. |
| **Reconexão** | queda de rede transitória | Banner "Reconectando…"; mantém UI; tenta `LiveProvider` reconnect (§14). |
| **Erro de mídia** | sem permissão/dispositivo | Mensagem acionável: "Habilite a câmera nas configurações do navegador". |
| **Sala cheia (quota)** | concorrência atingida | "A sala atingiu o limite de participantes" (quota de plano — §12). |
| **Sem entitlement** | matrícula inativa | Bloqueio com CTA de compra/regularização (Regra nº1). |
| **Encerrada** | `ended` | "A aula foi encerrada. A gravação estará disponível em breve." |
| **Replay em processamento** | `recording_status processing` | Card "Processando gravação…" (espelha `video_status`). |
| **Replay disponível** | `recording_status ready` | Player VOD Bunny normal (T4). |
| **Sem gravação** | gravação desligada/falhou | "Esta aula não foi gravada." / aviso de falha ao host (§7.7). |

### 4.3 Acessibilidade (WCAG AA)

- Navegação por teclado em todos os controles; foco visível; `aria-live` para entradas/saídas e novas
  mensagens de chat; contraste AA; legendas no replay VOD (VTT). Live captions = F3.

---

## 5. Agendamento e calendário

- **Agendamento** vive em `live_sessions` (data plane do tenant): `scheduled_start_at`, `scheduled_end_at`,
  `timezone`. Vinculado a uma `lessons.type='live'` (1 aula ↔ 1..N sessões; reagendar cria nova sessão ou
  atualiza a existente).
- **Calendário do aluno:** agrega as sessões `scheduled`/`live` dos cursos em que tem matrícula `active`.
  Exibe no dashboard do aluno e na sala de aula (currículo) com o estado de countdown (§4.2).
- **"Adicionar ao calendário":** gera **.ics** (e link Google Calendar) — geração leve no backend, sem novo
  fornecedor.
- **Reagendamento/cancelamento:** transição de estado (§15) + notificação aos matriculados (§11).
- **Fuso horário:** armazenado em UTC; exibido no fuso do usuário (i18n/next-intl). Sessões respeitam
  branding/i18n do tenant nos lembretes.

---

## 6. Controle de acesso por matrícula (entitlement + token de sala)

> **Regra nº1 + Regra de entitlement (PRD §4.2):** o backend é a **única** autoridade que emite o token de
> sala, **sempre** após validar matrícula ativa, e **sempre** com escopo de tenant.

### 6.1 Princípios

1. **Token emitido só pelo backend.** Análogo ao Embed Token da Bunny (ADR-0008): nunca no front; **TTL
   curto** (ex.: 1–4h, ≤ duração prevista + folga); assinado com a **API key/secret do LiveKit** (cifrada em
   repouso, igual às keys Bunny — CLAUDE.md anti-padrão "expor keys no front").
2. **Escopo de tenant no nome da sala.** Nome da sala namespaced: **`t_<tenantId>__ls_<liveSessionId>`**.
   Garante que dois tenants nunca colidam de sala e que o token só dá acesso àquela sala daquele tenant.
3. **Grants por papel/modo** no token: host → `canPublish + roomAdmin` (moderação); aluno → `canSubscribe`
   sempre, `canPublish` conforme modo (interativa: sim; host-mostly: só após "subir ao palco"). `identity`
   do participante = `user_id` do schema do tenant (nunca e-mail — alinha com ANALYTICS §2).
4. **Revalidação no use-case (defense in depth).** O guard de rota + o use-case revalidam papel, ownership e
   `enrollment.active` antes de emitir o token (RBAC §2.6).
5. **Token de sala ≠ entitlement de VOD.** O replay usa o fluxo de Embed Token Bunny já existente; são dois
   tokens distintos, ambos sob entitlement.

### 6.2 Fluxo de emissão (use-case `JoinLiveSession`)

```
POST /courses/:id/lessons/:lessonId/live/:sessionId/token
  onRequest: resolve tenant (search_path) + sessão/JWT (tenant_id, user_id, role)
  use-case JoinLiveSession (withTenant):
    1. carrega live_session; [guarda] status ∈ {lobby, live}
    2. valida entitlement: enrollment.active no course (ou role host) [Regra nº1]
    3. valida quota de concorrência do plano (valor injetado no onRequest — ver §12)
    4. resolve grants por role/modo
    5. liveProvider.createToken({ room: t_<tenant>__ls_<id>, identity: user_id, grants, ttl })
    6. registra intenção de presença; retorna { wsUrl, token }
```

### 6.3 Isolamento de gravação

- A gravação é escrita em **R2 sob prefixo por tenant**: `r2://live-recordings/<tenantId>/<liveSessionId>/…`
  (mesmo princípio de isolamento de storage; nenhuma FK cross-schema). O job de reingestão (§7) roda em
  `withTenant` e só toca a `lessons`/`live_sessions` do tenant dono.

---

## 7. Gravação → VOD (pipeline passo a passo)

> **Objetivo central:** a gravação vira **aula VOD normal** no Bunny (ADR-0008), reusando player, token,
> anti-seek, progresso e proteção. Sem duplicar o domínio de vídeo. Tudo idempotente e via `JobQueue`
> (pg-boss, ADR-0011), com `tenantId` no payload.

### 7.1 Visão (fluxo)

```
[Sala WebRTC]  --start/stopRecording-->  [Egress do provedor grava COMPOSITE MP4]
       │                                          │ (auto-upload)
       │ webhook room_finished                    ▼
       ▼                                   [Cloudflare R2: <tenantId>/<sessionId>/rec.mp4]
[live_sessions: live→ended]                       │ webhook egress_ended (URL do arquivo)
       │                                          ▼
       └──────────────── verifica HMAC ──> [API Fastify /webhooks/live]
                                                  │ responde 200 + enfileira (idempotente)
                                                  ▼
                                   [JobQueue pg-boss: live.recording.ingest {tenantId, sessionId}]
                                                  ▼
                              [Worker withTenant: Bunny.createVideo + fetch-from-URL(R2 presigned)]
                                                  │  (ou TUS, fallback — ver 7.4)
                                                  ▼
                                   [lessons.video_status: none→queued→processing]
                                                  │ webhook "vídeo pronto" (Bunny, HMAC) — ADR-0008
                                                  ▼
                                   [lessons.video_status: processing→ready ; video_guid setado]
                                   [live_sessions.recording_status: processing→ready]
                                                  ▼
                                   [Notificação "replay disponível" (§11)]
```

### 7.2 Passo a passo detalhado

1. **Início da gravação:** ao iniciar a sala (ou 1º participante), o backend chama
   `LiveProvider.startRecording(sessionId)` → o **Egress composite** do provedor grava a **mistura** (grade +
   screen share + áudio) e está configurado para **auto-upload** para o **R2** (S3-compatível) sob o prefixo
   do tenant. `live_sessions.recording_status: none → recording`, `recording_provider_ref` guarda o
   `egress_id`.
2. **Fim da gravação:** `stopRecording` no encerrar (ou automático no `room_finished`).
3. **Webhook `egress_ended`:** endpoint Fastify dedicado `/webhooks/live`:
   - **verifica HMAC/assinatura** do provedor (bytes brutos, comparação em tempo constante — mesmo padrão da
     Bunny, ADR-0008);
   - responde **200** rápido (NFR-PERF-20: ≤ 200 ms);
   - **enfileira** `live.recording.ingest` no pg-boss (idempotente por `egress_id`/`event_id`); nunca
     processa pesado no handler.
4. **Worker `live.recording.ingest` (`withTenant`):**
   - lê `live_sessions` + resolve a `lessons` alvo (§7.6);
   - gera **URL de leitura assinada (presigned) do R2** para o arquivo;
   - chama o **Bunny**: `createVideo` na **library do tenant** (ADR-0008) e dispara **fetch-from-URL** com a
     presigned URL **(método primário)**;
   - grava `video_guid` na `lessons`; `lessons.video_status: none → queued`;
   - `live_sessions.recording_status: recording → processing`.
5. **Encoding Bunny + webhook "vídeo pronto" (existente, ADR-0008):** o worker já existente de webhook Bunny
   move `video_status: processing → ready` e marca a sessão `recording_status: ready`.
6. **Disponibilização:** a aula passa a ter replay VOD; notificação "replay disponível" (§11).
7. **Retenção do master:** mantém o **MP4 original no R2** (mitiga lock-in e permite re-encode — alinha
   ARCHITECTURE §7 "manter masters no R2"). Política de retenção/limpeza configurável (§19).

### 7.3 Por que R2 como ponte (e não download no nosso backend)

- **Egress zero do R2** (ADR-0012) + **fetch-from-URL do Bunny** = a transferência ocorre **provedor → R2 →
  Bunny** sem passar banda pelo nosso backend (mesmo princípio do upload TUS direto ao edge no ADR-0008).
- Mantém o backend **leve** e barato; o trabalho pesado de mídia continua sendo da Bunny.

### 7.4 Fallback de ingestão

- Se `fetch-from-URL` falhar (ex.: presigned expirada, indisponibilidade), o worker faz **fallback para
  upload TUS resumable** do R2 → Bunny (streaming, sem materializar o arquivo inteiro em memória). Retentável
  via pg-boss com backoff.

### 7.5 Idempotência

- Dedup por `egress_id`/`event_id` (tabela de eventos análoga a `payment_events` — ver §15
  `live_recording_events`). Reprocessar webhook **não** cria vídeo duplicado nem dispara dupla notificação.

### 7.6 Mesma aula vs nova aula de replay (configurável)

| Estratégia | Efeito | Default |
|------------|--------|---------|
| **Mesma aula amadurece** | A própria `lessons.type='live'` recebe `video_guid`; após `ready` o player passa a exibir o VOD. Mantém posição no currículo e progresso. | **Sim (default)** |
| **Nova aula de replay** | Cria uma `lessons.type='video'` separada para o replay, deixando a aula live como "evento". | Opcional (toggle no Estúdio) |

### 7.7 Falhas

- `egress_ended` com status de falha, ou `ingest` esgotado de retries → `live_sessions.recording_status:
  recording/processing → failed`; notifica host (`video_failed`-like, §11); host pode tentar **reingestão
  manual** se o master existir no R2.

---

## 8. Presença / attendance (e relação com conclusão/certificado)

- **Registro:** `live_attendance` grava `joined_at`/`left_at` por participante (uma linha por intervalo de
  presença; reconexões geram novo intervalo). Origem da verdade: **webhooks** `participant_joined`/
  `participant_left` do provedor (server-side), não o cliente.
- **Tempo presente acumulado:** somatório dos intervalos → `attended_seconds` (derivado), comparável à
  duração da sessão.
- **Relação com conclusão (configurável — alinhado a AUTHORING_UX §8 / toggles):**
  - Default: **assistir o replay (VOD)** conclui a aula pela **regra anti-seek normal** (PRD §4.4:
    `watched_pct ≥ 90` **e** `real_watched_seconds ≥ duration×0.8`). Ou seja, quem perdeu o ao vivo conclui
    pelo replay como qualquer aula de vídeo.
  - **Toggle "presença ao vivo conclui a aula"** (por aula): se ligado, presença `attended_seconds ≥
    duration×<limiar>` (limiar default 0.8) marca `lesson_progress.status='completed'` para aquela aula,
    **respeitando o vocabulário canônico** (`not_started | in_progress | completed`).
- **Certificado:** **inalterado** — segue a regra existente (BUSINESS_RULES §7 / AUTHORING_UX §8): 100% das
  aulas **não-opcionais** concluídas (+ nota mínima se exigida). A aula live, quando vira VOD, conta
  exatamente como uma aula de vídeo. Aulas live podem ser marcadas **opcionais** (não contam para %), reusando
  `lessons.is_optional`.

---

## 9. Moderação

Reusa o vocabulário de moderação de comunidade (DATA_MODEL §6.3) onde aplicável e adiciona moderação de sala:

| Ação (host: IN autor/atribuído, AD, OW) | Efeito | Persistência/auditoria |
|------------------------------------------|--------|------------------------|
| **Mutar participante** | `LiveProvider.muteParticipant` (revoga publish de áudio) | evento de moderação |
| **Desligar câmera de participante** | revoga publish de vídeo | evento |
| **Remover (kick)** | `LiveProvider.removeParticipant` (desconecta); pode reentrar | evento |
| **Banir** | remove + invalida token futuro (`live_bans` por sessão/curso) | `live_bans` + auditoria |
| **Travar/destravar chat** | `live_sessions.chat_locked` | flag |
| **Ocultar mensagem de chat** | `live_chat_messages.hidden_at/hidden_by` (espelha `comment_*`) | soft moderação |
| **Baixar mão / limpar fila** | reseta sinal de raise hand | efêmero |
| **Encerrar para todos** | `endRoom` | transição de estado |

- **Escopo de tenant + RBAC (C6):** o host só modera sessões dos **seus** cursos. Banir/remover sensível →
  `platform.audit_log` (escopo de tenant via metadata `tenant_id`).

---

## 10. RBAC

Alinhado à [RBAC_MATRIX](RBAC_MATRIX.md) (papéis `SA/OW/AD/IN/AF/ST`). Proposta de novas linhas (a integrar
na §3 da matriz — domínio "Aulas ao vivo"):

| Ação / Recurso | SA | OW | AD | IN | AF | ST | Obs |
|----------------|----|----|----|----|----|----|-----|
| Agendar/editar aula ao vivo | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 (IN só nos seus cursos) |
| Iniciar/encerrar sala (host) | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 |
| Moderar sala (mutar/remover/banir) | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 + `audit_log` (banir) |
| Controlar gravação (rec/stop) | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 |
| Entrar na sala e publicar (aluno) | ❌ | 🔶 | 🔶 | 🔶 | ❌ | 🔶 | C8-live (entitlement + sessão ativa) |
| Assistir replay (VOD) | ❌ | 🔶 | 🔶 | 🔶 | ❌ | ✅ | C8 (entitlement — fluxo Bunny existente) |
| Ver/gerir LiveKit keys do tenant | ❌ | 🔶 | ❌ | ❌ | ❌ | ❌ | C9 (keys nunca no front) |

- **Nova condição C8-live:** o **token de sala** só é emitido pelo backend com `enrollment.active` (ou role
  host), sessão `lobby/live`, e quota não estourada (espelha C8 do VOD). Aluno suspenso/expirado: negado.
- **SA:** sem acesso silencioso; toca sala de tenant só via impersonação auditada (C3).

---

## 11. Notificações

Alinhado à [NOTIFICATIONS_MATRIX](NOTIFICATIONS_MATRIX.md) (canais MVP `in_app | email`; push = F2; disparo
idempotente via `JobQueue`; respeita `notification_preferences`, branding/i18n). A matriz já prevê
**"Live agendada / lembrete (F2)"** (§6 da matriz) — detalhamos e estendemos:

| Evento | E-mail | In-app | Push (F2) | Destinatário | Gatilho (transição) | Prio | Template |
|--------|:------:|:------:|:---------:|--------------|----------------------|:----:|----------|
| Aula ao vivo agendada | ✅ | ✅ | — | ST (matriculados) | `live_sessions →scheduled` | P2 | `live_scheduled` · `{course_title,live_title,start_at,add_to_calendar_url}` |
| Lembrete **24h** | ✅ | ✅ | ✅ | ST | T-24h da sessão | P2 | `live_reminder_24h` · `{live_title,start_at,join_url}` |
| Lembrete **10 min** | ✅ | ✅ | ✅ | ST | T-10min | P2 | `live_reminder_10m` · `{live_title,join_url}` |
| **AO VIVO agora** | — | ✅ | ✅ | ST | `live_sessions →live` | P1 | `live_started` · `{live_title,join_url}` |
| Aula reagendada/cancelada | ✅ | ✅ | — | ST | `→rescheduled/canceled` | P1 | `live_rescheduled`/`live_canceled` · `{live_title,new_start_at?}` |
| **Replay disponível** | ✅ | ✅ | ✅ | ST | `recording_status →ready` | P2 | `live_replay_ready` · `{lesson_title,course_title,lesson_url}` |
| Falha na gravação (host) | ✅ | ✅ | — | IN, OW/AD | `recording_status →failed` | P1 | `live_recording_failed` · `{live_title,error,retry_url}` |

> Os lembretes 24h/10min são **jobs agendados** no pg-boss no momento do agendamento; reagendar
> **reprograma** (cancela e recria) os jobs. Tenant suspenso pausa comunicações a alunos (regra geral da
> matriz).

---

## 12. Monetização / quota por plano

Alinhado a [MONETIZATION](MONETIZATION.md) (§A.2 features×quotas; `platform_plans.limits`) e ao serviço de
quotas central (DATA_MODEL §6.12). **Live é um recurso por tier** + **quotas numéricas**.

### 12.1 Chaves propostas em `platform_plans.limits` (control plane)

```jsonc
{
  // ... chaves existentes ...
  "features": [ /* ... */, "live_classes" ],     // booleana: tier tem live?
  "live_concurrent_participants": 50,            // teto de participantes simultâneos por sala
  "live_hours_month": 20,                         // horas de live por mês (host-time agregado)
  "live_recording_storage_gb": 50                 // (ou contabiliza junto do storage_gb existente)
}
```

> "ilimitado" = ausência da chave/`null` (fair-use), conforme convenção do MONETIZATION §A.2.

### 12.2 Proposta de liberação por tier (a confirmar com stakeholder)

| Recurso/Quota | Fase | Starter | Pro | Scale | Enterprise |
|---------------|------|---------|-----|-------|------------|
| **Aulas ao vivo interativas** | F2 | — | sim | sim | sim |
| **Participantes simultâneos/sala** | F2 | — | 50 | 100 | negociado |
| **Horas de live/mês** | F2 | — | 20 | 100 | negociado |

### 12.3 Enforcement (sem violar a Regra nº1)

- **Mesmo padrão do take rate (ADR-0013):** os limites vêm do **control plane** e são **injetados como valor**
  no `onRequest` (lookup `platform.tenants → plan_id → platform_plans.limits`), junto a `request.tenant`. O
  **use-case `JoinLiveSession`/`StartLiveSession`** depende de **números**, não consulta `platform`
  diretamente → preserva DIP + isolamento.
- **Uso corrente:** participantes simultâneos contados na sessão (data plane / estado do provedor); horas de
  live agregadas a partir de `live_sessions` (duração `live→ended`). O **serviço de quotas** (DATA_MODEL
  §6.12) compõe a decisão (autoriza/bloqueia + alerta 80%/100% — quota_warning/exceeded, §11 da matriz).
- **Política de overage:** participantes/concorrência = **hard block** ao atingir o teto (sala cheia, §4.2);
  horas de live = soft cap com alerta e CTA de upgrade (alinha overage do MONETIZATION §A.4). Custo de
  participante-minuto/recording é variável → calibrar quotas contra o custo do provedor (ver ADR-0015 +
  OPEN_QUESTIONS de egress).

---

## 13. Analytics (eventos com tenant_id)

Alinhado ao [ANALYTICS_AND_DASHBOARDS](ANALYTICS_AND_DASHBOARDS.md): naming `objeto_ação` no passado,
`snake_case`, **todo evento com `tenant_id`** (Regra nº1), eventos de consequência **server-side**, port
`AnalyticsProvider` (ADR-0014), schemas Zod em `packages/contracts`. Novos eventos propostos (domínio "live"):

| Evento | Lado | Quando dispara | Propriedades específicas |
|--------|------|----------------|--------------------------|
| `live_session_scheduled` | S | sessão agendada | `live_session_id, lesson_id, course_id, scheduled_start_at, mode` |
| `live_session_started` | S | `room_started` | `live_session_id, host_user_id` |
| `live_session_ended` | S | `room_finished` | `live_session_id, duration_sec, peak_participants` |
| `live_participant_joined` | S | `participant_joined` | `live_session_id, user_role, is_host` |
| `live_participant_left` | S | `participant_left` | `live_session_id, attended_sec` |
| `live_hand_raised` | C | aluno levanta a mão | `live_session_id` |
| `live_question_asked` | S | pergunta enviada (Q&A) | `live_session_id` |
| `live_chat_message_sent` | S | mensagem persistida | `live_session_id` (sem conteúdo/PII) |
| `live_recording_ingest_started` | S | job de reingestão inicia | `live_session_id` |
| `live_replay_ready` | S | `recording_status →ready` | `live_session_id, lesson_id, video_guid` |
| `live_recording_failed` | S | `recording_status →failed` | `live_session_id, error_code` |

- **KPIs derivados:** taxa de comparecimento (matriculados convidados → presentes), pico de concorrência,
  duração média, % de conclusão via replay vs via presença, custo por participante-minuto (FinOps).
- **Sem PII** no payload (sem conteúdo de chat, sem e-mail/tokens). Heartbeat de presença é agregável.

---

## 14. NFR (latência, concorrência, uptime, reconexão)

Estende [NON_FUNCTIONAL_REQUIREMENTS](NON_FUNCTIONAL_REQUIREMENTS.md). Tela **T-LIVE** (parente de T4).
IDs propostos `NFR-LIVE-NN`:

| ID | Métrica | Meta (p75/p95) | Limite "falha" |
|----|---------|----------------|----------------|
| **NFR-LIVE-01** | **Join-to-media** (clique "entrar" → 1º frame remoto), inclui validação de entitlement + emissão de token | ≤ 3,0 s (p75) | > 6,0 s |
| **NFR-LIVE-02** | Latência da API de **emissão de token de sala** (backend, p95) | ≤ 300 ms | > 800 ms |
| **NFR-LIVE-03** | **Latência de mídia** ponta-a-ponta (glass-to-glass) WebRTC | ≤ 400 ms típico | > 1 s sustentado |
| **NFR-LIVE-04** | **Latência de chat** (envio → entrega aos demais) | ≤ 500 ms (p95) | > 2 s |
| **NFR-LIVE-05** | **Reconexão automática** após queda transitória de rede (ICE restart) | ≤ 5 s para restabelecer | > 15 s → estado "Erro" |
| **NFR-LIVE-06** | **Webhook de entrada** (responder 200 + enfileirar) — reusa NFR-PERF-20 | ≤ 200 ms | > 500 ms |
| **NFR-LIVE-07** | **Disponibilidade da sala** (uptime do serviço de tempo real, herdado do provedor) | ≥ 99,9% | < 99,5% |
| **NFR-LIVE-08** | **Tempo até replay pronto** (encerrar → `recording_status=ready`) | ≤ 2× duração da aula (p75) | > 6× duração |
| **NFR-LIVE-09** | **Concorrência por sala** (publicadores) | conforme plano (≤ 50 recomendado) | teto duro 100 conexões |
| **NFR-LIVE-10** | **Buffering/qualidade adaptativa** (simulcast/SVC, queda graciosa em rede ruim) | adaptativo automático | congelamento > 3 s |

- **Reconexão (NFR-LIVE-05):** o cliente do provedor faz ICE restart automático; a UI mostra "Reconectando…"
  (§4.2) e preserva estado; presença reabre intervalo em `live_attendance`.
- **Concorrência:** o **teto duro de 100 conexões** vem do tier do provedor (LiveKit Cloud Build/Ship) — a
  quota de plano (§12) fica **abaixo** desse teto. Escala maior → tier superior ou self-host (ADR-0015).
- **Degradação graciosa:** simulcast/SVC para alunos em rede 4G ruim (alinha ao objetivo de NFR §2: aluno BR
  mid-tier).

---

## 15. Modelo de dados PROPOSTO

> **Proposta** para o DATA_MODEL §6 (a integração formal é do coordenador — ver ADR-0015 e §19). Convenções
> do DATA_MODEL: PK `uuid`, timestamps UTC, soft-delete onde fizer sentido, `status` com **vocabulário
> canônico**. **Nenhuma FK cruza schemas**; tudo no schema do **tenant**, acessado via `withTenant`. Marcação
> de fase: **[F2]**. Cada tabela com dado de tenant exige **teste de isolamento cross-tenant** (CLAUDE.md).

### 15.1 `live_sessions` [F2]

```sql
live_sessions(
  id uuid pk,
  lesson_id uuid fk -> lessons,            -- aula type='live' (1 aula ↔ 1..N sessões)
  course_id uuid fk -> courses,            -- desnormalizado p/ escopo/consulta (mesmo schema)
  host_user_id uuid fk -> users,           -- instrutor host
  title text,
  mode text,                               -- interactive | host_mostly        (broadcast = F3)
  scheduled_start_at timestamptz,
  scheduled_end_at timestamptz null,
  timezone text not null default 'America/Sao_Paulo',
  started_at timestamptz null,             -- room_started
  ended_at timestamptz null,               -- room_finished
  -- VOCABULÁRIO CANÔNICO (proposto p/ §6.0):
  status text not null default 'scheduled',
    -- scheduled | lobby | live | ended | canceled
  recording_enabled boolean not null default true,
  recording_status text not null default 'none',
    -- none | recording | processing | ready | failed
    -- (espelha lessons.video_status p/ a fase de VOD; ready ⇒ replay disponível)
  recording_provider_ref text null,        -- egress_id do provedor (idempotência/observabilidade)
  recording_master_key text null,          -- chave do MP4 master no R2 (<tenantId>/<sessionId>/...)
  room_provider_ref text null,             -- nome/sid da sala no provedor
  chat_locked boolean not null default false,
  attendance_completes_lesson boolean not null default false, -- toggle §8
  peak_participants int not null default 0,
  created_at timestamptz, updated_at timestamptz, deleted_at timestamptz null
)
-- índices: (lesson_id), (course_id, scheduled_start_at), (status)
```

**Marcação de fase (status):** `scheduled` (agendada) → `lobby` (sala aberta, host ainda não iniciou) →
`live` (ao vivo) → `ended` (encerrada) ; `canceled` a partir de `scheduled/lobby`. `recording_status` é
**separado** do `status` e governa a fase de **VOD** da gravação.

### 15.2 `live_attendance` [F2]

```sql
live_attendance(
  id uuid pk,
  live_session_id uuid fk -> live_sessions,
  user_id uuid fk -> users,
  enrollment_id uuid fk -> enrollments null,   -- null p/ host/staff
  role text,                                    -- host | participant
  joined_at timestamptz not null,
  left_at timestamptz null,                     -- fechado por participant_left/room_finished
  -- attended_seconds é derivado (sum de intervalos); pode materializar agregado por (session,user)
  created_at timestamptz
)
-- índices: (live_session_id), (user_id), (live_session_id, user_id)
-- 1 linha por intervalo de presença (reconexão = novo intervalo)
```

### 15.3 `live_chat_messages` [F2]

```sql
live_chat_messages(
  id uuid pk,
  live_session_id uuid fk -> live_sessions,
  user_id uuid fk -> users,
  body text,
  kind text not null default 'chat',            -- chat | question (Q&A) | system
  answered_at timestamptz null,                 -- Q&A: marcada como respondida pelo host
  answered_by uuid null,                         -- fk -> users
  hidden_at timestamptz null,                    -- moderação (espelha comment_*)
  hidden_by uuid null,                           -- fk -> users
  created_at timestamptz
)
-- índices: (live_session_id, created_at), (live_session_id, kind)
-- chat efêmero da sala usa o data channel do provedor (§17); PERSISTÊNCIA aqui é p/ histórico/replay/moderação
```

### 15.4 `live_bans` [F2]

```sql
live_bans(
  id uuid pk,
  course_id uuid fk -> courses null,             -- ban por curso (todas as sessões) ...
  live_session_id uuid fk -> live_sessions null, -- ... ou por sessão
  user_id uuid fk -> users,
  banned_by uuid fk -> users,
  reason text null,
  created_at timestamptz
)
-- emissão de token de sala consulta live_bans (defense in depth)
```

### 15.5 `live_recording_events` [F2] (idempotência de webhooks do provedor live)

```sql
live_recording_events(                            -- espelha payment_events (ADR-0010/0013)
  id uuid pk,
  provider text,                                  -- livekit (ou outro)
  event_id text unique,                           -- egress_id/event id do provedor (dedup)
  live_session_id uuid null,
  payload jsonb,
  processed_at timestamptz null,
  created_at timestamptz
)
```

> **Sem novas keys no control plane além das de provedor:** as **API key/secret do LiveKit** por tenant
> ficam **cifradas** (pgcrypto/KMS), como as keys da Bunny/pagamento. Opção de modelagem (a decidir na
> integração): coluna `live_keys_encrypted bytea` em `platform.tenants` **ou** uma tabela
> `platform.tenant_integration_keys`. Recomendação: **uma library/projeto LiveKit por tenant** (isolamento
> análogo à Video Library da Bunny). Ver §19.

### 15.6 Reuso (sem duplicação — DRY)

- **Vídeo/replay:** **não** há tabela nova de VOD — a gravação preenche `lessons.video_guid` +
  `lessons.video_status` (existentes) e reusa `lesson_progress` (anti-seek) e o player Bunny.
- **Moderação de chat:** espelha o padrão `comment_*` (`hidden_at/hidden_by`).
- **Idempotência:** espelha `payment_events`.

---

## 16. Port `LiveProvider` (SOLID)

Isola o provedor de tempo real e de gravação atrás de uma **interface** (DIP/ISP), igual a `VideoProvider`,
`PaymentProvider`, `JobQueue`, `AnalyticsProvider`. O **domínio e os use-cases não conhecem o LiveKit** —
dependem da port. Implementação concreta `livekit.live.provider.ts` na camada de infraestrutura.

```ts
// apps/api/src/modules/live/application/ports/live-provider.ts
import type { Brand } from "@core/brand";

export type RoomName = Brand<string, "RoomName">;        // t_<tenantId>__ls_<sessionId>
export type ParticipantIdentity = Brand<string, "ParticipantIdentity">; // = user_id

export interface RoomGrants {
  readonly canPublish: boolean;
  readonly canPublishData: boolean;     // chat/raise hand via data channel (§17)
  readonly canSubscribe: boolean;
  readonly roomAdmin: boolean;          // host: moderação
  readonly hidden?: boolean;
}

export interface CreateTokenInput {
  readonly room: RoomName;
  readonly identity: ParticipantIdentity;
  readonly name?: string;
  readonly grants: RoomGrants;
  readonly ttlSeconds: number;          // TTL curto (ADR-0008-like)
}

export interface RoomTokenResult {
  readonly token: string;               // JWT de sala (nunca gerado no front)
  readonly wsUrl: string;               // endpoint do provedor (por tenant)
}

export interface StartRecordingInput {
  readonly room: RoomName;
  readonly layout: "grid" | "speaker";  // composite
  readonly output: {                    // destino S3-compatível (R2), prefixo por tenant
    readonly bucket: string;
    readonly keyPrefix: string;         // <tenantId>/<sessionId>/
  };
}

export interface LiveProvider {
  ensureRoom(room: RoomName): Promise<void>;               // idempotente
  createToken(input: CreateTokenInput): Promise<RoomTokenResult>;
  startRecording(input: StartRecordingInput): Promise<{ recordingRef: string }>;
  stopRecording(recordingRef: string): Promise<void>;
  endRoom(room: RoomName): Promise<void>;
  muteParticipant(room: RoomName, identity: ParticipantIdentity): Promise<void>;
  removeParticipant(room: RoomName, identity: ParticipantIdentity): Promise<void>;
  verifyWebhook(rawBody: Buffer, signature: string): LiveWebhookEvent; // HMAC, tempo constante
}

export type LiveWebhookEvent =
  | { type: "room_started"; room: RoomName }
  | { type: "room_finished"; room: RoomName }
  | { type: "participant_joined"; room: RoomName; identity: ParticipantIdentity }
  | { type: "participant_left"; room: RoomName; identity: ParticipantIdentity }
  | { type: "egress_ended"; room: RoomName; recordingRef: string; fileUrl: string; status: "ok" | "failed" };
```

**Use-cases (1 classe = 1 caso de uso — SRP), camada `application`:**

- `ScheduleLiveSession`, `StartLiveSession`, `JoinLiveSession` (emite token), `EndLiveSession`,
  `ModerateParticipant`, `IngestLiveRecording` (worker), `HandleLiveWebhook`.

**Frontend:** o SDK do provedor (ex.: `@livekit/components-react`) fica **isolado em um componente cliente**
da feature (`features/live/`), consumindo apenas `{ token, wsUrl }` emitidos pelo backend — o resto do app
não conhece o provedor.

---

## 17. Tempo real / infra: chat e presença (revisita ADR-0011 — Redis?)

> **Pergunta central:** chat/presença ao vivo exigem **Redis/pub-sub**, contradizendo o ADR-0011 ("sem Redis
> no MVP")?

**Resposta: NÃO precisamos de Redis para esta feature.** Decisão detalhada e formal no
**[ADR-0015](../adr/0015-live-classes-interactive.md)**. Resumo:

- **Chat, raise hand e Q&A em tempo real usam o data channel nativo do provedor** (LiveKit data messages /
  `canPublishData`) — o **fan-out em tempo real é responsabilidade da SFU do provedor**, não nossa. Não
  precisamos manter conexões WebSocket nem um barramento pub-sub próprio para distribuir mensagens.
- **Presença em tempo real** vem dos eventos do SDK (no cliente) + **webhooks server-side** (`participant_*`)
  como fonte de verdade persistida (`live_attendance`). Sem Redis.
- **Persistência** (histórico de chat/Q&A, moderação, replay) usa **PostgreSQL** (tabelas §15) e o
  **pg-boss** para o trabalho assíncrono (ingestão, lembretes) — **dentro do ADR-0011** (mesmo Postgres, sem
  Redis).
- **Conclusão:** o ADR-0011 permanece válido. **Esta feature NÃO reintroduz Redis.** O ADR-0015 registra
  explicitamente essa revisita e o **gatilho futuro** que mudaria a decisão (ex.: chat fora da sala
  WebRTC/feed em tempo real de altíssima escala, ou broadcast massivo F3 com presença de milhares) — aí sim
  um pub-sub seria reavaliado, **por novo ADR**, atrás de uma port.

> **Implicação de "stack leve":** delegar o tempo real à SFU do provedor é o caminho mais leve — evita operar
> Redis e WebSocket server próprios no F2. Trade-off: dependência do provedor (mitigado pela port + opção
> self-host do LiveKit).

---

## 18. Faseamento recomendado

| Item | Fase | Observação |
|------|------|-----------|
| Sala interativa (vídeo/áudio/screen share) | **F2** | LiveKit Cloud + port |
| Chat + raise hand + Q&A (data channel) | **F2** | sem Redis (§17) |
| Gravação composite → R2 → Bunny (VOD) | **F2** | reusa pipeline ADR-0008 |
| Agendamento + calendário + lembretes 24h/10min | **F2** | jobs pg-boss |
| Presença/attendance + toggle de conclusão | **F2** | reusa anti-seek no replay |
| Moderação (mutar/remover/banir/travar chat) | **F2** | + auditoria |
| Quota por plano (horas/concorrência) | **F2** | `platform_plans.limits` |
| Analytics de live | **F2** | port `AnalyticsProvider` |
| **Self-host do LiveKit** (custo/escala) | **F3** | só se volume justificar; port já pronta |
| Broadcast massivo (HLS/RTMP, 1→muitos) | **F3** | audiências grandes |
| Breakout rooms, live captions, RTMP-out (YouTube) | **F3** | extensões |

---

## Dependências e pontos para o coordenador

Pontos que **cruzam** outros domínios/docs e precisam de decisão/integração antes da implementação. (A
edição formal dos outros docs é do coordenador.)

1. **PRD / ROADMAP (escopo):** esta feature **substitui** a linha "aula ao vivo (embed Zoom/YouTube Live)"
   (PRD §3.1/§3.5, ROADMAP F2) por **sala WebRTC nativa interativa com gravação→VOD**. Registrar a mudança de
   escopo no PRD e referenciar o ADR-0015. Confirmar que live interativo é **F2** (recomendado).

2. **ADR-0015 (decisão):** provedor **LiveKit** (Cloud→self-host), estratégia de **gravação→VOD via R2 +
   Bunny fetch-from-URL**, e **revisita ao ADR-0011** (sem Redis). Adicionar ao índice de ADRs (README).

3. **DATA_MODEL §6 (modelagem):** integrar as tabelas da §15 (`live_sessions`, `live_attendance`,
   `live_chat_messages`, `live_bans`, `live_recording_events`) e os **status canônicos** em §6.0
   (`live_sessions.status`, `live_sessions.recording_status`). Decidir onde guardar as **keys do LiveKit por
   tenant** (coluna em `platform.tenants` vs `platform.tenant_integration_keys`). Confirmar "1 projeto LiveKit
   por tenant" (isolamento análogo à Video Library Bunny). Lembrar: testes de isolamento cross-tenant para
   cada tabela nova.

4. **ARCHITECTURE §7/§9:** acrescentar o módulo `live/` (3 camadas) e o **fluxo de provisionamento** (criar
   projeto/keys LiveKit no onboarding, como já se cria a Bunny Library — ARCHITECTURE §3.4). Registrar o novo
   endpoint de webhook `/webhooks/live` e o job `live.recording.ingest`.

5. **RBAC_MATRIX:** integrar o domínio "Aulas ao vivo" (§10 deste doc) e a **condição C8-live**.

6. **NOTIFICATIONS_MATRIX:** detalhar os templates de live (§11) — expandir a linha "Live agendada/lembrete"
   existente para 24h/10min/ao-vivo/replay/falha, e adicionar ao **enum canônico de event types** (dep. 6 da
   matriz).

7. **MONETIZATION §A.2:** adicionar feature `live_classes` e quotas (`live_concurrent_participants`,
   `live_hours_month`) por tier (§12). **Stakeholder precisa definir os valores** (alinhar a OPEN_QUESTIONS
   #10 — tabela de planos) e a **política de overage** (custo de participante-minuto/recording do provedor vs
   margem).

8. **ANALYTICS_AND_DASHBOARDS §3:** adicionar a seção de eventos de live (§13) ao tracking plan e os schemas
   Zod em `packages/contracts` (DRY).

9. **NFR:** incorporar `NFR-LIVE-01..10` (§14) à matriz consolidada e aos gates de medição.

10. **BUSINESS_RULES_AND_STATES:** adicionar as máquinas de estado de `live_sessions.status` e
    `recording_status` (§15) ao catálogo de máquinas e ao mapa de orquestração (impacto em
    `lessons.video_status`/`lesson_progress`).

11. **Custos / FinOps (cruza com PRD §7 risco de egress):** participante-minuto + recording-minuto do
    provedor + egress de bandwidth são **novo custo variável**. Calibrar quotas e tier (LiveKit Cloud vs
    self-host) — ver ADR-0015 e os números de custo lá. Manter alertas de custo (como no egress Bunny).

12. **LGPD/dados no Brasil:** consentimento explícito de **gravação** (banner "esta aula está sendo
    gravada"), base legal para captura de áudio/vídeo de alunos, e **region pinning** do provedor (residência
    de dados) para tenants Enterprise sensíveis. Direito ao esquecimento: deleção da gravação no R2/Bunny e
    das linhas `live_*` dentro do schema do tenant. Validação jurídica (alinha OPEN_QUESTIONS #24).

13. **Provisionamento de pagamentos/recipient:** não há interação direta com split, mas o **provisionamento
    de projeto LiveKit** entra na saga de onboarding (ou passo "ativar lives") — definir com a coordenação
    (análogo ao "ativar pagamentos" do MONETIZATION dep. 6).
