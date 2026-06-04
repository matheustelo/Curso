# Fluxos de Usuário — Plataforma de Cursos SaaS Multitenant

- **Versão:** 1.0 · **Data:** 2026-06-04
- **Autor:** Interaction Design / Information Architecture
- **Status:** Proposta para revisão de produto/engenharia
- **Relacionados:** [PRD.md](../PRD.md) · [ROADMAP.md](../ROADMAP.md) · [ARCHITECTURE.md](../ARCHITECTURE.md) · [DATA_MODEL.md](../DATA_MODEL.md) · [INFORMATION_ARCHITECTURE.md](INFORMATION_ARCHITECTURE.md)

> **Personas** (alinhadas ao PRD §2): **Super-Admin** (nós, control plane) · **Admin do Tenant** (papéis `owner`/`admin`) · **Instrutor** (`instructor`) · **Afiliado** (`affiliate`) · **Aluno** (`student`).
> **Notação de fase:** **[MVP]** · **[F2]** · **[F3]**, conforme PRD/ROADMAP.
> **Convenção de back-end:** todo acesso a dados de tenant passa por `withTenant(tenantId, fn)` (`SET LOCAL search_path`). `tenant_id` vem da sessão/JWT (fonte de verdade), nunca de header. Erros mapeados para Problem Details (RFC 9457). Webhooks idempotentes.

---

## Índice

