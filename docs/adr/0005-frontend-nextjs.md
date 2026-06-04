# ADR-0005 — Frontend: Next.js App Router + RSC

- **Status:** Aceito · **Data:** 2026-06-04 · **Confiança:** 90%

## Contexto
Frontend multitenant (subdomínio por tenant), com áreas públicas (landing/catálogo) e área logada
(player, progresso). Preferência do stakeholder: **Next.js**. Stack previsível para IA.

## Decisão
**Next.js com App Router + React Server Components.**
- **Rendering:** público → SSG/ISR; área logada → RSC + SSR; interativo (player, quiz) → Client Components.
- **Multitenancy:** `middleware.ts` resolve tenant por subdomínio; branding via CSS variables.
- **Data fetching:** TanStack Query (client) + `fetch` em RSC (carga inicial).
- **Estado:** Zustand só para o pouco estado global de UI; resto é TanStack Query + `useState`.
- **UI:** shadcn/ui + Tailwind (componentes no repo → fáceis de a IA customizar).
- **i18n:** next-intl. **Auth:** cliente Better-Auth (mesma lib do backend → coerência).

## Justificativa
- App Router/RSC é o padrão atual que a IA assume; reduz JS no cliente e melhora performance.
- shadcn/ui + Tailwind é o padrão dominante e previsível; Zustand evita o peso do Redus.
- Casamento natural com deploy na Vercel.

## Consequências
- Pages Router é considerado legado (não usar).
- Estrutura: `app/` (rotas) + `features/` (vertical slices) + `components/ui` + `lib/`.

## Fontes
Relatório de arquitetura (frontend).
