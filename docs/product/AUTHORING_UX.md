# UX de Autoria / Instrutor (Estúdio)

- **Versão:** 1.0 · **Data:** 2026-06-04 · **Status:** Reconciliado
- **Papéis:** Instrutor (IN), Admin (AD), Owner (OW) — ver [RBAC_MATRIX](RBAC_MATRIX.md)
- **Relacionados:** [LEARNING_EXPERIENCE_UX](LEARNING_EXPERIENCE_UX.md) · [USER_FLOWS](USER_FLOWS.md) · [INFORMATION_ARCHITECTURE](INFORMATION_ARCHITECTURE.md) · [BUSINESS_RULES_AND_STATES](BUSINESS_RULES_AND_STATES.md) · [DATA_MODEL §2/§6](../DATA_MODEL.md) · [MONETIZATION](MONETIZATION.md)

> Fecha a lacuna registrada em [OPEN_QUESTIONS #26](../OPEN_QUESTIONS.md) e [README de produto §6](README.md).
> Este doc especifica a área `/manage` (Estúdio) consumida implicitamente por outros docs — em especial os
> **toggles de autoria** que a sala de aula do aluno respeita.

---

## 1. Objetivo e princípios

O **Estúdio** é onde IN/AD/OW criam, configuram, publicam e gerenciam cursos, aulas, avaliações, preços e
alunos. Princípios:

- **Rascunho seguro:** nada fica visível ao aluno até `published`. Edição não afeta quem já consome até publicar.
- **Autosave + estado explícito:** salva automaticamente; sempre indica `salvando…/salvo/erro` (sem perda de trabalho).
- **Pré-visualização fiel:** "ver como aluno" reflete drip, pré-requisitos e bloqueios reais.
- **Configuração em cascata:** padrões no **tenant** → sobrescritos no **curso** → sobrescritos na **aula**.
- **Isolamento (Regra nº1):** todo dado via `withTenant`; o IN só acessa **seus** cursos (RBAC).
- **WCAG AA** no editor (ver [NFR](NON_FUNCTIONAL_REQUIREMENTS.md)).

---

## 2. Mapa do Estúdio (telas)

Host/área `/manage` (ver [IA](INFORMATION_ARCHITECTURE.md), hosts = proposta a validar com eng.):

| Tela | Propósito | Papéis | Fase |
|------|-----------|--------|------|
| Dashboard do Estúdio | Visão geral (cursos, alunos, engajamento, pendências) | IN/AD/OW | MVP |
| Lista de cursos | Buscar/filtrar/criar curso; status | IN(seus)/AD/OW | MVP |
| Editor de curso › Currículo | Módulos/aulas (drag-drop) | IN/AD/OW | MVP |
| Editor de aula | Conteúdo por tipo (vídeo/texto/pdf/quiz/live) | IN/AD/OW | MVP |
| Configurações do curso | Capa, descrição, conclusão, comentários, drip, preço | IN/AD/OW | MVP |
| Quiz builder | Questões, política de tentativas, gabarito | IN/AD/OW | MVP |
| Biblioteca de mídia | Reuso de vídeos/arquivos | IN/AD/OW | F2 |
| Alunos do curso | Matricular/suspender, ver progresso | AD/OW (IN: leitura) | MVP |
| Ofertas/Preços | Venda avulsa/assinatura, cupons; order bump/upsell | AD/OW | MVP / F2 |
| Afiliados | Aprovar/gerir afiliados e comissões | AD/OW | MVP |
| Comentários (moderação) | Fila de moderação por curso | IN/AD/OW | MVP |

---

## 3. Gestão de cursos

- **Lista:** colunas título, status (`draft|published|archived`), nº de aulas, alunos, conclusão %, preço.
  Filtros por status/instrutor; busca. Ações: editar, duplicar, arquivar, ver como aluno, publicar.
- **Criar curso:** modal mínimo (título) → cria `draft` → abre editor. Slug autogerado (editável, único).
- **Configurações do curso** (abas): _Geral_ (título, slug, descrição rica, capa, idioma, instrutor),
  _Conteúdo_ (conclusão, comentários, drip), _Preço/Oferta_, _Certificado_, _Avançado_ (arquivar).

---

## 4. Editor de currículo (módulos e aulas)

- Árvore **Módulo → Aula** com **drag-and-drop** para reordenar (persiste `position`); reordenar não
  republica nem quebra progresso.
- Adicionar módulo/aula inline; aula nasce `draft` com tipo selecionável.
- Indicadores por aula: tipo, status de publicação, **`video_status`** (none/queued/processing/ready/failed),
  drip configurado, **aula opcional** (badge), quiz anexado.
- **Aula opcional:** toggle por aula → não conta para a % de conclusão do curso nem bloqueia certificado
  (ver §8 toggles). Estados vazios: "Nenhum módulo ainda — comece criando o Módulo 1".

---

## 5. Editor de aula por tipo

### 5.1 Vídeo (Bunny)
- **Upload** direto via TUS (resumable) com barra de progresso, retomada após falha, e cartão de
  **estado de processamento** (`queued→processing→ready`). Enquanto não `ready`, a aula pode ser editada
  mas não publicada para o aluno (ver [BUSINESS_RULES](BUSINESS_RULES_AND_STATES.md) — máquina de vídeo).
- Pós-`ready`: thumbnail (auto/custom), duração, legendas (upload VTT; auto-transcrição = F2), capítulos (F2).
- **Materiais/anexos** (PDFs no R2) e descrição rica da aula.
- Toggle **conclusão manual** (§8): se ligado, o aluno marca concluída; se desligado, conclusão automática
  por anti-seek (`≥90%` + `real_watched_seconds ≥ duration×0.8`).

### 5.2 Texto rico
- Editor WYSIWYG (headings, listas, imagens, embeds, código). Salva em `lessons.content jsonb`.

### 5.3 PDF / Download
- Upload de arquivo(s) → `lesson_assets` (R2). Opção "exigir visualização para concluir".

### 5.4 Quiz
- **Builder de questões:** múltipla escolha e V/F (MVP); dissertativa/lacuna/ordenar (F2).
- **Política de tentativas** (`max_attempts`: ilimitado/N), **nota mínima** (`pass_score`), **tempo**
  (`time_limit_sec`), **embaralhar** (F2).
- **Exibir gabarito** (§8): nunca / após enviar / só quando aprovado.
- Quiz pode ser uma **aula** (`type=quiz`) ou anexado a um curso (avaliação final).

### 5.5 Live (F2)
- Embed de Zoom/YouTube Live + data/hora; aparece no calendário do aluno.

---

## 6. Drip / liberação programada (UX)

- Por aula: **sem drip** | **data fixa** (`drip_release_at`) | **N dias após matrícula**
  (`drip_days_after_enroll`). Pré-visualização: "Esta aula libera em D+7".
- Pré-requisito sequencial (F2): "liberar só após concluir a aula anterior".
- A sala de aula do aluno renderiza o estado **bloqueado (drip/pré-req)** correspondente.

---

## 7. Comentários e moderação (autoria)

- Toggle **comentários** em cascata: tenant (padrão) → curso → aula (ver §8).
- **Fila de moderação** por curso: ocultar/denunciar/restaurar, marcar **resolvido**, responder como
  instrutor (badge). Espelha o modelo de [LEARNING_EXPERIENCE_UX](LEARNING_EXPERIENCE_UX.md) e
  `comment_*` ([DATA_MODEL §6.3](../DATA_MODEL.md)).

---

## 8. Matriz de toggles de autoria (resolve OPEN_QUESTIONS #26)

> Cascata: **tenant** (padrão global) → **curso** → **aula** (a mais específica vence). Campos a confirmar/
> consolidar no [DATA_MODEL §6](../DATA_MODEL.md) com a coordenação (ver §12).

| Toggle | Nível(is) | Valores | Default | Efeito no aluno | Campo (proposto) |
|--------|-----------|---------|---------|-----------------|------------------|
| **Conclusão manual de vídeo** | tenant/curso/aula | auto \| manual | `auto` | `auto`: conclui por anti-seek; `manual`: botão "marcar concluída" | `completion_mode` |
| **Aula opcional** | aula | sim \| não | `não` | opcional não conta para % nem para certificado | `lessons.is_optional` |
| **Comentários** | tenant/curso/aula | habilitado \| desabilitado | `habilitado` | mostra/oculta seção de comentários | `comments_enabled` |
| **Exibir gabarito (quiz)** | curso/quiz | nunca \| após_enviar \| se_aprovado | `após_enviar` | quando o aluno vê respostas certas | `quizzes.reveal_answers` |
| **Política de tentativas (quiz)** | quiz | ilimitado \| N | `ilimitado` | limita reenvios | `quizzes.max_attempts` |
| **Nota mínima (quiz)** | quiz | % | `null` (sem corte) | define aprovação/bloqueio | `quizzes.pass_score` |
| **Exigir visualização (texto/pdf)** | aula | sim \| não | `não` | marca conclusão ao abrir/baixar | `lessons.require_view` |
| **Certificado: exige nota mínima** | curso | sim \| não | `não` | condiciona emissão à aprovação no quiz | `courses.cert_requires_pass` |

> **Regra de conclusão de curso:** % = aulas concluídas / aulas **não-opcionais** publicadas. Certificado
> emitido a 100% das não-opcionais (e nota mínima, se exigida) — ver [BUSINESS_RULES](BUSINESS_RULES_AND_STATES.md).

---

## 9. Precificação, ofertas e alunos (resumo, detalhe em MONETIZATION)

- **Preço:** gratuito | avulso (`price_cents`) | assinatura. Cupons. Order bump/upsell = F2 (ofertas).
- **Afiliados:** aprovar/gerir, % de comissão, ver atribuição (ver [MONETIZATION](MONETIZATION.md)).
- **Alunos do curso:** matricular manual/CSV, suspender, ver progresso. Financeiro **global** não é
  visível ao IN (RBAC); engajamento sim.

---

## 10. Publicação e pré-visualização

- **Checklist de publicação** (bloqueia publicar se faltar): curso tem ≥1 aula publicável; vídeos `ready`;
  preço definido (se pago); capa e descrição presentes.
- Fluxo `draft → published` (e `archived`). Publicar uma aula nova num curso já publicado a disponibiliza
  conforme regra de drip.
- **"Ver como aluno":** abre a sala de aula em modo preview, respeitando drip/pré-req/bloqueios.

---

## 11. Estados, microinterações e analytics

- **Estados sempre presentes:** vazio (sem cursos/aulas), carregando, salvando/salvo/erro (autosave),
  upload em progresso, vídeo processando, erro de publicação (checklist).
- **Eventos de analytics** (ver [ANALYTICS_AND_DASHBOARDS](ANALYTICS_AND_DASHBOARDS.md), todos com `tenant_id`):
  `course_created`, `course_published`, `lesson_created`, `video_upload_started/completed`,
  `quiz_created`, `course_settings_updated` (com qual toggle), `student_enrolled_manual`,
  `comment_moderated`.

---

## 12. Dependências e pontos para o coordenador

1. **DATA_MODEL §6 — confirmar campos dos toggles:** vários toggles da §8 precisam de colunas/`jsonb`
   canônicos (`completion_mode`, `is_optional`, `comments_enabled`, `reveal_answers`, `require_view`,
   `cert_requires_pass`). Proponho padronizar como colunas em `courses`/`lessons`/`quizzes` + defaults em
   `tenant_settings` (cascata). **Requer alinhamento/ADR** se divergir do que já foi modelado.
2. **Cascata de configuração:** formalizar a precedência tenant→curso→aula (resolução no use-case, não no
   front) para evitar duplicação (DRY).
3. **Biblioteca de mídia (F2):** depende de `assets` reutilizáveis cross-curso — modelagem futura.
4. **Fronteira IN × AD em "Alunos/Preços":** confirmar se IN tem leitura de alunos dos seus cursos
   (proposta: sim) — alinhar com RBAC #22.
5. **Wireframes:** Editor de curso é uma das telas críticas listadas para wireframe (README §6).
