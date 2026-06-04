# ADR-0015 — Aulas ao vivo interativas (sala WebRTC) com gravação → VOD

- **Status:** Aceito (recomendação técnica) · **Data:** 2026-06-04 · **Confiança:** 82%
- **Fase:** **F2** (Should Have) — ver [ROADMAP](../ROADMAP.md)
- **Complementa:** [ADR-0008 (Bunny Stream)](0008-streaming-bunny.md), [ADR-0012 (apoio: R2/observabilidade)](0012-supporting-platform.md)
- **Revisita:** [ADR-0011 (filas/pg-boss — "sem Redis no MVP")](0011-queue-pgboss.md)
- **Relacionados:** [LIVE_CLASSES.md](../product/LIVE_CLASSES.md) · [ADR-0001 (isolamento)](0001-multitenancy-schema-per-tenant.md) · [ADR-0010 (pagamentos)](0010-payments-br.md) · [DATA_MODEL §6](../DATA_MODEL.md) · [ARCHITECTURE §7/§9](../ARCHITECTURE.md)

## Contexto

O stakeholder decidiu adicionar **aulas ao vivo INTERATIVAS** (sala WebRTC): alunos com câmera/áudio, chat,
levantar a mão e Q&A — **com gravação**, e a **gravação deve reentrar no pipeline VOD da Bunny** (ADR-0008),
virando uma aula gravada normal (replay com player, token, anti-seek e progresso existentes).

Isso vai além da linha original do PRD/ROADMAP ("aula ao vivo via embed Zoom/YouTube Live", F2), que é
unidirecional e não interativa. Precisamos de um **provedor de tempo real (SFU WebRTC)** com SDK web/React,
gravação na nuvem e webhooks, **multitenant**, **leve de operar** (CLAUDE.md), sem violar a **Regra nº1**
(isolamento por tenant) e sem reintroduzir Redis indevidamente (ADR-0011). Decisões a tomar:

1. **Qual provedor de vídeo interativo em tempo real.**
2. **Como a gravação reentra no VOD da Bunny.**
3. **O impacto em tempo real/infra** (necessidade de Redis/pub-sub para chat/presença) — revisita do ADR-0011.

## Opções consideradas

### Dimensão 1 — Provedor de tempo real (WebRTC)

> Preços de **2026** (participante-minuto = 1 minuto por participante; gravação cobrada à parte). Valores são
> referência para calibrar quotas; ver Fontes.

| Provedor | SDK web/React · capacidade | Gravação nuvem + export | Webhooks | Multitenancy | Preço (ref.) | Self-host | LGPD/BR | Prós / Contras |
|----------|----------------------------|-------------------------|----------|--------------|--------------|-----------|---------|----------------|
| **LiveKit Cloud** | SDK JS + `@livekit/components-react` maduros; ~100 conexões (Build) → 1.000 (Ship) → 5.000 (Scale) | **Egress composite/individual → MP4/HLS direto p/ S3/GCS/Azure** (inclui **R2**) | `room_started/finished`, `participant_*`, `egress_started/updated/ended` | Token JWT com **grants por sala/participante**; nome de sala namespaced | Free: 5.000 part-min + 100 conexões; **part-min $0.0005**; **recording $0.005/min**; egress bandwidth **$0.12/GB (Ship) / $0.10 (Scale)**; Ship $50/mo, Scale $500/mo | **Sim — open source (Apache-2.0)**, self-host do media server + egress | Region pinning (residência de dados) p/ enterprise; DPA | **+** Open source (anti-lock-in), egress→S3 casa com R2, webhooks completos, preço previsível. **−** Egress separado de deploy se self-host; sem região BR garantida no Cloud. |
| **Daily** | SDK + `@daily-co/daily-react` excelentes; até 100k em sessão (mas interativo prático menor) | Gravação cloud (composite/raw); cobrada em **minutos simples** + storage | REST + webhooks (presença, logs) | Salas + tokens de acesso (meeting tokens) | 10.000 part-min grátis/mês; depois **$0.004/part-min**; recording storage **$0.003/min** | **Não** (managed puro) | DPA/GDPR; sem região BR dedicada | **+** DX/onboarding muito rápido, prebuilt UI. **−** Sem self-host (lock-in), part-min ~8× LiveKit, export de gravação menos "S3-nativo". |
| **100ms** | SDK + templates prontos (sala/aula); recording com layout custom, breakout, waiting room | Recording composite com layout; webhooks | Webhooks por room | Templates/roles por projeto | **~$0.004/part-min** (vídeo); áudio -75% | Não (managed) | DPA | **+** Templates ricos (classroom), roles. **−** Managed (lock-in), part-min alto, menor tração open source. |
| **Agora** | SDKs muito maduros, escala global enorme | Cloud Recording (composite/individual) → bucket próprio; preço por **resolução** | Webhooks/notifications | AppID + tokens; recording exige orquestração | 10.000 min grátis; **part-min por 1k min**; recording **$1.49–$13.49 / 1k min** por resolução | Não | DPA | **+** Escala/robustez. **−** Modelo de billing/recording complexo (por resolução), DX mais pesada, custo de recording alto. |
| **Zoom Video SDK** | SDK; web baseado em WebTransport/WebCodecs/WASM (não WebRTC puro) | Cloud recording | Webhooks | Por conta/SDK | Por minuto; tiers | Não | DPA | **+** Marca/qualidade; recomendado pela Twilio na saída dela. **−** Web não-WebRTC (estabilidade/perf web inferior a SFU WebRTC), menos "developer-native" p/ embed custom. |
| **Vonage Video (ex-TokBox)** | OpenTok SDK | Archiving (composite/individual) | Webhooks | Sessions + tokens | Por minuto | Não | DPA | **+** Maduro, top em Gartner CPaaS. **−** **OpenTok em modo manutenção**; sinais de ecossistema pedem plano de migração — risco de longevidade. |
| **Twilio Video** | — | — | — | — | — | — | — | **Descartado: EOL em 05/dez/2024.** Twilio recomenda Zoom Video SDK. |