1. [Convenções dos diagramas](#1-convenções-dos-diagramas)
2. [Autenticação e contas](#2-autenticação-e-contas)
   - 2.1 Cadastro de aluno · 2.2 Login (aluno e equipe) · 2.3 Recuperação de senha · 2.4 Convite de equipe · 2.5 Convite/adesão de afiliado
3. [Onboarding e provisionamento de tenant](#3-onboarding-e-provisionamento-de-tenant)
   - 3.1 Assinatura + provisionamento · 3.2 Configuração de marca/domínio
4. [Autoria de conteúdo](#4-autoria-de-conteúdo)
   - 4.1 Curso → Módulo → Aula · 4.2 Upload de vídeo (Bunny/TUS) · 4.3 Publicar · 4.4 Drip · 4.5 Quiz builder
5. [Certificados](#5-certificados)
   - 5.1 Emissão · 5.2 Download · 5.3 Verificação pública
6. [Monetização — fluxos do aluno](#6-monetização--fluxos-do-aluno)
   - 6.1 Checkout (Pix/boleto/cartão) + cupom · 6.2 Matrícula e liberação · 6.3 Order bump/upsell [F2]
7. [Consumo de conteúdo](#7-consumo-de-conteúdo)
   - 7.1 Assistir aula (URL assinada, retomar, progresso) · 7.2 Comentar aula · 7.3 Comunidade [F2]
8. [Afiliados e split](#8-afiliados-e-split)
9. [Pagamento ↔ acesso: reembolso, cancelamento, suspensão](#9-pagamento--acesso-reembolso-cancelamento-suspensão)
10. [Super-Admin](#10-super-admin)
    - 10.1 Criar tenant · 10.2 Suspender/ativar tenant · 10.3 Impersonar
11. [Máquinas de estado de referência](#11-máquinas-de-estado-de-referência)
12. [Dependências e pontos para o coordenador](#dependências-e-pontos-para-o-coordenador)

---

## 1. Convenções dos diagramas

- `→` transição de passo · `⟂` ponto de decisão (if/else) · `⟳` retry/loop · `✔` estado de sucesso · `✖` estado de erro.
- **[Tela]** indica uma página/modal do inventário (ver INFORMATION_ARCHITECTURE.md).
- **(BE)** descreve o que ocorre no backend em alto nível.
- **(Job)** indica processamento assíncrono via pg-boss (worker).

---

## 2. Autenticação e contas

### 2.1 Cadastro de aluno [MVP]

Contexto: o aluno se cadastra **dentro do tenant** (subdomínio `tenant.app.com`). O mesmo e-mail pode existir em tenants diferentes (modelo workspace; `users.unique(email)` é por schema).

```
[Landing/Catálogo do tenant] → clica "Criar conta" / inicia checkout
        → [Tela: Cadastro]
```

1. Aluno acessa **[Cadastro]** (ou é levado a ela durante o checkout — ver §6.1).
2. Preenche nome, e-mail, senha. Aceita **Termos** e **Política de Privacidade** (consentimento LGPD — checkbox obrigatório, registrado com timestamp).
3. **Validações (client + server, schema Zod de `packages/contracts`):** e-mail válido; senha conforme política (mín. 8, etc.); consentimento marcado.
4. ⟂ **E-mail já existe neste tenant?**
   - Sim → ✖ erro "conta já existe" + CTA para login/recuperação (não revela detalhes além do necessário).
   - Não → segue.
5. (BE) `withTenant(tenant)`: cria `users(role='student', status='active')` via Better-Auth; registra consentimento; dispara **(Job)** e-mail de verificação/boas-vindas (Resend).
6. ⟂ **Verificação de e-mail exigida?**
   - Sim (recomendado) → ✔ **[Tela: Verifique seu e-mail]**; acesso pleno após clicar no link. Conteúdo gratuito pode ficar liberado; conteúdo pago depende de matrícula.
   - Não → ✔ sessão criada (cookie HttpOnly) → redireciona para **[Área do aluno / Meus cursos]**.
7. ✖ Erros: e-mail de verificação não chega → CTA "reenviar"; falha de criação → Problem Details + toast.

> Observação: durante o checkout, o cadastro é **embutido** (cria conta + matrícula na mesma jornada) para reduzir fricção (ver §6.1).

### 2.2 Login (aluno e equipe) [MVP]

Mesmo fluxo de UI para aluno, instrutor e admin do tenant; o **papel** define o destino pós-login. Super-Admin tem login **separado** no control plane (ver §10).

```
[Tela: Login do tenant] —submit→ (BE valida credenciais) ⟂ ok?
   ├─ não → ✖ erro genérico ("credenciais inválidas") + contador de tentativas
   └─ sim → ⟂ papel?
              ├─ student        → [Meus cursos]
              ├─ instructor      → [Painel do instrutor]
              ├─ admin/owner      → [Painel admin — Dashboard]
              └─ affiliate        → [Painel do afiliado]
```

1. Usuário em **[Login]** insere e-mail + senha.
2. (BE) Better-Auth valida no schema do tenant resolvido pelo subdomínio; cria sessão (cookie HttpOnly+Secure).
3. ⟂ **Credenciais válidas?** Não → ✖ erro genérico (não revela se e-mail existe); rate-limit após N tentativas → lockout temporário/captcha.
4. ⟂ **Conta `status='active'`?** Suspensa/banida → ✖ mensagem apropriada + canal de suporte.
5. ⟂ **2FA habilitado?** [F2] Sim → **[Tela: 2FA]** → valida código.
6. ⟂ **Tenant suspenso?** (status do SaaS) → todos exceto owner veem **[Tela: tenant indisponível]**; owner é direcionado a billing (ver §9/§10.2).
7. ✔ Redireciona conforme o papel (ver diagrama).

### 2.3 Recuperação de senha [MVP]

```
[Login] → "Esqueci minha senha" → [Tela: Solicitar reset]
   → (BE) sempre responde "se existir, enviaremos e-mail" (anti-enumeração)
   → (Job) e-mail com token (TTL curto, single-use)
   → [Tela: Definir nova senha] ⟂ token válido? sim→reset / não→✖ expirado
```

1. Usuário informa e-mail em **[Solicitar reset]**.
2. (BE) `withTenant`: se o e-mail existir, gera token de reset (hash, TTL ~30–60 min, uso único) e dispara **(Job)** e-mail. **Resposta idêntica** exista ou não (anti-enumeração).
3. Usuário clica no link → **[Definir nova senha]**.
4. ⟂ **Token válido e não usado?** Não → ✖ "link expirado/inválido" + CTA para solicitar novo.
5. Define nova senha (validação de política). (BE) atualiza hash, **invalida sessões ativas** e marca token consumido.
6. ✔ **[Tela: Senha alterada]** → redireciona para login.

### 2.4 Convite de equipe (Admin/Instrutor) [MVP]

```
[Admin → Equipe] → "Convidar membro" → [Modal: convite]
   → (BE) cria convite pendente + (Job) e-mail
   → convidado clica → ⟂ já tem conta no tenant?
        ├─ sim  → adiciona papel → [Login/aceite]
        └─ não  → [Tela: Aceitar convite / definir senha] → cria user
```

1. Admin/owner abre **[Equipe]** e clica "Convidar membro".
2. Informa e-mail + **papel** (`admin` | `instructor`). **Validação:** papel permitido; limite de assentos por plano (quota) verificado.
3. ⟂ **Quota de assentos atingida?** Sim → ✖ bloqueio + CTA upgrade de plano.
4. (BE) `withTenant`: cria registro de convite (token, TTL, papel) com `status='pending'`; **(Job)** e-mail (Resend).
5. Convidado abre link → ⟂ **já possui conta neste tenant?**
   - Sim → loga e o papel é anexado/atualizado → ✔ **[Painel correspondente]**.
   - Não → **[Aceitar convite]** define nome/senha → cria `users(role=convite)` → ✔ painel.
6. ⟂ **Convite expirado/revogado?** → ✖ "convite inválido". Admin pode reenviar/revogar em **[Equipe]**.
7. ✔ Membro aparece na lista de equipe com papel e status.

### 2.5 Convite/adesão de afiliado [MVP]

Dois caminhos: **convite direto** pelo admin, ou **autoinscrição** se o tenant abrir programa público.

```
Caminho A (convite):  [Admin → Afiliados] → "Convidar" → e-mail → aceite → vira affiliate
Caminho B (público):  [Página: Seja afiliado] → solicita → ⟂ aprovação manual?
                         ├─ auto-aprovar → ativo
                         └─ revisão     → status=pending → admin aprova/recusa
```

1. **A — Convite:** admin em **[Afiliados]** convida por e-mail (similar a §2.4, papel `affiliate`).
2. **B — Autoinscrição:** visitante/aluno acessa **[Seja afiliado]**, preenche dados (e dados de recebimento p/ split, quando aplicável).
3. ⟂ **Política do tenant: aprovação automática?**
   - Auto → cria `affiliates(status='active', code=<gerado>, commission_pct=<default do tenant>)`.
   - Manual → `affiliates(status='pending')`; admin recebe notificação e aprova/recusa em **[Afiliados]**.
4. (BE) `withTenant`: gera `code` único do afiliado; vincula a `users` (cria conta se necessário).
5. ✔ Afiliado acessa **[Painel do afiliado]** e gera links (ver §8).
6. ✖ Recusado → e-mail informando; pode reenviar solicitação se a política permitir.

---

## 3. Onboarding e provisionamento de tenant

### 3.1 Assinatura + provisionamento (saga idempotente) [MVP]

Quem inicia: prospect do SaaS (futuro Admin do Tenant) no **site da plataforma** (não em subdomínio de tenant). Billing via **Stripe Billing** no control plane.

```
[Site da plataforma → Planos] → escolhe plano → [Checkout Stripe Billing]
   → ✔ assinatura criada (webhook Stripe)
   → (BE platform) INSERT tenants(status='provisioning', idempotency_key)
   → (Job: saga provisioning)
        1. CREATE SCHEMA tenant_<slug>
        2. migrations no schema
        3. seed (papéis, admin, curso exemplo)
        4. criar Bunny Video Library + guardar keys cifradas
        5. registrar roteamento + smoke test
   → ⟂ todos os passos ok?
        ├─ sim → tenants.status='active' → (Job) e-mail boas-vindas ao admin
        └─ não → retry ⟳ (estado em provisioning_jobs); falha persistente → alerta Super-Admin
```

1. Prospect escolhe **plano** e **slug** desejado em **[Planos]** do site da plataforma.
2. ⟂ **Slug disponível?** (checagem em `platform.tenants.slug`) Não → ✖ sugere alternativos.
3. Completa **[Checkout Stripe Billing]** (cartão; trial opcional conforme plano).
4. (BE platform) Webhook do Stripe confirma assinatura → `INSERT tenants(status='provisioning', plan_id, idempotency_key)` + `platform_subscriptions`.
5. (Job) Saga de provisionamento (ver passos no diagrama; estado em `platform.provisioning_jobs`, idempotente com `IF NOT EXISTS` e migrations versionadas).
6. ⟂ **Saga concluída?**
   - Sucesso → `status='active'`; **(Job)** e-mail de boas-vindas com link para `tenant.app.com` e primeiro login do admin (define senha).
   - Falha → retry com backoff; se exceder, registra `last_error`, mantém `provisioning` e **alerta o Super-Admin** (ver §10).
7. ✔ Admin do tenant acessa o subdomínio, define senha → **[Painel admin — Onboarding inicial/wizard]**.
8. **Estados visíveis ao Super-Admin** em **[Tenants]**: `provisioning` (spinner/progress da saga), `active`, `suspended`, `cancelled`.

### 3.2 Configuração de marca/domínio [MVP / F2]

```
[Admin → Configurações → Marca]
   → logo + cores + favicon → preview ao vivo → salvar
   → (BE) persiste branding do tenant → tema via CSS variables no front

[Admin → Configurações → Domínio]
   → MVP: subdomínio tenant.app.com (automático)
   → F2: domínio próprio → instruções DNS (CNAME) → ⟂ DNS propagado? → emitir SSL automático
```

1. Admin abre **[Configurações → Marca]**.
2. Faz upload de **logo/favicon** (armazenados no R2) e define **cores** (paleta primária/secundária); **preview ao vivo**.
3. **Validações:** formato/tamanho de imagem; contraste mínimo (apoio à WCAG AA — avisa se contraste insuficiente).
4. (BE) `withTenant`/platform: persiste branding; front aplica via CSS variables por tenant.
5. **Subdomínio [MVP]:** já provisionado (`slug.app.com`) — somente exibido.
6. **Domínio próprio [F2]:** admin informa domínio → sistema exibe **registro CNAME** a configurar → ⟂ **DNS verificado?**
   - Sim → emite **SSL automático**; `tenants.custom_domain` setado; roteamento atualizado.
   - Não → ✖ "aguardando propagação"; permite reverificar.
7. ✔ Banner/identidade do tenant refletidos em todas as áreas (público + logado).

---

## 4. Autoria de conteúdo

### 4.1 Criar Curso → Módulo → Aula [MVP]

```
[Instrutor → Cursos] → "Novo curso" → [Editor do curso]
   → adiciona Módulos (drag-and-drop) → dentro de cada módulo, adiciona Aulas
   → cada aula: tipo (vídeo | texto | pdf) → editor específico
```

1. Instrutor/admin em **[Cursos]** clica "Novo curso".
2. Preenche título, slug (auto a partir do título, editável), descrição, capa (R2), **precificação** (`one_time` | `subscription` | `free` + `price_cents`/`currency`). Curso nasce `status='draft'`.
   - ⟂ **Slug único no tenant?** Não → ✖ sugere variação.
3. (BE) `withTenant`: `INSERT courses(status='draft', instructor_id=<user>)`.
4. No **[Editor do curso]**, adiciona **Módulos** (`modules.position`), reordenáveis por **drag-and-drop**.
5. Em cada módulo, adiciona **Aulas** (`lessons.position`, reordenáveis). Escolhe **tipo**:
   - **vídeo** → abre fluxo de upload (§4.2);
   - **texto** → editor rich-text (`content jsonb`);
   - **pdf/anexo** → upload para R2 (`lesson_assets`).
6. **Validações:** título obrigatório; ordem consistente; aula de vídeo exige `video_guid` antes de publicar.
7. (BE) Persiste hierarquia; reorder envia `position` atualizado em lote.
8. ✔ Estrutura salva como rascunho; nada visível a alunos até publicação (§4.3).

### 4.2 Upload de vídeo (Bunny / TUS) [MVP]

```
[Editor da aula tipo vídeo] → "Enviar vídeo"
   → (BE) cria vídeo na Library do tenant + gera assinatura/credencial de upload
   → upload DIRETO ao Bunny via TUS (resumable, do browser ao edge)
   → barra de progresso ⟂ falha de rede? → retoma (TUS) ⟳
   → ✔ upload concluído → status "processando" (encoding Bunny)
   → (Webhook Bunny "vídeo pronto" HMAC) → (Job) atualiza lesson.video_guid/duration → status "pronto"
```

1. No editor da aula (tipo vídeo), instrutor clica "Enviar vídeo".
2. (BE) `withTenant` → resolve `bunny_library_id` do tenant; cria objeto de vídeo na **Library do tenant**; gera **credencial/assinatura de upload TUS** (API key **nunca** vai ao front).
3. Browser faz **upload direto ao edge da Bunny via TUS** (resumable). **Barra de progresso** no editor.
4. ⟂ **Interrupção de rede?** → TUS **retoma** do ponto (⟳); usuário pode pausar/continuar.
5. ✔ Upload concluído → aula marca vídeo como **"processando"** (encoding pela Bunny).
6. (Webhook Bunny "vídeo pronto") → endpoint Fastify **verifica HMAC** (bytes brutos, comparação em tempo constante) → responde 200 → **enfileira (Job)**.
7. (Job) worker mapeia `VideoLibraryId → tenant`, `withTenant`: atualiza `lessons.video_guid`, `duration_seconds`, status do vídeo = **"pronto"**. **Idempotente** (dedup por evento).
8. ✖ Erros: encoding falhou → aula mostra "erro no processamento" + CTA reenviar; HMAC inválido → webhook rejeitado (sem efeito).
9. ✔ Instrutor vê preview e pode publicar a aula (§4.3).

### 4.3 Publicar [MVP]

```
[Editor do curso/aula] → "Publicar"
   → ⟂ pré-condições atendidas?
        (vídeo pronto / texto preenchido / anexo enviado)
        ├─ não → ✖ lista de pendências (bloqueia publicação)
        └─ sim → status='published' → visível conforme regras de acesso/drip
```

1. Instrutor clica "Publicar" (na aula e/ou no curso).
2. ⟂ **Pré-condições:** aula de vídeo com `video_guid` "pronto"; texto não vazio; quiz com ≥1 questão; curso com ≥1 aula publicada.
   - Falha → ✖ checklist de pendências, publicação bloqueada.
3. (BE) `withTenant`: seta `status='published'` na aula/curso.
4. ⟂ **Há regra de drip?** (ver §4.4) Sim → visibilidade/conclusão respeita a liberação programada.
5. ✔ Conteúdo passa a ser elegível para alunos (sujeito a matrícula + drip). Landing do curso fica indexável (SEO básico).
6. **Despublicar:** admin pode voltar a `draft`/`archived`; alunos perdem acesso ao item (sem apagar progresso).

### 4.4 Configurar Drip / liberação programada [MVP]

```
[Editor da aula/módulo → Liberação]
   ⟂ tipo de drip?
     ├─ Data fixa     → define drip_release_at (timestamptz)
     └─ Dias após matrícula → define drip_days_after_enroll (int)
   → (BE) persiste regra; cálculo de elegibilidade no acesso (§7.1)
```

1. Em **[Editor]**, na aba **Liberação**, instrutor escolhe a regra para a aula (ou módulo, propagando às aulas).
2. ⟂ **Tipo:**
   - **Data fixa** → seleciona data/hora (`drip_release_at`).
   - **Dias após matrícula** → define inteiro de dias (`drip_days_after_enroll`); referência = `enrollments.enrolled_at`.
   - **Imediato** → sem regra (nulls).
3. **Validações:** data futura coerente; dias ≥ 0.
4. (BE) persiste; a **elegibilidade** é calculada em tempo de acesso (não há job por aluno).
5. **Experiência do aluno:** itens não liberados aparecem **bloqueados** com selo "Disponível em DD/MM" ou "Disponível em N dias após sua matrícula" (ver §7.1).
6. ✔ Regra ativa; instrutor vê resumo do cronograma de liberação.

### 4.5 Montar quiz [MVP]

```
[Editor do curso → Avaliações] ou [Aula tipo quiz (F2)] → "Novo quiz"
   → adiciona questões (múltipla escolha / V-F) → define resposta correta
   → MVP: correção automática; (F2) pass_score / max_attempts / time_limit
   → publicar quiz
```

1. Instrutor abre **[Avaliações]** do curso (ou cria **aula tipo quiz** [F2]).
2. Cria quiz: título; vincula a `course_id` (e opcionalmente a uma `lesson_id`).
3. Adiciona **questões** (`quiz_questions`): tipo **múltipla escolha** ou **verdadeiro/falso** [MVP]; define `options` e `answer`; `position`.
4. [F2] Define `pass_score` (nota mínima), `max_attempts`, `time_limit_sec`, embaralhamento.
5. **Validações:** ≥1 questão; toda questão tem resposta correta marcada; pass_score 0–100.
6. (BE) `withTenant`: persiste `quizzes` + `quiz_questions`.
7. ✔ Publica o quiz; passa a contar para conclusão do curso quando aplicável (regra do certificado §5/PRD §4.5).

---

## 5. Certificados

### 5.1 Emissão (automática) [MVP]

```
(Aluno conclui última exigência do curso: 100% das aulas + (se houver) nota mínima no quiz)
   → (BE) detecta conclusão de curso → (Job) gera certificado
        - cria certificates(uuid_public, hash=SHA256(student+course+issued_at+secret))
        - renderiza PDF (Puppeteer/pdfme) → grava no R2 (storage_key)
   → ✔ aluno notificado (in-app + e-mail); aparece em [Meus certificados]
```

1. Gatilho: marcação de conclusão da última exigência (ver regra anti-seek §11 e PRD §4.4–4.5).
2. ⟂ **Curso 100% concluído E nota mínima atingida (quando houver quiz com pass_score)?**
   - Não → certificado **não** emitido; aluno vê o que falta.
   - Sim → segue.
3. (BE) `withTenant`: cria `certificates` com `uuid_public` (URL pública) + `hash` (integridade).
4. (Job) renderiza **PDF** (template básico do tenant) e grava no **R2** (`storage_key`); idempotente (1 certificado por `enrollment` — `enrollments 1───1 certificates`).
5. ✔ Aluno é notificado (in-app + e-mail) e vê em **[Meus certificados]**.
6. ✖ Falha na renderização → retry (Job); persistindo, alerta + estado de erro visível ao admin.

### 5.2 Baixar certificado [MVP]

1. Aluno em **[Meus certificados]** clica "Baixar".
2. (BE) valida que o certificado pertence ao aluno (entitlement) e que `revoked=false`.
3. Gera **URL assinada (TTL curto)** para o PDF no R2.
4. ⟂ **Revogado?** (ex.: pós-reembolso/fraude) → ✖ download bloqueado + motivo.
5. ✔ Download do PDF.

### 5.3 Verificação pública [MVP]

```
[Página pública: /certificados/verificar/{uuid_public}]  (ou via QR no PDF)
   → (BE) busca certificado por uuid_public → ⟂ existe e não revogado?
        ├─ sim → ✔ exibe: nome do aluno, curso, data, status válido + (opcional) confere hash
        └─ não → ✖ "certificado não encontrado / inválido / revogado"
```

1. Verificador (empregador/terceiro) acessa a URL pública (impressa no PDF + **QR code**).
2. (BE) consulta por `uuid_public` (rota pública, sem login).
3. ⟂ **Encontrado e `revoked=false`?**
   - Sim → ✔ página exibe dados do certificado (nome, curso, data de emissão, validade) e confirmação de autenticidade (hash conferido).
   - Não/Revogado → ✖ mensagem clara de inválido.
4. **Isolamento:** mesmo sendo público, a rota resolve o tenant/dado pelo `uuid_public` sem expor dados de outros tenants.

---

## 6. Monetização — fluxos do aluno

### 6.1 Checkout (Pix / boleto / cartão) + cupom [MVP]

```
[Landing do curso] → "Comprar" → [Checkout]
   → identificação (login ou cadastro embutido) → aplica cupom (opcional)
   → escolhe método: Pix | Boleto | Cartão (parcelas)
   → (BE) cria order(status='pending') via PaymentProvider (Pagar.me/Asaas)
   → ⟂ método?
        ├─ Pix    → exibe QR/copia-e-cola → aguarda webhook "paid"
        ├─ Boleto → exibe linha digitável/PDF → aguarda webhook "paid"
        └─ Cartão → autoriza na hora → ⟂ aprovado? sim→paid / não→✖ recusa
   → (Webhook pagamento "aprovado", idempotente) → matrícula + acesso (§6.2)
```

1. Aluno na **[Landing do curso]** clica "Comprar" → **[Checkout]**.
2. **Identificação:** se não logado, faz login ou **cadastro embutido** (cria conta na mesma jornada; consentimento LGPD).
3. **Cupom (opcional):** insere `code`. ⟂ **Cupom válido?** (`coupons`: existe, dentro de validade, `uses < max_uses`, aplicável ao curso) → aplica desconto; senão ✖ "cupom inválido/expirado".
4. **Afiliado:** se chegou via link de afiliado (cookie/`?ref=code`), o `affiliate_id` é anexado à futura order (atribuição — ver §8).
5. Escolhe **método de pagamento**. **Validações:** dados do cartão (tokenização no provedor; PCI no gateway), CPF para boleto/Pix, etc.
6. (BE) `withTenant`: cria `orders(status='pending', provider, payment_method, coupon_id, affiliate_id)` via **port `PaymentProvider`**.
7. ⟂ **Método:**
   - **Pix** → **[Checkout: aguardando Pix]** com QR + copia-e-cola; polling/SSE até confirmação.
   - **Boleto** → **[Checkout: boleto gerado]** com linha digitável + PDF; pode levar 1–2 dias.
   - **Cartão** → autorização imediata; ⟂ aprovado? Não → ✖ **[Tela: pagamento recusado]** + CTA tentar outro método.
8. (BE) **Webhook de pagamento** chega (verificação de assinatura + **idempotência** via `payment_events.event_id` unique) → marca `orders.status='paid'`, `paid_at` → dispara matrícula (§6.2) e comissão de afiliado (§8).
9. ✔ **[Tela: Compra confirmada]** → CTA "Acessar curso".
10. ✖ **Pix/boleto não pago:** order expira (TTL do provider); estado `pending → expired`; aluno pode recriar. E-mail de lembrete [F2 — recuperação de carrinho].

### 6.2 Matrícula e liberação de acesso [MVP]

```
(Webhook "paid") → (BE) cria/atualiza enrollment(status='active', source='purchase')
   → libera acesso ao curso → (Job) webhook de SAÍDA "compra aprovada" + e-mail
   → aluno acessa conteúdo (§7)
```

1. Disparado por `orders.status='paid'` (ou matrícula manual/CSV/auto — PRD §3.6).
2. (BE) `withTenant`: cria `enrollments(user_id, course_id, status='active', source)` (`unique(user_id, course_id)` — idempotente; se já existe suspensa, reativa).
3. Aplica `expires_at` se for assinatura/acesso temporário.
4. (Job) Webhooks de **saída** ("compra aprovada", "matrícula") para o CRM/automação do tenant; e-mail de confirmação ao aluno.
5. ✔ Curso aparece em **[Meus cursos]** com acesso liberado (sujeito a drip §4.4).
6. **Matrícula manual/CSV [MVP]:** admin em **[Alunos]** matricula avulso ou via **import CSV** → mesma criação de `enrollments(source='manual'|'bulk')` (sem order).

### 6.3 Order bump / upsell [F2]

```
[Checkout] → seção Order bump: oferta complementar (checkbox 1-clique)
   ⟂ aceitou? → soma ao pedido (mesma transação)
(pós-compra) [Tela: Upsell one-click] → "Adicionar à minha compra"
   ⟂ aceitou? → cobra no mesmo método/cartão tokenizado → cria order vinculada → matrícula extra
   ⟂ recusou? → [Downsell opcional] → segue para confirmação
```

1. **Order bump (no checkout):** oferta complementar exibida com checkbox 1-clique; ⟂ aceitou → adiciona item à mesma `order` (ou order irmã) antes do pagamento.
2. **Upsell pós-compra (one-click):** após confirmação, **[Upsell]** oferece produto adicional; ⟂ aceita → (BE) cobra **one-click** (cartão tokenizado) → cria nova `order` vinculada → nova `enrollment`.
3. ⟂ **Recusou upsell?** → **[Downsell]** opcional → senão segue para **[Compra confirmada]**.
4. (BE) cada item gera matrícula própria; afiliado/split recalculados por order.
5. ✔ Aluno recebe acesso a todos os itens adquiridos.

---

## 7. Consumo de conteúdo

### 7.1 Assistir aula (URL assinada, retomar, progresso) [MVP]

```
[Meus cursos] → [Player do curso] → seleciona aula
   → ⟂ entitlement válido? (enrollment ativo + drip liberado)
        ├─ não → ✖ tela bloqueada (matricule-se / disponível em...)
        └─ sim → (BE) gera Embed Token Bunny (TTL 1–12h) → player carrega HLS
   → player retoma de lesson_progress.position_seconds
   → timeupdate (throttle 5–15s) → POST /progress → atualiza watched_pct/position
   → ⟂ ≥90% assistido E anti-seek (tempo real ≥ duração×0.8)? → marca completed
   → ⟂ última exigência do curso concluída? → dispara certificado (§5.1)
```

1. Aluno abre **[Player do curso]** e seleciona uma aula no índice (módulos/aulas).
2. ⟂ **Entitlement válido?** (BE valida `enrollments.status='active'` **e** regra de **drip** §4.4 atendida):
   - Não matriculado → ✖ tela "matricule-se" (CTA checkout).
   - Drip não liberado → ✖ "Disponível em DD/MM" ou "em N dias após matrícula".
   - Suspenso/expirado → ✖ "acesso suspenso" + motivo (ver §9).
3. ⟂ **Tipo da aula:** vídeo → segue; texto → renderiza conteúdo; pdf → link de download (URL assinada R2).
4. (BE) Para vídeo: gera **Embed Token Bunny** (`SHA256(key+videoId+expires)`, **TTL curto 1–12h**); front renderiza player (HLS adaptativo; velocidade 0.5x–2x; legendas).
5. **Retomar:** player inicia em `lesson_progress.position_seconds`.
6. **Tracking:** `timeupdate` (throttle 5–15s) → `POST /progress` → (BE) atualiza `position_seconds`, `watched_pct`, `status='in_progress'`.
7. ⟂ **Conclusão:** `watched_pct ≥ 90%` **E** checagem **anti-seek** (tempo real assistido ≥ `duration × 0.8`) → `status='completed'`, `completed_at` (PRD §4.4).
8. ⟂ **Curso 100% (+ nota mínima quando houver)?** → dispara emissão de certificado (§5.1).
9. ✖ Token expira durante a sessão → player solicita **refresh** transparente do Embed Token (revalida entitlement).
10. **Anti-pirataria [MVP]:** MP4 progressivo off, restrição de referer, MediaCage Basic (transparente ao aluno).

### 7.2 Comentar numa aula [MVP]

```
[Player do curso → aba Comentários] → escreve → "Enviar"
   → ⟂ entitlement válido? não→✖ bloqueado / sim→ (BE) cria lesson_comments
   → exibe na thread (suporta resposta via parent_id)
   → autor/instrutor/admin podem editar/excluir (moderação)
```

1. Na aula, aluno abre aba **Comentários** e escreve uma dúvida.
2. ⟂ **Tem acesso à aula?** (entitlement) Não → ✖ bloqueado.
3. **Validações:** corpo não vazio; limites de tamanho; (anti-spam/rate-limit).
4. (BE) `withTenant`: `INSERT lesson_comments(lesson_id, user_id, body, parent_id?)`.
5. ✔ Comentário aparece na thread; **resposta** usa `parent_id` (1 nível). Notificação ao instrutor [F2].
6. **Moderação:** autor edita/exclui o próprio; instrutor/admin podem moderar (excluir/ocultar).

### 7.3 Interação de comunidade (fórum/feed/grupos) [F2]

```
[Comunidade do tenant] → escolhe espaço (curso/tópico/grupo)
   → cria post (forum_topics/forum_posts) → comentar/curtir
   → ⟂ acesso ao espaço? (matrícula no curso / grupo)
   → eventos/lives no calendário (embed Zoom/YouTube)
```

1. Aluno acessa **[Comunidade]** (feed geral, por curso ou por **grupo**).
2. ⟂ **Pode acessar o espaço?** (regra: matriculado no curso / membro do grupo).
3. Cria **tópico/post** (`forum_topics`, `forum_posts`); reage/comenta.
4. **Eventos/lives:** calendário com **lives nativas** (embed Zoom/YouTube Live).
5. (BE) `withTenant`: persiste; moderação por instrutor/admin.
6. ✔ Engajamento contabilizado para gamificação [F2] (pontos/badges).

---

## 8. Afiliados e split

```
[Painel do afiliado] → "Gerar link" → escolhe curso → link com ?ref=<code>
   → visitante clica → cookie de atribuição (janela N dias)
   → compra (§6.1) com affiliate_id atribuído
   → (Webhook "paid") → (BE) cria affiliate_commissions(status='pending')
       + aplica regra de split (splits/percent) no PaymentProvider (Pagar.me)
   → [Painel do afiliado] mostra comissão pendente → após liquidação → status='paid'
```

1. Afiliado em **[Painel do afiliado]** clica "Gerar link", escolhe o curso → recebe URL `tenant.app.com/curso/<slug>?ref=<code>` + materiais de divulgação.
2. **Atribuição:** visitante clica → (front) grava **cookie de atribuição** (janela de N dias, política do tenant); o `ref` é validado contra `affiliates.code`.
3. **Compra:** ao concluir checkout (§6.1), a `order` recebe `affiliate_id` (atribuição last-click padrão).
4. (BE) **Webhook "paid"** → cria `affiliate_commissions(amount_cents = order × commission_pct, status='pending')`.
5. **Split [MVP]:** regras em `splits` (course/recipient/percent) aplicadas via **Pagar.me** (split nativo entre produtor, afiliado e — F2 — coprodutores). O valor é dividido na liquidação do gateway.
6. ⟂ **Reembolso/chargeback posterior?** → comissão correspondente vira `status='reversed'` (ver §9).
7. **Acompanhamento:** afiliado vê em **[Comissões]**: vendas atribuídas, valores `pending`/`paid`/`reversed`, cliques (se rastreados), conversão.
8. ✔ Após liquidação pelo gateway → comissão `paid` (refletida no painel; pagamento operado pelo split).
9. ✖ Conflito de atribuição (sem afiliado / código inválido) → venda sem comissão; logado para auditoria.

---

## 9. Pagamento ↔ acesso: reembolso, cancelamento, suspensão

> Regra central (PRD §4.3): **pagamento governa acesso**. Webhooks idempotentes + reconciliação periódica.

### 9.1 Reembolso / chargeback [MVP]

```
(Origem A: aluno pede reembolso → admin aprova em [Pedidos])
(Origem B: chargeback/estorno vindo do gateway → webhook)
   → (BE) orders.status = refunded | chargeback
   → enrollments.status = refunded → revoga acesso ao curso
   → ⟂ certificado emitido? → opcional revogar (certificates.revoked=true)
   → affiliate_commissions correspondentes → reversed
   → (Job) webhook de SAÍDA "reembolso" + e-mail ao aluno
```

1. **Origem A (manual):** admin em **[Pedidos]** seleciona pedido → "Reembolsar" → confirma. (BE) chama `PaymentProvider.refund`.
2. **Origem B (gateway):** webhook de **reembolso/chargeback** chega (assinatura + idempotência).
3. (BE) `withTenant`: `orders.status='refunded'|'chargeback'`.
4. **Acesso:** `enrollments.status='refunded'` → aluno perde acesso ao curso (estado refletido no player §7.1).
5. ⟂ **Havia certificado?** → política do tenant pode **revogar** (`certificates.revoked=true`; verificação pública passa a invalidar).
6. **Afiliado:** comissões da order → `reversed` (§8.6).
7. (Job) webhook de **saída** "reembolso" + e-mail ao aluno.
8. ✔ Estados consistentes; reconciliação periódica confere divergências com o gateway.

### 9.2 Cancelamento de assinatura (do aluno) [MVP]

```
[Área do aluno → Minha assinatura] → "Cancelar"
   → ⟂ cancelar no fim do período? (padrão) → mantém acesso até current_period_end
   → (Webhook gateway) subscription canceled → na virada do período: enrollment expired
```

1. Aluno em **[Minha assinatura]** clica "Cancelar".
2. ⟂ **Política:** cancela ao fim do ciclo (mantém acesso até `current_period_end`) — padrão; ou imediato (raro).
3. (BE) chama `PaymentProvider`; `subscriptions.status='canceled'` (pendente fim de período).
4. (Webhook) na renovação não cobrada → na virada: `enrollments.status='expired'` → acesso encerrado.
5. ✔ Aluno informado da data de encerramento; pode reassinar.

### 9.3 Suspensão por inadimplência / reconciliação [MVP]

```
(Cobrança recorrente falhou → webhook past_due)
   → enrollments/subscriptions → suspended (após grace period, se houver)
   → player bloqueia acesso (§7.1) → e-mail "atualize seu pagamento"
   → ⟂ pagamento regularizado? → reativa (active)
(Job de reconciliação periódica corrige divergências pagamento↔acesso)
```

1. Cobrança recorrente falha → webhook `past_due`/falha.
2. ⟂ **Grace period configurado?** Sim → mantém ativo até fim do prazo; envia lembretes.
3. (BE) findo o prazo: `subscriptions.status='suspended'` + `enrollments.status='suspended'`.
4. Player bloqueia (§7.1); e-mail de regularização.
5. ⟂ **Regularizou?** → reativa (`active`).
6. **Reconciliação:** job agendado compara estado local × gateway e corrige inconsistências (idempotente).

### 9.4 Suspensão do TENANT (nível SaaS) — efeito no acesso

Quando o **Super-Admin** ou o billing do SaaS suspende o tenant (§10.2), **todos** os usuários daquele tenant perdem acesso (alunos veem "indisponível"; owner é direcionado a billing). Distinto da suspensão por inadimplência do aluno (acima).

---

## 10. Super-Admin (control plane)

> Super-Admin acessa o **painel da plataforma** (não um subdomínio de tenant), identidade global em `platform.super_admins`, com **MFA** e **auditoria** (`platform.audit_log`). Nunca há acesso implícito a dados de tenant sem registro.

### 10.1 Criar tenant (manualmente) [MVP]

```
[Super-Admin → Tenants] → "Novo tenant" → slug + plano + e-mail do admin
   → ⟂ slug livre? não→✖ / sim→ INSERT tenants(provisioning) → dispara saga (§3.1)
   → acompanha progresso da saga → ✔ active
```

1. Super-Admin em **[Tenants]** clica "Novo tenant" (criação assistida, ex.: vendas/onboarding manual).
2. Informa `slug`, `plan_id`, e-mail do admin inicial. ⟂ **Slug disponível?** Não → ✖.
3. (BE platform) `INSERT tenants(status='provisioning', idempotency_key)` → dispara a **saga** (§3.1).
4. Acompanha **progresso da saga** (passos em `provisioning_jobs`); ⟂ falha → opção de **reexecutar** passo (idempotente).
5. ✔ `status='active'`; e-mail de boas-vindas ao admin. Ação registrada em `audit_log`.

### 10.2 Suspender / ativar tenant [MVP]

```
[Super-Admin → Tenant detalhe] → "Suspender"
   → ⟂ confirmar (impacto: todos usuários perdem acesso)
   → tenants.status='suspended' → roteamento bloqueia área logada do tenant
   → audit_log registrado
"Ativar" → status='active' (reverte)
"Cancelar" → status='cancelled' (encerramento; dados retidos p/ DROP SCHEMA conforme LGPD §3.6)
```

1. Super-Admin abre **[Tenant detalhe]** e escolhe "Suspender" (motivo: inadimplência do SaaS, abuso, etc.).
2. ⟂ **Confirmação** explícita (impacto amplo — §9.4).
3. (BE platform) `tenants.status='suspended'`; resolução de tenant passa a bloquear acesso (owner → billing).
4. **Ativar:** reverte para `active`.
5. **Cancelar/excluir tenant (LGPD):** `status='cancelled'` → eventual `DROP SCHEMA tenant_<slug> CASCADE` (processo controlado, auditado).
6. Toda ação → `audit_log` (actor super-admin, ação, metadata, timestamp).

### 10.3 Impersonar (suporte) [MVP]

```
[Super-Admin → Tenant detalhe / Usuários] → "Impersonar"
   → ⟂ confirmar + motivo obrigatório → audit_log
   → emite sessão impersonada (escopo do tenant, marcação visível "modo suporte")
   → Super-Admin navega como o usuário (escopo limitado por política)
   → "Encerrar impersonação" → volta ao painel platform; fim registrado
```

1. Super-Admin escolhe usuário/tenant alvo e clica "Impersonar".
2. ⟂ **Motivo obrigatório** + confirmação → registra início em `audit_log` (actor real + alvo).
3. (BE) emite **sessão impersonada** escopada ao tenant (claims marcando impersonação); UI exibe **banner "modo suporte"** persistente.
4. Navega como o usuário (ações sensíveis podem ser restritas por política — ex.: não alterar senha, não ver dados de pagamento crus).
5. "Encerrar impersonação" → encerra sessão impersonada, retorna ao **[Painel super-admin]**; fim registrado no audit log.
6. **Isolamento:** a impersonação respeita `withTenant`; jamais cruza para outro tenant.

---

## 11. Máquinas de estado de referência

**Tenant (`platform.tenants.status`):**
```
provisioning ──(saga ok)──▶ active ──(suspender)──▶ suspended ──(ativar)──▶ active
     │                          │                                   │
     └──(saga falha)── (alerta) └──────────(cancelar)──────────────▶ cancelled ──▶ [DROP SCHEMA]
```

**Pedido (`orders.status`):**
```
pending ──(pago)──▶ paid ──(reembolso)──▶ refunded
   │                  └────(chargeback)──▶ chargeback
   └──(expira Pix/boleto)──▶ expired
```

**Matrícula / acesso (`enrollments.status`):**
```
active ──(reembolso)──▶ refunded
  │  ▲──(regulariza)─┐
  ├──(inadimplência)─┴▶ suspended
  └──(fim de período / expira)──▶ expired
```

**Conclusão de aula (`lesson_progress.status`):**
```
not_started ──(play)──▶ in_progress ──(≥90% + anti-seek)──▶ completed
```

**Comissão de afiliado (`affiliate_commissions.status`):**
```
pending ──(liquidação)──▶ paid
   └──(reembolso/chargeback da order)──▶ reversed
```

---

## Dependências e pontos para o coordenador

1. **Verificação de e-mail no cadastro (§2.1):** o PRD não fixa se a verificação é **obrigatória** antes do acesso. Decisão de produto necessária (sugestão: obrigatória para reduzir fraude, mas com conteúdo gratuito acessível antes). Impacta UX e Better-Auth.
2. **2FA de equipe/aluno (§2.2):** marcado como [F2] aqui; PRD só cita MFA para Super-Admin. Confirmar fase.
3. **Política de aprovação de afiliados (§2.5) e janela de atribuição/last-click (§8):** parâmetros por tenant não estão no DATA_MODEL (faltam campos de configuração de programa de afiliados e cookie window). Definir e refletir no modelo de dados/contracts.
4. **Order bump/upsell (§6.3):** é [F2] no PRD/ROADMAP — incluído como fluxo, mas depende de modelagem de "oferta" (bumps/upsells) ainda inexistente no DATA_MODEL.
5. **Revogação de certificado no reembolso (§9.1):** política (revogar ou manter) deve ser decidida por produto e, idealmente, configurável por tenant.
6. **Grace period de inadimplência (§9.3):** valor/existência não definidos; precisa de configuração (platform plan ou por tenant).
7. **Escopo da impersonação (§10.3):** definir quais ações ficam bloqueadas em modo suporte e granularidade da auditoria (LGPD).
8. **Atribuição de tenant em rotas públicas (verificação de certificado §5.3, landing):** confirmar como o `uuid_public`/slug resolve o tenant sem login mantendo isolamento.
9. **Recuperação de carrinho/Pix expirado (§6.1):** lembretes são [F2]; alinhar com módulo de e-mail marketing.
10. **Estados de assinatura do aluno vs. SaaS:** garantir que a UI distingue claramente "minha assinatura ao conteúdo" (aluno) de "assinatura do tenant à plataforma" (admin/owner) para evitar confusão.

> Próximos artefatos sugeridos: wireframes de baixa fidelidade das telas críticas (Checkout, Player, Editor de curso, Painel Super-Admin) e especificação dos schemas Zod em `packages/contracts` para os fluxos acima.
