# Onboarding & Ativação — First-Run Experience

- **Versão:** 1.0 · **Data:** 2026-06-07
- **Autor:** Product / Growth (onboarding & activation)
- **Status:** Proposta para revisão do coordenador de produto
- **Relacionados:** [USER_JOURNEYS](USER_JOURNEYS.md) · [INFORMATION_ARCHITECTURE](INFORMATION_ARCHITECTURE.md) · [MONETIZATION](MONETIZATION.md) · [AUTHORING_UX](AUTHORING_UX.md) · [LEARNING_EXPERIENCE_UX](LEARNING_EXPERIENCE_UX.md) · [NOTIFICATIONS_MATRIX](NOTIFICATIONS_MATRIX.md) · [ANALYTICS_AND_DASHBOARDS](ANALYTICS_AND_DASHBOARDS.md) · [BUSINESS_RULES_AND_STATES](BUSINESS_RULES_AND_STATES.md) · [BRANDING_WHITELABEL](../design/BRANDING_WHITELABEL.md) · [README de produto](README.md)

> **O que este documento é:** a especificação da **first-run experience** (primeira execução) que leva cada
> persona do "primeiro login" ao seu **Aha moment** e à **ativação**. Orquestra features e telas **já
> previstas** (wizard de onboarding [MVP] da IA §2.4, checklist gamificado de ativação dos USER_JOURNEYS,
> player "continue de onde parou"…). **Não inventa features novas.**
>
> **Foco:** (A) onboarding do **Admin do Tenant** (ativação B2B) e (B) onboarding do **Aluno** (ativação de
> aprendizado, ligada à North Star). Instrutor/Afiliado entram como **suporte** ao funil do Admin (§5).
>
> **Filtro de qualidade (CLAUDE.md):** todo estado de progresso de onboarding é **dado de tenant** →
> `withTenant`. Eventos de analytics carregam `tenant_id`. Nada vaza entre tenants.

---

## Índice

