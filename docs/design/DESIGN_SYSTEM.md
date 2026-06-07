# Design System — Plataforma de Cursos SaaS Multitenant White-Label

- **Versão:** 1.0 · **Data:** 2026-06-07
- **Autor:** Design Systems Lead (Figma + Design Tokens + shadcn/ui + Tailwind)
- **Status:** Proposta para revisão de produto/engenharia/design
- **Stack-alvo:** Next.js (App Router/RSC) · shadcn/ui (Radix) · Tailwind CSS · next-intl
- **Relacionados:** [ARCHITECTURE.md §5](../ARCHITECTURE.md) · [NON_FUNCTIONAL_REQUIREMENTS.md](../product/NON_FUNCTIONAL_REQUIREMENTS.md) · [INFORMATION_ARCHITECTURE.md](../product/INFORMATION_ARCHITECTURE.md) · [LEARNING_EXPERIENCE_UX.md](../product/LEARNING_EXPERIENCE_UX.md) · [README §2 (glossário)](../product/README.md) · [CLAUDE.md](../../CLAUDE.md)

> **Filtros de qualidade não-negociáveis:** SOLID/DRY/Clean Code · **WCAG 2.1 AA** desde o MVP · **isolamento por tenant** (Regra nº1) · branding white-label por tenant via CSS variables **sem quebrar contraste AA**.

---

## Índice