### Dimensão 2 — Gravação → VOD (Bunny)

| Estratégia | Como | Prós | Contras |
|-----------|------|------|---------|
| **A. Egress → R2 → Bunny fetch-from-URL** (recomendada) | Provedor grava composite MP4 e auto-upload p/ **R2** (S3-compatível, ADR-0012); webhook `egress_ended` → job pg-boss → **Bunny cria vídeo + busca por URL (presigned do R2)** → entra na máquina `video_status` normal (ADR-0008). | **Egress zero do R2** + fetch Bunny ⇒ **não passa banda pelo nosso backend** (mesmo princípio do TUS direto); reusa 100% do pipeline VOD; mantém **master no R2** (anti-lock-in, ARCHITECTURE §7); leve. | Depende do provedor suportar destino S3 (LiveKit/Daily/Agora suportam); 2 saltos (provedor→R2→Bunny). |
| **B. Egress → nosso backend → TUS p/ Bunny** | Backend baixa a gravação e re-sobe via TUS. | Controle total. | **Consome banda/CPU do backend** (viola "leve"); pior custo. Usar **só como fallback** se A falhar. |
| **C. Usar a gravação do provedor como player** (não Bunny) | Servir o arquivo do provedor/CDN dele. | Menos um salto. | **Quebra a unificação** (player/token/anti-seek/proteção Bunny), 2 pipelines de VOD, lock-in no provedor. **Rejeitada.** |

### Dimensão 3 — Tempo real (chat/presença): Redis/pub-sub? (revisita ADR-0011)

| Opção | Descrição | Prós | Contras |
|-------|-----------|------|---------|
| **R1. Data channel nativo do provedor** (recomendada) | Chat, raise hand e Q&A trafegam pelo **data channel/data messages da SFU**; fan-out em tempo real é do provedor. Presença via eventos do SDK + **webhooks** server-side. Persistência (histórico/moderação/replay) no **PostgreSQL**; assíncrono no **pg-boss**. | **Sem Redis** — ADR-0011 intacto; stack leve; sem operar WebSocket/pub-sub próprios. | Acopla o tempo real ao provedor (mitigado pela port + self-host). |
| **R2. Redis Pub/Sub + WebSocket próprio** | Barramento próprio para chat/presença. | Independe do provedor; necessário p/ chat fora da sala / feed massivo. | **Infra extra (Redis)** — conflita com "leve" e ADR-0011; sem ganho real p/ a sala WebRTC (o provedor já faz fan-out). |
| **R3. Postgres LISTEN/NOTIFY** | Pub/sub leve no Postgres existente. | Sem Redis. | Não escala p/ tempo real de muitas salas; pior latência que data channel; desnecessário aqui. |

