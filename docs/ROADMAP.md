# Roadmap (MoSCoW) — Plataforma de Cursos SaaS

- **Versão:** 1.0 · **Data:** 2026-06-04 · **Relacionados:** [PRD.md](PRD.md)

Priorização derivada da pesquisa de mercado e das decisões do stakeholder. O objetivo do MVP é fechar o
**loop completo de valor**: _criar curso → vender (checkout BR) → entregar com vídeo protegido →
acompanhar_ — o mínimo para um tenant gerar receita.

---

## 🟢 MVP — Must Have (fundação + loop de receita)

**Plataforma / Multitenant**
- Multitenant schema-per-tenant + provisionamento automatizado + subdomínio + branding básico.
- Painel Super-Admin (gestão de tenants, suspender/ativar, impersonar).
- Planos do SaaS + quotas + billing Stripe (status controla o tenant).
- Auth (alunos/instrutores/admin) + RBAC por tenant.

**Conteúdo & entrega**
- Curso → Módulo → Aula; tipos vídeo (Bunny) / texto / PDF; rascunho/publicação.
- Drip (por data e por dias após matrícula).
- Player Bunny (HLS, velocidade, retomar, legendas) + tracking de progresso.
- Proteção: Token Auth + MediaCage Basic + MP4 off + referer.

**Avaliação**
- Quiz simples (múltipla escolha / V-F) + certificado de conclusão + verificação pública (QR).

**Monetização (checkout dos alunos)**
- Checkout BR (Pix/boleto/cartão) + assinatura + cupom.
- **Afiliados + split** (Pagar.me).
- Webhooks de pagamento ↔ máquina de estados de acesso (idempotentes).

**Gestão & dados**
- Matrícula (manual/compra/CSV/auto) + progresso + dashboards básicos.
- Webhooks de saída (compra, reembolso, matrícula, conclusão).
- Analytics básico (conclusão, receita por curso).
- LGPD essencial (consentimento, exportação, deleção) + baseline WCAG AA.

---

## 🟡 Fase 2 — Should Have (retenção, conversão, diferenciação)

- **Order bump + upsell/downsell one-click + bundles + recuperação de carrinho.**
- **Co-produção** (split entre produtores).
- **Comunidade** (fórum/feed) + **lives** (Zoom/embed).
- **Gamificação** (pontos, badges, ranking).
- **Provas** (tentativas/nota/tempo) + **gradebook**.
- **Player próprio (Vidstack) + watermark dinâmico por aluno**; limite de sessões.
- **PWA + push.**
- **Analytics avançado** (coorte, MRR/churn, drop-off, heatmap, watch-time) + exportação.
- **E-mail marketing/automação** básica + pixels + integração RD/ActiveCampaign.
- **IA:** transcrição/legenda automática + geração de quizzes.
- **Domínio próprio** + i18n da interface + **API REST pública** (OpenAPI).

---

## 🔵 Fase 3 — Could Have (escala / premium / enterprise)

- **App mobile branded white-label** nas lojas.
- **Cohorts completos** (cronograma, peer review).
- **Tutor IA** (RAG sobre transcrições) + **resumos** + **recomendação**.
- **Trilhas/learning paths**, pré-requisitos avançados, níveis, streaks.
- **DRM Enterprise** (Widevine/FairPlay) + anti screen-grab.
- **Construtor de landing/funil** nativo + blog/SEO avançado.
- **SCORM/xAPI + LTI + SSO (SAML)** — abre B2B/educacional.
- **Marketplace de descoberta cross-tenant** (só se virar estratégia).
- **Multi-região** (se houver demanda de residência de dados).

---

## ⚪ Won't (por ora)

- Produção de vídeo in-house · Certificação acadêmica credenciada · Suite de e-mail nível Mailchimp ·
CMS/website builder genérico.

---

## Sequência sugerida de implementação (técnica)

1. **Fundação:** monorepo (pnpm/Turborepo) + `packages/config`, `core`, `contracts`, `db` + tsconfig
strict + Biome + CI.
2. **PoC crítico:** resolução de tenant (`search_path`) + Better-Auth + 1 endpoint Fastify + teste de
isolamento cross-tenant. _Valida o maior risco antes de escalar._
3. **Provisionamento de tenant** (saga) + Super-Admin mínimo.
4. **Catálogo** (cursos/módulos/aulas) + **vídeo Bunny** (upload TUS + webhook + player + progresso).
5. **Checkout BR + webhooks de pagamento + acesso/matrícula** + afiliados/split.
6. **Quiz + certificado** + analytics básico + LGPD.
7. Hardening (observabilidade, e2e, acessibilidade) → lançamento do MVP.
