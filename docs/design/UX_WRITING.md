# UX Writing & Content Design — Tom de Voz e Catálogo de Microcopy

- **Versão:** 1.0 · **Data:** 2026-06-07
- **Autor:** UX Writing / Content Design
- **Status:** Proposta para revisão do coordenador de produto
- **Documentos relacionados:** [NOTIFICATIONS_MATRIX.md](../product/NOTIFICATIONS_MATRIX.md) · [USER_FLOWS.md](../product/USER_FLOWS.md) · [LEARNING_EXPERIENCE_UX.md](../product/LEARNING_EXPERIENCE_UX.md) · [NON_FUNCTIONAL_REQUIREMENTS.md](../product/NON_FUNCTIONAL_REQUIREMENTS.md) · [README.md (glossário)](../product/README.md) · [EMAIL_TEMPLATES.md](../product/EMAIL_TEMPLATES.md) · [CLAUDE.md](../../CLAUDE.md)

> **Escopo.** Este documento é a **fonte de verdade de conteúdo** (UX writing) da plataforma: tom de voz,
> princípios de escrita PT-BR, convenções de i18n (chaves, ICU, sem strings hardcoded) e um **catálogo
> canônico de microcopy** (títulos, CTAs, erros, empty states, sucesso/confirmação e bloqueios).
> Complementa — não substitui — os requisitos de a11y/i18n dos [NFRs §3/§4](../product/NON_FUNCTIONAL_REQUIREMENTS.md)
> e os textos de e-mail em [EMAIL_TEMPLATES.md](../product/EMAIL_TEMPLATES.md). **Regra nº1 (isolamento de
> tenant):** nenhum texto pode revelar a existência de outros tenants nem dados cross-tenant.

---

## Índice

