# Branding & White-label — UX do Editor de Marca do Tenant

- **Versão:** 1.0 · **Data:** 2026-06-07
- **Autor:** Product Design (white-label & onboarding/activation)
- **Status:** Proposta para revisão do coordenador de produto
- **Relacionados:** [USER_JOURNEYS](../product/USER_JOURNEYS.md) · [INFORMATION_ARCHITECTURE](../product/INFORMATION_ARCHITECTURE.md) · [MONETIZATION](../product/MONETIZATION.md) · [AUTHORING_UX](../product/AUTHORING_UX.md) · [NOTIFICATIONS_MATRIX](../product/NOTIFICATIONS_MATRIX.md) · [DATA_MODEL §6.10](../DATA_MODEL.md) · [README de produto](../product/README.md)

> **O que este documento é:** a especificação de UX/produto do **editor de marca** que o Admin/Owner do
> tenant usa para deixar a escola "com a cara dele". Orquestra features já previstas (subdomínio + branding
> [MVP]; domínio próprio + SSL e white-label total [F2] — ver [MONETIZATION §A.2](../product/MONETIZATION.md)).
> **Não inventa features novas.** Onde algo depende de modelagem ainda não fechada, está marcado como
> proposta para o coordenador (§9).
>
> **Filtro de qualidade (CLAUDE.md):** todo dado de branding é dado **de tenant** → persistido via
> `withTenant` em `tenant_settings`. Nada de branding vive no `platform` exceto o que governa roteamento
> (`tenants.slug`, `tenants.custom_domain`). **Regra nº1 (isolamento) é inegociável.**

---

## Índice