1. [Princípios de ativação e North Star](#1-princípios-de-ativação-e-north-star)
2. [Definições de ativação (alinhadas ao PRD)](#2-definições-de-ativação-alinhadas-ao-prd)
3. [A) Onboarding do Admin do Tenant (B2B)](#3-a-onboarding-do-admin-do-tenant-b2b)
4. [B) Onboarding do Aluno (ativação de aprendizado)](#4-b-onboarding-do-aluno-ativação-de-aprendizado)
5. [Onboarding de Instrutor e Afiliado (suporte ao funil)](#5-onboarding-de-instrutor-e-afiliado-suporte-ao-funil)
6. [Empty states por tela-chave](#6-empty-states-por-tela-chave)
7. [Aha moments e celebrações](#7-aha-moments-e-celebrações)
8. [Notificações de onboarding (e-mail / in-app)](#8-notificações-de-onboarding-e-mail--in-app)
9. [Métricas de ativação e instrumentação](#9-métricas-de-ativação-e-instrumentação)
10. [Dependências e pontos para o coordenador](#dependências-e-pontos-para-o-coordenador)

---

## 1. Princípios de ativação e North Star

- **North Star (PRD §6):** **horas assistidas por aluno ativo/mês**. A first-run do **aluno** existe para
  acelerar a 1ª aula concluída (entrada na North Star); a first-run do **admin** existe para que **haja
  alunos e conteúdo** para gerar essas horas. Os dois loops se alimentam (USER_JOURNEYS §9).
- **Time-to-value mínimo:** cada persona deve chegar ao seu valor central no menor número de passos. O
  produto já semeia um **curso de exemplo** e um **admin** no provisionamento (USER_JOURNEYS §3 Fase A) —
  usamos isso como atalho, não como obstáculo.
- **Empty states que ensinam, não que culpam:** toda tela-chave vazia vira um mini-onboarding com **um único
  próximo passo claro** (princípio já presente em AUTHORING_UX §4/§11).
- **Progresso visível e reversível:** checklist com progresso; nada bloqueia o usuário de explorar; itens
  podem ser concluídos fora de ordem.
- **Celebrar os marcos certos:** confete/celebração reservados aos **Aha moments reais** (1ª venda, 1ª aula
  concluída, certificado), não a cada clique — para que a celebração mantenha significado (USER_JOURNEYS §4/§7).
- **Dispensável e retomável:** o usuário pode dispensar o guia e retomá-lo; o checklist persiste seu estado.

---

## 2. Definições de ativação (alinhadas ao PRD)

| Persona | Ativação (definição canônica) | Aha moment | Fonte |
|---------|-------------------------------|------------|-------|
| **Admin do Tenant** | Publicar **≥1 curso** **e** realizar **≥1 venda** nos primeiros **14 dias** | "Escola no ar com a própria marca" (Aha #1) → **1ª venda** (Aha #2, ativação plena) | PRD §6 / USER_JOURNEYS §4 Fase B |
| **Aluno** | **Concluir a 1ª aula** (≥90% + anti-seek) | "Primeira aula concluída"; valor pleno = **certificado** | USER_JOURNEYS §7 Fase C/E |
| **Instrutor** | **1ª aula em vídeo publicada** (upload TUS ok) | "Vídeo pronto e publicado, como o aluno verá" | USER_JOURNEYS §5 Fase B |
| **Afiliado** | **1º link gerado + 1ª comissão atribuída** | "Primeira comissão atribuída corretamente" | USER_JOURNEYS §6 Fase B |

> Estas definições **não são inventadas aqui** — vêm da matriz de momentos críticos (USER_JOURNEYS §8) e do
> PRD §6. Este doc apenas orquestra a UX que as realiza.

---

## 3. A) Onboarding do Admin do Tenant (B2B)

A fase **mais importante da retenção B2B** (USER_JOURNEYS §4 Fase B). Vive em
`tenant.app.com/manage/onboarding` (**Wizard de onboarding [MVP]** — [IA §2.4](INFORMATION_ARCHITECTURE.md))
e num **checklist de ativação** persistente no Dashboard `/manage`.

### 3.1 Ponto de entrada

- Após assinatura aprovada (Stripe) → provisionamento → smoke test verde → e-mail de boas-vindas com link do
  subdomínio (USER_JOURNEYS §4 Fase A). **Primeiro login** com credenciais semeadas → cai no
  **wizard de onboarding** (não no dashboard cru).

### 3.2 O checklist de ativação (gamificado) — "X de 5 para vender"

Reusa o **curso de exemplo** semeado como ponto de partida (o admin pode editar em vez de criar do zero).
Itens (cada um leva à tela real correspondente, em vez de uma versão paralela):

| # | Passo | Leva para (IA) | Conclusão detectada por | Fase |
|---|-------|----------------|--------------------------|------|
| 1 | **Configurar a marca** (logo + cores) | `/manage/configuracoes/marca` ([BRANDING_WHITELABEL](../design/BRANDING_WHITELABEL.md)) | `branding_logo_uploaded` ou cores salvas | MVP |
| 2 | **Criar/ajustar o 1º curso** (a partir do exemplo ou novo) | `/manage/cursos` → editor ([AUTHORING_UX](AUTHORING_UX.md)) | `course_created` / curso editado | MVP |
| 3 | **Subir a 1ª aula** (vídeo TUS / texto / PDF) e **publicar o curso** | editor de aula → checklist de publicação (AUTHORING_UX §10) | `video_upload_completed` + `course_published` | MVP |
| 4 | **Ativar pagamentos** (conectar recipient Pagar.me — KYC) | `/manage/configuracoes/pagamentos` | recipient válido (MONETIZATION §B.7) | MVP |
| 5 | **Publicar a página de vendas** e **fazer a 1ª venda** | `/manage/cursos/{id}/pagina-de-vendas` → checkout | `course_published` da landing → 1ª `order=paid` | MVP |

- **Progresso visível** (ex.: barra "3 de 5"); itens concluídos com check; ordem **flexível** (pode
  configurar marca por último).
- **"Ativar pagamentos" é o gargalo conhecido** (fricção de KYC — USER_JOURNEYS §4 Fase B): destacado com
  copy que explica o tempo de aprovação e que **não bloqueia** publicar conteúdo (só bloqueia *vender* —
  MONETIZATION §B.7/§6 / [README §4 #26](README.md)). Sem recipient → checkout indisponível, com aviso claro.
- **Item 5 (1ª venda)** é o único que o admin não conclui sozinho com um clique — depende de divulgar. O
  checklist oferece atalhos: "Copiar link da landing", "Criar cupom de lançamento", "Convidar 1º aluno
  (matrícula manual/CSV)".

### 3.3 Sub-fluxos guiados (dentro das telas reais)

- **Marca:** preview ao vivo + validação de contraste AA (BRANDING_WHITELABEL §5/§6) → **Aha #1** quando o
  admin vê a escola com a própria marca.
- **1º curso/aula:** usa o editor real com autosave e "ver como aluno" (AUTHORING_UX §4/§5/§10). Empty state
  do editor: "Nenhum módulo ainda — comece criando o Módulo 1" (§6).
- **Publicação:** **checklist de publicação** já existente (curso tem ≥1 aula publicável, vídeos `ready`,
  preço definido, capa+descrição — AUTHORING_UX §10) reaproveitado como parte do onboarding (DRY).

### 3.4 Aha moments do Admin

- **Aha #1:** escola no ar com a própria marca (após item 1) → celebração leve + CTA "ver minha escola".
- **Aha #2 (ativação plena):** **1ª venda** caindo + aluno matriculado automaticamente → **confete in-app**
  + e-mail "Primeira venda!" (USER_JOURNEYS §4 Fase B; um único evento de compra dispara 3 touchpoints —
  USER_JOURNEYS §9).

### 3.5 Pós-checklist

- Concluídos os 5 → o card vira um **resumo de próximos passos de crescimento** (convidar instrutor/afiliado,
  cupom, pixels) — sem inventar features, só apontando o que já existe na IA. O dashboard `/manage` assume o
  protagonismo (KPIs reais).

---

## 4. B) Onboarding do Aluno (ativação de aprendizado)

Persona de maior volume e **ligada diretamente à North Star** (USER_JOURNEYS §7). Vive na **Área do aluno**
`/app` (IA §2.3) e no **player** (LEARNING_EXPERIENCE_UX).

### 4.1 Primeiro acesso (pós-compra ou matrícula)

- **Gatilho:** compra aprovada (webhook `paid`) → matrícula `active` → e-mail de boas-vindas com **link da 1ª
  aula** (USER_JOURNEYS §7 Fase B/C; NOTIFICATIONS_MATRIX). Para `free`/matrícula manual: e-mail "comece por
  aqui".
- **Primeiro login em `/app`** → "Meus cursos". Se há **um único curso recém-comprado**, CTA primário e claro:
  **"Começar agora"** levando direto à 1ª aula (reduz "não sei por onde começar" — USER_JOURNEYS §7 Fase C).
- **Player na 1ª aula:** orientação mínima/contextual (sem tour pesado). A "primeira aula curta de
  boas-vindas" é uma **boa prática editorial** sugerida ao instrutor (não uma feature nova).

### 4.2 Retomar ("continue de onde parou")

- **Continue de onde parou** já é feature do player (HLS adaptativo, retomar via heartbeats —
  USER_JOURNEYS §7 Fase C; LEARNING_EXPERIENCE_UX). No retorno, "Meus cursos" mostra **card de retomada** com
  a próxima aula e a % de progresso → 1 clique para continuar.
- **Drip:** se a próxima aula está bloqueada por liberação programada, o estado **bloqueado** explica
  "libera em D+N" (AUTHORING_UX §6 / LEARNING_EXPERIENCE_UX).

### 4.3 Primeira aula concluída (ativação)

- **Ativação do aluno = concluir a 1ª aula** (≥90% + anti-seek — glossário README §2.4). Ao cruzar o
  limiar: marca `lesson_progress=completed`, atualiza % do curso e **celebra o marco** (microcelebração) +
  aponta a próxima aula (manter ritmo).
- Alimenta a **North Star** (horas assistidas via heartbeats/`lesson_progress`).

### 4.4 Continuidade até o Aha de valor

- **Quiz** (correção automática), **comentários por aula**, **progresso %** já previstos (IA §2.3).
- **Conclusão 100% → certificado** gerado automaticamente (PDF + QR de verificação) = **Aha de valor** do
  aluno (USER_JOURNEYS §7 Fase E) → tela de celebração + compartilhamento (marketing orgânico para o tenant).

---

## 5. Onboarding de Instrutor e Afiliado (suporte ao funil)

Entram como **suporte** ao funil de ativação do tenant (não são o foco solicitado, mas orquestram o loop de
receita — USER_JOURNEYS §9). Reusam padrões já especificados:

- **Instrutor:** convite por e-mail → primeiro login no **Estúdio** → vê o **curso de exemplo** → tour curto
  contextual → cria estrutura Curso→Módulo→Aula → **1º upload TUS** (ativação) (USER_JOURNEYS §5 Fase A/B;
  AUTHORING_UX §4/§5). Empty state do Estúdio: "Nenhum curso ainda — comece pelo exemplo ou crie do zero".
- **Afiliado:** convite/aprovação → **Painel do Afiliado** → **gerar 1º link** (+UTM) → baixar materiais →
  1ª comissão atribuída (ativação) (USER_JOURNEYS §6 Fase A/B; IA §2.5). Onboarding mostra **termos de
  comissão** (%, prazo de clearance) já no primeiro acesso.

---

## 6. Empty states por tela-chave

Cada empty state = **microcopy + ilustração + 1 CTA primário** que leva à ação. Espelha o princípio de
AUTHORING_UX §11.

| Tela (IA) | Persona | Empty state (copy + CTA) |
|-----------|---------|--------------------------|
| `/manage` (Dashboard) | Admin/Owner | "Sua escola está pronta. Faça **3 de 5** para começar a vender." → **abre checklist** (§3.2) |
| `/manage/cursos` | Admin/Instrutor | "Nenhum curso publicado ainda. Comece pelo **curso de exemplo** ou **crie o primeiro**." → editar exemplo / Criar curso |
| Editor de currículo | Admin/Instrutor | "Nenhum módulo ainda — comece criando o **Módulo 1**." (AUTHORING_UX §4) → Adicionar módulo |
| Editor de aula (vídeo) | Instrutor | "Arraste seu vídeo aqui ou **selecione um arquivo**." (upload TUS resiliente) → Upload |
| `/manage/configuracoes/marca` | Admin/Owner | "Envie seu logo para personalizar sua escola." (BRANDING_WHITELABEL §10) → Upload logo |
| `/manage/configuracoes/pagamentos` | Owner | "Ative os pagamentos para começar a vender (Pix/cartão/boleto)." → Conectar conta (KYC) |
| `/manage/afiliados` | Admin/Owner | "Convide afiliados para vender por você." → Convidar afiliado |
| `/manage/alunos` | Admin/Owner | "Nenhum aluno ainda. **Matricule manualmente** ou **divulgue sua página de vendas**." → Matricular / Copiar link |
| `/app` (Meus cursos) | Aluno | "Você ainda não tem cursos. **Explore o catálogo**." → Catálogo (ou, recém-comprado: **"Começar agora"**) |
| Player (aula bloqueada por drip) | Aluno | "Esta aula libera em **D+N**." (estado bloqueado, AUTHORING_UX §6) |
| `/app/certificados` | Aluno | "Conclua um curso para ganhar seu certificado." → Continuar curso |
| `/affiliate` (Painel) | Afiliado | "Gere seu primeiro link para começar a divulgar." → Gerar link |
| `/affiliate/comissoes` | Afiliado | "Nenhuma comissão ainda. Divulgue seus links." → Ir para links |

---

## 7. Aha moments e celebrações

| Persona | Microcelebração | Celebração plena (confete/marco) |
|---------|-----------------|----------------------------------|
| Admin | Marca publicada (Aha #1) → "Sua escola está no ar" | **1ª venda** (Aha #2) → confete in-app + e-mail "Primeira venda!" |
| Aluno | 1ª aula concluída → marco de progresso | **Certificado emitido** → tela de celebração + compartilhar (LinkedIn) |
| Instrutor | Upload concluído → "vídeo processando" | **Vídeo pronto e publicado** → preview "como o aluno verá" |
| Afiliado | 1º link gerado | **1ª comissão atribuída** → notificação "Você fez uma venda!" |

- **Regra de parcimônia:** confete só nos marcos plenos (coluna direita). Microcelebrações são discretas
  (toast/badge). Evita fadiga de celebração e mantém o sinal forte (§1).
- **Cross-persona:** a 1ª venda do aluno dispara **3 touchpoints** (aluno: acesso; afiliado: comissão;
  admin: venda) — um evento, três celebrações (USER_JOURNEYS §9). Idempotência via `payment_events`
  (não duplicar — NOTIFICATIONS_MATRIX).

---

## 8. Notificações de onboarding (e-mail / in-app)

Subconjunto de [NOTIFICATIONS_MATRIX](NOTIFICATIONS_MATRIX.md) (fonte de verdade; push = F2). Disparadas por
**transições idempotentes**.

| Evento | E-mail | In-app | Destinatário | Gatilho |
|--------|--------|--------|--------------|---------|
| Boas-vindas ao tenant | ✅ | ✅ (checklist) | Admin/Owner | Provisionamento `active` |
| "Seu curso foi publicado" | ✅ | ✅ | Admin/Instrutor | `course_published` |
| "Primeira venda!" | ✅ | ✅ (confete) | Admin/Owner | 1ª `order=paid` do tenant |
| Boas-vindas + link da 1ª aula | ✅ | ✅ | Aluno | Matrícula `active` |
| "Vídeo pronto para publicar" | ✅ | ✅ | Instrutor | `video_status=ready` |
| Convite de equipe/afiliado | ✅ | — | Instrutor/Afiliado | Convite enviado |
| Lembrete de retomada | ✅ | ✅ | Aluno | Inatividade após início (F2: push) |
| "Parabéns, seu certificado" | ✅ | ✅ (celebração) | Aluno | Conclusão 100% |

> **Branding dos e-mails de onboarding** usa logo/cor do tenant (BRANDING_WHITELABEL §7). No MVP o `From` é
> da plataforma; remetente próprio é F2.

---

## 9. Métricas de ativação e instrumentação

Alinhadas ao PRD §6 e ao [ANALYTICS_AND_DASHBOARDS](ANALYTICS_AND_DASHBOARDS.md) (PostHog, todos os eventos
com `tenant_id`).

### 9.1 North Star e ativação

| Métrica | Definição | Persona | Fonte |
|---------|-----------|---------|-------|
| **North Star** | Horas assistidas por aluno ativo/mês | Aluno | heartbeats / `lesson_progress` |
| **Ativação B2B** | % tenants com ≥1 curso publicado **e** ≥1 venda em **14 dias** | Admin | `course_published` + `order=paid` |
| **Tempo até 1º curso publicado** | t(provisionamento active → `course_published`) | Admin | timestamps |
| **Tempo até 1ª venda** | t(provisionamento active → 1ª `order=paid`) | Admin | timestamps |
| **Ativação do aluno** | % de alunos que concluem a **1ª aula** | Aluno | `lesson_progress=completed` (1ª) |
| **Taxa de conclusão de curso** | % de matrículas que chegam a 100% | Aluno | enrollments/progresso |
| **Ativação do instrutor** | % de instrutores com 1ª aula em vídeo publicada | Instrutor | `video_upload_completed`+`lesson published` |
| **Ativação do afiliado** | % com 1º link + 1ª comissão | Afiliado | link gerado + `affiliate_commissions` |

### 9.2 Eventos de funil de onboarding (instrumentar)

`onboarding_started`, `onboarding_step_completed` (com `step`), `onboarding_checklist_completed`,
`onboarding_dismissed`, `branding_published`, `payments_activated`, `first_sale`, `student_first_login`,
`student_first_lesson_started`, `student_first_lesson_completed`, `certificate_issued`.
(Reaproveita eventos já catalogados em AUTHORING_UX §11 e ANALYTICS_AND_DASHBOARDS quando existirem; novos
eventos seguem o tracking plan com schemas Zod em `packages/contracts` — ADR-0014.)

### 9.3 Funis de análise sugeridos
- **Funil de ativação B2B:** primeiro login → marca → curso criado → aula publicada → pagamentos ativados →
  1ª venda. Identifica o **maior drop-off** (hipótese: "ativar pagamentos"/KYC — §3.2).
- **Funil de ativação do aluno:** matrícula → 1º login → 1ª aula iniciada → 1ª aula concluída → conclusão →
  certificado.

---

## Dependências e pontos para o coordenador

1. **Persistência do estado do checklist (§3.2):** onde guardar o progresso de onboarding do admin? Proposta:
   derivar de eventos/estado real (curso publicado? recipient ativo? 1ª venda?) **sem nova tabela**, OU uma
   coluna `onboarding_state jsonb` em `tenant_settings` ([DATA_MODEL §6.10](../DATA_MODEL.md)) para itens
   dispensados/ordem. **Requer decisão** (preferência: derivado, com `withTenant`).
2. **Detecção de conclusão dos itens (§3.2):** confirmar os eventos/queries que marcam cada passo como
   concluído (ex.: "ativar pagamentos" = recipient Pagar.me válido — cruza [README §4 #26](README.md)).
3. **Wizard vs. checklist (§3):** o `/manage/onboarding` (IA §2.4) é um **wizard linear** ou o checklist
   persistente no dashboard? Recomendação: wizard leve no 1º acesso **+** checklist persistente depois
   (mesma fonte de estado). Confirmar com design de telas.
4. **Item "1ª venda" depende de ação externa (§3.2):** não é concluível só no produto (precisa divulgar).
   Confirmar copy/atalhos (cupom de lançamento, matrícula manual como "venda assistida"?) e se matrícula
   manual conta como ativação ou só `order=paid` (alinhar definição PRD §6).
5. **Tour do player do aluno (§4.1):** decidir o peso (orientação contextual mínima vs. tour). Recomendação:
   mínimo, para não atrasar a 1ª aula. A "aula de boas-vindas curta" é orientação **editorial** ao instrutor,
   não feature.
6. **Lembretes de retomada (§8):** parâmetros (após quantos dias de inatividade) cruzam com
   [NOTIFICATIONS_MATRIX](NOTIFICATIONS_MATRIX.md) (janela/anti-ruído) — push é F2.
7. **Novos eventos de analytics (§9.2):** vários eventos de onboarding podem não existir no tracking plan
   atual — confirmar/adicionar no [ANALYTICS_AND_DASHBOARDS](ANALYTICS_AND_DASHBOARDS.md) com schemas Zod
   (ADR-0014). Garantir `tenant_id` em todos (Regra nº1).
8. **Celebração/confete (§7):** confirmar biblioteca/componente e acessibilidade (respeitar
   `prefers-reduced-motion` — NFR a11y).
9. **Catálogo público opcional (§6):** o empty state de `/app` aponta para o catálogo; mas o catálogo público
   pode ser desligável por tenant ([IA Dep. #9](INFORMATION_ARCHITECTURE.md)). Ajustar o CTA conforme
   `public_catalog_enabled` ([DATA_MODEL §6.10](../DATA_MODEL.md)).
```
