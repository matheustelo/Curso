# Compliance & Privacidade (LGPD) — Plataforma de Cursos SaaS Multitenant

- **Versão:** 1.0 · **Data:** 2026-06-07
- **Status:** Proposto — requisitos de produto + **minutas-base** para revisão jurídica
- **Mantido por:** Coordenação de Produto + (futuro) DPO/Encarregado
- **Relacionados:** [NON_FUNCTIONAL_REQUIREMENTS.md](../product/NON_FUNCTIONAL_REQUIREMENTS.md) (§5 LGPD UX) ·
  [OPEN_QUESTIONS.md](../OPEN_QUESTIONS.md) (#24 retenção, #25 CMP, #29 gravação live) ·
  [DATA_MODEL.md](../DATA_MODEL.md) · [product/README.md](../product/README.md) (§3 dois domínios de
  pagamento) · [LIVE_CLASSES.md](../product/LIVE_CLASSES.md) (§6, §7, §12) · [CLAUDE.md](../../CLAUDE.md)
  (Regra nº1 — isolamento por tenant) · ADR-0001 (schema-per-tenant) · ADR-0008 (Bunny) · ADR-0010
  (pagamentos BR) · ADR-0015 (lives)

---

> ## ⚠️ AVISO — Isto NÃO é aconselhamento jurídico
>
> Este documento é um **documento de requisitos de produto/engenharia de privacidade** e contém **minutas-base
> (esqueletos/seções)** destinadas a **acelerar e orientar a revisão por um(a) advogado(a) qualificado(a)** em
> Direito Digital/Proteção de Dados no Brasil. **Nenhum texto aqui constitui parecer, contrato ou política
> juridicamente vigente.** As bases legais, prazos de retenção e cláusulas propostos são **hipóteses
> fundamentadas** que **DEVEM** ser validadas, ajustadas e assinadas por profissional habilitado antes de
> qualquer publicação ou uso em produção. Em caso de conflito entre este documento e a orientação do
> advogado/DPO, **prevalece a orientação jurídica**. Os números de retenção, em especial, dependem de
> confirmação fiscal/contábil (CTN, legislação estadual de ICMS, regras municipais de ISS).

---

## Índice

1. [Escopo, papéis e contexto multitenant (controlador × operador)](#1-escopo-papéis-e-contexto-multitenant-controlador--operador)
2. [Mapa de dados pessoais & bases legais](#2-mapa-de-dados-pessoais--bases-legais)
3. [Tabela de retenção + janela win-back antes do `DROP SCHEMA`](#3-tabela-de-retenção--janela-win-back-antes-do-drop-schema)
4. [Direitos do titular e como o produto os atende](#4-direitos-do-titular-e-como-o-produto-os-atende)
5. [Consentimento: cookies/CMP e gravação de aula ao vivo](#5-consentimento-cookiescmp-e-gravação-de-aula-ao-vivo)
6. [Minutas-base (estruturas)](#6-minutas-base-estruturas)
   - 6.1 [Termos de Uso](#61-termos-de-uso-plataforma)
   - 6.2 [Política de Privacidade](#62-política-de-privacidade)
   - 6.3 [DPA — Acordo de Tratamento de Dados (plataforma × tenant)](#63-dpa--acordo-de-tratamento-de-dados-plataforma--tenant)
   - 6.4 [Política de Moderação de Conteúdo](#64-política-de-moderação-de-conteúdo)
7. [Matriz MVP vs. depois (essencial × evolutivo)](#7-matriz-mvp-vs-depois-essencial--evolutivo)
8. [Fontes](#8-fontes)
9. [Dependências e pontos para o coordenador](#9-dependências-e-pontos-para-o-coordenador)

---

## 1. Escopo, papéis e contexto multitenant (controlador × operador)

A LGPD (Lei 13.709/2018, art. 5º) distingue dois papéis centrais:

- **Controlador:** a quem competem as **decisões** sobre o tratamento de dados pessoais (finalidade e meios).
- **Operador:** quem realiza o tratamento **em nome do controlador**, seguindo suas instruções.

O operador responde **solidariamente** quando descumpre a lei ou contraria instruções lícitas do controlador,
caso em que se **equipara ao controlador** (art. 42, §1º). Por isso a delimitação correta de papéis é o
alicerce de toda a estratégia de compliance — e, num SaaS multitenant, ela **não é trivial**.

### 1.1 O modelo dos três relacionamentos

Nosso produto é **schema-per-tenant** (ADR-0001): cada escola/produtor (**tenant**) tem seu próprio schema
`tenant_<slug>`; a plataforma (nós) opera o control plane (`platform`). Há **três fluxos de dados** com papéis
distintos:

| # | Relação | Dados | Nosso papel (plataforma) | Papel do tenant |
|---|---------|-------|--------------------------|-----------------|
| **A** | **Tenant → Plataforma** (B2B): a escola é nossa cliente | Dados de cadastro/billing do tenant (Owner/Admin), faturas Stripe | **Controlador** (decidimos como tratar nossos clientes B2B) | Titular/cliente |
| **B** | **Aluno → Tenant** (B2C): o aluno é cliente da escola | Dados de alunos: conta, progresso, compras, comentários, presença/gravação de live | **Operador** (tratamos os dados dos alunos **em nome** do tenant, conforme DPA) | **Controlador** (decide finalidades sobre seus alunos) |
| **C** | **Plataforma — fins próprios** sobre dados que transitam | Logs de segurança/auditoria, telemetria agregada/pseudonimizada, prevenção a fraude, métricas de saúde do serviço | **Controlador** (para nossos próprios fins legítimos de operação e segurança) | Co-interessado |

> **Posição recomendada (a validar pelo advogado):** No relacionamento **B (alunos)**, **o tenant é o
> Controlador e a plataforma é Operadora**. Isso é coerente com o isolamento por tenant (Regra nº1): nós
> fornecemos a infraestrutura, mas **quem decide** quais cursos vender, a quem, e como se comunicar com os
> alunos é o tenant. Esse enquadramento **precisa estar explícito no DPA** (§6.3) e na Política de Privacidade
> (§6.2), porque define quem responde primariamente pelos direitos do titular-aluno.

### 1.2 Por que essa distinção importa na prática

- **Direitos do titular (art. 18):** um aluno exerce direitos perante o **Controlador (tenant)**; a plataforma,
  como **Operadora**, fornece as **ferramentas técnicas** (exportação/eliminação self-service) e **executa** as
  instruções do tenant. O produto deve permitir que o **tenant** atenda seus alunos (ver §4).
- **Vazamento cross-tenant = incidente de segurança (art. 46/48):** sob a ótica LGPD, misturar dados de
  tenants é falha de segurança que pode gerar responsabilização **solidária** da plataforma. Reforça a
  **Regra nº1** (CLAUDE.md) como obrigação **legal**, não só técnica.
- **Suboperadores:** Bunny (vídeo/CDN), Cloudflare R2 (storage), Pagar.me/Asaas (pagamento aluno), Stripe
  (billing SaaS), Resend (e-mail), LiveKit (lives F2), PostHog/Sentry (telemetria), provedores de IA (F2/F3)
  são **suboperadores** que precisam constar de uma lista e estar cobertos por cláusula de suboperação no DPA.

### 1.3 Encarregado (DPO)

A LGPD (art. 41) exige a indicação de um **Encarregado pelo Tratamento de Dados Pessoais**. **[MVP]** Indicar
e publicar **canal de contato do Encarregado** (e-mail dedicado, ex.: `dpo@<plataforma>`) na Política de
Privacidade e no rodapé. O tenant também deve indicar seu próprio canal aos alunos (campo em `tenant_settings`
— ver §9).

---

## 2. Mapa de dados pessoais & bases legais

> **Leitura:** "Papel" indica se a **plataforma** atua como Controlador (C) ou Operador (O) naquele dado (ver
> §1). Para dados em que somos **Operador**, a **base legal é definida pelo tenant-Controlador**; a coluna
> "Base legal sugerida" é a hipótese recomendada que o DPA deve refletir. Bases conforme **LGPD art. 7º**
> (dados pessoais) e **art. 11** (dados sensíveis).

### 2.1 Dados do **aluno** (schema do tenant — plataforma = Operadora)

| Categoria de dado | Exemplos (tabelas DATA_MODEL) | Finalidade | Base legal sugerida (art. 7º) | Papel plataforma |
|-------------------|-------------------------------|-----------|-------------------------------|------------------|
| **Conta/identidade** | `users` (email, name, password_hash), `sessions` | Autenticar, dar acesso ao curso, suporte | **Execução de contrato** (VII) | O |
| **Matrícula e acesso** | `enrollments`, `subscriptions` (aluno) | Conceder/gerir acesso ao conteúdo comprado | **Execução de contrato** (VII) | O |
| **Progresso de aprendizagem** | `lesson_progress`, `quiz_attempts`, `enrollment_milestones`, `gamification_*` | Entregar a experiência (retomar, certificar, gamificar) | **Execução de contrato** (VII); gamificação opt-out (`leaderboard_optout`) | O |
| **Certificados** | `certificates` (nome, curso, hash, QR público) | Emitir e verificar certificado | **Execução de contrato** (VII) | O |
| **Pagamento/financeiro (aluno)** | `orders`, `payment_events`, `coupons` (dados de cartão **não** ficam conosco — tokenização no gateway, NFR-SEC-18) | Processar compra, antifraude, conciliação | **Execução de contrato** (VII) + **obrigação legal** fiscal (II) p/ registro transacional | O |
| **CPF / dados fiscais** | em `orders`/checkout quando exigido p/ NF | Emissão de nota fiscal, antifraude, watermark (F2) | **Obrigação legal/fiscal** (II); minimização — só quando exigido (NFR-LGPD-12) | O |
| **Comunidade/UGC** | `lesson_comments`, `comment_reports`, `live_chat_messages` | Interação, dúvidas, moderação | **Execução de contrato** (VII) + **legítimo interesse** (IX) p/ moderação/segurança | O |
| **Notificações/preferências** | `notifications`, `notification_preferences` | Comunicar eventos do curso | Contrato (transacional) / **consentimento** (I) p/ marketing | O |
| **Imagem/voz em live (F2)** | stream WebRTC + `live_attendance` + **gravação** (R2/Bunny via `live_sessions`) | Aula ao vivo interativa e replay | **Consentimento específico** (I) p/ captura/gravação — ver §5.2 | O |
| **Telemetria de produto** | PostHog (pseudonimizado), RUM | Medir uso, melhorar UX | **Consentimento** (analytics, via CMP) — ver §5.1 | C/O (ver §1-C) |
| **Pixels de marketing** | Meta, GA4 | Atribuição/remarketing do tenant | **Consentimento** (marketing, via CMP) | O (em nome do tenant) |

### 2.2 Dados do **tenant** (control plane — plataforma = Controladora)

| Categoria | Exemplos (DATA_MODEL `platform`) | Finalidade | Base legal (art. 7º) | Papel |
|-----------|----------------------------------|-----------|----------------------|-------|
| **Cadastro do tenant / equipe B2B** | `tenants`, Owner/Admin em `users` do tenant | Prestar o serviço SaaS, suporte, cobrança | **Execução de contrato** (VII) | C |
| **Billing SaaS** | `platform_subscriptions`, `platform_invoices` (Stripe) | Cobrar a mensalidade, antifraude, fiscal | Contrato (VII) + **obrigação legal** fiscal (II) | C |
| **Recipients de pagamento (KYC)** | `platform_payment_recipients`, `affiliates.*_recipient_id_encrypted` | Repasse/split, prevenção a fraude | Contrato (VII) + **obrigação legal** (II) | C/O |

### 2.3 Dados de **segurança e auditoria** (fins próprios da plataforma — Controladora)

| Categoria | Exemplos | Finalidade | Base legal (art. 7º) | Papel |
|-----------|----------|-----------|----------------------|-------|
| **Auditoria de atos sensíveis** | `platform.audit_log` (impersonação, suspensão) | Segurança, accountability, prova | **Legítimo interesse** (IX) + **obrigação legal** (II) | C |
| **Logs técnicos / observabilidade** | Pino, Sentry (com PII scrubbing — NFR-OBS-02), traces | Operar, depurar, segurança | **Legítimo interesse** (IX) | C |
| **Prevenção a fraude/abuso** | rate-limit, idempotência, `live_bans` | Proteger o serviço e usuários | **Legítimo interesse** (IX) | C |

> **Dados sensíveis (art. 11):** o produto **não** deve coletar dados sensíveis por padrão. **Imagem e voz**
> capturadas em aulas ao vivo (F2) merecem tratamento reforçado (consentimento específico e destacado — §5.2),
> pois imagem/voz são dados pessoais e há **direito de imagem** autônomo (CF art. 5º X; Código Civil art. 20)
> que coexiste com a LGPD. O **CPF** não é "sensível" na LGPD, mas é dado de alto risco → minimização e
> cifragem em repouso.

---

## 3. Tabela de retenção + janela win-back antes do `DROP SCHEMA`

> **Status: PROPOSTA — números pendentes de validação jurídica/contábil** (OPEN_QUESTIONS #24). O princípio
> LGPD é **necessidade/finalidade** (art. 15–16): o dado deve ser eliminado quando a finalidade se exaure,
> **salvo** hipóteses do art. 16 (obrigação legal, estudo por órgão de pesquisa, uso exclusivo do
> controlador anonimizado, ou exercício regular de direitos). Os prazos fiscais derivam do **CTN** (decadência
> e prescrição de **5 anos**, arts. 173 e 174); **a guarda de documentos fiscais eletrônicos pode chegar a
> ~10–11 anos em legislações estaduais de ICMS** — confirmar com contabilidade.

### 3.1 Tabela de retenção proposta (por categoria)

| Categoria | Prazo de retenção proposto | Gatilho de contagem | Após o prazo | Fase |
|-----------|----------------------------|---------------------|--------------|------|
| **Conta do aluno (ativa)** | Enquanto a conta existir + relação ativa | — | — | MVP |
| **Conta após exclusão solicitada** | Eliminação/anonimização em **≤ 15 dias** (SLA art. 18); meta interna ≤ 72h | Pedido do titular | Anonimizar (se houver retenção legal) ou eliminar | MVP |
| **Conta inativa (sem login)** | **24 meses** sugeridos → notificar + eliminar/anonimizar | Último login | Anonimização | F2 |
| **Pagamento / fiscal (`orders`, NF, invoices)** | **mín. 5 anos**; **até ~10–11 anos** se a NF-e estadual exigir | 1º dia do exercício seguinte ao fato gerador | Manter **registro transacional mínimo anonimizado** (art. 16, II) | MVP |
| **Conteúdo do tenant (cursos/aulas/assets)** | Enquanto o tenant existir; pós-saída ver §3.2 | — | Excluir no offboarding (após win-back) | MVP |
| **Gravações de aula ao vivo (R2 master + Bunny VOD)** | **Padrão sugerido 12 meses** (configurável por tenant); master no R2 pode ter prazo menor | `recording_status=ready` | Excluir R2+Bunny + linhas `live_*` | F2 |
| **Comentários/UGC** | Enquanto o curso/tenant existir | — | Excluir/anonimizar autor no offboarding/exclusão | MVP |
| **Logs de auditoria (`audit_log`)** | **Sugerido 5 anos** (accountability/prova) | Evento | Eliminar/anonimizar | MVP |
| **Logs técnicos/observabilidade (Sentry/Pino)** | **30–90 dias** (configurável, sem PII — NFR-OBS-07) | Evento | Eliminar | MVP |
| **Telemetria de produto (PostHog)** | **12–24 meses**, pseudonimizada | Evento | Agregar/eliminar | MVP |
| **Marketing / consentimento de marketing** | Até revogação do opt-in; registro de consentimento por **≥ 5 anos** (prova) | Opt-in / revogação | Eliminar dado, manter prova do consentimento | MVP |
| **Registro de consentimento (cookies/política)** | **≥ 5 anos** após o término da relação (prova de accountability) | Coleta | Eliminar | MVP |
| **Sessões/tokens** | Expiração curta (TTL) + limpeza | Logout/expiração | Eliminar | MVP |

### 3.2 Offboarding do tenant — janela win-back antes do `DROP SCHEMA`

Quando um tenant cancela (status `cancelled`), o schema `tenant_<slug>` contém dados pessoais de **alunos**
(de quem o tenant é Controlador). **Não** se pode dar `DROP SCHEMA` imediato sem oportunizar
recuperação/portabilidade. Fluxo proposto (state machine de offboarding):

```
cancelled
   │  (T0: cancelamento confirmado)
   ▼
[GRACE / WIN-BACK]  ── duração proposta: 30 dias ──────────────────────────────┐
   • acesso ao painel suspenso para novas vendas; dados PRESERVADOS            │
   • alunos: avaliar manter acesso a conteúdo já comprado (OQ #14 — jurídico)  │
   • e-mails de win-back ao Owner ("reative em N dias e nada é perdido")       │
   • export completo do tenant disponível (portabilidade do Controlador)       │
   ▼ (sem reativação)                                                          │
[EXPORT WINDOW]  ── +30 dias ──────────────────────────────────────────────────┤
   • somente leitura/exportação; sem operação                                  │
   • backup final criptografado retido por prazo curto                         │
   ▼ (sem ação)                                                                │
[ELIMINAÇÃO]                                                                    │
   • DROP SCHEMA tenant_<slug>                                                  │
   • purge de assets em R2 (prefixo <tenantId>/) e Bunny library do tenant     │
   • revogar keys (Bunny/LiveKit) e recipients                                 │
   • PRESERVAR somente: registros fiscais mínimos anonimizados (art. 16 II) +  │
     audit_log de offboarding                                                  │
   ◄───────────────────────────────────────────────────────────────────────────┘
```

> **Total sugerido: ~60 dias** entre cancelamento e `DROP SCHEMA` (30 win-back + 30 export). **Números a
> validar (comercial + jurídico).** Implementação: job agendado (pg-boss) com `tenantId` no payload; cada
> transição registrada em `audit_log`. **Importante:** alunos são titulares cujo Controlador é o tenant — o
> contrato B2B/DPA deve definir o que acontece com o acesso do aluno pago quando o tenant sai (OQ #14).

---

## 4. Direitos do titular e como o produto os atende

LGPD **art. 18**: confirmação de tratamento, **acesso**, **correção**, **anonimização/bloqueio/eliminação**,
**portabilidade**, informação sobre compartilhamento, revogação de consentimento. **Acesso/confirmação:
resposta em até 15 dias** (art. 18 §5º / declaração simplificada imediata). Como somos **Operador** dos dados
do aluno, atendemos **fornecendo as ferramentas** e **executando** as instruções do tenant-Controlador.

| Direito (art. 18) | Como o produto atende | Tabelas/fluxos | Fase |
|-------------------|-----------------------|----------------|------|
| **Confirmação + Acesso** | Painel do aluno com seus dados; **exportação** legível por máquina (JSON/CSV) via job assíncrono com link seguro e expiração (NFR-LGPD-06) | `users`, `enrollments`, `lesson_progress`, `orders`, `certificates`… via export | MVP |
| **Correção/retificação** | Edição self-service de perfil (NFR-LGPD-10) | `users` | MVP |
| **Portabilidade** | Mesmo pacote de exportação em formato interoperável; **liga ao DATA_IMPORT_EXPORT/UX LGPD** | export | MVP |
| **Eliminação / esquecimento** | Fluxo de "excluir conta" com confirmação clara das consequências; **eliminação ou anonimização** no schema do tenant (NFR-LGPD-07) | soft-delete → purge | MVP |
| **Eliminação × retenção legal** | Quando há retenção fiscal, **anonimiza** preservando registro transacional mínimo (NFR-LGPD-08; art. 16 II) — explicado ao usuário | `orders` anonimizado | MVP |
| **Revogação de consentimento** | Central de cookies (alterar a qualquer tempo) + opt-out de marketing (`notification_preferences`) + `leaderboard_optout` | CMP, preferências | MVP |
| **Eliminação de gravação de live** | Direito ao esquecimento alcança a **gravação**: deletar arquivo no R2 + VOD na Bunny + linhas `live_*` (LIVE_CLASSES dep. 12) | `live_*`, R2, Bunny | F2 |
| **Informação sobre compartilhamento** | Política de Privacidade lista suboperadores (§6.2) | — | MVP |
| **Revisão de decisões automatizadas (art. 20)** | Anti-seek/gamificação/antifraude — informar e permitir contestação humana quando houver efeito relevante | regras de negócio | F2 |

**Operacional (proposto):** canal único de solicitação (formulário "Privacidade/LGPD") que roteia ao
**Controlador correto** — pedido de aluno → tenant (com a plataforma executando tecnicamente); pedido de
tenant (B2B) → plataforma. **SLA ≤ 15 dias**, meta ≤ 72h (NFR-LGPD-09). Registrar cada pedido e resposta
(accountability).

---

## 5. Consentimento: cookies/CMP e gravação de aula ao vivo

### 5.1 Cookies / CMP — consent gating de PostHog, Meta, GA4 (OPEN_QUESTIONS #25)

O **Guia Orientativo de Cookies da ANPD** estabelece princípios que o banner **DEVE** seguir:

- **Granularidade:** consentimento **por categoria/finalidade** — Necessários (sempre ativos), **Analytics**,
  **Marketing/Pixels** (NFR-LGPD-01).
- **Simetria ("rejeitar tão fácil quanto aceitar"):** se houver "Aceitar tudo", deve haver "Rejeitar tudo"
  com **igual proeminência**; é **vedado** esconder/dificultar a recusa.
- **Consent gating real:** scripts de Analytics/Marketing (PostHog, Meta Pixel, GA4) **só carregam após o
  opt-in** da categoria correspondente — não cosmético (NFR-LGPD-02). Necessários podem operar sem
  consentimento (base = legítimo interesse/execução).
- **Registro auditável:** timestamp, versão da política, escopo da escolha; **revisável/alterável** a qualquer
  tempo via link no rodapé/perfil (NFR-LGPD-03).
- **Por tenant:** o banner respeita o branding e o escopo do tenant; consentimento não vaza entre tenants
  (Regra nº1).

**Decisão pendente (OQ #25):** CMP **próprio** (controle total, integra com o consent gating de PostHog/Sentry
Replay) **vs.** CMP de **terceiro**. **[MVP]** Banner + gating funcional é **essencial**; um CMP de prateleira
acelera. Recomenda-se um **wrapper/port de consentimento** no front para que a fonte (próprio ou terceiro) seja
trocável e o gating dependa de uma única API de "consent state".

### 5.2 Consentimento de gravação de aula ao vivo (F2 — OPEN_QUESTIONS #29)

Imagem e voz são **dados pessoais** e atraem também o **direito de imagem** (CF art. 5º X; CC art. 20). Para a
captura/gravação do aluno em sala WebRTC:

- **Base legal:** **consentimento específico, livre e informado** do aluno para captura de áudio/vídeo e
  para a **gravação**/uso no replay (a doutrina admite consentimento, execução de contrato ou legítimo
  interesse para uso de imagem, mas para **gravação de pessoas identificáveis** o caminho mais seguro é
  **consentimento destacado**). Consentimento de menores exige tratamento específico (art. 14) — **verificar
  faixa etária do público** com o tenant.
- **Transparência (banner "esta aula está sendo gravada"):** indicador **⏺ gravando** visível a todos durante a
  sessão (LIVE_CLASSES §4.1) + aviso/checkbox no **device check** antes de entrar.
- **Opção de participar sem câmera/mic:** aluno pode assistir/participar só por chat se não consentir com a
  captura da própria imagem (não pode ser coagido — consentimento livre).
- **Direito ao esquecimento:** deleção da gravação alcança R2 + Bunny + `live_*` (§4).
- **Residência de dados (region pinning):** avaliar PoP/região BR do provedor de live para tenants sensíveis
  (LIVE_CLASSES dep. 12; OQ #30).

---

## 6. Minutas-base (estruturas)

> **São esqueletos de seções, não texto jurídico final.** Servem para o advogado preencher/ajustar. Cada
> documento precisa de **versão + data + histórico de alterações** e ser **versionado** (o consentimento
> registra a versão aceita).

### 6.1 Termos de Uso (plataforma)

> Atenção: pode haver **dois níveis** de Termos — (a) **Termos B2B** (plataforma ↔ tenant, "MSA"/contrato de
> assinatura) e (b) **Termos do usuário final/aluno** (tenant ↔ aluno, que a plataforma disponibiliza como
> template white-label). Recomendado separar.

1. **Identificação e aceite** (quem é a plataforma; aceite por uso/cadastro; capacidade civil)
2. **Definições** (tenant, aluno, conteúdo, área restrita, planos)
3. **Descrição do serviço** (SaaS de cursos; white-label; o que está/não está incluso)
4. **Cadastro e conta** (veracidade dos dados, segurança da senha, 2FA p/ papéis sensíveis)
5. **Planos, pagamento e renovação** — **distinguir os dois domínios** (README §3): **Billing SaaS (Stripe,
   tenant→plataforma)** vs. **Checkout do aluno (Pagar.me/Asaas, aluno→tenant)**; take rate/split
6. **Obrigações e responsabilidades do tenant** (licitude do conteúdo que publica; é Controlador dos alunos;
   tributos da sua operação; respeito a direitos de terceiros)
7. **Obrigações da plataforma** (disponibilidade/SLA — cruzar NFR §6; suporte; segurança)
8. **Propriedade intelectual** (do tenant sobre seu conteúdo; da plataforma sobre o software; licença de uso)
9. **Conteúdo de terceiros e moderação** (remissão à Política de Moderação §6.4; takedown)
10. **Proteção de vídeo/anti-pirataria** (URLs assinadas, **watermark com identificador do aluno** — informar,
    cruza NFR-LGPD-14)
11. **Privacidade e dados pessoais** (remissão à Política de Privacidade e ao DPA)
12. **Suspensão e encerramento** (inadimplência/dunning; offboarding e **win-back/exportação** §3.2)
13. **Limitação de responsabilidade; isenções** (indisponibilidade de terceiros — Bunny/gateway/LiveKit)
14. **Alterações dos termos** (notificação; versão/data)
15. **Lei aplicável e foro** (Brasil)

### 6.2 Política de Privacidade

1. **Quem somos e papéis** (declarar: plataforma = **Operadora** dos dados de aluno; **Controladora** do
   B2B e de segurança — §1)
2. **Contato do Encarregado/DPO** (e-mail dedicado)
3. **Dados coletados, finalidades e bases legais** (refletir a tabela §2)
4. **Cookies e tecnologias de rastreamento** (categorias; remete ao CMP — §5.1)
5. **Compartilhamento e suboperadores** (lista: Bunny, R2, Pagar.me, Asaas, Stripe, Resend, LiveKit, PostHog,
   Sentry, IA; finalidade de cada)
6. **Transferência internacional** (se houver — provedores fora do BR; salvaguardas; art. 33)
7. **Retenção** (resumo da tabela §3, em linguagem clara)
8. **Direitos do titular e como exercê-los** (§4; prazo de 15 dias; canal)
9. **Segurança** (cifragem, isolamento por tenant, scrubbing de PII)
10. **Menores** (se aplicável — art. 14)
11. **Gravação de aulas ao vivo** (F2 — §5.2)
12. **Alterações** (versão/data) · **Foro/lei aplicável**

### 6.3 DPA — Acordo de Tratamento de Dados (plataforma × tenant)

> Anexo ao contrato B2B. Formaliza que, quanto aos **alunos**, **tenant = Controlador** e **plataforma =
> Operadora**. É a peça que distribui responsabilidades e habilita a suboperação.

1. **Objeto e papéis** (Controlador/Operador; escopo dos dados de aluno tratados)
2. **Objeto e duração do tratamento; natureza e finalidade; categorias de titulares e dados** (anexo descritivo)
3. **Obrigações do Operador** (tratar só sob instrução do Controlador; confidencialidade; segurança art. 46;
   auxiliar nos direitos do titular e em relatórios de impacto)
4. **Suboperadores** (lista autorizada; obrigação de impor as mesmas obrigações; notificação de mudanças)
5. **Segurança da informação** (medidas técnicas: isolamento schema-per-tenant, cifragem de keys, URLs
   assinadas, controle de acesso, logs)
6. **Incidentes de segurança** (notificação ao Controlador **sem demora**; apoio à comunicação à ANPD/titulares
   — art. 48; prazos)
7. **Direitos dos titulares** (Operador fornece ferramentas e executa; Controlador é o respondente primário)
8. **Transferência internacional** (salvaguardas)
9. **Retenção e devolução/eliminação ao término** (espelha §3.2: win-back, exportação, `DROP SCHEMA`, purge R2/Bunny)
10. **Auditoria** (direito do Controlador de auditar/receber evidências de conformidade)
11. **Responsabilidade e indenização** · **Vigência e rescisão** (acompanha o contrato principal)

### 6.4 Política de Moderação de Conteúdo

> Cobre UGC (comentários, chat de live) e o **conteúdo publicado pelo tenant**. Relevante para o **Marco Civil
> da Internet (Lei 12.965/2014)** quanto à responsabilidade por conteúdo de terceiros e **notice-and-takedown**.

1. **Escopo e definições** (conteúdo do tenant vs. UGC do aluno; ilegal vs. abusivo)
2. **Conteúdo proibido** (ilegal — ex.: abuso infantil, incitação a crime, violação de PI/direito autoral,
   fraude; abusivo — assédio, discurso de ódio, spam)
3. **Responsabilidade do tenant** (é o **publicador/Controlador** do seu conteúdo; declara ter direitos; isenta
   a plataforma)
4. **Ferramentas de moderação** (denúncia `comment_reports`; ocultar/`hidden_at`; banir `live_bans`; travar chat)
5. **Notice-and-takedown** (canal de denúncia de conteúdo ilegal; prazo de análise; remoção; contraditório)
6. **Responsabilidade do provedor (plataforma)** (atua mediante notificação; Marco Civil; preservação de
   registros quando exigido por ordem judicial)
7. **Sanções** (remoção de conteúdo, suspensão de usuário, suspensão/encerramento do tenant reincidente)
8. **Direitos autorais/DMCA-like** (procedimento para titulares de direitos)
9. **Transparência e recurso** (informar o usuário moderado; via de contestação)

---

## 7. Matriz MVP vs. depois (essencial × evolutivo)

| Item | MVP (LGPD essencial) | Depois (F2/F3) |
|------|----------------------|----------------|
| Papéis Controlador/Operador formalizados (DPA básico) | ✅ | — |
| Política de Privacidade + Termos publicados | ✅ | Revisão p/ novos módulos (IA, live) |
| Canal do Encarregado/DPO | ✅ | — |
| Banner de cookies + **consent gating** (PostHog/Meta/GA4) | ✅ | CMP avançado/terceiro |
| Registro auditável de consentimento | ✅ | — |
| Exportação de dados (acesso/portabilidade) self-service | ✅ | Imediato/automático |
| Exclusão/anonimização de conta + retenção fiscal | ✅ | Decisões automatizadas (art. 20) |
| Correção self-service de perfil | ✅ | — |
| Tabela de retenção publicada + jobs de purga | ✅ (prazos validados) | Automação fina por categoria |
| Win-back + export + `DROP SCHEMA` no offboarding | ✅ (fluxo) | Otimizações |
| Scrubbing de PII em logs/Sentry | ✅ | — |
| Política de Moderação + takedown | ✅ (básico) | Fluxo de contestação/transparência |
| **Consentimento de gravação de live + deleção** | — | ✅ **F2** (com a feature) |
| **Aviso de IA / dados a provedores de IA** | — | ✅ **F2/F3** |
| Residência de dados/region pinning | — | ✅ Enterprise/F2 |
| Transferência internacional formalizada | ✅ (se já houver provedor fora do BR) | Reavaliar |

---

## 8. Fontes

- LGPD — Lei 13.709/2018 (texto): [Planalto](https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709.htm)
- Art. 5º (controlador/operador): [LGPD Brasil — art. 5](https://lgpd-brasil.info/capitulo_01/artigo_05)
- Art. 7º (bases legais): [LGPD Brasil — art. 7](https://lgpd-brasil.info/capitulo_02/artigo_07) ·
  [TRF5 — requisitos para tratamento](https://www.trf5.jus.br/index.php/lgpd/lgpd-requisitos-para-o-tratamento-de-dados)
- Art. 18 (direitos do titular, prazo 15 dias): [LGPD Brasil — art. 18](https://lgpd-brasil.info/capitulo_03/artigo_18) ·
  [TRF5 — direitos do titular](https://www.trf5.jus.br/index.php/lgpd/lgpd-direitos-do-titular)
- Retenção fiscal (decadência/prescrição 5 anos; guarda de NF-e estendida): [Access — prazos de guarda](https://www.accesscorp.com/pt-br/blog/prazos-para-guarda-de-documentos-tributarios/) ·
  [Portal Tributário — prescrição e decadência](https://www.portaltributario.com.br/artigos/prescricaoedecadencia.htm) ·
  [LegisWeb — novo prazo de guarda de documentos fiscais eletrônicos (ICMS)](https://www.legisweb.com.br/noticia/?id=30563)
- Guia Orientativo de Cookies da ANPD (PDF oficial): [gov.br/anpd](https://www.gov.br/anpd/pt-br/centrais-de-conteudo/materiais-educativos-e-publicacoes/guia-orientativo-cookies-e-protecao-de-dados-pessoais.pdf)
- Gravação/imagem e voz (direito de imagem + LGPD): [Migalhas — proteção de dados no uso de imagem e voz](https://www.migalhas.com.br/depeso/394075/protecao-de-dados-pessoais-no-uso-de-imagem-e-voz) ·
  [Jacobs — direito de imagem em aulas remotas](https://www.jacobsconsultoria.com.br/post/o-direito-de-imagem-dos-docentes-e-discentes-nas-aulas-remotas)

> As demais referências legais (CTN arts. 173/174; CF art. 5º X; CC art. 20; Marco Civil — Lei 12.965/2014;
> ECA/art. 14 LGPD p/ menores) devem ser confirmadas pelo advogado responsável.

---

## 9. Dependências e pontos para o coordenador

1. **Decisão de papel (jurídico — alta prioridade):** confirmar o enquadramento **tenant = Controlador /
   plataforma = Operadora** para dados de aluno (§1.1). Tudo (DPA, Política, atendimento a direitos) deriva
   disso. **Bloqueia** a redação final dos documentos legais.
2. **Prazos de retenção (jurídico + contábil — OQ #24):** validar os **números** da §3, em especial o prazo de
   guarda fiscal (5 vs. ~10–11 anos de NF-e) e a duração da **janela win-back** (proposto 30+30 dias) antes do
   `DROP SCHEMA`. Sem isso, os jobs de purga ficam parametrizados em placeholder.
3. **CMP (produto/eng — OQ #25):** decidir CMP próprio vs. terceiro e implementar a **port de consentimento**
   para o gating de PostHog/Meta/GA4 e Sentry Replay. Impacta NFR-LGPD-01/02 e a stack de analytics.
4. **Gravação de live (jurídico — OQ #29, F2):** confirmar base legal (consentimento específico) e o fluxo de
   banner/device-check/opt-out de câmera; definir prazo de retenção da gravação e o purge no esquecimento.
   Coordenar com LIVE_CLASSES dep. 12 e OQ #30 (region pinning).
5. **Acesso do aluno quando o tenant sai/inadimplente (jurídico/produto — OQ #14):** definir contratualmente
   (Termos/DPA) o destino do acesso do **aluno pagante** quando o tenant é suspenso/cancelado — interage com a
   janela win-back (§3.2).
6. **Encarregado/DPO (negócio):** indicar a pessoa/contato e publicar; criar campo em `tenant_settings` para o
   contato de privacidade do **próprio tenant** (cada tenant é Controlador e deve oferecer canal aos seus
   alunos).
7. **Transferência internacional (jurídico):** mapear quais suboperadores tratam dados **fora do Brasil**
   (provavelmente Stripe, possivelmente PostHog/Sentry/LiveKit conforme região) e formalizar salvaguardas
   (LGPD art. 33). Cruza com OQ #30 (residência de dados).
8. **Watermark e IA (produto/jurídico, F2/F3):** garantir que os Termos informem o **watermark com
   identificador do aluno** (NFR-LGPD-14) e o **uso de IA** com dados (NFR-LGPD-15) — incluir nas revisões
   futuras dos documentos.
9. **DATA_IMPORT_EXPORT / UX LGPD:** o pacote de exportação (§4) precisa de spec de formato e do fluxo de UX
   de "baixar meus dados" / "excluir conta" — coordenar com o time de produto/UX (referenciado em NFR §5.2).
10. **Versionamento legal:** estabelecer onde ficam as versões publicadas dos documentos e como o
    consentimento registra a **versão aceita** (necessário para accountability — §5.1).
