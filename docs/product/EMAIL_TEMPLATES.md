# E-mails Transacionais — Templates, Estrutura e Deliverability

- **Versão:** 1.0 · **Data:** 2026-06-07
- **Autor:** UX Writing / Content Design
- **Status:** Proposta para revisão do coordenador de produto
- **Relacionados:** [NOTIFICATIONS_MATRIX.md](NOTIFICATIONS_MATRIX.md) (fonte dos eventos/placeholders) · [UX_WRITING.md](../design/UX_WRITING.md) (tom de voz) · [USER_FLOWS.md](USER_FLOWS.md) · [NON_FUNCTIONAL_REQUIREMENTS.md](NON_FUNCTIONAL_REQUIREMENTS.md) (i18n/LGPD/a11y) · [ADR-0012](../adr/0012-supporting-platform.md) (Resend) · [CLAUDE.md](../../CLAUDE.md)

> **Escopo.** Especifica os **e-mails transacionais** da plataforma: estrutura (assunto, preheader, corpo,
> CTA, rodapé), placeholders, versões **texto + HTML**, **branding por tenant** e requisitos de
> **deliverability/LGPD**. A matriz de quando/para quem disparar é a [NOTIFICATIONS_MATRIX](NOTIFICATIONS_MATRIX.md);
> aqui está **o conteúdo**. Provedor: **Resend** ([ADR-0012](../adr/0012-supporting-platform.md)) via port
> `EmailProvider`, **React Email** para HTML. **Regra nº1:** todo envio carrega `tenantId` no payload do
> job ([CLAUDE.md](../../CLAUDE.md)); branding/locale resolvidos **por tenant/destinatário**; nenhum
> e-mail mistura dados ou marca de tenants diferentes.

---

## Índice