1. [Objetivo, princípios e personas](#1-objetivo-princípios-e-personas)
2. [O que é customizável por fase (matriz)](#2-o-que-é-customizável-por-fase-matriz)
3. [Limites por plano (quais tiers liberam o quê)](#3-limites-por-plano-quais-tiers-liberam-o-quê)
4. [Anatomia do editor de marca (telas e layout)](#4-anatomia-do-editor-de-marca-telas-e-layout)
5. [Preview ao vivo](#5-preview-ao-vivo)
6. [Cores, tema e validação de contraste (AA)](#6-cores-tema-e-validação-de-contraste-aa)
7. [E-mail branding](#7-e-mail-branding)
8. [Domínio próprio + SSL (F2) — fluxo de verificação](#8-domínio-próprio--ssl-f2--fluxo-de-verificação)
9. [Persistência e mapeamento ao DATA_MODEL](#9-persistência-e-mapeamento-ao-data_model)
10. [Estados, microinterações e analytics](#10-estados-microinterações-e-analytics)
11. [Dependências e pontos para o coordenador](#dependências-e-pontos-para-o-coordenador)

---

## 1. Objetivo, princípios e personas

O **white-label real** é uma das três promessas centrais do produto (LMS profundo + checkout BR +
**marca própria de verdade**, PRD/§USER_JOURNEYS Persona 2). O editor de marca é onde o **Aha #1 do Admin
do Tenant** acontece: *"ver a escola com a própria marca no ar"* ([USER_JOURNEYS §4 Fase B](../product/USER_JOURNEYS.md)).

**Personas e papéis** (ver [RBAC_MATRIX](../product/RBAC_MATRIX.md) e glossário do [README](../product/README.md)):

| Papel | Acesso ao editor de marca |
|-------|---------------------------|
| **Owner (OW)** | Acesso total (marca, domínio, white-label, e-mail branding). |
| **Admin (AD)** | Acesso total à marca; **domínio próprio/SSL** e **remoção de marca da plataforma** podem ficar restritos ao Owner (decisão §9). |
| **Instrutor (IN)** | **Sem acesso** (escopo de conteúdo apenas — [IA §4.2](../product/INFORMATION_ARCHITECTURE.md)). |

**Localização na IA:** `tenant.app.com/manage/configuracoes/marca` (Logo/cores/favicon) e
`/manage/configuracoes/dominio` (subdomínio [MVP] / domínio próprio [F2]) — ver
[IA §2.4](../product/INFORMATION_ARCHITECTURE.md).

**Princípios de design**
- **Preview-first:** o tenant **vê o resultado antes de salvar** (espelha o princípio de "ver como aluno"
  do Estúdio — [AUTHORING_UX §10](../product/AUTHORING_UX.md)).
- **Acessibilidade obrigatória (WCAG AA):** o editor é AA **e** impede o tenant de produzir uma marca que
  quebre o contraste AA do produto final (NFR a11y — [AUTHORING_UX §1](../product/AUTHORING_UX.md),
  [NON_FUNCTIONAL_REQUIREMENTS](../product/NON_FUNCTIONAL_REQUIREMENTS.md)). Ver §6.
- **Defaults seguros:** o tenant nasce com um tema neutro funcional (semeado no provisionamento); branding é
  *progressive enhancement*, nunca um pré-requisito para vender.
- **Cascata de tokens:** valores de marca viram **CSS variables por tenant** aplicadas em todas as
  superfícies do tenant (B/C/D/E da [IA §1](../product/INFORMATION_ARCHITECTURE.md)). 1 fonte → muitas telas (DRY).
- **Gate por plano transparente:** recursos travados por tier aparecem **visíveis com cadeado + CTA de
  upgrade** (não escondidos), reforçando o caminho de expansão (USER_JOURNEYS Super-Admin Fase D).

---

## 2. O que é customizável por fase (matriz)

Legenda: **[MVP]** lançamento · **[F2]** Fase 2 · **[F3]** Fase 3.

| Elemento de marca | Fase | O que o tenant controla | Onde aparece (superfícies) |
|-------------------|------|-------------------------|----------------------------|
| **Logo** (header) | MVP | Upload PNG/SVG; versão clara/escura (proposta §9) | Header do site público, área do aluno, painel, checkout, e-mail |
| **Favicon** | MVP | Upload (gera tamanhos) | Aba do navegador / PWA icon base |
| **Cor primária** | MVP | Color picker + hex; vira `--brand-primary` | Botões, links, destaques, player, e-mail |
| **Cor secundária** | MVP | Color picker + hex; vira `--brand-secondary` | Acentos, badges, gráficos |
| **Tema claro/escuro** | MVP (claro) / F2 (par claro+escuro) | No MVP, paleta única derivada das 2 cores; par de temas = F2 | Todas as superfícies do tenant |
| **Subdomínio** | MVP | Confirma/edita `slug` → `slug.app.com` | URL de toda a operação do tenant |
| **E-mail branding (logo/cor no template)** | MVP | Logo + cor primária aplicadas ao template transacional Resend | E-mails transacionais (boas-vindas, compra, certificado…) |
| **Nome de exibição / título da escola** | MVP | `tenants.name` exibido em headers e `<title>` | Site público, área do aluno, e-mails |
| **Domínio próprio + SSL** | **F2** | `escola.com.br` via CNAME + cert automático | Substitui o subdomínio em todas as superfícies |
| **Remetente de e-mail próprio (domínio verificado)** | **F2** | `contato@escola.com.br` como `From` (DKIM/SPF) | E-mails transacionais saem do domínio do tenant |
| **White-label total (remoção da marca da plataforma)** | **F2** | Remove "Powered by", remove marca da plataforma do footer/login/e-mails | Footer público, tela de login, rodapé de e-mail, página de certificado |
| **Template de certificado** | MVP básico / **F3** customização | MVP: template padrão com logo+cor do tenant; edição visual = F3 | PDF do certificado + página pública de verificação |
| **CSS/tema avançado, fontes custom** | F3 (a confirmar) | Fora de escopo MVP/F2 | — |

> **Sobre "remoção da marca da plataforma":** no MVP **todos** os tiers já recebem branding (logo/cores) nas
> próprias superfícies. O que o white-label F2 adiciona é a **remoção da assinatura da plataforma** ("Powered
> by", footer, marca no login/e-mail) — liberada só para Scale/Enterprise (§3). É a diferença entre *"minha
> marca aparece"* (MVP, todos) e *"a marca da plataforma desaparece"* (F2, tiers altos).

---

## 3. Limites por plano (quais tiers liberam o quê)

Espelha [MONETIZATION §A.2](../product/MONETIZATION.md) (fonte de verdade dos tiers). As features booleanas
ficam em `platform_plans.limits.features[]` ([DATA_MODEL §6.11](../DATA_MODEL.md)); o enforcement chega ao
use-case via **injeção no `onRequest`** (mesmo padrão do take rate — não consulta `platform` direto, ADR-0013).

| Recurso de marca | Feature flag (proposta) | Starter | Pro | Scale | Enterprise |
|------------------|-------------------------|---------|-----|-------|------------|
| Subdomínio + branding (logo/cores/favicon) | — (base, todos) | ✅ | ✅ | ✅ | ✅ |
| E-mail branding (logo/cor no template) | — (base, todos) | ✅ | ✅ | ✅ | ✅ |
| **Domínio próprio + SSL** | `custom_domain` | — | ✅ | ✅ | ✅ |
| **Remetente de e-mail próprio (domínio verificado)** | `email_sender_domain` | — | ✅* | ✅ | ✅ |
| **White-label total (remoção da marca da plataforma)** | `whitelabel_full` | — | — | ✅ | ✅ |
| Par de temas claro+escuro | `theme_dark` (F2) | — | ✅ | ✅ | ✅ |
| Template de certificado customizável (F3) | `cert_template_custom` | — | — | add-on | ✅ |

\* Remetente próprio em Pro **depende** do domínio próprio estar verificado (o e-mail só pode sair de domínio
verificado por DKIM/SPF). Confirmar acoplamento `custom_domain` ↔ `email_sender_domain` (§9).

**UX do gate por plano**
- Recursos travados aparecem **em estado bloqueado** (cadeado + tooltip "Disponível no plano **Pro**")
  com **CTA "Fazer upgrade"** → leva a `/manage/plano-saas` (USER_JOURNEYS Super-Admin Fase D — upsell
  self-service).
- **Downgrade que perde branding:** se o tenant em Scale com domínio próprio + white-label faz downgrade
  para Starter, o sistema **bloqueia/avisa** (mesmo padrão de downgrade que excede quota —
  [MONETIZATION §A.4](../product/MONETIZATION.md)): "Ao voltar para Starter você perderá seu domínio
  `escola.com.br` e a marca da plataforma voltará a aparecer. Resolva antes de continuar." Mostra o **delta
  a resolver**.

---

## 4. Anatomia do editor de marca (telas e layout)

Duas telas dentro de `Config — Marca` e `Config — Domínio` (IA §2.4), com **layout de duas colunas**:
**painel de edição** (esquerda) + **preview ao vivo** (direita, §5).

### 4.1 `/manage/configuracoes/marca` — Identidade visual [MVP]

Seções (acordeão/abas, autosave com estado explícito — §10):

1. **Logotipo**
   - Upload (PNG/SVG/WebP), tamanho/peso máximos validados; recorte/zoom opcional.
   - Variante para fundo escuro (proposta §9: 2º slot `logo_dark_url`).
   - Estado vazio: placeholder com a inicial do `tenants.name` + "Envie seu logo".
2. **Favicon**
   - Upload quadrado; preview em "aba do navegador" simulada. Gera tamanhos a partir de 1 arquivo.
3. **Cores**
   - **Cor primária** e **secundária**: color picker + campo hex + amostras de paleta sugerida.
   - **Validação de contraste AA inline** (§6) ao lado de cada cor.
   - Botão "Restaurar tema padrão".
4. **Nome de exibição da escola** (`tenants.name`) — usado em headers, `<title>`, assinatura de e-mail.
5. **Marca da plataforma** [F2 / Scale+]
   - Toggle "Remover 'Powered by' e a marca da plataforma" (bloqueado por tier — §3).

**Barra de ações:** estado de autosave (`salvando… / salvo / erro`), botão **"Pré-visualizar como visitante/
aluno"** (abre superfície real em modo preview), botão **"Publicar marca"** se optarmos por staging
(decisão §9: autosave-direto vs. rascunho+publicar).

### 4.2 `/manage/configuracoes/dominio` — Endereço da escola

1. **Subdomínio [MVP]:** mostra `slug.app.com`; edição de `slug` com **checagem de disponibilidade** e aviso
   de impacto (muda URLs públicas/SEO — confirmar política de redirect 301, §9).
2. **Domínio próprio [F2]:** wizard de verificação (§8), com estados `none → pending → verifying → active →
   failed`.

---

## 5. Preview ao vivo

- **Painel de preview** à direita renderiza, **em tempo real** a cada alteração, as superfícies-chave usando
  as CSS variables do tenant. Sem salvar para ver: o preview reflete o **estado em edição**.
- **Seletor de superfície** (tabs no topo do preview):
  - **Site público** (home/landing) · **Checkout** · **Área do aluno (player)** · **E-mail transacional**.
  - Cada uma usa componentes reais (shadcn/tokens), não mockups — garante fidelidade.
- **Seletor de viewport:** desktop / mobile (mobile-first é regra do checkout — MONETIZATION §B.1).
- **Modo claro/escuro:** preview alterna para validar contraste nos dois fundos (no MVP, deriva da paleta;
  no F2, par de temas).
- **"Abrir em nova aba como visitante/aluno":** abre a superfície real em **modo preview** (não publicado),
  espelhando o padrão "ver como aluno" do Estúdio ([AUTHORING_UX §10](../product/AUTHORING_UX.md)).

---

## 6. Cores, tema e validação de contraste (AA)

A acessibilidade é **NFR obrigatório** (WCAG AA). O editor não só **é** AA, como **impede o tenant de
configurar uma marca que viole AA** nas combinações de cor que o produto realmente usa.

### 6.1 O que é validado

Para cada cor escolhida, calculamos a **razão de contraste** (fórmula WCAG 2.x — luminância relativa) contra
os fundos/textos onde ela é efetivamente usada:

| Par avaliado | Limiar AA | Onde se aplica |
|--------------|-----------|----------------|
| Texto sobre cor primária (ex.: rótulo de botão branco sobre botão primário) | **≥ 4,5:1** (texto normal) | Botões/CTAs |
| Cor primária como texto/link sobre fundo claro e escuro | **≥ 4,5:1** | Links, destaques |
| Componentes/bordas/ícones (UI não-textual) | **≥ 3:1** | Estados de foco, ícones, gráficos |
| Texto grande (títulos) | **≥ 3:1** | Headings com cor de marca |

### 6.2 UX da validação (inline, em tempo real)

- **Badge de status por cor:** `AA ✓` (verde) / `AA ✗` (vermelho) / `AA grande apenas` (amarelo), com a
  **razão calculada** ("4,8:1 — passa AA").
- **Em falha:** mensagem acionável + **sugestão automática** do tom mais próximo que passa AA
  ("Sua cor tem contraste 3,1:1. Sugerimos `#0B5FFF` (4,6:1). [Aplicar sugestão]").
- **Texto de botão automático:** o sistema escolhe texto branco ou preto sobre a cor primária para **maximizar
  contraste** automaticamente (o tenant não precisa pensar nisso).
- **Bloqueio brando (recomendado):** salvar uma cor que falha AA é permitido **com aviso persistente**
  ("Esta cor pode dificultar a leitura para alguns alunos"), **exceto** combinações que tornariam CTAs de
  pagamento ilegíveis → **bloqueio duro** nessas (decisão de severidade em §9). Justificativa: não punir o
  tenant, mas proteger a conversão e a a11y do produto.
- **Preview de daltonismo** (proposta, F2): simular protanopia/deuteranopia no painel de preview.

### 6.3 Como vira tema

- As 2 cores semente derivam uma **escala de tokens** (hover, ativo, desabilitado, superfícies) via função
  determinística → CSS variables (`--brand-primary`, `--brand-primary-foreground`, `--brand-secondary`, …).
- 1 fonte de verdade (as cores em `tenant_settings`) → todas as superfícies (DRY). O front lê os tokens; não
  há cores hardcoded por tela.

---

## 7. E-mail branding

E-mail transacional é via **Resend** ([NOTIFICATIONS_MATRIX](../product/NOTIFICATIONS_MATRIX.md)). O branding
de e-mail reusa os mesmos tokens da marca (DRY) — não há um segundo editor de cores.

| Item | Fase | Comportamento |
|------|------|---------------|
| **Logo no cabeçalho do e-mail** | MVP | Usa `tenant_settings.logo_url` (variante clara). |
| **Cor primária no e-mail** | MVP | Botões/CTAs do template usam `--brand-primary`; contraste AA aplicado (§6). |
| **Nome do remetente (`From` name)** | MVP | `tenants.name` (ex.: "Escola Acme"). |
| **Endereço do remetente** | MVP | Domínio **da plataforma** verificado (ex.: `nao-responda@app.com`), com `reply_to` configurável pelo tenant. |
| **Remetente de domínio próprio** (`contato@escola.com.br`) | **F2** | Requer verificação DKIM/SPF do domínio do tenant no Resend; gated por tier (§3). |
| **Rodapé com marca da plataforma** | MVP / removível **F2** | "Enviado por [plataforma]" no rodapé; removido no white-label total (Scale+, §3). |
| **Preview de e-mail** | MVP | É uma das superfícies do preview ao vivo (§5). |

> **Importante:** no MVP o **From** é da plataforma (deliverability garantida por nós); só no F2 o tenant
> envia do próprio domínio, e **apenas** após verificação de domínio (mesmo gate técnico do domínio próprio,
> §8). Sem verificação → continua saindo do domínio da plataforma com `reply_to` do tenant.

---

## 8. Domínio próprio + SSL (F2) — fluxo de verificação

Feature **[F2]**, liberada para **Pro/Scale/Enterprise** (§3). Persiste em `tenants.custom_domain`
([DATA_MODEL §1](../DATA_MODEL.md)). Wizard de 4 estados, idempotente e reentrante.

### 8.1 Wizard

1. **Informar domínio** — tenant digita `escola.com.br` (validação de formato; um domínio por tenant no MVP/F2).
2. **Configurar DNS** — instruções claras com **registros a criar** (copiáveis):
   - `CNAME` do domínio (ou ALIAS/A para apex) apontando para o host da plataforma.
   - Registro `TXT` de verificação de propriedade (token único por tenant).
   - (Para e-mail próprio §7) registros `DKIM`/`SPF`/`DMARC` do Resend.
3. **Verificar** — botão "Verificar agora" dispara checagem de DNS (job assíncrono). Estado `verifying`
   com polling/banner.
4. **SSL automático** — após propriedade verificada, **emissão automática de certificado** (ACME/Let's
   Encrypt via plataforma/edge); estado `active` quando o cert está válido e o domínio serve HTTPS.

### 8.2 Máquina de estados do domínio (proposta — §9)

| Estado | Significado | UX |
|--------|-------------|-----|
| `none` | Só subdomínio | Estado inicial |
| `pending` | Domínio informado, DNS não verificado | Mostra registros + "Verificar" |
| `verifying` | Checagem em andamento | Spinner/polling; "pode levar até X min (propagação DNS)" |
| `active` | DNS ok + SSL emitido | Banner verde; subdomínio passa a redirecionar (301) para o domínio próprio |
| `failed` | DNS/SSL falhou | Erro acionável + retry idempotente + link de ajuda |

### 8.3 Regras
- **Sessão/cookies em dois hosts:** a sessão (Better-Auth) precisa funcionar tanto em `slug.app.com` quanto
  em `custom_domain` — **dependência de engenharia** já registrada ([IA Dep. #5](../product/INFORMATION_ARCHITECTURE.md)).
- **Renovação de SSL** automática (sem ação do tenant); alerta interno (Super-Admin) se a renovação falhar.
- **Remoção do domínio** volta o estado para `none` e reativa o subdomínio (com aviso de impacto em SEO).
- **Resolução de tenant:** o domínio **sugere** o tenant; o **claim do JWT (`tenant_id`)** é a autoridade
  (Regra nº1 — [IA §5.1](../product/INFORMATION_ARCHITECTURE.md)). Domínio próprio **não** afeta isolamento.

---

## 9. Persistência e mapeamento ao DATA_MODEL

Todo branding é **dado de tenant** → `withTenant` → `tenant_settings` (1 linha por tenant,
[DATA_MODEL §6.10](../DATA_MODEL.md)). O roteamento (subdomínio/domínio) vive em `platform.tenants` porque
governa resolução de host (control plane), **sem** FK para o schema do tenant.

### 9.1 Campos já existentes (DATA_MODEL §6.10 / §1)

```sql
-- tenant_settings (data plane) — JÁ EXISTE
logo_url text null
primary_color text null
secondary_color text null
-- (default_locale, supported_locales, políticas... — não-branding)

-- platform.tenants (control plane) — JÁ EXISTE
slug text unique          -- subdomínio (MVP)
custom_domain text null   -- domínio próprio (F2)
name text                 -- nome de exibição da escola
```

### 9.2 Campos propostos (extensão — requer alinhamento/ADR, §11)

Para suportar o escopo deste doc sem violar isolamento (tudo no data plane, exceto roteamento):

```sql
-- tenant_settings (data plane) — PROPOSTA
favicon_url text null                 -- §4.1 (hoje não há coluna)
logo_dark_url text null               -- variante p/ fundo escuro (§4.1)
email_reply_to text null              -- reply_to do e-mail transacional (§7)
whitelabel_full boolean default false -- estado efetivo (gated por feature flag do plano)

-- platform.tenants (control plane) — PROPOSTA (apenas o que governa roteamento/verificação)
custom_domain_status text null        -- none|pending|verifying|active|failed (§8.2)
custom_domain_verify_token text null  -- TXT de verificação de propriedade
email_sender_domain text null         -- domínio verificado p/ From próprio (F2, §7)
```

### 9.3 Feature flags de plano (control plane — `platform_plans.limits.features[]`)

Booleanos consumidos via injeção no `onRequest` (ADR-0013), **não** consultados pelo use-case direto:
`custom_domain`, `email_sender_domain`, `whitelabel_full`, `theme_dark` (F2), `cert_template_custom` (F3).

### 9.4 Onde os assets binários vivem
- Logo/favicon = **arquivos** → bucket de objetos (R2, mesmo padrão de `lesson_assets`/`cover_url`); o que
  fica em `tenant_settings` é a **URL**. Confirmar bucket/prefixo por tenant e política de cache/CDN (§11).

---

## 10. Estados, microinterações e analytics

**Estados sempre presentes** (espelha [AUTHORING_UX §11](../product/AUTHORING_UX.md)):
- **Vazio:** sem logo/favicon → placeholders guiados ("Envie seu logo para personalizar sua escola").
- **Autosave:** `salvando… / salvo / erro` explícito; sem perda de trabalho.
- **Upload:** barra de progresso + validação de formato/tamanho com erro acionável.
- **Validação de contraste:** badge AA por cor (§6), em tempo real.
- **Domínio (F2):** `pending/verifying/active/failed` com retry idempotente (§8).
- **Bloqueado por plano:** cadeado + CTA de upgrade (§3).

**Eventos de analytics** (PostHog, todos com `tenant_id` — [ANALYTICS_AND_DASHBOARDS](../product/ANALYTICS_AND_DASHBOARDS.md)):
`branding_logo_uploaded`, `branding_colors_updated`, `branding_contrast_warning_shown`,
`branding_contrast_suggestion_applied`, `branding_published`, `custom_domain_started`,
`custom_domain_verified`, `custom_domain_failed`, `whitelabel_enabled`.
Conecta ao **Aha #1 do Admin** (escola no ar com a própria marca — USER_JOURNEYS §8) e alimenta o
**checklist de ativação** (item "Configurar marca" — ver [ONBOARDING_ACTIVATION](../product/ONBOARDING_ACTIVATION.md)).

---

## Dependências e pontos para o coordenador

1. **Extensão do DATA_MODEL §6.10 / §1 (proposta §9.2):** `favicon_url`, `logo_dark_url`, `email_reply_to`,
   `whitelabel_full` em `tenant_settings`; `custom_domain_status`, `custom_domain_verify_token`,
   `email_sender_domain` em `platform.tenants`. **Requer confirmação/ADR** — hoje `tenant_settings` só tem
   `logo_url`, `primary_color`, `secondary_color`. Sem `favicon_url`, o favicon (escopo MVP da IA §2.4) não
   tem onde persistir.
2. **Severidade da validação de contraste (§6.2):** bloqueio **brando** (aviso, padrão) vs **duro** (impede
   salvar) — recomendamos brando geral + duro só em CTAs de pagamento. Decisão de produto/a11y.
3. **Autosave-direto vs. rascunho+publicar (§4.1):** o branding aplica ao vivo ao salvar, ou há um estado de
   "rascunho de marca" publicado por botão? Recomendação: autosave-direto no MVP (simplicidade), staging F2.
4. **RBAC fino (Owner × Admin) para domínio/white-label (§1):** domínio próprio, remetente próprio e remoção
   de marca da plataforma ficam restritos ao **Owner** ou liberados ao **Admin**? Alinhar com
   [RBAC_MATRIX](../product/RBAC_MATRIX.md) e [IA Dep. #4](../product/INFORMATION_ARCHITECTURE.md).
5. **Domínio próprio + cookies/SSL (engenharia, F2):** sessão Better-Auth em `slug.app.com` **e**
   `custom_domain`; emissão/renovação de SSL automática (ACME na edge). Já registrado em
   [IA Dep. #5](../product/INFORMATION_ARCHITECTURE.md) e marcado como validação de engenharia no
   [README §4 #22](../product/README.md).
6. **Acoplamento e-mail próprio ↔ domínio próprio (§3/§7):** remetente de domínio próprio (`email_sender_domain`)
   exige domínio verificado no Resend (DKIM/SPF). Confirmar se Pro libera e-mail próprio só após o domínio
   próprio estar `active`, e a UX de verificação combinada DNS+DKIM.
7. **Política de redirect ao trocar subdomínio/ativar domínio (§4.2/§8.2):** 301 do host antigo para o novo,
   preservação de SEO/canonical (cruza com [IA §5.4](../product/INFORMATION_ARCHITECTURE.md)).
8. **Storage de assets de marca (§9.4):** confirmar bucket/prefixo por tenant (R2), limites de tamanho,
   sanitização de SVG (segurança) e estratégia de CDN/cache-busting ao trocar logo.
9. **Template de certificado (§2):** MVP usa logo+cor do tenant no template padrão; customização visual é
   **F3** ([IA Dep. #7](../product/INFORMATION_ARCHITECTURE.md), [README §4 #23](../product/README.md)).
   Confirmar o que é editável no MVP.
10. **Downgrade que remove branding (§3):** confirmar a UX de bloqueio/aviso (paridade com a política de
    downgrade que excede quota — [MONETIZATION §A.4](../product/MONETIZATION.md)).
```
