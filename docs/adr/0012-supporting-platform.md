# ADR-0012 — Plataforma de apoio: storage, e-mail, analytics, observabilidade, IA, certificados

- **Status:** Aceito · **Data:** 2026-06-04 · **Confiança:** 85%

Agrupa decisões de menor risco, todas atrás de **interfaces (ports)** quando aplicável (DIP), para
trocabilidade.

## Armazenamento de arquivos (PDFs, materiais, certificados)
- **Decisão:** **Cloudflare R2** (API S3-compatível, **egress zero**) — ideal para downloads frequentes.
- Alternativa: Bunny Storage (manter um só fornecedor) — atenção ao egress SA. Vídeo permanece no Bunny.

## E-mail transacional
- **Decisão:** **Resend** (melhor DX, React Email integra com Next.js, free tier real, boa
entregabilidade). Escalar para **AWS SES** acima de ~100k/mês; **Postmark** para transacionais críticos.
- Abstração `EmailProvider`. Manter transacional separado de marketing (reputação de IP).

## E-mail marketing / automação
- **Decisão:** **integrar** RD Station (BR) ou ActiveCampaign via **event bus** interno; não construir.

## Analytics
- **Decisão:** **PostHog** (produto/funis, self-host possível, LGPD-friendly) para eventos de
aprendizagem; Plausible/Umami para web analytics leve, se necessário.

## Observabilidade
- **Decisão:** **Pino** (logs JSON, sempre com `tenantId`+`requestId`) + **OpenTelemetry** (traces) +
**Sentry** (exceções front/back). Erros via Problem Details (RFC 9457) em handler único.

## IA (Fase 2/3)
- **Decisão:** transcrição/legenda via **Bunny Transcribe AI** (Whisper); **tutor IA** via RAG sobre as
transcrições com **pgvector** + provedor LLM atrás de `AIProvider` (Anthropic/OpenAI intercambiáveis).

## Certificados (PDF)
- **Decisão:** geração via **Puppeteer** (template HTML/CSS) ou **pdfme** (templates editáveis) em
**worker assíncrono**; armazenar no R2; **verificação pública** (UUID + QR + hash) com página de
validação.

## Notificações/push (F2)
- **Decisão:** Web Push (VAPID) + **Novu** (open-source, orquestração) ou OneSignal.

## Fontes
Relatório de streaming/integrações (R2 vs S3, e-mail, certificados, IA); relatório de arquitetura
(observabilidade).