1. [Princípios visuais](#1-princípios-visuais)
2. [Arquitetura de design tokens](#2-arquitetura-de-design-tokens)
3. [Tokens de cor e tematização por tenant (white-label)](#3-tokens-de-cor-e-tematização-por-tenant-white-label)
4. [Tipografia](#4-tipografia)
5. [Espaçamento, raios, sombras, breakpoints, z-index e motion](#5-espaçamento-raios-sombras-breakpoints-z-index-e-motion)
6. [Dark mode](#6-dark-mode)
7. [Grid, layout e responsividade](#7-grid-layout-e-responsividade)
8. [Inventário de componentes](#8-inventário-de-componentes)
9. [Estados obrigatórios e padrões de a11y por componente](#9-estados-obrigatórios-e-padrões-de-a11y-por-componente)
10. [Ícones e ilustração](#10-ícones-e-ilustração)
11. [Convenções de nomenclatura e organização no código](#11-convenções-de-nomenclatura-e-organização-no-código)
12. [Conexão com NFRs (performance/CLS/budget)](#12-conexão-com-nfrs-performancecls-budget)
13. [Dependências e pontos para o coordenador](#dependências-e-pontos-para-o-coordenador)

---

## 1. Princípios visuais

O design system serve a uma plataforma **white-label**: a marca visível é a do **tenant** (escola/produtor), não a nossa. Nossa identidade é a **previsibilidade, a clareza e a acessibilidade** do chassi.

1. **White-label primeiro.** Nenhuma cor de marca hardcoded em componente. A "personalidade" vem do tenant (logo + paleta + tipografia opcional). O sistema garante estrutura, ritmo e contraste; o tenant injeta a cor.
2. **Acessível por construção (AA), não por retrofit.** Cada componente nasce navegável por teclado, com foco visível, ARIA correto e contraste ≥ 4,5:1 (texto) / 3:1 (UI/ícones). Ver [NFR-A11Y-01..24](../product/NON_FUNCTIONAL_REQUIREMENTS.md#3-acessibilidade-wcag-21-aa).
3. **Conteúdo no centro.** A UI é discreta para destacar curso, vídeo e progresso. Cromia neutra dominante; cor de marca reservada para ação e ênfase (não para decorar).
4. **Mobile-first.** O aluno BR é majoritariamente mobile em 4G mid-tier ([NFR-COMP-02](../product/NON_FUNCTIONAL_REQUIREMENTS.md#8-compatibilidade-responsividade-e-pwa)). Projeta-se em 320px e expande-se para cima.
5. **Estável (sem CLS).** Skeletons reservam espaço, imagens têm `aspect-ratio`, fontes com `font-display: swap` com fallback métrico. Meta CLS ≤ 0,1 ([NFR-PERF-03](../product/NON_FUNCTIONAL_REQUIREMENTS.md#2-performance-e-core-web-vitals)).
6. **Leve.** Budget de JS/CSS apertado ([NFR-PERF-06/07](../product/NON_FUNCTIONAL_REQUIREMENTS.md#2-performance-e-core-web-vitals)); preferir RSC e CSS sobre JS; componentes pesados via `dynamic import`.
7. **Previsível para IA.** Tokens semânticos, variantes via `cva`, nomes explícitos e 1 conceito por arquivo — alinhado à filosofia de previsibilidade do [CLAUDE.md](../../CLAUDE.md) e [ARCHITECTURE §13](../ARCHITECTURE.md).
8. **Estados completos sempre.** Toda superfície de dados prevê `loading / empty / error` além de `default` ([NFR-REL-10](../product/NON_FUNCTIONAL_REQUIREMENTS.md#6-confiabilidade-e-disponibilidade)).
9. **Nunca só cor.** Status sempre combina cor + ícone + texto ([NFR-A11Y-08](../product/NON_FUNCTIONAL_REQUIREMENTS.md#3-acessibilidade-wcag-21-aa)).
10. **Propriedades lógicas.** Usar `inline-start/end`, não `left/right`, preparando RTL futuro ([NFR-I18N-05](../product/NON_FUNCTIONAL_REQUIREMENTS.md#4-internacionalização-i18n-e-localização)).

---

## 2. Arquitetura de design tokens

Adotamos uma hierarquia de **3 camadas** (alinhada ao W3C Design Tokens Community Group), que é a chave para o white-label e para o dark mode sem duplicação:

```
┌─ Camada 1: TOKENS PRIMITIVOS (global, imutáveis) ──────────────┐
│  Escalas cruas. Não usados diretamente em componentes.          │
│  --gray-50..950, --blue-500, --space-4, --radius-8, --text-md   │
└────────────────────────────────────────────────────────────────┘
            ↓ referenciados por
┌─ Camada 2: TOKENS SEMÂNTICOS (a interface de design) ──────────┐
│  Significado, não valor. É o que os componentes consomem.       │
│  --color-bg, --color-fg, --color-primary, --color-border,       │
│  --color-destructive, --space-card, --radius-control            │
│  ↳ ALTERNAM entre light/dark e RECEBEM override do tenant.      │
└────────────────────────────────────────────────────────────────┘
            ↓ referenciados por
┌─ Camada 3: TOKENS DE COMPONENTE (opcional, quando necessário) ─┐
│  --button-primary-bg, --card-shadow, --player-controls-bg       │
│  Só quando um componente precisa de exceção controlada.         │
└────────────────────────────────────────────────────────────────┘
```

**Regras de ouro:**

- **Componentes nunca referenciam primitivos.** Um botão usa `--color-primary`, jamais `--blue-500`. Isso permite trocar tema/tenant/modo num único ponto (**DRY**).
- **O tenant só sobrescreve a camada 2 (semântica)** — e apenas um subconjunto curado (cor de marca, raio, e poucos mais). Primitivos e tokens de componente ficam protegidos.
- **Fonte única de verdade:** os tokens vivem em `packages/ui/tokens/` (JSON estilo W3C) e são compilados (Style Dictionary) para: (a) CSS variables consumidas pelo Tailwind, (b) `tailwind.config` (mapeamento `theme.extend`), (c) export TS opcional para casos fora do Tailwind. Figma consome o **mesmo** JSON via Tokens Studio. Não se edita CSS variável à mão — gera-se.
- **Tailwind como consumidor, não como dono.** As cores no `tailwind.config` apontam para `hsl(var(--color-*))`, então `bg-primary`, `text-fg`, `border-border` funcionam e respondem a tema/tenant automaticamente.

### 2.1 Convenção de nomes de token

`--{categoria}-{papel}-{variante?}-{estado?}` em **kebab-case**:

- `--color-primary`, `--color-primary-foreground`, `--color-primary-hover`
- `--color-bg`, `--color-bg-subtle`, `--color-fg`, `--color-fg-muted`
- `--color-success`, `--color-warning`, `--color-destructive`, `--color-info` (+ `-foreground` de cada)
- `--space-4`, `--radius-control`, `--shadow-overlay`, `--z-modal`, `--motion-duration-fast`

---

## 3. Tokens de cor e tematização por tenant (white-label)

### 3.1 Cor armazenada como canais HSL

Cada cor semântica é uma CSS variable **em canais HSL sem a função** (`221 83% 53%`), e o consumo aplica `hsl(var(--...))`. Isso permite:
- derivar variações (hover/active) por manipulação de **L**uminosidade sem novo token;
- compor alpha (`hsl(var(--color-primary) / 0.1)`) para estados sutis sem mais um token.

```css
/* packages/ui/styles/tokens.generated.css — base neutra (gerada, não editar à mão) */
:root {
  /* Superfícies / texto */
  --color-bg:               0 0% 100%;
  --color-bg-subtle:        220 14% 96%;
  --color-fg:               222 47% 11%;
  --color-fg-muted:         220 9% 46%;
  --color-border:           220 13% 91%;
  --color-ring:             221 83% 53%;   /* anel de foco — default = primary */

  /* Marca (DEFAULT do sistema; SOBRESCRITO pelo tenant) */
  --color-primary:            221 83% 53%;
  --color-primary-foreground: 0 0% 100%;

  /* Feedback (NÃO sobrescritos pelo tenant — semântica universal) */
  --color-success:            142 71% 36%;  --color-success-foreground: 0 0% 100%;
  --color-warning:            38 92% 40%;   --color-warning-foreground: 0 0% 100%;
  --color-destructive:        0 72% 45%;    --color-destructive-foreground: 0 0% 100%;
  --color-info:               221 83% 45%;  --color-info-foreground: 0 0% 100%;

  /* Tokens de raio/espaço usados como CSS var (ver §5) */
  --radius: 0.625rem;
}
```

### 3.2 Como o tenant sobrescreve cor e logo (sem quebrar AA)

O `middleware.ts` resolve o tenant por subdomínio ([IA §5.1](../product/INFORMATION_ARCHITECTURE.md#5-estrutura-de-urls)) e o RSC raiz injeta um bloco `<style>` com **apenas os tokens semânticos curados** do tenant, derivados de `tenant_settings` (logo, cor de marca, raio). Isso usa a **mesma técnica de CSS variables** já prevista em [ARCHITECTURE §5](../ARCHITECTURE.md).

```tsx
// app/layout.tsx (RSC) — branding por tenant injetado no servidor (chave de cache inclui tenantId+locale)
<style id="tenant-theme" dangerouslySetInnerHTML={{ __html: `
  :root{
    --color-primary: ${tenant.brand.primaryHsl};
    --color-primary-foreground: ${tenant.brand.primaryFgHsl};
    --color-ring: ${tenant.brand.primaryHsl};
    --radius: ${tenant.brand.radius ?? '0.625rem'};
  }
` }} />
```

**Subconjunto sobrescrevível pelo tenant (curado — ISP de tokens):**

| Token | Sobrescreve? | Por quê |
|-------|--------------|---------|
| `--color-primary` (+ `-foreground`) | ✅ | cor de ação da marca |
| `--color-ring` | ✅ (= primary) | foco coerente com a marca |
| `--radius` | ✅ (opcional) | "tom" da marca (arredondado vs reto) |
| `--color-bg/-fg/-border` | ⚠️ apenas em paletas pré-aprovadas | risco alto de contraste; ver §3.3 |
| `--color-success/warning/destructive/info` | ❌ | semântica universal; não é branding |
| primitivos e tokens de componente | ❌ | protegidos |

O `--color-primary-foreground` **é calculado/validado**, não escolhido livremente: dado o `primary`, o sistema escolhe preto ou branco como texto sobreposto pelo maior contraste — garantindo legibilidade de botão independentemente da cor da marca.

### 3.3 Garantia de contraste AA (gate, não confiança no admin)

Atende **[NFR-A11Y-03/20/21](../product/NON_FUNCTIONAL_REQUIREMENTS.md#3-acessibilidade-wcag-21-aa)** — "a a11y não pode depender do bom gosto do admin":

1. **Validação em tempo real no editor de marca** (`/manage/configuracoes/marca`): ao escolher a cor, calcula-se o contraste (APCA/WCAG) de:
   - `primary-foreground` sobre `primary` (texto de botão) ≥ 4,5:1;
   - `primary` sobre `bg` (link/borda de foco/ícone) ≥ 3:1.
   O componente **bloqueia salvar** (ou exige confirmar um ajuste sugerido) se reprovar, com mensagem textual.
2. **Auto-ajuste de luminosidade:** se a cor da marca não atinge contraste contra a superfície (ex.: amarelo claro como botão), o sistema deriva um `--color-primary-strong` (mesma matiz, L reduzida) para uso em texto/links/foco, preservando a cor pura para o fundo do botão. O token primário "para texto sobre fundo claro" pode ser distinto do "para preenchimento" — desacopla cor de marca de cor acessível.
3. **Paletas pré-aprovadas** ([NFR-A11Y-21](../product/NON_FUNCTIONAL_REQUIREMENTS.md#3-acessibilidade-wcag-21-aa)) como ponto de partida, todas testadas em light e dark.
4. **Gate de CI:** `axe-core` nas telas-chave roda contra um tenant de teste com paleta "agressiva" para provar que o chassi resiste.
5. **Utilitário compartilhado** `assertContrast()` em `packages/ui/a11y/` — **DRY**: a mesma função valida no editor, no seed de tenant e no teste.

### 3.4 Logo e favicon

- Logo via `next/image` com `width/height` explícitos (sem CLS), `priority` no header, `alt` = nome do tenant. Slot reservado por `aspect-ratio` para evitar reflow durante o carregamento.
- Variante de logo para dark mode (`logoDark`) opcional em `tenant_settings`; fallback para a logo única.
- Favicon/manifest por tenant ([NFR-COMP-07](../product/NON_FUNCTIONAL_REQUIREMENTS.md#8-compatibilidade-responsividade-e-pwa) PWA F2).

---

## 4. Tipografia

- **Famílias:** máximo 2 ([NFR-PERF-09](../product/NON_FUNCTIONAL_REQUIREMENTS.md#2-performance-e-core-web-vitals)), `next/font` (self-host, sem FOUT, `font-display: swap`, subsetting). Default: **Inter** (UI/corpo) + opcional display. Tenant pode trocar por uma lista curada de fontes self-hosted (F2) — nunca URL arbitrária (segurança/perf).
- **Fallback métrico** para minimizar CLS de fonte.

### 4.1 Escala tipográfica (token `--text-*`, type scale 1.25)

| Token | rem / line-height | Uso |
|-------|-------------------|-----|
| `text-xs` | 0,75 / 1rem | legendas, metadados |
| `text-sm` | 0,875 / 1,25rem | corpo secundário, labels |
| `text-base` | 1 / 1,5rem | corpo padrão |
| `text-lg` | 1,125 / 1,75rem | subtítulo |
| `text-xl` | 1,25 / 1,75rem | título de card |
| `text-2xl` | 1,5 / 2rem | título de seção |
| `text-3xl` | 1,875 / 2,25rem | título de página |
| `text-4xl` | 2,25 / 2,5rem | hero (landing) |

- **Pesos:** 400 / 500 / 600 / 700.
- **Medida de leitura:** corpo longo limitado a `max-w-prose` (~65ch).
- **`rem` em tudo** → respeita zoom 200% ([NFR-A11Y-09](../product/NON_FUNCTIONAL_REQUIREMENTS.md#3-acessibilidade-wcag-21-aa)).
- **Hierarquia semântica:** o estilo é desacoplado do nível de heading — usar `h1..h6` corretos para estrutura ([NFR-A11Y-06](../product/NON_FUNCTIONAL_REQUIREMENTS.md#3-acessibilidade-wcag-21-aa)) e classe de estilo separada quando necessário.

---

## 5. Espaçamento, raios, sombras, breakpoints, z-index e motion

### 5.1 Espaçamento (base 4px — escala Tailwind)

`0, 1(4), 2(8), 3(12), 4(16), 6(24), 8(32), 12(48), 16(64), 24(96)`. Tokens semânticos: `--space-card` (24), `--space-section` (48/64), `--space-inline` (gap de controles, 8/12). Alvos de toque ≥ 44px ([NFR-A11Y-10](../product/NON_FUNCTIONAL_REQUIREMENTS.md#3-acessibilidade-wcag-21-aa)).

### 5.2 Raios (derivados de `--radius`, controlado pelo tenant)

| Token | Cálculo | Uso |
|-------|---------|-----|
| `radius-sm` | `calc(var(--radius) - 4px)` | inputs internos, badges |
| `radius-control` (DEFAULT) | `var(--radius)` | botões, inputs, cards |
| `radius-lg` | `calc(var(--radius) + 4px)` | modais, painéis |
| `radius-full` | `9999px` | avatar, pill, progress |

### 5.3 Sombras (sutis; elevação semântica)

`--shadow-sm` (cards em repouso) · `--shadow-md` (hover de card, popover) · `--shadow-lg`/`--shadow-overlay` (modais, dropdowns). No dark mode, sombras quase desaparecem; a elevação vem de **borda + superfície mais clara** (ver §6).

### 5.4 Breakpoints (mobile-first, padrão Tailwind)

| Token | Largura | Alvo |
|-------|---------|------|
| (base) | ≥ 320px | mobile (default, projeto primário) |
| `sm` | ≥ 640px | mobile grande |
| `md` | ≥ 768px | tablet |
| `lg` | ≥ 1024px | desktop / 2 colunas de gestão |
| `xl` | ≥ 1280px | desktop largo |
| `2xl` | ≥ 1536px | telas amplas (limitar `max-w` de conteúdo) |

Sem scroll horizontal em ≥ 320px ([NFR-COMP-03](../product/NON_FUNCTIONAL_REQUIREMENTS.md#8-compatibilidade-responsividade-e-pwa)).

### 5.5 Z-index (escala fechada — evita guerra de z-index)

| Token | Valor | Camada |
|-------|-------|--------|
| `--z-base` | 0 | fluxo normal |
| `--z-sticky` | 100 | topbar/sidebar fixos |
| `--z-dropdown` | 200 | menus, selects, popovers |
| `--z-overlay` | 300 | scrim de modal/drawer |
| `--z-modal` | 400 | diálogos, sheets |
| `--z-toast` | 500 | notificações |
| `--z-tooltip` | 600 | tooltips (sempre no topo) |

### 5.6 Motion

| Token | Valor | Uso |
|-------|-------|-----|
| `--motion-duration-fast` | 120ms | hover, foco |
| `--motion-duration-base` | 200ms | abrir/fechar, fade |
| `--motion-duration-slow` | 320ms | sheets, transições de página |
| `--motion-ease-standard` | `cubic-bezier(.2,0,0,1)` | entradas/saídas |

**Obrigatório:** envolver toda animação não-essencial em `@media (prefers-reduced-motion: reduce)` que zera/encurta a transição ([NFR-A11Y-11](../product/NON_FUNCTIONAL_REQUIREMENTS.md#3-acessibilidade-wcag-21-aa)). Animar apenas `transform`/`opacity` (sem reflow, protege INP/CLS).

---

## 6. Dark mode

**Sim, suportado** desde a base. Estratégia: classe `.dark` no `<html>` (estratégia `class` do Tailwind), alternada por preferência do usuário (perfil) com default em `prefers-color-scheme`, persistida sem flash (script inline no `<head>`).

- **Implementação por tokens:** o bloco `.dark` redefine **somente os tokens semânticos da camada 2** — zero mudança em componentes. Mesma técnica usada pelo tenant override.
- **Compatibilidade com white-label:** a cor `--color-primary` do tenant vale nos dois modos; se o contraste reprovar no dark (ex.: marca muito escura), aplica-se o `--color-primary-strong` derivado (§3.3) só no `.dark`. A validação de contraste (§3.3) roda **nos dois modos**.
- **Elevação:** no dark, superfícies elevadas ficam mais **claras** (não mais sombreadas); bordas ganham peso.

```css
.dark{
  --color-bg: 222 47% 11%;   --color-bg-subtle: 217 33% 17%;
  --color-fg: 210 40% 98%;   --color-fg-muted: 215 20% 65%;
  --color-border: 217 33% 24%;
  /* feedback recalibrado para contraste em fundo escuro */
  --color-success: 142 64% 48%; --color-destructive: 0 72% 58%; /* ... */
}
```

Player, certificados (PDF) e checkout podem **forçar light** quando a fidelidade de cor importa (decisão por tela — ver §13).

---

## 7. Grid, layout e responsividade

- **Container:** `max-w` por contexto — conteúdo de leitura `max-w-3xl`, listas de gestão `max-w-7xl`, hero full-bleed. Padding lateral `px-4 md:px-6 lg:px-8`.
- **Grid:** CSS Grid/Flex com `gap` de token. Catálogo/cards: `grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4`.
- **Shells por área** ([IA §2](../product/INFORMATION_ARCHITECTURE.md#2-sitemap-por-área)):
  - **Público (B):** header (logo + nav + CTA login) · main · footer (legal/LGPD).
  - **Aluno `/app` (C):** topbar minimalista; **sala de aula** = player full-width + sidebar de aulas que **colapsa em drawer/accordion abaixo do player no mobile** ([LEARNING_EXPERIENCE_UX §2](../product/LEARNING_EXPERIENCE_UX.md)).
  - **Gestão `/manage` (D):** sidebar colapsável (vira drawer no mobile) + topbar com breadcrumb + main com tabelas/forms.
  - **Afiliado `/affiliate` (E)** e **Super-Admin `admin.app.com` (F):** mesmo shell de painel.
- **Skip-to-content** link + landmarks (`header/nav/main/footer`) em todos os shells ([NFR-A11Y-06](../product/NON_FUNCTIONAL_REQUIREMENTS.md#3-acessibilidade-wcag-21-aa)).
- **Reflow a 320px** sem perda ([NFR-A11Y-09](../product/NON_FUNCTIONAL_REQUIREMENTS.md#3-acessibilidade-wcag-21-aa)); propriedades lógicas para RTL futuro.

---

## 8. Inventário de componentes

Dois níveis: **base shadcn/ui** (`components/ui/`, primitivos genéricos sobre Radix) e **compostos de domínio** (`features/<feature>/components/`, conhecem regras de negócio). Compostos **compõem** os base — nunca reimplementam (**DRY**).

### 8.1 Base (shadcn/ui — `components/ui/`)

| Componente | Variantes principais | Notas |
|------------|----------------------|-------|
| `Button` | `default · secondary · outline · ghost · destructive · link` × `sm/md/lg/icon` | variantes via `cva`; `asChild` p/ links |
| `Input`, `Textarea`, `Select`, `Combobox`, `Checkbox`, `Radio`, `Switch`, `Slider` | — | sempre com `Label` + `aria-describedby` p/ erro |
| `Label`, `Form` (RHF + Zod) | — | mensagens via `packages/contracts` |
| `Card` | `default · interactive` | base de CourseCard etc. |
| `Badge` | `neutral · success · warning · destructive · info · outline` | cor + ícone + texto |
| `Dialog`, `Sheet`/`Drawer`, `Popover`, `DropdownMenu`, `Tooltip`, `HoverCard` | — | Radix: foco preso, ESC, retorno de foco |
| `Tabs`, `Accordion`, `Collapsible` | — | roving tabindex |
| `Table`, `DataTable` | — | `<th scope>`, keyset pagination, virtualização |
| `Toast`/`Sonner` | `success · error · info · warning` | `aria-live` |
| `Avatar`, `Skeleton`, `Separator`, `ScrollArea`, `Breadcrumb`, `Pagination`, `Progress`, `Alert`, `AlertDialog` | — | — |
| `Command` (⌘K) | — | busca/ações |

### 8.2 Compostos de domínio (`features/*/components/`)

| Componente | Feature | Composição / variantes | Estados-chave |
|------------|---------|------------------------|---------------|
| **`CourseCard`** | courses/catalog | `Card`+`Progress`+`Badge`+`Button`; variantes `catalog` (preço/CTA comprar) e `enrolled` (progresso + CTA Começar/Retomar/Ver certificado) — [LEARNING_EXPERIENCE_UX §3.2](../product/LEARNING_EXPERIENCE_UX.md) | loading(skeleton)/empty/error; capa com `aspect-ratio` |
| **`LessonItem`** | courses/player | item da sidebar: ícone de estado (`✓` concluída / em progresso / bloqueada-drip) + título + duração + mini-barra `watched_pct` | locked (drip), completed, active, loading |
| **`VideoPlayer`** | video | MVP: iframe Bunny (HLS); F2: player próprio (Vidstack) com watermark. Controles próprios opcionais | loading(skeleton 16:9)/error(token)/processing(vídeo)/locked(entitlement) |
| **`PriceTag`** | monetization | exibe preço via `Intl.NumberFormat` (centavos→BRL); variantes `single · subscription · free · discount` (riscado + cupom) | — |
| **`QuizQuestion`** | quizzes | single/multiple choice, dissertativa; com/sem gabarito (toggle de autoria) | answered/correct/incorrect/submitting; revisão |
| **`QuizRunner`** | quizzes | orquestra questões + submissão idempotente | in_progress/submitted/graded |
| **`ProgressBar`** | shared/progress | sobre `Progress`; variantes `lesson · module · course`; `aria-live` em mudança | indeterminado (loading) |
| **`LiveBadge`** [F2] | live | "AO VIVO" pulsante + contagem; variantes `live · scheduled · ended` | — |
| **`CertificateCard`** | certificates | baixar PDF / verificar | loading/empty/revoked |
| **`EnrollmentStatusBadge`** | enrollments | mapeia `active/suspended/refunded/expired` ([README §2.3](../product/README.md)) cor+ícone+texto | — |
| **`OrderStatusBadge`** | payments | `pending/paid/refunded/chargeback` | — |
| **`CheckoutSummary` / `PaymentMethodTabs`** | payments | Pix/boleto/cartão (iframe gateway) + cupom | loading/error/recusado/aguardando-pix |
| **`UploadDropzone`** | video/studio | upload TUS resumível ([NFR-REL-08](../product/NON_FUNCTIONAL_REQUIREMENTS.md#6-confiabilidade-e-disponibilidade)) | idle/uploading(%)/paused/error/done |
| **`CourseStructureEditor`** | studio | drag-and-drop de módulos/aulas **com alternativa por teclado** ([NFR-A11Y §3.3](../product/NON_FUNCTIONAL_REQUIREMENTS.md#3-acessibilidade-wcag-21-aa)) | saving/error |
| **`KpiStat`, `RevenueChart`** | analytics | gráfico + **tabela/resumo textual alternativo** ([NFR-A11Y §3.3](../product/NON_FUNCTIONAL_REQUIREMENTS.md#3-acessibilidade-wcag-21-aa)) | loading/empty |
| **`CommentThread`** | community | comentário por aula (UI otimista + rollback) | loading/empty/error/sending |
| **`BrandColorPicker`** | settings | seletor de marca com validação de contraste AA (§3.3) | invalid(bloqueia) |
| **`ImpersonationBanner`** [SA] | platform | banner fixo visível ([NFR-LGPD-16](../product/NON_FUNCTIONAL_REQUIREMENTS.md#5-privacidade--lgpd-na-perspectiva-do-usuário)) | — |
| **`ConsentBanner`** | privacy | granular (Necessários/Analytics/Marketing) com igual proeminência ([NFR-LGPD-01](../product/NON_FUNCTIONAL_REQUIREMENTS.md#5-privacidade--lgpd-na-perspectiva-do-usuário)) | — |
| **`EmptyState`, `ErrorState`, `TenantUnavailable`** | shared | estados reutilizáveis com CTA e branding do tenant ([NFR-REL-09/10](../product/NON_FUNCTIONAL_REQUIREMENTS.md#6-confiabilidade-e-disponibilidade)) | — |
| **`DripLockPanel`** | courses/player | substitui player em aula bloqueada com countdown ([LEARNING_EXPERIENCE_UX §2](../product/LEARNING_EXPERIENCE_UX.md)) | — |

---

## 9. Estados obrigatórios e padrões de a11y por componente

**Estados obrigatórios universais** para componentes interativos e de dados:
`default · hover · focus(-visible) · active · disabled · loading · empty · error`.
(`empty` só onde houver coleção; `loading` via skeleton — nunca spinner que cause CLS.)

| Componente | Estados (além do default) | A11y: foco · ARIA · teclado · contraste |
|------------|---------------------------|------------------------------------------|
| **Button** | hover, **focus-visible (ring 2px, ≥3:1)**, active, disabled (`aria-disabled`, opacidade + cursor), loading (spinner + `aria-busy`, largura preservada, texto "Carregando") | `<button>` real; icon-only exige `aria-label`; `Enter`/`Space`; nunca remover outline sem substituto ([NFR-A11Y-02/04](../product/NON_FUNCTIONAL_REQUIREMENTS.md#3-acessibilidade-wcag-21-aa)) |
| **Input/Form field** | focus, disabled, **error** (`aria-invalid`, mensagem textual via `aria-describedby`, não só borda vermelha) | `<label>` associado; placeholder ≠ label; `autocomplete` no checkout ([NFR-A11Y-04/12](../product/NON_FUNCTIONAL_REQUIREMENTS.md#3-acessibilidade-wcag-21-aa)) |
| **CourseCard** | hover (elevação), focus (card inteiro focável se linkável), loading(skeleton), empty(catálogo vazio + CTA), error | imagem `alt` = título; CTA é link/botão real; progresso com `aria-label` legível |
| **LessonItem** | active(aria-current), completed, **locked** (drip: `aria-disabled` + motivo textual), loading | estado nunca só por cor (ícone+texto); navegável por teclado na lista |
| **VideoPlayer** | loading(skeleton 16:9), error(token, "Tentar novamente"), processing("vídeo sendo processado"), locked(entitlement→CTA) | controles focáveis com `aria-label` e estado anunciado; atalhos (espaço/setas) sem conflito com leitor; **legendas/CC**; sem autoplay com áudio ([NFR-A11Y-14..18](../product/NON_FUNCTIONAL_REQUIREMENTS.md#3-acessibilidade-wcag-21-aa)) |
| **ProgressBar** | indeterminado(loading) | `role="progressbar"` + `aria-valuenow/min/max`; `aria-live="polite"` anuncia "Progresso: X%" ([LEARNING_EXPERIENCE_UX §3.3](../product/LEARNING_EXPERIENCE_UX.md)) |
| **QuizQuestion** | answered, correct, incorrect, disabled(pós-submit), submitting | `radiogroup`/`group` + `fieldset/legend`; feedback textual + ícone, não só cor; erro associado |
| **Badge (status)** | — | cor + **ícone + texto** sempre ([NFR-A11Y-08](../product/NON_FUNCTIONAL_REQUIREMENTS.md#3-acessibilidade-wcag-21-aa)); não é elemento interativo |
| **Dialog/Sheet/Drawer** | open/closed | foco preso, foco inicial no 1º controle, retorno de foco ao gatilho, ESC fecha, scrim `aria-hidden` no fundo; `role="dialog"` + `aria-labelledby` |
| **DropdownMenu/Select/Combobox** | open, highlighted, disabled | setas navegam, `Home/End`, type-ahead, ESC; `aria-expanded`/`aria-activedescendant` |
| **Tabs/Accordion** | selected, disabled | roving tabindex; `aria-selected`/`aria-expanded`; setas |
| **Toast** | success/error/info/warning, auto-dismiss | `aria-live="polite"` (assertive p/ erro); não depender só de cor; pausável no hover/foco |
| **DataTable** | loading(skeleton rows), empty, error, sorting, paginating | `<th scope>`, ordenação anunciada, paginação por teclado, virtualização preserva semântica ([NFR-A11Y §3.3](../product/NON_FUNCTIONAL_REQUIREMENTS.md#3-acessibilidade-wcag-21-aa)) |
| **UploadDropzone** | idle, uploading(%), paused, error(retomável), done | input file acessível + botão; progresso via `aria-live`; instruções textuais |
| **CourseStructureEditor** | dragging, saving, error | **alternativa por teclado** (mover ↑/↓ por botão) além do drag ([NFR-A11Y §3.3](../product/NON_FUNCTIONAL_REQUIREMENTS.md#3-acessibilidade-wcag-21-aa)) |
| **LiveBadge** [F2] | live(pulsa), scheduled, ended | animação respeita `prefers-reduced-motion`; "AO VIVO" textual |

> **UI otimista** (curtir, comentar, marcar progresso) com **rollback** em falha; ações críticas (submeter quiz, pagar) **não** são otimistas — confirmam no servidor ([LEARNING_EXPERIENCE_UX](../product/LEARNING_EXPERIENCE_UX.md)). Botões de mutação desabilitam no clique + idempotency key ([NFR-REL-14](../product/NON_FUNCTIONAL_REQUIREMENTS.md#6-confiabilidade-e-disponibilidade)).

---

## 10. Ícones e ilustração

- **Biblioteca única:** **lucide-react** (combina com shadcn, tree-shakeable, traço consistente). Proibido misturar bibliotecas. Tamanhos por token (16/20/24); `aria-hidden="true"` quando decorativo, `aria-label`/título quando informativo ([NFR-A11Y-05](../product/NON_FUNCTIONAL_REQUIREMENTS.md#3-acessibilidade-wcag-21-aa)).
- **Ícones funcionais** (status, contorno) com contraste ≥ 3:1 ([NFR-A11Y-03](../product/NON_FUNCTIONAL_REQUIREMENTS.md#3-acessibilidade-wcag-21-aa)).
- **Ilustração / empty states:** SVG leve, **sem texto rasterizado**, que aceite `currentColor`/token de marca para herdar o tema do tenant. Carregam abaixo da dobra (lazy). Ilustração nunca carrega informação essencial sem texto de apoio.
- **Imagens de curso/landing:** `next/image` (AVIF/WebP, `srcset`, lazy abaixo da dobra, `priority` no LCP) com `aspect-ratio` reservado ([NFR-PERF-28](../product/NON_FUNCTIONAL_REQUIREMENTS.md#2-performance-e-core-web-vitals)).

---

## 11. Convenções de nomenclatura e organização no código

Alinhado a [ARCHITECTURE §5](../ARCHITECTURE.md) (vertical slices) e [CLAUDE.md](../../CLAUDE.md) (kebab-case arquivos, PascalCase componentes, 1 conceito/arquivo).

```
apps/web/src/
├─ app/                         # rotas (App Router) — só composição
├─ components/ui/               # base shadcn (genérico, SEM regra de negócio)
│  ├─ button.tsx  card.tsx  dialog.tsx  ...
├─ features/<feature>/          # vertical slice de UI (espelha módulos da API)
│  ├─ components/               # compostos de domínio (CourseCard, QuizRunner...)
│  ├─ hooks/                    # useCourseProgress, ... (TanStack Query)
│  └─ lib/
├─ components/layout/           # shells: PublicShell, AppShell, ManageShell...
├─ lib/                         # utils, cn(), formatters (Intl), providers
└─ messages/pt-BR.json          # i18n (next-intl) — ZERO string hardcoded

packages/ui/                    # (opcional) tokens + componentes compartilháveis
├─ tokens/*.json                # FONTE ÚNICA de tokens (W3C) → Figma + código
├─ styles/tokens.generated.css  # gerado por Style Dictionary
└─ a11y/assert-contrast.ts      # utilitário DRY de contraste
```

**Regras de fronteira (lint/Clean Code):**

- `components/ui/*` **não** importa de `features/*` (dependência só para dentro, como o backend).
- Componente de domínio que conhece um tipo de dado consome o **schema Zod de `packages/contracts`** — **não** redefine DTO ([CLAUDE.md DRY](../../CLAUDE.md)).
- **Nomes:** arquivo `kebab-case` (`course-card.tsx`), componente `PascalCase` (`CourseCard`), variantes via **`cva`** com nomes semânticos. Sufixos: `*.tsx` (componente), `*.stories.tsx` (Storybook), `*.test.tsx`.
- **Server vs Client:** RSC por padrão; `"use client"` só onde há interação (player, quiz, forms, dropdowns). Componentes pesados via `dynamic import` ([NFR-PERF-29](../product/NON_FUNCTIONAL_REQUIREMENTS.md#2-performance-e-core-web-vitals)).
- **`cn()`** (clsx + tailwind-merge) para classe condicional; nunca concatenar strings de classe à mão.

---

## 12. Conexão com NFRs (performance/CLS/budget)

| Diretriz do DS | NFR atendido |
|----------------|--------------|
| Skeletons reservam espaço; `aspect-ratio` em imagens/player; `next/font` com fallback métrico | CLS ≤ 0,1 ([NFR-PERF-03](../product/NON_FUNCTIONAL_REQUIREMENTS.md#2-performance-e-core-web-vitals)) |
| Animar só `transform`/`opacity`; `prefers-reduced-motion` | INP ≤ 200ms ([NFR-PERF-02](../product/NON_FUNCTIONAL_REQUIREMENTS.md#2-performance-e-core-web-vitals)), [NFR-A11Y-11](../product/NON_FUNCTIONAL_REQUIREMENTS.md#3-acessibilidade-wcag-21-aa) |
| RSC por padrão, `dynamic import`, base shadcn tree-shakeable, 1 lib de ícones | JS inicial ≤ 170/250KB ([NFR-PERF-06/29](../product/NON_FUNCTIONAL_REQUIREMENTS.md#2-performance-e-core-web-vitals)) |
| Máx. 2 fontes, self-host, `swap`, subsetting | Fontes ≤ 100KB ([NFR-PERF-09](../product/NON_FUNCTIONAL_REQUIREMENTS.md#2-performance-e-core-web-vitals)) |
| Tokens via CSS var (1 arquivo); tema por var (não rebuild) | CSS inicial ≤ 60/80KB ([NFR-PERF-07](../product/NON_FUNCTIONAL_REQUIREMENTS.md#2-performance-e-core-web-vitals)) |
| `next/image` AVIF/WebP, lazy + `priority` LCP | LCP ([NFR-PERF-01/28](../product/NON_FUNCTIONAL_REQUIREMENTS.md#2-performance-e-core-web-vitals)) |
| DataTable: keyset, skeleton, virtualização, debounce server-side | Listas grandes ([NFR-PERF-21..26](../product/NON_FUNCTIONAL_REQUIREMENTS.md#2-performance-e-core-web-vitals)) |
| Skeleton do player 16:9 + estados error/processing | TTF-Play / graceful degradation ([NFR-PERF-11](../product/NON_FUNCTIONAL_REQUIREMENTS.md#2-performance-e-core-web-vitals), [NFR-REL-05](../product/NON_FUNCTIONAL_REQUIREMENTS.md#6-confiabilidade-e-disponibilidade)) |
| Validação de contraste AA no branding | [NFR-A11Y-03/20/21](../product/NON_FUNCTIONAL_REQUIREMENTS.md#3-acessibilidade-wcag-21-aa) |
| 0 string hardcoded (next-intl), `Intl.*` para moeda/data/número | [NFR-I18N-01/07/08/09](../product/NON_FUNCTIONAL_REQUIREMENTS.md#4-internacionalização-i18n-e-localização) |
| Chave de cache de tema/branding inclui `tenantId`+`locale` | Isolamento ([Regra nº1](../../CLAUDE.md)), [NFR-PERF-27](../product/NON_FUNCTIONAL_REQUIREMENTS.md#2-performance-e-core-web-vitals) |

**Gates de qualidade do DS:** `axe-core` (0 violações críticas) + Lighthouse a11y ≥ 95 nas telas-chave T1/T3/T4/T5/T7; Storybook + teste de contraste de paletas; lint de string hardcoded ([NFR §11](../product/NON_FUNCTIONAL_REQUIREMENTS.md#11-como-medimos-e-validamos-gates-de-ci-e-monitoração)).

---

## Dependências e pontos para o coordenador

1. **Subconjunto de tokens sobrescrevíveis pelo tenant (§3.2):** confirmar se o MVP libera **apenas `--color-primary` + `--radius` + logo** (proposta segura) ou também superfícies (`bg/fg`) via paletas pré-aprovadas. Cruza [NFR-A11Y-20/21](../product/NON_FUNCTIONAL_REQUIREMENTS.md#3-acessibilidade-wcag-21-aa) e o editor `/manage/configuracoes/marca` ([IA §2.4](../product/INFORMATION_ARCHITECTURE.md#24-painel-admininstrutor--tenantappcommanage-auth-owneradmininstructor)).
2. **Dark mode no MVP ou F2:** o chassi já o suporta por tokens; decidir se é exposto ao usuário no MVP, e se telas como **player/checkout/certificado** forçam light (fidelidade de cor). Definir armazenamento da preferência (perfil vs cookie).
3. **Fonte por tenant (§4):** confirmar se branding inclui troca de família tipográfica (lista curada self-hosted) no MVP ou F2 — impacta budget de fontes ([NFR-PERF-09](../product/NON_FUNCTIONAL_REQUIREMENTS.md#2-performance-e-core-web-vitals)) e o pipeline de provisionamento.
4. **Pipeline de tokens (Figma ↔ código):** validar adoção de **Style Dictionary + Tokens Studio** sobre JSON W3C como fonte única, e onde mora (`packages/ui`). Requer ADR se virar decisão estrutural ([CLAUDE.md](../../CLAUDE.md)).
5. **`packages/ui` vs `apps/web`:** decidir se os componentes base/tokens ficam num pacote compartilhado (reuso entre `web` e futuros surfaces) ou dentro do app no MVP. Cruza monorepo ([ARCHITECTURE §1/ADR-0007](../ARCHITECTURE.md)).
6. **Algoritmo de contraste:** WCAG 2.x (4,5:1/3:1) é o gate obrigatório; avaliar **APCA** como recomendação adicional. Definir tolerância e a UX de bloqueio vs sugestão no `BrandColorPicker`.
7. **Storybook/Chromatic:** confirmar Storybook como ambiente de catálogo + teste visual/a11y dos estados obrigatórios (§9) e se entra no pipeline de CI.
8. **Componentes [F2] (LiveBadge, watermark, anotações, comunidade, gamificação):** especificados aqui em alto nível; spec visual detalhada (ex.: watermark × legibilidade — [NFR-A11Y-19 vs NFR-SEC-12](../product/NON_FUNCTIONAL_REQUIREMENTS.md)) fica para a fase F2 com UX+a11y juntos.
9. **Página "Tenant indisponível/suspenso" (§8 `TenantUnavailable`):** confirmar copy e o que o owner vê (billing) vs demais (bloqueio) — cruza [IA dependência #8](../product/INFORMATION_ARCHITECTURE.md#dependências-e-pontos-para-o-coordenador).
10. **Conjunto mínimo de telas-chave para gates de a11y/perf:** confirmar T1/T3/T4/T5/T7 ([NFR §9 dependência](../product/NON_FUNCTIONAL_REQUIREMENTS.md#12-dependências-e-pontos-para-o-coordenador)) como escopo auditado automaticamente, para dimensionar o esforço de Storybook/axe.