1. [Arquitetura de envio e princípios](#1-arquitetura-de-envio-e-princípios)
2. [Anatomia padrão de um e-mail](#2-anatomia-padrão-de-um-e-mail)
3. [Branding por tenant (white-label)](#3-branding-por-tenant-white-label)
4. [Deliverability, LGPD e descadastro](#4-deliverability-lgpd-e-descadastro)
5. [Placeholders e helper de layout](#5-placeholders-e-helper-de-layout)
6. [Templates — Conta e segurança](#6-templates--conta-e-segurança)
7. [Templates — Pagamento e acesso](#7-templates--pagamento-e-acesso)
8. [Templates — Aprendizado e conteúdo](#8-templates--aprendizado-e-conteúdo)
9. [Templates — Afiliados e financeiro](#9-templates--afiliados-e-financeiro)
10. [Templates — Plataforma / Tenant (control plane)](#10-templates--plataforma--tenant-control-plane)
11. [Templates — Aulas ao vivo (F2)](#11-templates--aulas-ao-vivo-f2)
12. [Templates — Ciclo de vida / marketing (P3)](#12-templates--ciclo-de-vida--marketing-p3)
13. [Matriz de assuntos (resumo)](#13-matriz-de-assuntos-resumo)
14. [Dependências e pontos para o coordenador](#dependências-e-pontos-para-o-coordenador)

---

## 1. Arquitetura de envio e princípios

- **Provedor:** **Resend** atrás da port `EmailProvider` (DIP) — trocável por SES/Postmark sem mexer no
  domínio ([ADR-0012](../adr/0012-supporting-platform.md)). Render HTML com **React Email** (integra Next.js);
  cada template tem versão **HTML** e **texto puro** (multipart/alternative).
- **Disparo assíncrono:** sempre via `JobQueue` (pg-boss) com `tenantId` no payload, a partir de
  **transições idempotentes** das máquinas de estado ([NOTIFICATIONS_MATRIX §0](NOTIFICATIONS_MATRIX.md));
  reprocessar webhook **não** reenvia e-mail (dedup por `event_id`).
- **Transacional ≠ marketing:** domínios/subdomínios e reputação de IP separados ([ADR-0012](../adr/0012-supporting-platform.md)).
  **Transacional (P0–P1)** não traz descadastro; **marketing/ciclo de vida (P3)** exige opt-in + link de
  descadastro (§4).
- **i18n:** corpo, assunto e preheader localizados pelo `{locale}` do destinatário ([NFR-I18N-10](NON_FUNCTIONAL_REQUIREMENTS.md));
  PT-BR no MVP, arquitetura pronta para ES/EN. ICU para plural/datas; moeda via `Intl` (centavos+moeda).
- **Tom:** segue [UX_WRITING.md](../design/UX_WRITING.md) — claro, acolhedor, "você", neutro à marca,
  sem culpa, com próximo passo. Sem emoji em assuntos P0 (pagamento/segurança) — entregabilidade.
- **A11y do e-mail:** HTML semântico, `lang` no `<html>`, `alt` no logo (`{tenant_name}`), contraste
  AA do botão sobre `{brand_primary_color}` (fallback se o tenant escolheu cor de baixo contraste — §3),
  largura ~600px, responsivo, **dark-mode-safe** (cores explícitas), CTA como link estilizado (não só imagem).
- **Preferências:** honra `notification_preferences(channel='email', category, enabled)` para **P2/P3**;
  **P0/P1 não são suprimíveis** ([NOTIFICATIONS_MATRIX dep. 1](NOTIFICATIONS_MATRIX.md)).

---

## 2. Anatomia padrão de um e-mail

| Parte | Regra de conteúdo |
|-------|-------------------|
| **From** | `{tenant_name} <no-reply@{tenant_mail_domain}>` (ou domínio compartilhado verificado no MVP — §4). Nunca o nome da nossa empresa. |
| **Reply-To** | `{support_email}` do tenant (respostas vão ao tenant, não a nós). |
| **Assunto** | ≤ ~50 caracteres, específico, sem clickbait/CAPS/“!!!”. Inclui o essencial à frente (curso, status). |
| **Preheader** | 40–100 caracteres; complementa o assunto (não repete). Oculto visualmente no corpo. |
| **Cabeçalho** | Logo do tenant (`{tenant_logo_url}`, `alt="{tenant_name}"`), sobre fundo neutro. |
| **Saudação** | "Olá, {recipient_name}," (neutro — [UX_WRITING §3.1](../design/UX_WRITING.md)). |
| **Corpo** | 1 mensagem principal, frases curtas, 1 ideia por parágrafo. Dados relevantes em lista/“card”. |
| **CTA** | **1 CTA primário** (botão), verbo+objeto; URL = `{cta_url}` específico. CTA secundário só como link textual. |
| **Fechamento** | Linha de apoio ("Precisa de ajuda? Fale com a gente em {support_email}."). |
| **Rodapé** | Identificação do tenant, `{year}`, links legais; **descadastro só em P3** (§4). |

> **Regra de ouro:** todo e-mail tem **um** objetivo e **um** CTA primário. Informação secundária vira
> link textual, nunca um segundo botão concorrente.

---

## 3. Branding por tenant (white-label)

Resolvido por tenant no momento do render ([NOTIFICATIONS_MATRIX §8](NOTIFICATIONS_MATRIX.md)):

- **Logo:** `{tenant_logo_url}` no cabeçalho; `alt="{tenant_name}"`. Fallback: nome do tenant em texto
  estilizado se não houver logo.
- **Cor:** `{brand_primary_color}` aplica-se ao botão CTA e acentos. **Validação de contraste**: se a cor do
  tenant não atingir 4,5:1 com o texto do botão, o sistema escolhe texto branco/preto automaticamente ou cai
  para uma cor segura ([NFR-A11Y-03/20](NON_FUNCTIONAL_REQUIREMENTS.md)). A11y não pode depender do gosto do admin.
- **Domínio/remetente:** `{tenant_name}` no From; Reply-To `{support_email}`. Domínio de envio: ver §4.
- **URLs:** todos os links apontam para o host do tenant (`{tenant_url}` / subdomínio / domínio próprio F2).
- **Locale:** `{locale}` do destinatário define idioma e formatos.
- **Sem co-branding nosso:** o aluno nunca vê “powered by”/nossa marca (white-label). Exceção: e-mails do
  **control plane** ao **owner** (§10) podem usar a marca da plataforma (são B2B, nossos).

---

## 4. Deliverability, LGPD e descadastro

### 4.1 Entregabilidade

- **Autenticação de domínio:** SPF, **DKIM** e **DMARC** configurados no domínio de envio (via Resend).
  MVP: domínio **compartilhado verificado** (ex.: `mail.{plataforma}` com From exibindo `{tenant_name}`);
  **F2:** domínio próprio por tenant com verificação DNS (cruza [USER_FLOWS §3.2](USER_FLOWS.md)).
- **Separação de fluxos:** **subdomínio/IP transacional** distinto do de marketing — preserva reputação
  ([ADR-0012](../adr/0012-supporting-platform.md)).
- **Conteúdo:** proporção texto/imagem saudável, **sempre** versão texto puro, sem anexos pesados (links em
  vez de anexar PDF — certificado/boleto via URL), sem encurtadores suspeitos, links em domínio próprio.
- **Idempotência/anti-spam:** não reenviar em reprocessamento; agrupar P2/P3 (digest) para reduzir volume
  ([NOTIFICATIONS_MATRIX dep. 7](NOTIFICATIONS_MATRIX.md)).
- **Bounce/complaint:** tratar via webhooks do Resend; suprimir endereços com hard bounce/spam complaint.

### 4.2 LGPD — transacional vs marketing

| | **Transacional (P0–P1)** | **Marketing / ciclo de vida (P3)** |
|---|---|---|
| Base legal | Execução de contrato / legítimo interesse | **Consentimento** (opt-in separado, não pré-marcado) |
| Descadastro | **Não** traz link de marketing unsubscribe | **Obrigatório** `{unsubscribe_url}` + `List-Unsubscribe` header |
| Exemplos | Compra aprovada, reset de senha, certificado, matrícula | Recomendações, nudge de inatividade, carrinho abandonado |
| Supressível por preferências | Não (P0/P1) | Sim (honra `notification_preferences`) |

- **Rodapé transacional:** identificação do tenant (`{tenant_name}`), motivo do envio em 1 linha
  ("Você recebeu este e-mail porque tem uma conta em {tenant_name}."), `{year}`, links de Política de
  Privacidade/Termos. **Sem** unsubscribe de marketing (mas pode haver link para gerenciar preferências de
  notificação não-críticas).
- **Rodapé marketing (P3):** tudo do transacional **+** `{unsubscribe_url}` visível **+** header
  `List-Unsubscribe`/`List-Unsubscribe-Post` (one-click). Endereço/identificação do remetente conforme boa
  prática anti-spam.
- **Segurança:** e-mails de credencial (reset/verificação/senha alterada) **nunca** incluem a senha; tokens
  em link de uso único e TTL curto; informam IP/horário quando relevante ([NFR-SEC-04/06](NON_FUNCTIONAL_REQUIREMENTS.md)).
  Não pedir dados sensíveis por e-mail (anti-phishing).

---

## 5. Placeholders e helper de layout

### 5.1 Placeholders comuns (de [NOTIFICATIONS_MATRIX §8](NOTIFICATIONS_MATRIX.md))

- **Tenant/marca:** `{tenant_name}`, `{tenant_logo_url}`, `{brand_primary_color}`, `{tenant_url}`,
  `{support_email}`, `{locale}`.
- **Destinatário:** `{recipient_name}`, `{recipient_email}`, `{recipient_role}`.
- **Curso/aula:** `{course_title}`, `{course_url}`, `{lesson_title}`, `{lesson_url}`, `{instructor_name}`.
- **Comercial:** `{amount}`, `{currency}`, `{order_id}`, `{payment_method}`, `{net_amount}`.
- **Sistema:** `{cta_url}`, `{expires_at}`, `{unsubscribe_url}` (só P3), `{year}`, `{event_id}`.

> Cada template abaixo lista **placeholders próprios** além destes. Valores monetários chegam em centavos +
> moeda e são formatados via `Intl` no render. Datas em UTC → fuso do destinatário.

### 5.2 Layout compartilhado (DRY)

Um único componente `EmailLayout` (React Email) provê **header** (logo), **footer** (legal + condicional
unsubscribe) e tokens de marca; cada template injeta só **assunto, preheader, corpo e CTA**. Evita duplicar
estrutura por e-mail.

```
EmailLayout(tenant, locale, isMarketing)
 ├─ Header: logo {tenant_logo_url} (alt={tenant_name})
 ├─ {slot: corpo do template}
 ├─ CTA primário (cor = contraste-safe de {brand_primary_color})
 └─ Footer: "{tenant_name} · © {year}" · Privacidade · Termos
           [se isMarketing] · {unsubscribe_url} + header List-Unsubscribe
```

> **Convenção dos templates abaixo:** cada um traz **Template-id** (= nome em NOTIFICATIONS_MATRIX),
> **Prio**, **Assunto**, **Preheader**, **Corpo (HTML resumido)**, **Versão texto** e **CTA**. Placeholders
> entre `{}`. Listo a versão **texto** explícita só onde o conteúdo difere de forma relevante; nos demais, a
> versão texto é a transcrição linear do corpo + URL do CTA + rodapé.

---

## 6. Templates — Conta e segurança

### 6.1 `welcome` — Boas-vindas / confirmação de conta · P1

- **Placeholders próprios:** `{name}`, `{verify_url}`, `{tenant_brand}`.
- **Assunto:** `Boas-vindas a {tenant_name}`
- **Preheader:** `Sua conta está pronta. Confirme seu e-mail para começar.`
- **Corpo (HTML):**
  > Olá, {name}, <br>
  > Que bom ter você na {tenant_name}! Sua conta foi criada com sucesso. <br>
  > Para garantir o acesso completo, confirme seu e-mail. <br>
  > **[Confirmar e-mail]** ({verify_url})
- **CTA:** `Confirmar e-mail` → `{verify_url}`
- **Texto:** `Olá, {name}. Sua conta na {tenant_name} foi criada. Confirme seu e-mail: {verify_url}. Precisa de ajuda? {support_email}.`
- **Rodapé:** transacional (sem unsubscribe).

### 6.2 `email_verify` — Verificação de e-mail · P0

- **Placeholders próprios:** `{verify_url}`, `{expires_at}`.
- **Assunto:** `Confirme seu e-mail`
- **Preheader:** `Falta só um passo para ativar sua conta.`
- **Corpo:** "Confirme que este e-mail é seu para ativar sua conta. O link expira em {expires_at}." + botão.
- **CTA:** `Confirmar e-mail` → `{verify_url}`
- **Nota:** se não foi você, ignore este e-mail. Token de uso único. **Sem emoji.** Rodapé transacional.

### 6.3 `password_reset` — Redefinição de senha · P0

- **Placeholders próprios:** `{reset_url}`, `{expires_at}`, `{ip}`.
- **Assunto:** `Redefinir sua senha`
- **Preheader:** `Use o link para criar uma nova senha. Expira em breve.`
- **Corpo:**
  > Recebemos um pedido para redefinir a senha da sua conta na {tenant_name}. <br>
  > Para criar uma nova senha, use o botão abaixo. O link expira em {expires_at}. <br>
  > Se você não fez este pedido, ignore este e-mail — sua senha continua a mesma. (Solicitação registrada do IP {ip}.)
- **CTA:** `Criar nova senha` → `{reset_url}`
- **Segurança:** anti-enumeração (mesma resposta na UI exista ou não — [USER_FLOWS §2.3](USER_FLOWS.md)); a senha **nunca** vai no e-mail. Rodapé transacional.

### 6.4 `password_changed` — Senha alterada · P0

- **Placeholders próprios:** `{when}`, `{ip}`, `{support_url}`.
- **Assunto:** `Sua senha foi alterada`
- **Preheader:** `Se não foi você, fale com o suporte imediatamente.`
- **Corpo:** "A senha da sua conta na {tenant_name} foi alterada em {when} (IP {ip}). Se foi você, está tudo certo. Se não reconhece esta ação, proteja sua conta agora."
- **CTA:** `Falar com o suporte` → `{support_url}`
- Rodapé transacional.

### 6.5 `new_login` — Novo login/dispositivo (F2) · P1

- **Placeholders próprios:** `{device}`, `{ip}`, `{location}`, `{when}`.
- **Assunto:** `Novo acesso à sua conta`
- **Preheader:** `Detectamos um login em um novo dispositivo.`
- **Corpo:** "Houve um acesso à sua conta na {tenant_name} em {when}, de {device} ({location}, IP {ip}). Se foi você, ignore. Se não, redefina sua senha."
- **CTA:** `Proteger minha conta` → `{cta_url}` · Rodapé transacional.

### 6.6 `team_invite` — Convite para equipe · P1

- **Placeholders próprios:** `{inviter}`, `{role}`, `{accept_url}`.
- **Assunto:** `Você foi convidado para a equipe de {tenant_name}`
- **Preheader:** `{inviter} convidou você como {role}.`
- **Corpo:** "{inviter} convidou você para participar da {tenant_name} como **{role}**. Aceite o convite para acessar o painel."
- **CTA:** `Aceitar convite` → `{accept_url}` (TTL — [USER_FLOWS §2.4](USER_FLOWS.md)). Rodapé transacional.

### 6.7 `role_changed` — Alteração de papel · P1

- **Placeholders próprios:** `{new_role}`, `{changed_by}`.
- **Assunto:** `Seu acesso em {tenant_name} foi atualizado`
- **Corpo:** "{changed_by} atualizou seu papel na {tenant_name} para **{new_role}**. Suas permissões podem ter mudado."
- **CTA:** `Acessar painel` → `{cta_url}` · Rodapé transacional.

### 6.8 `account_status` — Conta suspensa/reativada · P1

- **Placeholders próprios:** `{status}`, `{reason}`.
- **Assunto (suspensa):** `Sua conta foi suspensa` · **(reativada):** `Sua conta foi reativada`
- **Corpo:** condicional por `{status}`; quando suspensa, explica `{reason}` sem culpa e oferece suporte.
- **CTA:** `Falar com o suporte` → `{support_url}` · Rodapé transacional.

### 6.9 `data_export_ready` — Dados exportados prontos (LGPD) · P1

- **Placeholders próprios:** `{download_url}`, `{expires_at}`.
- **Assunto:** `Seus dados estão prontos para download`
- **Corpo:** "Concluímos a exportação dos seus dados na {tenant_name}. O link abaixo expira em {expires_at}."
- **CTA:** `Baixar meus dados` → `{download_url}` ([NFR-LGPD-06](NON_FUNCTIONAL_REQUIREMENTS.md)). Rodapé transacional.

### 6.10 `data_deleted` — Conta/dados deletados (LGPD) · P1

- **Placeholders próprios:** `{when}`.
- **Assunto:** `Confirmação de exclusão de dados`
- **Corpo:** "Concluímos a exclusão dos seus dados na {tenant_name} em {when}, conforme seu pedido. Você não tem mais acesso a cursos e certificados associados."
- **CTA:** nenhum (ou `Falar com o suporte`). **Sem link de acesso** (conta encerrada). Rodapé transacional mínimo.

---

## 7. Templates — Pagamento e acesso

### 7.1 `purchase_approved` — Compra aprovada · P0 · (aluno)

- **Placeholders próprios:** `{student_name}`, `{course_title}`, `{order_id}`, `{amount}`, `{access_url}`.
- **Assunto:** `Compra confirmada: {course_title}`
- **Preheader:** `Seu acesso já está liberado. Comece agora.`
- **Corpo (HTML):**
  > Olá, {student_name}, <br>
  > Seu pagamento foi aprovado e seu acesso a **{course_title}** já está liberado. <br>
  > <br>
  > _Pedido_ {order_id} · _Valor_ {amount} <br>
  > **[Acessar curso]** ({access_url})
- **CTA:** `Acessar curso` → `{access_url}`
- **Texto:** `Olá, {student_name}. Pagamento aprovado! Acesso a {course_title} liberado. Pedido {order_id}, valor {amount}. Acesse: {access_url}.`
- Rodapé transacional.

### 7.2 `sale_received` — Nova venda recebida · P1 · (owner/admin, instrutor autor)

- **Placeholders próprios:** `{course_title}`, `{buyer_name}`, `{amount}`, `{net_amount}`.
- **Assunto:** `Nova venda: {course_title}`
- **Corpo:** "{buyer_name} comprou **{course_title}**. Valor {amount} · Líquido estimado {net_amount}."
- **CTA:** `Ver pedido` → `{cta_url}` · Tom operacional (B2B). Rodapé transacional.

### 7.3 `payment_failed` — Pagamento recusado · P1 · (aluno)

- **Placeholders próprios:** `{course_title}`, `{reason}`, `{retry_url}`.
- **Assunto:** `Não conseguimos aprovar seu pagamento`
- **Preheader:** `Você pode tentar novamente em alguns instantes.`
- **Corpo:** "Não foi possível aprovar o pagamento de **{course_title}**. Nenhuma cobrança foi feita. Você pode tentar outro cartão ou forma de pagamento." (Não expor `{reason}` técnico do banco; usar mensagem neutra — [UX_WRITING §6.3](../design/UX_WRITING.md).)
- **CTA:** `Tentar novamente` → `{retry_url}` · Rodapé transacional.

### 7.4 `payment_pending` — Pagamento pendente (Pix/boleto) · P1 · (aluno)

- **Placeholders próprios:** `{course_title}`, `{method}`, `{pix_code}`/`{boleto_url}`, `{expires_at}`.
- **Assunto:** `Falta pouco: conclua o pagamento de {course_title}`
- **Preheader:** `Seu {method} está esperando. Vence em {expires_at}.`
- **Corpo (condicional por método):**
  - **Pix:** "Para liberar seu acesso, pague o Pix abaixo. Ele expira em {expires_at}." + código copia-e-cola (`{pix_code}`) + botão.
  - **Boleto:** "Seu boleto foi gerado e vence em {expires_at}. A compensação pode levar 1 a 2 dias úteis." + botão para o PDF.
- **CTA:** `Ver detalhes do pagamento` → `{cta_url}` (Pix) / `Ver boleto` → `{boleto_url}`.
- **Texto (Pix):** inclui o código `{pix_code}` em linha própria para fácil cópia. Rodapé transacional.

### 7.5 `payment_expired` — Pix/boleto expirado · P2 · (aluno)

- **Placeholders próprios:** `{course_title}`, `{new_checkout_url}`.
- **Assunto:** `Seu pagamento de {course_title} expirou`
- **Corpo:** "O prazo do seu Pix/boleto para **{course_title}** terminou. Você pode gerar um novo pagamento quando quiser."
- **CTA:** `Refazer pagamento` → `{new_checkout_url}` · P2: respeita preferências. Rodapé transacional.

### 7.6 `enrollment_active` — Matrícula confirmada / acesso liberado · P0 · (aluno)

- **Placeholders próprios:** `{course_title}`, `{access_url}`, `{start_here}`.
- **Assunto:** `Tudo pronto! Seu acesso a {course_title} está liberado`
- **Preheader:** `Comece pela primeira aula.`
- **Corpo:** "Sua matrícula em **{course_title}** está ativa. Sugerimos começar por aqui: {start_here}."
- **CTA:** `Começar agora` → `{access_url}` · Rodapé transacional.

### 7.7 `enrollment_granted` — Matrícula manual/cortesia · P1 · (aluno)

- **Placeholders próprios:** `{course_title}`, `{granted_by}`, `{access_url}`.
- **Assunto:** `Você recebeu acesso a {course_title}`
- **Corpo:** "{granted_by} liberou seu acesso a **{course_title}** na {tenant_name}. Aproveite!"
- **CTA:** `Acessar curso` → `{access_url}` · Rodapé transacional.

### 7.8 `refund_processed` — Reembolso processado · P0 · (aluno)

- **Placeholders próprios:** `{course_title}`, `{amount}`, `{refund_id}`.
- **Assunto:** `Seu reembolso foi processado`
- **Corpo:** "Processamos o reembolso de {amount} referente a **{course_title}**. O valor pode levar alguns dias para aparecer, conforme sua forma de pagamento. _Reembolso_ {refund_id}." (Informar que o acesso ao curso foi encerrado.)
- **CTA:** `Falar com o suporte` → `{support_url}` · Rodapé transacional.

### 7.9 `refund_issued_admin` — Reembolso emitido (aviso ao tenant) · P1 · (owner/admin)

- **Placeholders próprios:** `{course_title}`, `{buyer_name}`, `{amount}`.
- **Assunto:** `Reembolso emitido: {course_title}`
- **Corpo:** "Um reembolso de {amount} foi emitido para {buyer_name} ({course_title}). O acesso foi encerrado e a comissão de afiliado, se houver, foi revertida."
- **CTA:** `Ver pedido` → `{cta_url}` · Tom operacional. Rodapé transacional.

### 7.10 `chargeback_opened` — Chargeback aberto · P0 · (owner/admin)

- **Placeholders próprios:** `{order_id}`, `{buyer_name}`, `{amount}`.
- **Assunto:** `Chargeback aberto no pedido {order_id}`
- **Corpo:** "Foi aberta uma disputa (chargeback) de {amount} no pedido {order_id} de {buyer_name}. O acesso foi suspenso. Você pode acompanhar e contestar pelo painel."
- **CTA:** `Ver pedido` → `{cta_url}` · Rodapé transacional.

### 7.11 `access_suspended` — Acesso suspenso · P1 · (aluno)

- **Placeholders próprios:** `{course_title}`, `{reason}`, `{resolve_url}`.
- **Assunto:** `Seu acesso a {course_title} foi suspenso`
- **Corpo:** "Seu acesso a **{course_title}** está suspenso. Regularize o pagamento para voltar a estudar." (Mensagem neutra; sem culpa.)
- **CTA:** `Regularizar pagamento` → `{resolve_url}` · Rodapé transacional.

### 7.12 `access_expired` — Acesso expirado · P2 · (aluno)

- **Placeholders próprios:** `{course_title}`, `{renew_url}`.
- **Assunto:** `Seu acesso a {course_title} expirou`
- **Corpo:** "O período de acesso a **{course_title}** terminou. Se quiser continuar, é possível renovar."
- **CTA:** `Renovar acesso` → `{renew_url}` · P2. Rodapé transacional.

### 7.13 `sub_payment_failed` — Assinatura: cobrança falhou (dunning) · P1 · (aluno)

- **Placeholders próprios:** `{plan}`, `{attempt}`, `{update_card_url}`, `{grace_until}`.
- **Assunto:** `Atualize seu pagamento para manter o acesso`
- **Preheader:** `Tivemos um problema na cobrança da sua assinatura.`
- **Corpo:** "Não conseguimos renovar sua assinatura **{plan}** (tentativa {attempt}). Atualize seus dados de pagamento até {grace_until} para não perder o acesso."
- **CTA:** `Atualizar pagamento` → `{update_card_url}` · Rodapé transacional.

### 7.14 `sub_renewed` — Assinatura: renovada · P2 · (aluno)

- **Placeholders próprios:** `{plan}`, `{period_end}`, `{amount}`.
- **Assunto:** `Sua assinatura foi renovada`
- **Corpo:** "Renovamos sua assinatura **{plan}** ({amount}). Acesso garantido até {period_end}."
- **CTA:** `Acessar conteúdo` → `{cta_url}` · P2. Rodapé transacional.

### 7.15 `sub_canceled` — Assinatura: cancelada · P1 · (aluno)

- **Placeholders próprios:** `{plan}`, `{access_until}`.
- **Assunto:** `Sua assinatura foi cancelada`
- **Corpo:** "Sua assinatura **{plan}** foi cancelada. Você mantém acesso até {access_until}. Pode reassinar quando quiser."
- **CTA:** `Reassinar` → `{cta_url}` · Rodapé transacional.

### 7.16 `trial_ending` — Trial terminando · P2 · (aluno)

- **Placeholders próprios:** `{plan}`, `{trial_end}`, `{upgrade_url}`.
- **Assunto:** `Seu período de teste termina em breve`
- **Corpo:** "Seu teste do **{plan}** termina em {trial_end}. Continue com acesso total assinando agora."
- **CTA:** `Assinar agora` → `{upgrade_url}` · P2. Rodapé transacional.

---

## 8. Templates — Aprendizado e conteúdo

### 8.1 `video_failed` — Falha no processamento de vídeo · P1 · (instrutor, owner/admin)

- **Placeholders próprios:** `{lesson_title}`, `{error}`, `{reupload_url}`.
- **Assunto:** `Falha ao processar o vídeo de "{lesson_title}"`
- **Corpo:** "O processamento do vídeo da aula **{lesson_title}** não foi concluído. Você pode reenviar o arquivo para tentar de novo." (`{error}` resumido, sem stack.)
- **CTA:** `Reenviar vídeo` → `{reupload_url}` · Tom operacional. Rodapé transacional.

> **Nota:** `video_ready` é **in-app apenas** (sem e-mail, conforme [NOTIFICATIONS_MATRIX §2](NOTIFICATIONS_MATRIX.md)).

### 8.2 `lesson_unlocked` — Nova aula liberada (drip) · P2 · (aluno)

- **Placeholders próprios:** `{lesson_title}`, `{course_title}`, `{lesson_url}`, `{release_at}`.
- **Assunto:** `Nova aula liberada em {course_title}`
- **Preheader:** `"{lesson_title}" já está disponível.`
- **Corpo:** "A aula **{lesson_title}** de **{course_title}** acabou de ser liberada. Bons estudos!"
- **CTA:** `Assistir agora` → `{lesson_url}` · P2 (respeita preferências). Rodapé transacional.

### 8.3 `course_published` — Curso publicado (para inscritos/lista) · P2 · (aluno opt-in)

- **Placeholders próprios:** `{course_title}`, `{landing_url}`.
- **Assunto:** `Novo curso disponível: {course_title}`
- **Corpo:** "A {tenant_name} acaba de publicar **{course_title}**. Confira o conteúdo."
- **CTA:** `Conhecer o curso` → `{landing_url}` · P2/opt-in. Rodapé transacional (lista de interesse — pode oferecer gerenciar preferências).

### 8.4 `comment_reply` — Nova resposta em comentário · P2 · (aluno/instrutor autor do tópico)

- **Placeholders próprios:** `{replier_name}`, `{lesson_title}`, `{thread_url}`.
- **Assunto:** `{replier_name} respondeu seu comentário`
- **Corpo:** "{replier_name} respondeu seu comentário na aula **{lesson_title}**."
- **CTA:** `Ver resposta` → `{thread_url}` · P2 (preferência por tipo — [LEARNING_EXPERIENCE_UX §4.4](LEARNING_EXPERIENCE_UX.md)). Rodapé transacional + link "gerenciar notificações".

### 8.5 `comment_moderated` — Comentário moderado/removido · P2 · (aluno autor)

- **Placeholders próprios:** `{lesson_title}`, `{reason}`.
- **Assunto:** `Seu comentário foi removido`
- **Corpo:** "Seu comentário na aula **{lesson_title}** foi removido por um moderador. Motivo: {reason}." (Tom respeitoso, sem hostilidade.)
- **CTA:** `Ver diretrizes da comunidade` → `{cta_url}` (se houver) · P2. Rodapé transacional.

### 8.6 `certificate_issued` — Certificado emitido · P1 · (aluno)

- **Placeholders próprios:** `{course_title}`, `{cert_url}`, `{verify_url}`, `{qr}`.
- **Assunto:** `Seu certificado de {course_title} está pronto 🎉`
- **Preheader:** `Baixe e compartilhe sua conquista.`
- **Corpo (HTML):**
  > Parabéns, {recipient_name}! <br>
  > Você concluiu **{course_title}**. Seu certificado já está disponível. <br>
  > **[Baixar certificado]** ({cert_url}) <br>
  > Link de verificação: {verify_url}
- **CTA:** `Baixar certificado` → `{cert_url}` (URL assinada — o PDF **não** vai anexado, por entregabilidade/peso).
- **Texto:** `Parabéns, {recipient_name}! Certificado de {course_title} pronto. Baixe: {cert_url}. Verificação: {verify_url}.`
- Rodapé transacional.

### 8.7 `certificate_revoked` — Certificado revogado · P1 · (aluno)

- **Placeholders próprios:** `{course_title}`, `{reason}`.
- **Assunto:** `Seu certificado de {course_title} foi revogado`
- **Corpo:** "Informamos que seu certificado de **{course_title}** foi revogado. Motivo: {reason}. A página pública de verificação passará a indicar isso."
- **CTA:** `Falar com o suporte` → `{support_url}` · Rodapé transacional.

> **In-app apenas (sem e-mail):** `new_question`, `quiz_result`, `progress_milestone`, `coupon_feedback`
> ([NOTIFICATIONS_MATRIX §1–2](NOTIFICATIONS_MATRIX.md)).

---

## 9. Templates — Afiliados e financeiro

### 9.1 `affiliate_approved` — Afiliação aprovada · P1 · (afiliado)

- **Placeholders próprios:** `{program}`, `{links_url}`, `{materials_url}`.
- **Assunto:** `Você foi aprovado como afiliado de {tenant_name}`
- **Corpo:** "Sua participação no programa **{program}** foi aprovada. Gere seus links e acesse os materiais de divulgação."
- **CTA:** `Gerar meus links` → `{links_url}` (secundário: `Materiais` → `{materials_url}`). Rodapé transacional.

### 9.2 `affiliate_status` — Afiliação recusada/suspensa · P1 · (afiliado)

- **Placeholders próprios:** `{reason}`.
- **Assunto:** `Atualização sobre sua afiliação em {tenant_name}`
- **Corpo:** condicional: recusada/suspensa, com `{reason}` e próximo passo (reenviar solicitação, se a política permitir).
- **CTA:** `Falar com o suporte` → `{support_url}` · Rodapé transacional.

### 9.3 `commission_earned` — Nova comissão gerada · P2 · (afiliado)

- **Placeholders próprios:** `{course_title}`, `{amount}`, `{order_id}`.
- **Assunto:** `Você ganhou uma nova comissão`
- **Corpo:** "Uma venda de **{course_title}** gerou {amount} em comissão para você (pedido {order_id}). Ela ficará pendente até a liquidação."
- **CTA:** `Ver comissões` → `{cta_url}` · P2. Rodapé transacional.

### 9.4 `commission_paid` — Comissão paga · P1 · (afiliado)

- **Placeholders próprios:** `{amount}`, `{period}`, `{payout_ref}`.
- **Assunto:** `Sua comissão foi paga`
- **Corpo:** "Pagamos {amount} em comissões referentes a {period}. _Referência_ {payout_ref}."
- **CTA:** `Ver extrato` → `{cta_url}` · Rodapé transacional.

### 9.5 `commission_reversed` — Comissão revertida · P1 · (afiliado)

- **Placeholders próprios:** `{amount}`, `{reason}`, `{order_id}`.
- **Assunto:** `Uma comissão foi revertida`
- **Corpo:** "A comissão de {amount} do pedido {order_id} foi revertida. Motivo: {reason} (ex.: reembolso ou chargeback)."
- **CTA:** `Ver comissões` → `{cta_url}` · Rodapé transacional.

### 9.6 `coproduction_payout` — Repasse de co-produção (F2) · P1 · (instrutor/produtor)

- **Placeholders próprios:** `{course_title}`, `{amount}`, `{period}`.
- **Assunto:** `Repasse de co-produção: {course_title}`
- **Corpo:** "Seu repasse de co-produção de **{course_title}** referente a {period} foi de {amount}."
- **CTA:** `Ver detalhes` → `{cta_url}` · Rodapé transacional.

---

## 10. Templates — Plataforma / Tenant (control plane)

> **B2B, destinatário owner/super-admin.** Aqui o e-mail **pode** usar a marca da plataforma (não é
> white-label do tenant). From: nossa plataforma.

### 10.1 `tenant_welcome` — Tenant provisionado (boas-vindas) · P1 · (owner)

- **Placeholders próprios:** `{owner_name}`, `{admin_url}`, `{subdomain}`.
- **Assunto:** `Sua escola está no ar!`
- **Corpo:** "Olá, {owner_name}. Seu ambiente foi provisionado e está disponível em {subdomain}. Acesse o painel para configurar marca, criar cursos e ativar pagamentos."
- **CTA:** `Acessar o painel` → `{admin_url}` ([USER_FLOWS §3.1](USER_FLOWS.md)). Rodapé da plataforma.

### 10.2 `provisioning_failed` — Falha de provisionamento · P0 · (super-admin)

- **Placeholders próprios:** `{tenant}`, `{step}`, `{error}`.
- **Assunto:** `[Alerta] Falha no provisionamento de {tenant}`
- **Corpo:** interno/operacional: tenant, passo `{step}`, erro `{error}`, link para reexecutar a saga.
- **CTA:** `Abrir tenant` → `{cta_url}` · Interno (não white-label).

### 10.3 `saas_invoice` — Fatura do SaaS emitida · P1 · (owner)

- **Placeholders próprios:** `{amount}`, `{due_at}`, `{pay_url}`.
- **Assunto:** `Sua fatura de {tenant_name} está disponível`
- **Corpo:** "Sua fatura da plataforma no valor de {amount} vence em {due_at}."
- **CTA:** `Ver fatura` → `{pay_url}` (Stripe Billing — control plane). Rodapé da plataforma.

### 10.4 `saas_payment_failed` — Cobrança do SaaS falhou · P0 · (owner)

- **Placeholders próprios:** `{amount}`, `{grace_until}`, `{update_url}`.
- **Assunto:** `Falha na cobrança da sua assinatura`
- **Corpo:** "Não conseguimos cobrar {amount} da sua assinatura da plataforma. Atualize o pagamento até {grace_until} para evitar a suspensão da sua escola."
- **CTA:** `Atualizar pagamento` → `{update_url}` · Rodapé da plataforma.

### 10.5 `tenant_suspended` — Tenant suspenso (inadimplência SaaS) · P0 · (owner)

- **Placeholders próprios:** `{reason}`, `{reactivate_url}`.
- **Assunto:** `Sua escola foi suspensa`
- **Corpo:** "Sua escola na plataforma foi suspensa. Motivo: {reason}. Regularize para reativar o acesso de todos os seus usuários."
- **CTA:** `Reativar agora` → `{reactivate_url}` · Rodapé da plataforma.

### 10.6 `tenant_reactivated` — Tenant reativado · P1 · (owner)

- **Assunto:** `Sua escola foi reativada`
- **Corpo:** "Tudo certo! Sua escola voltou a funcionar normalmente em {when}." · CTA `Acessar painel` → `{cta_url}`.

### 10.7 `quota_warning` — Plano/quota próximo do limite · P2 · (owner/admin)

- **Placeholders próprios:** `{resource}`, `{used}`, `{limit}`, `{upgrade_url}`.
- **Assunto:** `Você está perto do limite de {resource}`
- **Corpo:** "Você já usou {used} de {limit} {resource} do seu plano. Considere um upgrade para não interromper suas operações."
- **CTA:** `Ver planos` → `{upgrade_url}` · P2. Rodapé da plataforma.

### 10.8 `quota_exceeded` — Quota excedida (ação bloqueada) · P1 · (owner/admin)

- **Placeholders próprios:** `{resource}`, `{limit}`, `{upgrade_url}`.
- **Assunto:** `Limite de {resource} atingido`
- **Corpo:** "Você atingiu o limite de {limit} {resource} do seu plano. Faça upgrade para liberar mais."
- **CTA:** `Fazer upgrade` → `{upgrade_url}` · Rodapé da plataforma.

### 10.9 `tenant_canceled` — Cancelamento de tenant agendado · P0 · (owner)

- **Placeholders próprios:** `{access_until}`, `{export_url}`, `{purge_at}`.
- **Assunto:** `Cancelamento agendado da sua escola`
- **Corpo:** "Sua escola será encerrada. Você mantém acesso até {access_until}. Exporte seus dados antes de {purge_at}, quando serão removidos permanentemente."
- **CTA:** `Exportar dados` → `{export_url}` (LGPD). Rodapé da plataforma.

> **In-app apenas:** `impersonation_notice` (transparência ao owner — config; sem e-mail por padrão,
> [NOTIFICATIONS_MATRIX §4](NOTIFICATIONS_MATRIX.md)).

---

## 11. Templates — Aulas ao vivo (F2)

> Feature **F2** ([ADR-0015](../adr/0015-live-classes-interactive.md)). Destinatário: aluno matriculado.
> Lembretes são sensíveis a tempo — assunto direto, CTA = entrar/calendário.

| Template | Prio | Assunto | Corpo (resumo) | CTA |
|----------|:----:|---------|----------------|-----|
| `live_scheduled` | P2 | Aula ao vivo agendada: {live_title} | "{live_title} de {course_title} acontece em {start_at}. Adicione ao seu calendário." | `Adicionar ao calendário` → `{add_to_calendar_url}` |
| `live_reminder_24h` | P2 | Amanhã: {live_title} | "Sua aula ao vivo {live_title} é em {start_at}." | `Ver detalhes` → `{join_url}` |
| `live_reminder_10m` | P2 | Começa em 10 min: {live_title} | "A aula ao vivo {live_title} está prestes a começar." | `Entrar na sala` → `{join_url}` |
| `live_rescheduled` | P1 | {live_title} foi remarcada | "A aula {live_title} foi remarcada para {new_start_at}." | `Ver novo horário` → `{cta_url}` |
| `live_canceled` | P1 | {live_title} foi cancelada | "A aula ao vivo {live_title} foi cancelada." | `Ver curso` → `{course_url}` |
| `live_replay_ready` | P2 | Replay disponível: {live_title} | "A gravação de {live_title} ({course_title}) já está disponível." | `Assistir replay` → `{lesson_url}` |
| `live_recording_failed` | P1 | Falha na gravação de {live_title} | (instrutor/admin) "Não foi possível gerar a gravação de {live_title}." | `Tentar novamente` → `{retry_url}` |

> `live_started` ("ao vivo agora") é **in-app/push apenas** (sem e-mail — latência).

---

## 12. Templates — Ciclo de vida / marketing (P3)

> **P3 — exige opt-in (LGPD) + `{unsubscribe_url}` + header `List-Unsubscribe`** (§4.2). Honra
> `notification_preferences`. Tom mais relacional, ainda neutro à marca.

### 12.1 `onboarding_drip` — Sequência de boas-vindas pós-matrícula · P3 · (aluno)

- **Placeholders próprios:** `{course_title}`, `{next_step}`.
- **Cadência:** D0/D1/D3 ([NOTIFICATIONS_MATRIX §6](NOTIFICATIONS_MATRIX.md)).
- **Assunto (ex. D1):** `Como vai seu primeiro dia em {course_title}?`
- **Corpo:** dica + próximo passo (`{next_step}`), encorajador.
- **CTA:** `Continuar curso` → `{cta_url}` · **Rodapé marketing** (unsubscribe).

### 12.2 `inactivity_nudge` — Lembrete de inatividade · P3 · (aluno)

- **Placeholders próprios:** `{course_title}`, `{resume_url}`.
- **Assunto:** `Que tal continuar {course_title}?`
- **Corpo:** "Faz um tempo que você não estuda **{course_title}**. Retome de onde parou — leva poucos minutos."
- **CTA:** `Retomar` → `{resume_url}` · Rodapé marketing.

### 12.3 `abandoned_cart` — Carrinho abandonado (F2) · P3 · (aluno/lead)

- **Placeholders próprios:** `{course_title}`, `{resume_checkout_url}`, `{coupon}` (opcional).
- **Assunto:** `Você esqueceu algo: {course_title}`
- **Corpo:** "Sua compra de **{course_title}** não foi concluída. Finalize quando quiser." (+ cupom opcional `{coupon}`.)
- **CTA:** `Concluir compra` → `{resume_checkout_url}` · Rodapé marketing.

### 12.4 `weekly_digest` — Resumo semanal do instrutor · P3 · (instrutor, owner/admin)

- **Placeholders próprios:** `{sales}`, `{enrollments}`, `{completion}`, `{top_course}`.
- **Assunto:** `Seu resumo da semana em {tenant_name}`
- **Corpo:** card com {sales}, {enrollments}, {completion}, {top_course}.
- **CTA:** `Ver painel completo` → `{cta_url}` · Rodapé marketing (gerenciar digest).

### 12.5 `course_recommendation` — Recomendação de curso (F3) · P3 · (aluno opt-in)

- **Placeholders próprios:** `{courses[]}`.
- **Assunto:** `Cursos selecionados para você`
- **Corpo:** lista de `{courses[]}` (título + capa + link).
- **CTA:** `Ver recomendações` → `{cta_url}` · Rodapé marketing.

---

## 13. Matriz de assuntos (resumo)

| Template | Prio | Assunto (PT-BR) |
|----------|:----:|-----------------|
| `welcome` | P1 | Boas-vindas a {tenant_name} |
| `email_verify` | P0 | Confirme seu e-mail |
| `password_reset` | P0 | Redefinir sua senha |
| `password_changed` | P0 | Sua senha foi alterada |
| `new_login` | P1 | Novo acesso à sua conta |
| `team_invite` | P1 | Você foi convidado para a equipe de {tenant_name} |
| `role_changed` | P1 | Seu acesso em {tenant_name} foi atualizado |
| `account_status` | P1 | Sua conta foi suspensa / reativada |
| `data_export_ready` | P1 | Seus dados estão prontos para download |
| `data_deleted` | P1 | Confirmação de exclusão de dados |
| `purchase_approved` | P0 | Compra confirmada: {course_title} |
| `sale_received` | P1 | Nova venda: {course_title} |
| `payment_failed` | P1 | Não conseguimos aprovar seu pagamento |
| `payment_pending` | P1 | Falta pouco: conclua o pagamento de {course_title} |
| `payment_expired` | P2 | Seu pagamento de {course_title} expirou |
| `enrollment_active` | P0 | Tudo pronto! Seu acesso a {course_title} está liberado |
| `enrollment_granted` | P1 | Você recebeu acesso a {course_title} |
| `refund_processed` | P0 | Seu reembolso foi processado |
| `refund_issued_admin` | P1 | Reembolso emitido: {course_title} |
| `chargeback_opened` | P0 | Chargeback aberto no pedido {order_id} |
| `access_suspended` | P1 | Seu acesso a {course_title} foi suspenso |
| `access_expired` | P2 | Seu acesso a {course_title} expirou |
| `sub_payment_failed` | P1 | Atualize seu pagamento para manter o acesso |
| `sub_renewed` | P2 | Sua assinatura foi renovada |
| `sub_canceled` | P1 | Sua assinatura foi cancelada |
| `trial_ending` | P2 | Seu período de teste termina em breve |
| `video_failed` | P1 | Falha ao processar o vídeo de "{lesson_title}" |
| `lesson_unlocked` | P2 | Nova aula liberada em {course_title} |
| `course_published` | P2 | Novo curso disponível: {course_title} |
| `comment_reply` | P2 | {replier_name} respondeu seu comentário |
| `comment_moderated` | P2 | Seu comentário foi removido |
| `certificate_issued` | P1 | Seu certificado de {course_title} está pronto 🎉 |
| `certificate_revoked` | P1 | Seu certificado de {course_title} foi revogado |
| `affiliate_approved` | P1 | Você foi aprovado como afiliado de {tenant_name} |
| `affiliate_status` | P1 | Atualização sobre sua afiliação em {tenant_name} |
| `commission_earned` | P2 | Você ganhou uma nova comissão |
| `commission_paid` | P1 | Sua comissão foi paga |
| `commission_reversed` | P1 | Uma comissão foi revertida |
| `coproduction_payout` | P1 | Repasse de co-produção: {course_title} |
| `tenant_welcome` | P1 | Sua escola está no ar! |
| `provisioning_failed` | P0 | [Alerta] Falha no provisionamento de {tenant} |
| `saas_invoice` | P1 | Sua fatura de {tenant_name} está disponível |
| `saas_payment_failed` | P0 | Falha na cobrança da sua assinatura |
| `tenant_suspended` | P0 | Sua escola foi suspensa |
| `tenant_reactivated` | P1 | Sua escola foi reativada |
| `quota_warning` | P2 | Você está perto do limite de {resource} |
| `quota_exceeded` | P1 | Limite de {resource} atingido |
| `tenant_canceled` | P0 | Cancelamento agendado da sua escola |
| `live_scheduled` (F2) | P2 | Aula ao vivo agendada: {live_title} |
| `live_reminder_24h` (F2) | P2 | Amanhã: {live_title} |
| `live_reminder_10m` (F2) | P2 | Começa em 10 min: {live_title} |
| `live_rescheduled` (F2) | P1 | {live_title} foi remarcada |
| `live_canceled` (F2) | P1 | {live_title} foi cancelada |
| `live_replay_ready` (F2) | P2 | Replay disponível: {live_title} |
| `live_recording_failed` (F2) | P1 | Falha na gravação de {live_title} |
| `onboarding_drip` (P3) | P3 | Como vai seu primeiro dia em {course_title}? |
| `inactivity_nudge` (P3) | P3 | Que tal continuar {course_title}? |
| `abandoned_cart` (F2/P3) | P3 | Você esqueceu algo: {course_title} |
| `weekly_digest` (P3) | P3 | Seu resumo da semana em {tenant_name} |
| `course_recommendation` (F3/P3) | P3 | Cursos selecionados para você |

---

## Dependências e pontos para o coordenador

1. **Domínio de envio (MVP vs F2):** decidir se o MVP usa **domínio compartilhado verificado** (From com
   `{tenant_name}`) ou já oferece **domínio próprio por tenant** com verificação DNS (SPF/DKIM/DMARC).
   Impacta entregabilidade e o fluxo de marca/domínio ([USER_FLOWS §3.2](USER_FLOWS.md)). Recomendação:
   compartilhado no MVP, próprio em F2.
2. **Reply-To e suporte:** confirmar que todo tenant tem `{support_email}` válido no onboarding; sem ele,
   definir fallback (e nunca cair no nosso e-mail — white-label).
3. **Catálogo canônico de `event types` ↔ templates:** alinhar com [NOTIFICATIONS_MATRIX dep. 6](NOTIFICATIONS_MATRIX.md):
   um **enum único** em `packages/contracts` mapeando evento → `template_id` (DRY entre e-mail, in-app,
   webhooks). Este doc usa os mesmos `template_id` da matriz.
4. **Anexar vs linkar PDF (certificado/boleto):** definimos **linkar** (URL assinada) por entregabilidade
   e peso. Confirmar com produto/segurança o TTL dos links em e-mail (mais longo que o da sessão?).
5. **i18n dos templates:** PT-BR no MVP, estrutura pronta para ES/EN ([NFR-I18N-10](NON_FUNCTIONAL_REQUIREMENTS.md)).
   Definir onde vivem os catálogos de e-mail (mesmos `messages/*` do app ou pacote dedicado de e-mail) e
   como o React Email consome ICU. Cruza com [UX_WRITING §4](../design/UX_WRITING.md).
6. **List-Unsubscribe e CMP:** padronizar header `List-Unsubscribe`/`List-Unsubscribe-Post` (one-click) nos
   P3 e a integração com a central de preferências/consentimento ([NFR-LGPD-01/05](NON_FUNCTIONAL_REQUIREMENTS.md)).
7. **Digest/throttling (P2/P3):** definir agrupamento (ex.: resumo diário de respostas de comentários) para
   reduzir ruído e custo ([NOTIFICATIONS_MATRIX dep. 7](NOTIFICATIONS_MATRIX.md)). Impacta `comment_reply`,
   `lesson_unlocked`, `commission_earned`.
8. **Dunning de assinatura/SaaS:** número de tentativas e janelas (`grace_until`, `attempt`) em
   `sub_payment_failed` e `saas_payment_failed` dependem dos parâmetros de [BUSINESS_RULES] ainda pendentes
   ([README §4, itens 18 e dep. 5 da matriz](README.md)).
9. **Co-branding do control plane:** confirmar que e-mails ao **owner** (§10) usam a marca da plataforma
   (B2B), enquanto **todos** os e-mails ao aluno/instrutor/afiliado são white-label do tenant.
10. **Validação de contraste do botão (a11y):** especificar a regra de fallback de cor do CTA quando
    `{brand_primary_color}` reprova em AA ([NFR-A11Y-03/20](NON_FUNCTIONAL_REQUIREMENTS.md)) — alinhar com o
    seletor de marca do tenant.
11. **Revisão jurídica/LGPD:** rodapés, base legal por categoria e textos de exclusão/exportação de dados
    (§6.9/6.10, §4.2) devem passar por revisão jurídica (dep. 24 do [README](README.md)).
