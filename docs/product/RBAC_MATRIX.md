# Matriz de Permissões (RBAC)

- **Versão:** 1.0 · **Data:** 2026-06-04
- **Status:** Proposto para revisão do coordenador
- **Relacionados:** [PRD.md](../PRD.md) · [ARCHITECTURE.md](../ARCHITECTURE.md) (§6 Auth/RBAC) · [DATA_MODEL.md](../DATA_MODEL.md) · [BUSINESS_RULES_AND_STATES.md](BUSINESS_RULES_AND_STATES.md) · [NOTIFICATIONS_MATRIX.md](NOTIFICATIONS_MATRIX.md)

> RBAC por tenant, aplicado em guards/hooks Fastify (ponto único → DRY) e **revalidado nos use-cases**
> (defense in depth). Papéis do data plane: `owner | admin | instructor | affiliate | student`
> (`users.role`). Super-Admin é identidade global no schema `platform` com impersonação auditada.

---

## Índice

- [0. Convenções e legenda](#0-convenções-e-legenda)
- [1. Papéis e escopo](#1-papéis-e-escopo)
- [2. Regras de escopo (tenant boundary)](#2-regras-de-escopo-tenant-boundary)
- [3. Matriz granular de permissões](#3-matriz-granular-de-permissões)
  - [3.1 Plataforma / Control plane](#31-plataforma--control-plane-super-admin)
  - [3.2 Configuração do tenant e marca](#32-configuração-do-tenant-e-marca)
  - [3.3 Equipe e usuários](#33-equipe-e-usuários)
  - [3.4 Cursos e conteúdo](#34-cursos-e-conteúdo)
  - [3.5 Vídeo](#35-vídeo)
  - [3.6 Avaliações e certificados](#36-avaliações-e-certificados)
  - [3.7 Matrículas e alunos](#37-matrículas-e-alunos)
  - [3.8 Monetização e financeiro](#38-monetização-e-financeiro)
  - [3.9 Afiliados e split](#39-afiliados-e-split)
  - [3.10 Comunidade](#310-comunidade)
  - [3.11 Marketing](#311-marketing)
  - [3.12 Analytics e dados (LGPD)](#312-analytics-e-dados-lgpd)
  - [3.13 Aprendizado (visão do aluno)](#313-aprendizado-visão-do-aluno)
  - [3.14 Integrações e API](#314-integrações-e-api)
- [4. Condições (notas C1..Cn)](#4-condições-notas-c1cn)
- [Dependências e pontos para o coordenador](#dependências-e-pontos-para-o-coordenador)

---

## 0. Convenções e legenda

- **✅ Permitido** · **❌ Negado** · **🔶 Condicional** (ver nota Cn na coluna observações ou §4).
- Colunas: **SA** Super-Admin · **OW** Owner · **AD** Admin · **IN** Instrutor · **AF** Afiliado ·
  **ST** Aluno (student).
- Super-Admin (SA) age no **control plane**; sobre dados de tenant só via **impersonação auditada**
  (🔶 = sempre com `audit_log`). SA **não** tem acesso silencioso a conteúdo de tenant.
- Toda permissão é **restrita ao tenant** do ator (exceto SA, global). Ver §2.

---

## 1. Papéis e escopo

| Papel | Schema | Descrição | Escopo |
|-------|--------|-----------|--------|
| **Super-Admin (SA)** | `platform` | Operador do SaaS (nós) | Global; impersonação auditada |
| **Owner (OW)** | `tenant_<slug>` | Dono da escola; superusuário do tenant | Todo o tenant |
| **Admin (AD)** | `tenant_<slug>` | Gestor delegado pelo owner | Todo o tenant, exceto atos exclusivos do owner |
| **Instrutor (IN)** | `tenant_<slug>` | Cria/gerencia conteúdo | Seus cursos (e atribuídos) |
| **Afiliado (AF)** | `tenant_<slug>` | Promove cursos por comissão | Seus links/comissões |
| **Aluno (ST)** | `tenant_<slug>` | Consome cursos | Suas matrículas/dados |

---

## 2. Regras de escopo (tenant boundary)

1. **Tudo é restrito ao tenant.** Todo acesso a dados de tenant passa por `withTenant(tenantId, fn)`;
   `tenant_id` vem da sessão/JWT. Nenhum papel de tenant enxerga outro tenant.
2. **Super-Admin é global, mas auditado.** SA opera em `platform` (tenants, planos, billing SaaS).
   Para tocar dados de um tenant, **impersona** com registro em `platform.audit_log`. Não há acesso
   implícito (PRD §4.7).
3. **Instrutor é escopado aos próprios cursos.** Vê/edita cursos onde é `instructor_id` (ou atribuído).
   Não vê financeiro global nem gerencia equipe.
4. **Afiliado e Aluno são escopados a si mesmos.** Afiliado vê suas comissões/links; Aluno vê suas
   matrículas/progresso/certificados/dados.
5. **Owner > Admin.** Owner detém atos sensíveis exclusivos (transferir propriedade, fechar conta SaaS,
   trocar plano, expor/rotacionar chaves de integração). Admin faz a gestão operacional.
6. **Defense in depth.** Mesmo com guard de rota, o use-case revalida papel/escopo (ownership) antes
   de agir.

---

## 3. Matriz granular de permissões

### 3.1 Plataforma / Control plane (Super-Admin)

| Ação / Recurso | SA | OW | AD | IN | AF | ST | Obs |
|----------------|----|----|----|----|----|----|-----|
| Provisionar tenant | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | Saga onboarding |
| Suspender/ativar tenant | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | `audit_log` |
| Cancelar/purgar tenant (DROP SCHEMA) | ✅ | 🔶 | ❌ | ❌ | ❌ | ❌ | C1 (owner solicita; SA executa) |
| Gerir planos do SaaS + quotas | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | — |
| Ver/gerir billing do SaaS (Stripe) | ✅ | 🔶 | ❌ | ❌ | ❌ | ❌ | C2 (owner vê/paga o próprio) |
| Observabilidade global | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | — |
| Impersonar usuário de tenant | 🔶 | ❌ | ❌ | ❌ | ❌ | ❌ | C3 (sempre auditado) |
| Ler `audit_log` global | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | — |
| Gerir super-admins/MFA | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | — |

### 3.2 Configuração do tenant e marca

| Ação / Recurso | SA | OW | AD | IN | AF | ST | Obs |
|----------------|----|----|----|----|----|----|-----|
| Configurar branding (logo/cores) | 🔶 | ✅ | ✅ | ❌ | ❌ | ❌ | C3 |
| Configurar subdomínio | 🔶 | ✅ | ✅ | ❌ | ❌ | ❌ | C3 |
| Configurar domínio próprio (F2) | 🔶 | ✅ | 🔶 | ❌ | ❌ | ❌ | C4 (admin se delegado) |
| Trocar plano do SaaS | 🔶 | ✅ | ❌ | ❌ | ❌ | ❌ | C2 |
| Configurar i18n/idioma do tenant (F2) | 🔶 | ✅ | ✅ | ❌ | ❌ | ❌ | C3 |
| Transferir propriedade do tenant | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | Ato exclusivo do owner |

### 3.3 Equipe e usuários

| Ação / Recurso | SA | OW | AD | IN | AF | ST | Obs |
|----------------|----|----|----|----|----|----|-----|
| Convidar/criar membro de equipe | 🔶 | ✅ | ✅ | ❌ | ❌ | ❌ | C3 |
| Atribuir/alterar papéis (instrutor/admin) | 🔶 | ✅ | 🔶 | ❌ | ❌ | ❌ | C5 (admin não cria/promove a owner) |
| Atribuir papel **owner** | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | Só owner (transferência) |
| Remover/desativar membro | 🔶 | ✅ | 🔶 | ❌ | ❌ | ❌ | C5 |
| Gerir afiliados (aprovar/comissão/status) | 🔶 | ✅ | ✅ | ❌ | ❌ | ❌ | C3 |
| Editar próprio perfil | 🔶 | ✅ | ✅ | ✅ | ✅ | ✅ | Todos no próprio |
| Resetar senha de aluno (suporte) | 🔶 | ✅ | ✅ | ❌ | ❌ | ❌ | C3 |

### 3.4 Cursos e conteúdo

| Ação / Recurso | SA | OW | AD | IN | AF | ST | Obs |
|----------------|----|----|----|----|----|----|-----|
| Criar curso | 🔶 | ✅ | ✅ | ✅ | ❌ | ❌ | C3 |
| Editar curso | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 (instrutor só os seus) |
| Publicar/despublicar curso | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 |
| Arquivar/desarquivar curso | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 |
| Definir preço/pricing do curso | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6+C7 (preço pode exigir admin) |
| Criar/editar/reordenar módulos e aulas | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 |
| Publicar/despublicar aula | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 |
| Configurar drip/pré-requisitos | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 |
| Subir anexos/PDF (lesson_assets) | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 |
| Excluir curso/aula | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 (soft-delete) |

### 3.5 Vídeo

| Ação / Recurso | SA | OW | AD | IN | AF | ST | Obs |
|----------------|----|----|----|----|----|----|-----|
| Iniciar upload (pré-assinatura TUS) | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 |
| Substituir/re-encode vídeo | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 |
| Gerar Embed Token (reprodução) | ❌ | 🔶 | 🔶 | 🔶 | ❌ | 🔶 | C8 (só com entitlement; sistema gera) |
| Ver/gerir Bunny library/keys | ❌ | 🔶 | ❌ | ❌ | ❌ | ❌ | C9 (chaves nunca no front) |

### 3.6 Avaliações e certificados

| Ação / Recurso | SA | OW | AD | IN | AF | ST | Obs |
|----------------|----|----|----|----|----|----|-----|
| Criar/editar quiz e questões | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 |
| Configurar nota mínima/tentativas/tempo | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 |
| Responder quiz (tentativa) | ❌ | ❌ | ❌ | ❌ | ❌ | 🔶 | C10 (aluno matriculado) |
| Ver gradebook/notas dos alunos (F2) | 🔶 | ✅ | ✅ | 🔶 | ❌ | 🔶 | C6/C11 (aluno só a própria) |
| Emitir certificado (gatilho) | ❌ | ❌ | ❌ | ❌ | ❌ | 🔶 | C12 (automático na conclusão) |
| Revogar certificado | 🔶 | ✅ | ✅ | ❌ | ❌ | ❌ | C3 + `audit_log` |
| Configurar template de certificado (F3) | 🔶 | ✅ | ✅ | ❌ | ❌ | ❌ | C3 |
| Verificar certificado (página pública) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | Público (UUID/QR) |

### 3.7 Matrículas e alunos

| Ação / Recurso | SA | OW | AD | IN | AF | ST | Obs |
|----------------|----|----|----|----|----|----|-----|
| Matricular manualmente / em massa (CSV) | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 (instrutor nos seus cursos) |
| Configurar auto-enroll por regra | 🔶 | ✅ | ✅ | ❌ | ❌ | ❌ | C3 |
| Suspender/reativar matrícula manual | 🔶 | ✅ | ✅ | ❌ | ❌ | ❌ | C3 |
| Ver progresso de todos os alunos | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 (instrutor nos seus) |
| Ver progresso próprio | ❌ | ✅ | ✅ | ✅ | ❌ | ✅ | Próprio |

### 3.8 Monetização e financeiro

| Ação / Recurso | SA | OW | AD | IN | AF | ST | Obs |
|----------------|----|----|----|----|----|----|-----|
| Ver financeiro do tenant (receita/pedidos) | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C3 / C13 (instrutor só seus cursos, se habilitado) |
| Configurar gateway/credenciais pagamento | 🔶 | ✅ | 🔶 | ❌ | ❌ | ❌ | C4 + C9 |
| Criar/editar cupons | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 |
| Emitir reembolso | 🔶 | ✅ | ✅ | ❌ | ❌ | ❌ | C3 + `audit_log` |
| Ver/exportar relatórios financeiros | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C13 |
| Comprar curso / iniciar checkout | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | Aluno (e visitante público) |
| Configurar order bump/upsell (F2) | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 |

### 3.9 Afiliados e split

| Ação / Recurso | SA | OW | AD | IN | AF | ST | Obs |
|----------------|----|----|----|----|----|----|-----|
| Gerir programa de afiliados (políticas) | 🔶 | ✅ | ✅ | ❌ | ❌ | ❌ | C3 |
| Aprovar/recusar afiliado | 🔶 | ✅ | ✅ | ❌ | ❌ | ❌ | C3 |
| Configurar split/co-produção (F2) | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 |
| Ver/gerar links de afiliado próprios | ❌ | ✅ | ✅ | ✅ | ✅ | ❌ | Afiliado nos próprios |
| Ver painel de comissões próprio | ❌ | ✅ | ✅ | 🔶 | ✅ | ❌ | Afiliado/instrutor nas suas |
| Ver todas as comissões do tenant | 🔶 | ✅ | ✅ | ❌ | ❌ | ❌ | C3 |
| Pagar/reverter comissão (gatilho) | 🔶 | ✅ | ✅ | ❌ | ❌ | ❌ | C3 (e via reconciliação) |
| Acessar materiais de divulgação | ❌ | ✅ | ✅ | ✅ | ✅ | ❌ | — |

### 3.10 Comunidade

| Ação / Recurso | SA | OW | AD | IN | AF | ST | Obs |
|----------------|----|----|----|----|----|----|-----|
| Comentar em aula | ❌ | ✅ | ✅ | ✅ | ❌ | 🔶 | C10 (aluno matriculado) |
| Responder/comentar como instrutor | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 |
| Moderar comentários (editar/remover) | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 (instrutor nos seus cursos) |
| Editar/excluir comentário próprio | ❌ | ✅ | ✅ | ✅ | ❌ | ✅ | Próprio |
| Gerir fórum/grupos (F2) | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 |

### 3.11 Marketing

| Ação / Recurso | SA | OW | AD | IN | AF | ST | Obs |
|----------------|----|----|----|----|----|----|-----|
| Editar landing de curso / SEO | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 |
| Configurar pixels/tracking (Meta/GA4) | 🔶 | ✅ | ✅ | ❌ | ❌ | ❌ | C3 |
| Configurar e-mail marketing/automação (F2) | 🔶 | ✅ | ✅ | ❌ | ❌ | ❌ | C3 |

### 3.12 Analytics e dados (LGPD)

| Ação / Recurso | SA | OW | AD | IN | AF | ST | Obs |
|----------------|----|----|----|----|----|----|-----|
| Ver analytics do tenant | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C13 |
| Exportar dados do tenant (CSV) | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C13 |
| Solicitar exportação dos próprios dados | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | LGPD (todos no próprio) |
| Solicitar deleção/anonimização própria | ❌ | 🔶 | 🔶 | 🔶 | 🔶 | ✅ | C14 (owner não se autodeleta sem transferir) |
| Executar deleção de aluno (LGPD) | 🔶 | ✅ | ✅ | ❌ | ❌ | ❌ | C3 + `audit_log` |
| Ler trilha de auditoria do tenant (F2) | 🔶 | ✅ | 🔶 | ❌ | ❌ | ❌ | C4 |

### 3.13 Aprendizado (visão do aluno)

| Ação / Recurso | SA | OW | AD | IN | AF | ST | Obs |
|----------------|----|----|----|----|----|----|-----|
| Assistir aula (com entitlement) | ❌ | 🔶 | 🔶 | 🔶 | ❌ | ✅ | C8 (staff só preview/impersonação) |
| Registrar progresso (heartbeats) | ❌ | 🔶 | 🔶 | 🔶 | ❌ | ✅ | C8 |
| Baixar certificado próprio | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | Aluno |
| Preview de curso não publicado | 🔶 | ✅ | ✅ | 🔶 | ❌ | ❌ | C6 (autor/admin) |

### 3.14 Integrações e API

| Ação / Recurso | SA | OW | AD | IN | AF | ST | Obs |
|----------------|----|----|----|----|----|----|-----|
| Configurar webhooks de saída | 🔶 | ✅ | ✅ | ❌ | ❌ | ❌ | C3 |
| Gerar/rotacionar API keys do tenant (F2) | 🔶 | ✅ | 🔶 | ❌ | ❌ | ❌ | C4 + C9 |
| Receber webhooks de entrada (sistema) | n/a | n/a | n/a | n/a | n/a | n/a | Sem ator humano; HMAC + idempotência |

---

## 4. Condições (notas C1..Cn)

- **C1 — Cancelar/purgar tenant:** owner **solicita** o cancelamento; SA **executa** o purge
  (`DROP SCHEMA`) após janela de retenção. Owner não dispara DROP diretamente.
- **C2 — Billing do SaaS:** owner vê/paga a **própria** assinatura (portal Stripe); SA gere todas.
- **C3 — Impersonação/escopo admin:** quando SA, **somente** via impersonação auditada
  (`platform.audit_log`). Para OW/AD a ação é direta dentro do próprio tenant.
- **C4 — Delegação ao Admin:** ações marcadas como exclusivas/sensíveis podem ser delegadas pelo owner
  ao admin via configuração de permissões finas (ver Dependências). Default: negado ao admin.
- **C5 — Gestão de equipe pelo Admin:** admin gerencia membros, **mas não** cria/promove a `owner` nem
  remove o owner. Admin não promove outro a admin se a política exigir owner (configurável).
- **C6 — Escopo do instrutor:** instrutor age **apenas** em cursos onde é `instructor_id` (ou
  explicitamente atribuído). Sobre cursos de terceiros: negado.
- **C7 — Definição de preço:** publicar/precificar pode ser restrito a OW/AD (proteção de receita);
  default permite instrutor nos próprios cursos — configurável por tenant.
- **C8 — Reprodução = entitlement:** Embed Token só é gerado pelo backend com `enrollment.active`,
  vídeo `ready`, drip/pré-requisito satisfeitos (BUSINESS_RULES §11.2). Staff sem matrícula só vê via
  preview/impersonação (auditada). Aluno suspenso/expirado: negado.
- **C9 — Segredos:** chaves Bunny/pagamento **nunca** expostas ao front (CLAUDE.md anti-padrão);
  cifradas em repouso; manipuláveis só por backend/owner via fluxo seguro.
- **C10 — Aluno matriculado:** responder quiz e comentar exigem matrícula `active` no curso da aula.
- **C11 — Gradebook:** aluno vê **apenas** as próprias notas; staff vê conforme escopo (C6).
- **C12 — Emissão de certificado:** disparada pelo **sistema** na conclusão (BUSINESS_RULES §7), não
  por ação manual de usuário; aluno é o titular/destinatário.
- **C13 — Financeiro/analytics do instrutor:** instrutor vê dados **apenas dos próprios cursos** e
  somente se o tenant habilitar visibilidade financeira a instrutores (default: negado a financeiro
  global; permitido a métricas de engajamento dos próprios cursos).
- **C14 — Deleção da própria conta:** todos podem solicitar; owner **não** pode autodeletar/anonimizar
  sem antes transferir a propriedade do tenant (evita tenant órfão).

---

## Dependências e pontos para o coordenador

1. **Permissões finas vs papéis fixos:** o modelo assume papéis fixos (`users.role` único). Várias
   células 🔶 (C4/C7/C13) pressupõem **delegação configurável** owner→admin/instrutor. Definir se haverá
   tabela de permissões/flags por tenant ou se ficamos só com papéis fixos no MVP.
2. **Papel múltiplo:** `users.role` é único no DATA_MODEL. Um usuário que é instrutor **e** afiliado?
   Definir se papéis são exclusivos ou compostos (impacta modelagem).
3. **Visibilidade financeira ao instrutor (C13):** confirmar default (engajamento sim, financeiro
   global não) e granularidade (por curso).
4. **Definição de preço por instrutor (C7):** confirmar se instrutor pode precificar ou se é exclusivo
   de OW/AD.
5. **Impersonação (C3):** confirmar escopo (read-only vs full) e exigência de MFA do super-admin.
6. **Visitante público (não autenticado):** landing/catálogo/checkout/verificação de certificado são
   públicos; mapear como pseudo-papel "guest" se necessário no front.
7. **Trilha de auditoria do tenant (3.12):** é F2; confirmar quem lê (owner sempre; admin condicional).