## Decisão

1. **Provedor: LiveKit.** Usar **LiveKit Cloud** no F2 (operação leve, sem ops de mídia) com **caminho de
   migração para LiveKit self-hosted** (open source Apache-2.0) quando volume/custo justificarem (F3). Tudo
   atrás da **port `LiveProvider`** (DIP/ISP — SOLID), espelhando `VideoProvider`/`PaymentProvider`. **Plano
   B: Daily** (managed puro, DX excelente) — substituível trocando a implementação da port.
   - **Isolamento multitenant:** 1 projeto/credenciais LiveKit por tenant (análogo à Video Library da Bunny,
     ADR-0008); nome de sala **namespaced por tenant** (`t_<tenantId>__ls_<sessionId>`); **token de sala
     emitido só pelo backend**, TTL curto, após validar entitlement (Regra nº1, PRD §4.2). Keys **cifradas em
     repouso** (pgcrypto/KMS) — nunca no front.

2. **Gravação → VOD: estratégia A.** Egress **composite** do LiveKit grava **MP4 → Cloudflare R2** (egress
   zero); webhook `egress_ended` (HMAC) → responde 200 → enfileira job pg-boss → **Bunny cria vídeo na library
   do tenant via fetch-from-URL** (presigned R2) → segue a máquina `lessons.video_status`
   (`queued→processing→ready`) já existente. **Fallback B (TUS)** se a fetch falhar. **Master mantido no R2**
   (anti-lock-in). A aula `type='live'` **amadurece** em VOD (default) reusando player/token/anti-seek/
   progresso — **sem duplicar o domínio de vídeo (DRY)**.

3. **Tempo real: opção R1 — SEM Redis. ADR-0011 permanece válido.** Chat/raise hand/Q&A pelo **data channel
   nativo do LiveKit**; presença por eventos do SDK + **webhooks** (`participant_*`), persistida em PostgreSQL;
   trabalho assíncrono (ingestão, lembretes 24h/10min) no **pg-boss**. **Esta feature NÃO reintroduz Redis.**
   Gatilho futuro que reabriria a decisão (por **novo ADR**): chat/feed em tempo real **fora** da sala WebRTC
   ou **broadcast massivo (F3)** com presença de milhares.

## Justificativa

- **Gravação→VOD elegante e leve:** Egress LiveKit → **R2** (egress zero, já no stack) → **Bunny
  fetch-from-URL** reusa **todo** o pipeline VOD (ADR-0008) sem trafegar mídia pelo nosso backend — coerente
  com o padrão TUS-direto e com "stack leve".
- **Anti-lock-in / SOLID:** LiveKit é **open source** (self-host possível) e fica atrás da **port
  `LiveProvider`** — troca de provedor é local; o domínio depende de interface, não do SDK.
- **Multitenancy/Regra nº1:** token de sala com grants por participante + sala namespaced + entitlement
  validado no backend espelham o modelo de Embed Token da Bunny — isolamento consistente.
- **Webhooks + pg-boss:** `room_*`/`egress_ended` encaixam no padrão webhook (HMAC, idempotência via
  `live_recording_events` espelhando `payment_events`) + `JobQueue` já decidido (ADR-0011).
- **Custo previsível e calibrável:** participante-minuto $0.0005 + recording $0.005/min são baixos e
  mapeáveis a **quotas de plano** (`platform_plans.limits.live_*`), com o mesmo padrão de injeção de limite no
  `onRequest` usado para `take_rate_bps` (ADR-0013) — **sem violar isolamento**.
- **ADR-0011 intacto:** a SFU faz o fan-out de tempo real; não precisamos de Redis. Mantém o princípio "leve".

## Consequências

- **Novo módulo `live/`** (3 camadas: domain/application/infrastructure) com use-cases SRP
  (`Schedule/Start/Join/End LiveSession`, `ModerateParticipant`, `IngestLiveRecording`, `HandleLiveWebhook`).
- **Provisionamento:** criar projeto/keys LiveKit por tenant no onboarding (ou passo "ativar lives"),
  espelhando a criação da Bunny Library (ARCHITECTURE §3.4); keys cifradas no control plane.
