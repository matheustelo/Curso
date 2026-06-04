# ADR-0008 — Streaming de vídeo: Bunny Stream

- **Status:** Aceito · **Data:** 2026-06-04 · **Confiança:** 88%

## Contexto
Streaming das aulas com bom custo, segurança e simplicidade. Preferência do stakeholder: **Bunny.net**.
Modelo multitenant exige isolamento de conteúdo por tenant.

## Opções consideradas (validação da escolha)
- **Bunny Stream:** encoding x264 grátis, player + DRM básico (MediaCage) grátis, storage US$0.01/GB,
delivery a partir de US$0.005/GB (Volume). Mais barato em escala. Ressalva: egress Standard na América
do Sul ~US$0.045/GB.
- **Cloudflare Stream:** preço por minuto (previsível; penaliza catálogo grande).
- **Mux:** dev-first, analytics fortes; encoding pago encarece uploads.
- **Vimeo OTT / AWS IVS+MediaConvert+CloudFront:** mais caros/complexos para SaaS white-label.

## Decisão
**Bunny Stream**, com **1 Video Library por tenant** (AccessKey/Pull Zone/token key próprios),
provisionada no onboarding.
- **Upload:** pre-signed + **TUS** (resumable) direto do browser.
- **Reprodução:** Embed Token (TTL curto) gerado no backend após validar entitlement.
- **Webhooks:** verificação HMAC + idempotência → worker mapeia `VideoLibraryId → tenant`.
- **Tracking:** player.js (`timeupdate` throttle) → `lesson_progress`.

## Justificativa
- Melhor custo/simplicidade (encoding + player + DRM básico inclusos).
- 1 library/tenant dá isolamento alinhado ao schema-per-tenant.
- `VideoProvider` abstrato (SOLID) evita lock-in e permite player próprio/DRM Enterprise depois.

## Consequências / riscos
- **Validar egress BR** (Volume vs Standard) — maior risco de custo (ver OPEN_QUESTIONS #1).
- Keys por tenant **cifradas em repouso**; nunca no frontend.
- Manter masters/originais no R2 para re-encode futuro (mitiga lock-in).

## Fontes
docs.bunny.net (Stream, Token Auth, Webhooks, Pricing, Transcribe AI, pre-signed/TUS); comparativos
Bunny/Cloudflare/Mux. Ver relatório de streaming.
