# Questões em Aberto / Itens a Validar

- **Versão:** 1.0 · **Data:** 2026-06-04

As decisões de produto e arquitetura estão fechadas em ≥95% de assertividade. Os itens abaixo **não
bloqueiam** o início da implementação, mas precisam ser validados/decididos durante a execução do MVP.

---

## 1. Validações operacionais (não bloqueiam design)

| # | Item | Por que importa | Ação |
|---|------|-----------------|------|
| 1 | **Custo de egress de vídeo no Brasil** | A rede Standard da Bunny cobra ~US$0.045/GB na América do Sul (~9x a Volume Network a US$0.005/GB). É o **maior risco financeiro** do projeto. | Contato comercial Bunny: confirmar cobertura/qualidade da **Volume Network** no BR. Definir cap de resolução (ex.: 720p padrão) + alertas de custo. |
| 2 | **PoC Better-Auth em schema-per-tenant** | Os exemplos da lib assumem DB único; precisamos validar auth + resolução de tenant + `search_path` ponta a ponta. | PoC no início da implementação (passo 2 do roadmap). Plano B: auth próprio Fastify (`@fastify/jwt` + `@fastify/cookie` + argon2). |
| 3 | **Estratégia de pooling PgBouncer** | `SET LOCAL search_path` em transaction pooling é o padrão de ouro; validar limites de pool × tenants ativos. | Definir `max` por pool e testar isolamento sob carga. |

---

## 2. Decisões de produto a refinar com dados (pós-MVP)

| # | Item | Default adotado | Quando revisar |
|---|------|-----------------|----------------|
| 4 | **Anti-pirataria** (você respondeu "não sei") | Faseado: MVP = Token + MediaCage Basic; F2 = watermark por aluno; DRM Enterprise sob demanda. Ver [ADR-0009](adr/0009-anti-piracy.md). | Se surgir cliente premium exigindo DRM "Hollywood-grade". |
| 5 | **Escopo de IA no MVP** | IA fica na **Fase 2** (transcrição + geração de quiz); MVP sem IA para focar no loop de receita. | Reavaliar se IA virar prioridade de marketing. |
| 6 | **Mobile** | **PWA primeiro** (F2); app nativo white-label só na F3. | Demanda de tenants por app branded nas lojas. |
| 7 | **Billing do SaaS** (como o tenant paga você) | Stripe Billing por plano (mensalidade fixa). | Se quiser modelo % sobre vendas (estilo Hotmart) no futuro. |
| 8 | **E-mail marketing** | Integrar terceiros (RD Station/ActiveCampaign) via event bus; não construir. | Depende da operação de marketing. |
| 9 | **Idiomas** | Lançar em **PT-BR**; arquitetura i18n pronta. | Expansão LATAM (ES/EN). |

---

## 3. Itens explicitamente fora de escopo (decididos)

SCORM/xAPI, SSO/SAML, LTI, marketplace cross-tenant, multi-região, produção de vídeo in-house e
certificação acadêmica credenciada estão em **Fase 3 ou Won't** (ver [ROADMAP.md](ROADMAP.md)).

---

> Atualize este arquivo conforme os itens forem resolvidos, promovendo decisões relevantes a ADRs.