- **Novo endpoint** `/webhooks/live` (HMAC, responde 200, enfileira) e job `live.recording.ingest`.
- **Novas tabelas** no schema do tenant (DATA_MODEL §6, fase F2): `live_sessions`, `live_attendance`,
  `live_chat_messages`, `live_bans`, `live_recording_events` — com **status canônicos** e **teste de
  isolamento cross-tenant** para cada uma (gate de CI, CLAUDE.md).
- **Reuso (DRY):** gravação preenche `lessons.video_guid`/`video_status` existentes; moderação espelha
  `comment_*`; idempotência espelha `payment_events`.
- **Quotas e analytics:** novas chaves em `platform_plans.limits` (`live_classes`,
  `live_concurrent_participants`, `live_hours_month`) e eventos de analytics com `tenant_id` (ADR-0014).
- **Custo variável novo** (participante-minuto + recording + egress bandwidth) → FinOps/alertas, calibrar tier
  Cloud vs self-host conforme volume.
- **LGPD:** consentimento de gravação, base legal de captura de áudio/vídeo de alunos, region pinning para
  enterprise, e deleção (R2/Bunny + linhas `live_*`) no direito ao esquecimento.

## Riscos

- **Custo de participante-minuto/recording/egress** acima do previsto em salas grandes/longas — mitigar com
  quotas duras de concorrência, cap de gravação e alertas (mesmo risco do egress Bunny, PRD §7).
- **Latência/qualidade no Brasil** (PoP do provedor) e residência de dados — validar PoP/region pinning;
  self-host regional é o plano de contingência.
- **Dependência do provedor para tempo real** (data channel) — mitigada por LiveKit ser open source + port.
- **Complexidade operacional do Egress self-hosted** (se migrar) — adiada para F3; Cloud no F2.
- **Confiança do plano B (Daily)** quanto a export S3-nativo é menor que LiveKit; revalidar se virar plano A.

## Nível de confiança

**82%.** Alto na arquitetura (port + R2→Bunny + webhooks/pg-boss + sem Redis) e no encaixe com ADR-0008/0011.
Os ~18% residuais concentram-se em: **valores comerciais** de quota/overage (dependem do stakeholder —
OPEN_QUESTIONS #10), **PoP/residência de dados no BR** (validação com o provedor), e **calibração de custo**
real em produção. Nenhum desses bloqueia iniciar o desenho do módulo na F2.

## Fontes

- LiveKit — Pricing (Build/Ship/Scale; part-min $0.0005, recording $0.005/min, egress bandwidth $0.10–0.12/GB;
  open source): https://livekit.com/pricing
- LiveKit — Egress (composite/track → MP4/HLS para S3/GCS/Azure; self-host separado):
  https://docs.livekit.io/transport/media/ingress-egress/egress/ · https://github.com/livekit/egress
- LiveKit — Webhooks/events (`room_started/finished`, `participant_*`, `egress_started/updated/ended`):
  https://docs.livekit.io/intro/basics/rooms-participants-tracks/webhooks-events/
- LiveKit — Tokens & grants (JWT, grants por sala/participante, canPublish/canSubscribe/roomAdmin):
  https://docs.livekit.io/frontends/authentication/tokens/
- LiveKit — Region pinning (residência de dados): https://docs.livekit.io/deploy/admin/regions/region-pinning/
- Daily — Pricing (10k part-min grátis; $0.004/part-min; recording storage $0.003/min):
  https://www.daily.co/pricing/video-sdk/
- 100ms — Pricing/recursos (recording com layout, templates, breakout, webhooks): https://www.100ms.live/pricing
- Agora — Cloud Recording pricing (por resolução, $1.49–$13.49/1k min): https://docs.agora.io/en/cloud-recording/overview/pricing
- Twilio — Programmable Video End of Life (05/dez/2024; recomenda Zoom Video SDK):
  https://help.twilio.com/articles/24158233644443-Programmable-Video-End-of-Life-Extension
- Vonage Video API — release notes/status 2026 (OpenTok em manutenção; plano de migração recomendado):
  https://developer.vonage.com/en/blog/q1-2026-vonage-developer-recap
- Bunny Stream — pre-signed/TUS resumable + **fetch video from URL** (ingestão a partir de URL/S3):
  https://bunny.net/blog/bunny-stream-introducing-pre-signed-and-resumable-uploads/ ·
  https://docs.bunny.net/docs/stream-uploading-videos-through-our-http-api