1. [Tom de voz e princípios de escrita](#1-tom-de-voz-e-princípios-de-escrita)
2. [White-label: neutralidade à marca do tenant](#2-white-label-neutralidade-à-marca-do-tenant)
3. [Acessibilidade do texto e linguagem inclusiva](#3-acessibilidade-do-texto-e-linguagem-inclusiva)
4. [Convenções de i18n (chaves, ICU, formatação)](#4-convenções-de-i18n-chaves-icu-formatação)
5. [Catálogo de microcopy — títulos e CTAs](#5-catálogo-de-microcopy--títulos-e-ctas)
6. [Catálogo de mensagens de erro](#6-catálogo-de-mensagens-de-erro)
7. [Catálogo de empty states](#7-catálogo-de-empty-states)
8. [Estados de sucesso e confirmação](#8-estados-de-sucesso-e-confirmação)
9. [Mensagens de bloqueio (drip, acesso, quota)](#9-mensagens-de-bloqueio-drip-acesso-quota)
10. [Glossário de termos para o usuário final](#10-glossário-de-termos-para-o-usuário-final)
11. [Dependências e pontos para o coordenador](#dependências-e-pontos-para-o-coordenador)

---

## 1. Tom de voz e princípios de escrita

A plataforma é **white-label**: o usuário final (aluno) percebe a marca do **tenant** (a escola), não a
nossa. Por isso o tom é **neutro à marca**, mas sempre **claro, acolhedor e humano** em PT-BR.

### 1.1 Atributos de voz

| Atributo | Somos | Não somos |
|----------|-------|-----------|
| **Claro** | Direto, frases curtas, uma ideia por frase. | Prolixos, jargão técnico, ambíguos. |
| **Acolhedor** | Empáticos, encorajadores, do lado do aluno. | Bajuladores, infantis, falsos-amigos. |
| **Confiável** | Honestos sobre o que aconteceu e o próximo passo. | Vagos, evasivos, alarmistas. |
| **Respeitoso** | Tratamos o tempo e a inteligência do usuário. | Condescendentes, culpabilizadores. |
| **Neutro à marca** | Funcionais; deixamos a personalidade vir do tenant. | "Divertidões" que conflitam com a marca da escola. |

### 1.2 Princípios (não-negociáveis)

1. **Pessoa e tratamento:** sempre **2ª pessoa "você"** (nunca "tu" nem "vós"). Voz **ativa**. Evite
   imperativos secos isolados; prefira convidar ("Comece agora", não apenas "Iniciar").
2. **Clareza acima de esperteza:** sem trocadilhos que não traduzam, sem gírias regionais, sem humor que
   envelhece. Microcopy não é lugar de piada.
3. **Conciso:** títulos ≤ ~6 palavras; botões 1–3 palavras; mensagens de erro 1–2 frases. Corte
   "por favor", "simplesmente", "apenas", redundâncias.
4. **Orientado à ação e ao próximo passo:** todo estado de erro/bloqueio diz **o que aconteceu** e **o
   que fazer agora**. Nunca um beco sem saída.
5. **Nunca culpe o usuário:** "Não foi possível salvar" (não "Você preencheu errado"). O sistema assume a
   responsabilidade ([NFR-REL-16](../product/NON_FUNCTIONAL_REQUIREMENTS.md)).
6. **Honestidade sobre o sistema:** não prometa o que não controlamos (ex.: tempo exato de encoding de
   vídeo, prazo de compensação de boleto). Use faixas ("alguns minutos", "1 a 2 dias úteis").
7. **Sem ansiedade desnecessária:** evite caixa-alta gritada, excesso de exclamação e vermelho onde não
   há erro real. Reserve a urgência para o que é de fato urgente (pagamento, segurança).
8. **Consistência:** o **mesmo conceito usa sempre a mesma palavra** (ver §10). Não alterne
   "matrícula/inscrição", "aula/lição", "certificado/diploma".

### 1.3 Mecânica de escrita PT-BR

- **Capitalização:** **sentence case** em títulos, rótulos e botões ("Marcar como concluída", não
  "Marcar Como Concluída"). Nomes próprios e siglas mantêm-se (Pix, PDF, CPF).
- **Pontuação:** frases completas com ponto final; títulos e botões **sem** ponto final. Reticências só
  para ação contínua ("Carregando…", "Gerando seu certificado…") — use o caractere `…`.
- **Números/moeda/data:** sempre via `Intl` (ver §4.2). Moeda em BRL: `R$ 1.234,56`. Data: `dd/mm/aaaa`.
  Horas no fuso do usuário (default `America/São_Paulo`).
- **Emoji:** uso **parcimonioso** e apenas em momentos de celebração/positividade (conclusão, certificado:
  🎉 🏅). **Nunca** em erros, segurança ou pagamento. Emoji **nunca** carrega significado sozinho (a11y —
  ver §3). Não usar emoji em e-mails de assunto P0 (pagamento/segurança) para não prejudicar entregabilidade.
- **Abreviações:** evite ("config." → "configurações"). Exceções consagradas: "Pix", "CPF", "PDF".

---

## 2. White-label: neutralidade à marca do tenant

O conteúdo da plataforma é renderizado **dentro da identidade do tenant**. Diretrizes:

- **Nunca** cite o nome da nossa empresa/plataforma para o aluno. Onde for preciso nomear a escola, use a
  variável `{tenant_name}`, nunca um nome fixo.
- **Não assuma o segmento** do tenant. Use termos genéricos de educação ("curso", "aula", "instrutor",
  "aluno") que funcionam para qualquer nicho. Evite vocabulário que pressuponha público específico.
- **Personalidade vem do branding** (logo, cor, tipografia via CSS variables — [LEARNING_EXPERIENCE_UX §1](../product/LEARNING_EXPERIENCE_UX.md)),
  não do texto. O copy é o "esqueleto neutro"; a marca veste.
- **Suporte:** sempre direcione ao suporte **do tenant** (`{support_email}`), nunca ao nosso.
- **Painel admin/estúdio/super-admin:** o tom pode ser ligeiramente mais **operacional/objetivo** (público
  profissional), mantendo clareza e ausência de culpa. O **super-admin (control plane)** é interface
  interna nossa e pode citar a plataforma.

---

## 3. Acessibilidade do texto e linguagem inclusiva

Alinhado a [NFR-A11Y-04/07/08/12](../product/NON_FUNCTIONAL_REQUIREMENTS.md) (WCAG 2.1 AA).

### 3.1 Linguagem inclusiva

- **Neutralidade de gênero quando viável:** prefira construções neutras ("Boas-vindas!" em vez de
  "Bem-vindo/Bem-vinda"; "Olá, {nome}" em vez de saudação com gênero; "pessoa instrutora" só se soar
  natural — caso contrário use o papel "instrutor" como termo técnico neutro do sistema). **Não** use
  "x"/"@"/"e" como desinência (prejudica leitores de tela e legibilidade). Quando o gênero do usuário for
  conhecido e necessário, respeite-o.
- **Sem capacitismo / termos pejorativos:** evite "burro", "deficiente" como adjetivo genérico, "louco".
- **Tratamento etário/cultural neutro:** sem suposições sobre idade, religião, classe.

### 3.2 Texto a serviço da a11y (regras para o conteúdo de componentes)

- **`aria-label` em ícones-botão:** todo botão só-ícone tem rótulo textual descritivo da **ação**
  (ex.: ícone ♥ → `aria-label="Curtir comentário"`; ícone ⋯ → `aria-label="Mais ações do comentário"`;
  ícone ⛶ → `aria-label="Tela cheia"`). Catálogo de aria-labels em §5.4.
- **`aria-live` para mudanças de estado:** anuncie transições importantes em texto: "Aula concluída",
  "Progresso do curso: 50%", "Comentário publicado", "Salvando…/Salvo", "Falha ao enviar — tente
  novamente". Use `polite` para confirmações e `assertive` apenas para erros que exigem ação imediata.
- **Erros associados ao campo:** mensagens de validação são **textuais e específicas**, vinculadas via
  `aria-describedby`, nunca só borda vermelha ([NFR-A11Y-12](../product/NON_FUNCTIONAL_REQUIREMENTS.md)).
- **Não dependa só de cor:** todo status tem **texto + ícone** além da cor (ex.: "Válido ✓" em verde, mas
  o "✓" e a palavra carregam o significado).
- **Alt text:** imagens informativas têm `alt` descritivo; decorativas têm `alt=""`. Logo do tenant:
  `alt="{tenant_name}"`.
- **Texto de link descritivo:** evite "clique aqui"; descreva o destino ("Ver meus certificados").
- **`lang`:** o `<html lang>` acompanha o locale (§4); trechos em outro idioma recebem `lang` próprio.

### 3.3 Microcopy de a11y reutilizável (chaves)

| Chave i18n | Texto PT-BR | Uso |
|------------|-------------|-----|
| `a11y.skipToContent` | Pular para o conteúdo | Skip link ([NFR-A11Y-06]) |
| `a11y.loading` | Carregando conteúdo | Região `aria-busy` |
| `a11y.menuOpen` | Abrir menu | Toggle de navegação |
| `a11y.menuClose` | Fechar menu | Toggle de navegação |
| `a11y.closeDialog` | Fechar | Botão de fechar modal |
| `a11y.required` | Obrigatório | Sufixo de campo obrigatório (além do `*`) |

---

## 4. Convenções de i18n (chaves, ICU, formatação)

Base: [NFR-I18N-01..10](../product/NON_FUNCTIONAL_REQUIREMENTS.md), **next-intl**, lançamento PT-BR com
arquitetura pronta para ES/EN (F2).

### 4.1 Estrutura de chaves

- **Zero strings hardcoded** na UI: todo texto vem de `messages/pt-BR.json` (futuros `es.json`, `en.json`).
  Gate de lint reprova literais em JSX ([NFR-I18N-01](../product/NON_FUNCTIONAL_REQUIREMENTS.md)).
- **Namespace por feature/vertical slice** (DRY, alinhado à arquitetura — [CLAUDE.md](../../CLAUDE.md)):
  `auth.*`, `checkout.*`, `player.*`, `course.*`, `comments.*`, `certificate.*`, `quiz.*`, `dashboard.*`,
  `studio.*`, `admin.*`, `affiliate.*`, `billing.*`, `errors.*`, `empty.*`, `blocked.*`, `success.*`,
  `a11y.*`.
- **Padrão de chave:** `namespace.contexto.elemento[.variante]` em `camelCase`, sem PT no nome da chave.
  Exemplos: `player.completeButton`, `errors.payment.cardDeclined`, `empty.courses.title`,
  `empty.courses.cta`.
- **Variantes de estado:** `*.title`, `*.description`/`*.body`, `*.cta`, `*.secondaryCta`, `*.helper`.
- **Não duplicar** texto entre back e front: textos que viajam em payload (ex.: rótulo de erro de domínio)
  referenciam **chaves** resolvidas no front; o backend envia `code` + dados, não a string final
  (Problem Details RFC 9457 — ver §6.1).

### 4.2 ICU MessageFormat (pluralização, gênero, seleção)

Use **ICU**, nunca concatenação de strings ([NFR-I18N-03](../product/NON_FUNCTIONAL_REQUIREMENTS.md)).

```jsonc
// messages/pt-BR.json (exemplos)
{
  "course": {
    "lessonsCount": "{count, plural, =0 {Nenhuma aula} one {# aula} other {# aulas}}",
    "progress": "Progresso do curso: {pct, number, ::percent}",
    "lessonOfTotal": "Aula {current, number} de {total, number}"
  },
  "comments": {
    "count": "{count, plural, =0 {Sem comentários} one {# comentário} other {# comentários}}",
    "repliesCount": "{count, plural, one {# resposta} other {# respostas}}"
  },
  "blocked": {
    "dripDays": "Esta aula libera em {days, plural, one {# dia} other {# dias}} ({date})."
  },
  "quiz": {
    "attemptsLeft": "{count, plural, =0 {Sem tentativas restantes} one {# tentativa restante} other {# tentativas restantes}}"
  }
}
```

- **Interpolação tipada:** variáveis sempre nomeadas (`{count}`, `{course_title}`), nunca posicionais.
- **Datas/moeda/números:** formatação via skeletons ICU (`::percent`, `::currency/BRL`) ou `Intl` no
  componente; **nunca** formatar manualmente ([NFR-I18N-07/08/09](../product/NON_FUNCTIONAL_REQUIREMENTS.md)).
  Valor monetário trafega como **inteiro em centavos** + moeda.
- **Seleção (`select`)** para gênero/estado quando inevitável; preferir neutro (§3.1).

### 4.3 Boas práticas de tradução

- **Não quebrar frases** em múltiplas chaves para "montar" sentenças (ordem de palavras muda por idioma).
- **Contexto para tradutores:** comentar a chave (`_comment`) com onde/como aparece e limites de caracteres.
- **Placeholders preservam o `{token}`** literal entre idiomas; nunca traduzir o nome do token.
- **Propriedades lógicas/RTL-ready:** o texto não assume direção; layout usa propriedades lógicas
  ([NFR-I18N-05](../product/NON_FUNCTIONAL_REQUIREMENTS.md)).

---

## 5. Catálogo de microcopy — títulos e CTAs

> Tabelas com **chave i18n + texto PT-BR + contexto**. Reutilize as chaves; não recrie sinônimos.

### 5.1 CTAs primários (verbo + objeto, sentence case)

| Chave | PT-BR | Contexto |
|-------|-------|----------|
| `cta.start` | Começar | Curso 0% ([LEARNING_EXPERIENCE_UX §3.2]) |
| `cta.resume` | Continuar | Curso/aula em progresso |
| `cta.continueWatching` | Continuar assistindo | Card do dashboard |
| `cta.nextLesson` | Próxima aula | Rodapé do player |
| `cta.markComplete` | Marcar como concluída | Aula não-vídeo / override |
| `cta.markIncomplete` | Marcar como não concluída | Desfazer conclusão manual |
| `cta.publish` | Publicar | Estúdio |
| `cta.save` | Salvar | Formulários |
| `cta.saveDraft` | Salvar rascunho | Estúdio |
| `cta.buy` | Comprar | Landing do curso |
| `cta.checkout` | Ir para o pagamento | Checkout |
| `cta.retryPayment` | Tentar outro pagamento | Pagamento recusado |
| `cta.enroll` | Matricular-se | Curso gratuito / catálogo |
| `cta.downloadCertificate` | Baixar certificado | Tela de conquista |
| `cta.viewCertificate` | Ver certificado | Card de curso 100% |
| `cta.copyVerifyLink` | Copiar link de verificação | Certificado |
| `cta.share` | Compartilhar | Certificado |
| `cta.submitQuiz` | Enviar quiz | Quiz |
| `cta.retryQuiz` | Refazer | Quiz com tentativas restantes |
| `cta.startQuiz` | Iniciar quiz | Tela de início do quiz |
| `cta.publishComment` | Publicar | Composer de comentário |
| `cta.reply` | Responder | Comentário |
| `cta.tryAgain` | Tentar novamente | Recuperação de erro |
| `cta.goToCatalog` | Ver catálogo | Empty state sem cursos |
| `cta.upgradePlan` | Fazer upgrade do plano | Quota (admin) |
| `cta.contactSupport` | Falar com o suporte | Erros recuperáveis |
| `cta.backHome` | Voltar ao início | Páginas de erro |

### 5.2 CTAs secundários / destrutivos

| Chave | PT-BR | Contexto |
|-------|-------|----------|
| `cta.cancel` | Cancelar | Fechar sem salvar |
| `cta.delete` | Excluir | Ação destrutiva (com confirmação) |
| `cta.remove` | Remover | Tirar item de lista |
| `cta.restartFromStart` | Recomeçar do início | Retomada de vídeo |
| `cta.cancelAutoplay` | Cancelar | Overlay de autoplay |
| `cta.playNow` | Reproduzir agora | Overlay de autoplay |
| `cta.report` | Denunciar | Comentário |
| `cta.hide` | Ocultar | Moderação |
| `cta.resendEmail` | Reenviar e-mail | Verificação não chegou |

### 5.3 Títulos de tela/seção (sentence case, ≤ 6 palavras)

| Chave | PT-BR | Tela |
|-------|-------|------|
| `dashboard.greeting` | Olá, {name}! | Dashboard ([LEARNING_EXPERIENCE_UX §3.2]) |
| `dashboard.continueSection` | Continuar assistindo | Seção |
| `dashboard.myCoursesSection` | Meus cursos | Seção |
| `auth.signupTitle` | Criar sua conta | Cadastro |
| `auth.loginTitle` | Entrar | Login |
| `auth.verifyEmailTitle` | Verifique seu e-mail | Pós-cadastro |
| `auth.resetRequestTitle` | Redefinir senha | Solicitar reset |
| `auth.resetNewTitle` | Defina uma nova senha | Definir senha |
| `checkout.title` | Finalizar compra | Checkout |
| `checkout.awaitingPix` | Aguardando pagamento | Pix |
| `certificate.congratsTitle` | Parabéns, {name}! | Conquista |
| `quiz.resultPassedTitle` | Você passou! | Resultado |
| `quiz.resultFailedTitle` | Quase lá | Resultado reprovado |

### 5.4 Catálogo de aria-labels (botões só-ícone)

| Chave | PT-BR (`aria-label`) | Ícone |
|-------|----------------------|-------|
| `a11y.player.playPause` | Reproduzir ou pausar | ▶/⏸ |
| `a11y.player.fullscreen` | Tela cheia | ⛶ |
| `a11y.player.captions` | Legendas | CC |
| `a11y.player.speed` | Velocidade de reprodução | 1x |
| `a11y.comment.like` | Curtir comentário | ♥ |
| `a11y.comment.more` | Mais ações do comentário | ⋯ |
| `a11y.notifications.open` | Abrir notificações | 🔔 |
| `a11y.search.lessons` | Buscar aula | 🔍 |
| `a11y.settings` | Preferências | ⚙ |

---

## 6. Catálogo de mensagens de erro

### 6.1 Princípios de erro

- **Estrutura:** **o que aconteceu** (sem jargão) → **por quê** (se útil) → **o que fazer agora** (ação).
- **Sem culpa, sem stack, sem código técnico cru.** Erros 500/inesperados mostram mensagem humana + **ID de
  correlação copiável** ("Informe o código {requestId} ao suporte") — [NFR-OBS-05](../product/NON_FUNCTIONAL_REQUIREMENTS.md).
- **Mapeamento backend → front:** o handler único emite **Problem Details (RFC 9457)** com `type`/`code`;
  o front resolve a **chave i18n** correspondente. O backend **não** envia a string final (DRY/i18n).
  Use-cases lançam `DomainError` — nunca tratam HTTP ([CLAUDE.md](../../CLAUDE.md)).
- **Anti-enumeração** em auth: nunca revele se um e-mail existe ([NFR-SEC-05/06](../product/NON_FUNCTIONAL_REQUIREMENTS.md)).
- **Associação a11y:** erros de campo via `aria-describedby`; erros globais via `aria-live="assertive"`.

### 6.2 Validação (formulários)

| Chave | PT-BR | Contexto |
|-------|-------|----------|
| `errors.validation.required` | Preencha este campo. | Campo obrigatório vazio |
| `errors.validation.email` | Digite um e-mail válido. | E-mail malformado |
| `errors.validation.passwordWeak` | Use ao menos 8 caracteres, com letras e números. | Senha fraca |
| `errors.validation.passwordMismatch` | As senhas não coincidem. | Confirmação |
| `errors.validation.consentRequired` | Para continuar, aceite os Termos e a Política de Privacidade. | Checkbox LGPD |
| `errors.validation.cpfInvalid` | Verifique o CPF informado. | CPF inválido |
| `errors.validation.maxLength` | Texto muito longo (máximo {max} caracteres). | Limite de caracteres |
| `errors.validation.slugTaken` | Esse endereço já está em uso. Tente outro. | Slug duplicado (estúdio) |
| `errors.validation.couponInvalid` | Cupom inválido ou expirado. | Cupom no checkout |
| `errors.validation.fileType` | Formato não suportado. Use {formats}. | Upload |
| `errors.validation.fileTooBig` | Arquivo muito grande (máximo {max}). | Upload |

### 6.3 Pagamento

| Chave | PT-BR | Contexto |
|-------|-------|----------|
| `errors.payment.cardDeclined` | Seu pagamento não foi aprovado. Tente outro cartão ou forma de pagamento. | Cartão recusado |
| `errors.payment.insufficientFunds` | Não foi possível concluir o pagamento. Verifique os dados ou tente outra forma. | Saldo/limite (sem expor motivo do banco) |
| `errors.payment.gatewayUnavailable` | O pagamento está indisponível no momento. Tente novamente em instantes. | Gateway fora ([NFR-REL-06]) |
| `errors.payment.pixExpired` | Este código Pix expirou. Gere um novo para concluir. | Pix expirado |
| `errors.payment.boletoExpired` | Este boleto venceu. Gere um novo para concluir. | Boleto expirado |
| `errors.payment.duplicateAttempt` | Já recebemos sua solicitação. Aguarde a confirmação. | Anti duplo-clique ([NFR-REL-14]) |
| `errors.payment.generic` | Não foi possível processar o pagamento. Nenhuma cobrança foi feita. | Falha genérica |

### 6.4 Acesso / entitlement

| Chave | PT-BR | Contexto |
|-------|-------|----------|
| `errors.access.notEnrolled` | Você ainda não tem acesso a este conteúdo. | Sem matrícula (ver também §9) |
| `errors.access.suspended` | Seu acesso está suspenso. Regularize o pagamento para continuar. | Inadimplência ([LEARNING_EXPERIENCE_UX §2.10]) |
| `errors.access.expired` | Seu acesso expirou em {date}. | Matrícula expirada |
| `errors.access.refunded` | Este acesso foi encerrado. | Reembolso/cancelamento |
| `errors.access.forbidden` | Você não tem permissão para acessar esta página. | 403 (RBAC) |
| `errors.access.sessionExpired` | Sua sessão expirou. Entre novamente para continuar. | Sessão ([NFR-SEC-02]) |
| `errors.access.tenantUnavailable` | Este ambiente está temporariamente indisponível. | Tenant suspenso (aluno — §9.4 dos flows; nunca expõe motivo do SaaS) |

### 6.5 Vídeo / player

| Chave | PT-BR | Contexto |
|-------|-------|----------|
| `errors.video.loadFailed` | Não foi possível carregar o vídeo. | Falha de player/token ([LEARNING_EXPERIENCE_UX §2.11]) |
| `errors.video.tokenExpired` | Sessão de vídeo expirada, recarregando… | Token expirou em sessão |
| `errors.video.processing` | Estamos preparando este vídeo. Isso pode levar alguns minutos. | Encoding em andamento ([LEARNING_EXPERIENCE_UX §2.9]) |
| `errors.video.unavailable` | Este vídeo está temporariamente indisponível. Tente novamente em instantes. | Bunny fora ([NFR-REL-05]) |
| `errors.video.encodingFailed` | Não foi possível processar este vídeo. | Visão do instrutor (com CTA reenviar) |

### 6.6 Rede / sistema

| Chave | PT-BR | Contexto |
|-------|-------|----------|
| `errors.network.offline` | Você está sem conexão. Verifique sua internet. | Offline ([NFR-COMP-10]) |
| `errors.network.timeout` | A conexão demorou demais. Tente novamente. | Timeout |
| `errors.network.saveFailed` | Não foi possível salvar. Tente novamente. | Falha de escrita |
| `errors.network.progressRetry` | Salvando progresso… | Heartbeat reenfileirado ([LEARNING_EXPERIENCE_UX §2.5]) |
| `errors.generic.title` | Algo deu errado | Página de erro 500 ([NFR-REL-09]) |
| `errors.generic.body` | Tivemos um problema inesperado. Tente novamente em instantes. | Corpo do 500 |
| `errors.generic.correlationId` | Se o problema continuar, informe o código {requestId} ao suporte. | ID de correlação |
| `errors.notFound.title` | Página não encontrada | 404 |
| `errors.notFound.body` | O conteúdo que você procura não existe ou foi movido. | 404 |
| `errors.rateLimited` | Muitas tentativas. Aguarde um momento e tente de novo. | Rate limit ([NFR-SEC-05]) |

---

## 7. Catálogo de empty states

> Fórmula: **título acolhedor** (estado, sem culpa) + **descrição breve** (o que é/por quê) + **CTA**
> (próximo passo, quando houver). Toda lista/painel tem empty state ([NFR-REL-10](../product/NON_FUNCTIONAL_REQUIREMENTS.md)).

| Chave | Título | Descrição | CTA |
|-------|--------|-----------|-----|
| `empty.courses` | Você ainda não tem cursos | Quando você se matricular, seus cursos aparecem aqui. | Ver catálogo |
| `empty.coursesProgress` | Nada em andamento | Comece um curso para vê-lo por aqui. | Ver meus cursos |
| `empty.certificates` | Nenhum certificado ainda | Conclua um curso para receber seu certificado. | Ver meus cursos |
| `empty.comments` | Seja o primeiro a comentar | Tire uma dúvida ou compartilhe o que achou desta aula. | — (composer em foco) |
| `empty.materials` | Esta aula não tem materiais | Quando houver anexos, eles ficam disponíveis aqui. | — |
| `empty.notifications` | Tudo em dia | Você não tem notificações novas. | — |
| `empty.students` (admin) | Nenhum aluno ainda | Seus alunos aparecem aqui após a primeira matrícula. | Matricular aluno |
| `empty.studentsSearch` (admin) | Nenhum resultado | Não encontramos alunos para "{query}". Ajuste a busca. | Limpar busca |
| `empty.orders` (admin) | Nenhuma venda ainda | Quando você vender, os pedidos aparecem aqui. | — |
| `empty.coursesStudio` (instrutor) | Crie seu primeiro curso | Estruture módulos e aulas para começar a ensinar. | Novo curso |
| `empty.affiliateLinks` (afiliado) | Nenhum link gerado | Gere um link para começar a divulgar e ganhar comissões. | Gerar link |
| `empty.commissions` (afiliado) | Sem comissões ainda | Suas comissões aparecem aqui quando houver vendas atribuídas a você. | — |
| `empty.quizQuestions` (estúdio) | Nenhuma questão | Adicione ao menos uma questão para publicar o quiz. | Adicionar questão |
| `empty.search.generic` | Nada encontrado | Não encontramos resultados para "{query}". | Limpar busca |
| `empty.leaderboard` (F2) | O ranking começa em breve | Comece a aprender para aparecer aqui. | — |

---

## 8. Estados de sucesso e confirmação

> Sucesso é **breve, positivo e específico**. Confirmar = dizer o que aconteceu e (se útil) o próximo
> passo. Use toast `aria-live="polite"`. Reserve modal/tela só para marcos (conclusão, certificado).

### 8.1 Toasts e confirmações inline

| Chave | PT-BR | Contexto |
|-------|-------|----------|
| `success.saved` | Alterações salvas | Salvar genérico |
| `success.commentPublished` | Comentário publicado | Comentar |
| `success.commentResolved` | Marcado como resolvido | Thread resolvida |
| `success.reportSent` | Denúncia enviada. Obrigado por avisar. | Moderação |
| `success.lessonCompleted` | Aula concluída | `aria-live` ao concluir ([LEARNING_EXPERIENCE_UX §2.6]) |
| `success.progressAnnounce` | Progresso do curso: {pct, number, ::percent} | `aria-live` de barra |
| `success.linkCopied` | Link copiado | Copiar link de verificação |
| `success.coursePublished` | Curso publicado | Estúdio |
| `success.videoUploaded` | Vídeo enviado. Processando… | Upload ([USER_FLOWS §4.2]) |
| `success.studentEnrolled` | Aluno matriculado | Admin |
| `success.couponApplied` | Cupom aplicado | Checkout |
| `success.passwordChanged` | Senha alterada | Segurança |
| `success.preferencesSaved` | Preferências atualizadas | Configurações |

### 8.2 Telas/modais de marco

| Chave | PT-BR | Contexto |
|-------|-------|----------|
| `success.purchaseConfirmed.title` | Compra confirmada! | Pós-checkout ([USER_FLOWS §6.1]) |
| `success.purchaseConfirmed.body` | Tudo certo com seu pagamento. Seu acesso já está liberado. | — |
| `success.purchaseConfirmed.cta` | Acessar curso | — |
| `success.milestone50` | Metade do caminho! 🎉 | Marco 50% ([LEARNING_EXPERIENCE_UX §3.5]) |
| `success.courseCompleted.title` | Parabéns, você concluiu o curso! | 100% |
| `certificate.generating` | Gerando seu certificado… | Estado intermediário (CTA download desabilitado) |
| `certificate.ready.body` | Seu certificado está pronto. | Conquista |
| `success.dataExportReady.body` | Seus dados estão prontos para download. | LGPD ([NFR-LGPD-06]) |

### 8.3 Confirmações destrutivas (modal)

> Padrão: **título-pergunta** + consequência clara + botão que **nomeia a ação** (não "OK").

| Chave | Título | Corpo | CTA confirmar |
|-------|--------|-------|---------------|
| `confirm.deleteComment` | Excluir comentário? | Esta ação não pode ser desfeita. | Excluir |
| `confirm.unpublishCourse` | Despublicar curso? | Os alunos perderão acesso até você publicar de novo. O progresso é mantido. | Despublicar |
| `confirm.cancelSubscription` | Cancelar assinatura? | Você mantém acesso até {date}. | Cancelar assinatura |
| `confirm.deleteAccount` | Excluir sua conta? | Você perderá acesso aos seus cursos e certificados. Esta ação é definitiva. | Excluir conta |
| `confirm.refundOrder` (admin) | Reembolsar pedido? | O acesso do aluno será encerrado e a comissão do afiliado, revertida. | Reembolsar |
| `confirm.suspendTenant` (SA) | Suspender tenant? | Todos os usuários deste ambiente perderão acesso. | Suspender |

---

## 9. Mensagens de bloqueio (drip, acesso, quota)

> **Bloqueio ≠ erro.** É um estado **informativo** com motivo e caminho. Distinguir claramente
> **bloqueio de conteúdo** (drip/pré-requisito) de **bloqueio de acesso** (pagamento) — [LEARNING_EXPERIENCE_UX §2.8–2.10].

### 9.1 Drip / pré-requisito (conteúdo)

| Chave | PT-BR | Contexto |
|-------|-------|----------|
| `blocked.dripDate` | Esta aula libera em {date}. | Drip por data fixa |
| `blocked.dripDays` | Esta aula libera em {days, plural, one {# dia} other {# dias}} ({date}). | Drip por dias após matrícula |
| `blocked.dripBadge` | Disponível em {date} | Selo na lista lateral |
| `blocked.prerequisite` | Conclua a aula "{lesson_title}" para liberar esta. | Pré-requisito (F2) |
| `blocked.cta.nextAvailable` | Ir para a próxima aula disponível | CTA do painel de bloqueio |
| `blocked.title` | Conteúdo bloqueado | Título do painel |

### 9.2 Sem matrícula / catálogo

| Chave | PT-BR | Contexto |
|-------|-------|----------|
| `blocked.enroll.title` | Matricule-se para assistir | Aluno sem matrícula no player ([USER_FLOWS §7.1]) |
| `blocked.enroll.bodyPaid` | Garanta seu acesso para liberar todas as aulas deste curso. | Curso pago |
| `blocked.enroll.cta` | Ver opções de compra | CTA |
| `blocked.commentsDisabled` | Os comentários estão desativados nesta aula. | Config do instrutor |

### 9.3 Acesso suspenso/expirado (pagamento)

| Chave | PT-BR | Contexto |
|-------|-------|----------|
| `blocked.access.suspended.title` | Acesso suspenso | Inadimplência ([USER_FLOWS §9.3]) |
| `blocked.access.suspended.body` | Regularize o pagamento para voltar a estudar. | — |
| `blocked.access.suspended.cta` | Regularizar pagamento | — |
| `blocked.access.expired.title` | Seu acesso expirou | — |
| `blocked.access.expired.body` | Seu acesso terminou em {date}. | — |
| `blocked.access.expired.cta` | Renovar acesso | Quando aplicável |

### 9.4 Quota / plano (admin/owner)

> Voltado a **admin/owner** (público profissional), tom objetivo. Inclui empurrão suave a upgrade.

| Chave | PT-BR | Contexto |
|-------|-------|----------|
| `blocked.quota.warning` | Você usou {used} de {limit} {resource}. | Aviso ~80–100% ([NOTIFICATIONS_MATRIX §5]) |
| `blocked.quota.exceeded.title` | Limite do plano atingido | Ação bloqueada |
| `blocked.quota.exceeded.body` | Você atingiu o limite de {resource} do seu plano. Faça upgrade para continuar. | — |
| `blocked.quota.exceeded.cta` | Fazer upgrade do plano | — |
| `blocked.seats.exceeded` | Você atingiu o limite de membros da equipe do seu plano. | Convite de equipe ([USER_FLOWS §2.4]) |
| `blocked.payments.notActivated` | Ative os pagamentos para começar a vender. | Recipient/KYC pendente ([README §4, item 26]) |

---

## 10. Glossário de termos para o usuário final

> **Use sempre o termo da esquerda** na UI (alinhado ao [glossário canônico do README §2](../product/README.md)).
> A coluna "evite" lista sinônimos proibidos para garantir consistência.

| Use | Evite | Observação |
|-----|-------|------------|
| curso | treinamento, programa | Termo neutro de catálogo |
| aula | lição, vídeo-aula, módulo | "Aula" é a unidade; "módulo" agrupa aulas |
| módulo | capítulo, unidade | Agrupador de aulas |
| matrícula / matricular-se | inscrição, assinatura | Acesso a um curso; não confundir com assinatura recorrente |
| assinatura | plano (do aluno) | Cobrança recorrente do aluno por acesso |
| certificado | diploma, certificação | Documento emitido na conclusão |
| instrutor | professor, mentor, tutor | Papel `instructor` (neutro) |
| aluno | estudante, usuário | Papel `student` na UI voltada ao próprio aluno; "você" |
| progresso | avanço, andamento | % de conclusão |
| concluída (aula) | finalizada, completa | Estado `completed` |
| acesso | permissão, liberação | Entitlement; "seu acesso" |
| comentário | post, mensagem | Na sala de aula |
| suporte | atendimento, SAC | Sempre `{support_email}` do tenant |
| pagamento | cobrança (para o aluno) | "Cobrança" usar mais no contexto admin/assinatura |
| cupom | voucher, código promocional | Campo `code` |

---

## Dependências e pontos para o coordenador

1. **Locale e gênero do usuário:** confirmar se o perfil captura **gênero/pronome** opcional. Sem isso,
   mantemos a estratégia neutra (§3.1); com isso, podemos personalizar saudações via ICU `select`.
   Cruza com [NFR-I18N-04](../product/NON_FUNCTIONAL_REQUIREMENTS.md) e DATA_MODEL (perfil).
2. **Catálogo i18n canônico:** este documento define **chaves**; falta consolidá-las em
   `messages/pt-BR.json` e expor um helper de tipos em `packages/contracts` (ou pacote `i18n`) para evitar
   chaves órfãs (DRY back/front). Recomendo um gate de lint de "chave faltante/não usada".
3. **Códigos de erro de domínio ↔ chaves i18n:** padronizar o mapa `DomainError.code → errors.*` (RFC 9457)
   junto ao handler único (§6.1). Precisa de enum canônico compartilhado (cruza com
   [NOTIFICATIONS_MATRIX dep. 6](../product/NOTIFICATIONS_MATRIX.md) sobre enum de eventos).
4. **Texto de regularização de inadimplência (§9.3):** depende do contrato exato dos estados de
   `enrollments` e do fluxo de pagamento (cruza [LEARNING_EXPERIENCE_UX §9 — Estados](../product/LEARNING_EXPERIENCE_UX.md)).
   Confirmar com pagamentos a URL/ação de regularização por método (Pix/cartão/assinatura).
5. **Nomenclatura por nicho (white-label):** alguns tenants podem querer renomear "curso/aula/turma".
   Decidir se haverá **rótulos configuráveis por tenant** (overrides de label) ou termo fixo no MVP.
   Recomendação MVP: fixo (este glossário), configurável em F2.
6. **Voz/intensidade de gamificação (F2):** celebrações (§8.2) e badges precisam de revisão de copy quando
   a gamificação entrar; valores e nomes de badge em [LEARNING_EXPERIENCE_UX §5](../product/LEARNING_EXPERIENCE_UX.md).
7. **ES/EN (F2):** PT-BR é a fonte; ao traduzir, validar pluralização ICU e limites de caractere em botões
   (alemão/espanhol expandem). Recomendo testes de "string overflow" no design system.
8. **Revisão jurídica/LGPD do copy sensível:** textos de consentimento, exclusão de conta e exportação de
   dados (§7/§8) devem passar por revisão jurídica ([NFR-LGPD](../product/NON_FUNCTIONAL_REQUIREMENTS.md), dep. 24 do README).
