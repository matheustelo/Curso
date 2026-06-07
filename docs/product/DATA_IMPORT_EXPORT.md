# Importação, Exportação e Portabilidade de Dados (Migração + LGPD)

- **Versão:** 1.0 · **Data:** 2026-06-07
- **Autor:** Migração de Dados & Portabilidade (EdTech)
- **Status:** Proposta para revisão de produto/engenharia
- **Relacionados:** [DATA_MODEL.md](../DATA_MODEL.md) · [ARCHITECTURE.md](../ARCHITECTURE.md) · [USER_FLOWS.md](USER_FLOWS.md) · [BUSINESS_RULES_AND_STATES.md](BUSINESS_RULES_AND_STATES.md) · [NON_FUNCTIONAL_REQUIREMENTS.md](NON_FUNCTIONAL_REQUIREMENTS.md) · [RBAC_MATRIX.md](RBAC_MATRIX.md) · [NOTIFICATIONS_MATRIX.md](NOTIFICATIONS_MATRIX.md) · [README.md](README.md) (glossário) · [CLAUDE.md](../../CLAUDE.md)

> **Diferencial comercial:** facilitar a **vinda de tenants** que hoje usam Hotmart/Kiwify/Eduzz/Teachable
> (importação assistida) e garantir **portabilidade** (anti-lock-in) e **conformidade LGPD** (acesso,
> portabilidade e esquecimento). Este documento especifica os fluxos de **importação** e **exportação**.
>
> **Regra nº1 (não-negociável):** TODO acesso a dados de tenant passa por `withTenant(tenantId, fn)`
> (`SET LOCAL search_path`). `tenantId` vem da sessão/JWT, nunca de header. Import/export **nunca** cruzam
> tenants; cada job carrega `tenantId` no payload (igual aos demais jobs pg-boss — ARCHITECTURE §9).
>
> **Notação de fase:** **[MVP]** · **[F2]** · **[F3]** (alinhada ao PRD/ROADMAP). O **MVP** entrega o
> **LGPD essencial** (export/erase do aluno + export do tenant para anti-lock-in) e a **importação básica**
> (alunos CSV + matrículas + estrutura de curso). Import avançado (vídeo em lote, conectores por provedor)
> é majoritariamente **[F2]**.

---

## Índice

1. [Princípios e invariantes](#1-princípios-e-invariantes)
2. [Modelo de jobs, idempotência e armazenamento](#2-modelo-de-jobs-idempotência-e-armazenamento)
3. [Importação — visão geral e matriz de fases](#3-importação--visão-geral-e-matriz-de-fases)
   - 3.1 Assistente de importação (UX, passos, preview, dry-run)
   - 3.2 Mapeamento de colunas, validação e normalização
   - 3.3 Deduplicação por e-mail e idempotência
   - 3.4 Relatório de erros
4. [Importação por entidade](#4-importação-por-entidade)
   - 4.1 Alunos (CSV) · 4.2 Cursos/Módulos/Aulas · 4.3 Matrículas · 4.4 Progresso/Pedidos
5. [Importação de vídeos (lote → Bunny via TUS)](#5-importação-de-vídeos-lote--bunny-via-tus)
6. [Viabilidade por provedor de origem (limitações)](#6-viabilidade-por-provedor-de-origem-limitações)
7. [Exportação — portabilidade do tenant (anti-lock-in)](#7-exportação--portabilidade-do-tenant-anti-lock-in)
8. [Exportação/portabilidade do aluno (LGPD — acesso/portabilidade)](#8-exportaçãoportabilidade-do-aluno-lgpd--acessoportabilidade)
9. [Direito ao esquecimento (LGPD — exclusão/anonimização)](#9-direito-ao-esquecimento-lgpd--exclusãoanonimização)
10. [Saída do tenant (offboarding) e DROP SCHEMA](#10-saída-do-tenant-offboarding-e-drop-schema)
11. [Segurança, isolamento e auditoria](#11-segurança-isolamento-e-auditoria)
12. [Máquinas de estado (job de import/export)](#12-máquinas-de-estado-job-de-importexport)
13. [Contratos (Zod) e formatos canônicos](#13-contratos-zod-e-formatos-canônicos)
14. [Dependências e pontos para o coordenador](#dependências-e-pontos-para-o-coordenador)

---

## 1. Princípios e invariantes

1. **Isolamento absoluto (Regra nº1).** Cada operação roda em `withTenant`. Arquivos gerados/consumidos no
   R2 são prefixados por `<tenantId>/...` e nunca compartilhados entre tenants. Todo job de import/export
   exige **teste de isolamento cross-tenant** (gate de CI — CLAUDE.md).
2. **Assíncrono por padrão.** Import/export são pesados e variáveis: processados no **worker (pg-boss)**.
   A API só **valida, enfileira e responde** com um `job_id`; o front faz polling/SSE do progresso.
3. **Idempotência ponta-a-ponta.** Reprocessar um arquivo/linha é **no-op** quando já aplicado
   (dedupe por chave natural). Espelha o padrão de `payment_events.event_id` (ARCHITECTURE §9,
   BUSINESS_RULES §0).
4. **Dry-run antes de gravar.** Toda importação tem um modo **simulação** (valida + reporta o que faria)
   **sem** efeitos colaterais, separado do **commit**.
5. **DRY de contratos.** Schemas de linhas/colunas vivem em `packages/contracts` (Zod) — fonte única
   back+front (ARCHITECTURE §1, ADR-0004). Reuso dos use-cases existentes (criar curso/matrícula) em vez de
   inserts crus paralelos.
6. **Reuso dos use-cases de domínio.** A importação **não** insere direto na tabela: chama os mesmos
   use-cases (`CreateCourse`, `EnrollStudent`, …) para herdar regras/validação/eventos. Para volume, há um
   caminho de **bulk** otimizado nos repositórios (port), mas sob as mesmas invariantes.
7. **Sem PII em logs.** Logs estruturados (Pino) carregam `tenantId`/`jobId`/contadores, **nunca** e-mail,
   CPF ou conteúdo de linha (NFR de segurança/LGPD).
8. **RBAC.** Import e export do tenant: **Owner/Admin**. Export/erase do **aluno**: o próprio aluno
   (self-service) ou Admin a pedido do titular. Offboarding/`DROP SCHEMA`: **Super-Admin** (control plane).

---

## 2. Modelo de jobs, idempotência e armazenamento

### 2.1 Tabelas de controle (data plane do tenant)

> Proposta de modelagem (ver [Dependências](#dependências-e-pontos-para-o-coordenador) #1 — não existe hoje
> no DATA_MODEL). Todas no **schema do tenant**, acessadas via `withTenant`. Sem FK cross-schema.

```sql
import_jobs(
  id uuid pk,
  kind text,                         -- students | courses | enrollments | progress | orders | videos
  source_provider text null,         -- hotmart | kiwify | eduzz | teachable | generic_csv
  status text,                        -- uploaded | validating | preview_ready | committing | partially_done | done | failed | canceled
  source_file_key text,               -- CSV/ZIP original no R2 (<tenantId>/imports/<jobId>/source.csv)
  mapping jsonb,                       -- mapa coluna_origem -> campo_destino (definido no assistente)
  options jsonb,                       -- { dedupe_by:"email", dry_run:false, send_welcome:false, ... }
  totals jsonb,                        -- { rows, valid, invalid, created, updated, skipped }
  report_file_key text null,           -- relatório de erros (CSV) no R2
  created_by uuid,                     -- fk -> users (Owner/Admin)
  created_at, updated_at, finished_at timestamptz null
)
import_rows(                           -- 1 linha por registro processado (idempotência + auditoria do lote)
  id uuid pk, job_id uuid fk,
  row_number int,
  natural_key text,                    -- ex.: lower(trim(email)) | course.slug | (email|course_slug)
  status text,                         -- valid | invalid | created | updated | skipped | error
  errors jsonb null,                   -- [{ field, code, message }]
  unique(job_id, row_number)
)
export_jobs(
  id uuid pk,
  scope text,                          -- tenant | student
  subject_ref text null,               -- user_id quando scope=student
  format text,                          -- zip_csv | json
  status text,                          -- queued | running | ready | expired | failed
  artifact_key text null,               -- arquivo no R2 (<tenantId>/exports/<jobId>/export.zip)
  expires_at timestamptz null,          -- link expira (ver §7.4)
  requested_by uuid,
  created_at, finished_at timestamptz null
)
```

### 2.2 Idempotência

- **Job:** `import_jobs.id` é a `idempotency_key` do lote; reenfileirar o mesmo job é seguro (retoma do
  ponto). O **commit** processa em **chunks** (ex.: 500 linhas/tx) com checkpoint em `import_rows`.
- **Linha:** a chave natural (`natural_key`) torna a reaplicação no-op:
  - alunos → `lower(trim(email))` (mesma regra do unique por schema, DATA_MODEL §2.1);
  - cursos/aulas → `slug` (ou `external_id` se mapeado);
  - matrículas → `(email, course_slug)` resolvido para `(user_id, course_id)` (unique de `enrollments`).
- Reupload do **mesmo CSV** após falha parcial: linhas já `created` viram `skipped` (idempotente), o resto
  é completado.

### 2.3 Armazenamento (R2)

- Arquivos de origem e relatórios em **Cloudflare R2** (egress zero — ADR-0012), sempre sob prefixo
  `<tenantId>/...`. Acesso por **URL assinada (TTL curto)**, como certificados/anexos (USER_FLOWS §5.2/§7.1).
- Artefatos de export têm **expiração** (`expires_at`); após expirar, job → `expired` e o objeto é
  removido por job de limpeza (lifecycle do bucket + reconciliação).

---

## 3. Importação — visão geral e matriz de fases

| Entidade | Fonte (MVP) | Caminho | Fase |
|---|---|---|---|
| **Alunos** | CSV | upload + assistente + dedupe e-mail | **[MVP]** |
| **Matrículas** | CSV (email × curso) | resolve `user`/`course`, cria `enrollment(source='bulk')` | **[MVP]** |
| **Cursos/Módulos/Aulas** (estrutura) | CSV/JSON | cria hierarquia (vídeo entra depois) | **[MVP]** (texto/estrutura) |
| **Progresso de aulas** | CSV | preenche `lesson_progress` (migração de histórico) | **[F2]** |
| **Pedidos/histórico financeiro** | CSV | `orders` históricos (status `paid`, sem cobrar) | **[F2]** |
| **Vídeos** (URL/upload em lote → Bunny) | URLs / ZIP | TUS / fetch-from-URL para a Library do tenant | **[F2]** |
| **Conectores por provedor** (API/export nativo) | Hotmart/Kiwify/Eduzz/Teachable | normalização específica | **[F2/F3]** |

> **MVP mínimo viável de migração:** alunos + matrículas (com dedupe) e estrutura de curso. Isso já permite
> "trazer a base" de outra plataforma. **Vídeos em lote e progresso/pedidos históricos** são **F2** (maior
> custo/risco e dependência de export do provedor de origem — ver §6).

### 3.1 Assistente de importação (UX, passos, preview, dry-run)

Local: **[Admin → Alunos / Cursos → Importar]** (Studio). Wizard de 5 passos, com estado salvo em
`import_jobs` para retomar.

```
Passo 1 — Origem        → escolhe entidade (Alunos | Cursos | Matrículas) + provedor de origem
                          (Hotmart/Kiwify/Eduzz/Teachable/CSV genérico) → mostra template/colunas esperadas
Passo 2 — Upload        → arrasta CSV (ou ZIP); (BE) grava em R2 <tenantId>/imports/<jobId>/source.csv
                          → detecta encoding/delimitador/cabeçalho
Passo 3 — Mapeamento    → mapeia coluna_origem → campo_destino (auto-sugerido por heurística/perfil do provedor)
                          → define opções (dedupe por e-mail, enviar e-mail de boas-vindas?, política de update)
Passo 4 — Preview/Dry-run → (Job) valida TODAS as linhas SEM gravar → mostra:
                          ✔ válidas (amostra) · ✖ inválidas (com motivo) · ⟂ duplicadas (ação) · totais
                          → permite baixar "relatório de validação" (CSV)
Passo 5 — Confirmar      → "Importar N registros" → (Job) commit em chunks idempotentes
                          → progresso ao vivo (barra + contadores) → ✔ concluído + relatório final
```

Princípios de UX:
- **Template oficial por entidade** disponível para download (CSV com cabeçalhos canônicos + exemplo).
- **Preview sempre antes do commit** (passo 4 nunca grava). O usuário vê exatamente o que será criado /
  atualizado / ignorado.
- **Dry-run = passo 4**; **commit = passo 5**. Internamente, são duas execuções do mesmo pipeline com a
  flag `options.dry_run`.
- **Retomável:** fechar o navegador não perde o job; ele aparece em "Importações recentes" com status.
- **Cancelável:** antes do commit, cancelar é livre; durante o commit, o job para no próximo checkpoint
  (linhas já aplicadas permanecem — operação idempotente permite reexecutar).
- **Acessibilidade/i18n:** mensagens de erro claras e localizadas (next-intl); contadores legíveis (NFR a11y).

### 3.2 Mapeamento de colunas, validação e normalização

- **Mapeamento:** o assistente sugere o mapa (`mapping jsonb`) a partir do **perfil do provedor** (ex.:
  Hotmart exporta `Email`, `Nome`, `Telefone`, `Produto`) e o usuário ajusta. Campos não mapeados são
  ignorados (e listados).
- **Validação (Zod, `packages/contracts`):** por linha — e-mail válido; nome presente; CPF válido se
  fornecido (normaliza máscara); datas em ISO-8601 (aceita formatos comuns BR e converte); valores
  monetários → centavos (`integer`) + `currency` (DATA_MODEL convenções).
- **Normalização:**
  - e-mail → `lower(trim())`;
  - telefone → E.164 quando possível (best-effort, não bloqueia);
  - `course_slug` → slugify se vier título; colisão de slug recebe sufixo (mesma regra de autoria,
    USER_FLOWS §4.1).
- **Linhas inválidas não bloqueiam o lote:** ficam marcadas `invalid` no relatório; as válidas seguem
  (política configurável "tudo-ou-nada" vs "parcial" — default **parcial** com relatório).

### 3.3 Deduplicação por e-mail e idempotência

- **Chave de dedupe:** `lower(trim(email))`, coerente com `users.unique(email)` **por schema**
  (DATA_MODEL §2.1, BUSINESS_RULES §10.7).
- **E-mail já existe no tenant:**
  - **Alunos:** não duplica; aplica `política de update` escolhida no passo 3 (`skip` = mantém / `update` =
    atualiza nome/telefone/campos não sensíveis). **Nunca** sobrescreve `password_hash` nem rebaixa papel.
  - **Matrículas:** resolve para o `user` existente e cria/idempotentemente reativa `enrollment`
    (`unique(user_id, course_id)`).
- **Duplicatas dentro do próprio CSV:** detectadas no dry-run; mantém a primeira, marca as demais `skipped`.
- **Senhas:** import **não** define senha. Aluno importado entra por **fluxo de definição de senha**
  (link "criar/definir senha") ou login social — evita armazenar/transportar senhas de terceiros (LGPD/segurança).
  E-mail de boas-vindas/convite é **opt-in** no passo 3 (evita disparo em massa indesejado).

### 3.4 Relatório de erros

- Ao fim de **dry-run** e de **commit**, gera-se um **CSV de relatório** no R2 (`report_file_key`), com:
  `row_number, status, natural_key, field, error_code, message`.
- Códigos de erro estáveis (i18n no front): `INVALID_EMAIL`, `MISSING_NAME`, `INVALID_CPF`,
  `COURSE_NOT_FOUND`, `DUPLICATE_IN_FILE`, `ALREADY_EXISTS`, `QUOTA_EXCEEDED`, `INVALID_DATE`, …
- **Resumo no topo:** totais por status. Erros de domínio reaproveitam `DomainError` → Problem Details
  (RFC 9457) quando expostos via API (CLAUDE.md, ARCHITECTURE §4.3).
- **Quota:** se a importação exceder `platform_plans.limits.max_students` (serviço de quotas, DATA_MODEL
  §6.12), o dry-run avisa **antes** e o commit bloqueia o excedente com `QUOTA_EXCEEDED` (não corrompe
  dados existentes — BUSINESS_RULES §10.11).

---

## 4. Importação por entidade

### 4.1 Alunos (CSV) [MVP]

- **Colunas canônicas:** `email` (obrig.), `name` (obrig.), `phone`, `cpf`, `created_at` (data de cadastro
  original, opcional), `tags` (opcional → futura segmentação).
- **Efeito:** cria `users(role='student', status='active')` via use-case (Better-Auth gerencia identidade);
  registra **consentimento** com a base legal/origem (importação por migração — não substitui consentimento
  do titular, mas registra a fonte para auditoria LGPD). Sem senha (ver §3.3).
- **Idempotente** por e-mail; relatório por linha.

### 4.2 Cursos / Módulos / Aulas (estrutura) [MVP]

- Importa a **hierarquia** `courses → modules → lessons` (texto/estrutura). Formatos:
  - **CSV achatado** com colunas `course_title, course_slug, module_title, module_position, lesson_title,
    lesson_position, lesson_type, content/text, video_source_url?`; ou
  - **JSON aninhado** (mais fiel para conteúdo rich-text).
- **Aulas de vídeo:** a linha pode trazer `video_source_url` (origem). No **MVP**, isso apenas registra a
  URL/pendência; a **ingestão do vídeo no Bunny é F2** (§5). Até o vídeo ficar `ready`, a aula segue regra
  normal de publicação (não publica sem `video_status='ready'` — BUSINESS_RULES §2/§3).
- **Aulas de texto/PDF:** texto vai para `content jsonb`; PDFs em lote (ZIP) viram `lesson_assets` no R2.
- Curso importado nasce `draft` (admin revisa e publica — USER_FLOWS §4.3).

### 4.3 Matrículas (CSV) [MVP]

- **Colunas canônicas:** `email`, `course_slug` (ou `course_external_id`), `enrolled_at` (opcional),
  `expires_at` (opcional), `status` (default `active`).
- **Resolução:** `email → user` (cria se ausente + flag no relatório), `course_slug → course`
  (erro `COURSE_NOT_FOUND` se inexistente).
- **Efeito:** `enrollments(source='bulk')` via use-case (idempotente por `unique(user_id, course_id)`;
  se já existe suspensa, reativa — alinhado a USER_FLOWS §6.2).
- **Não gera pedido** (matrícula de cortesia/migração; reembolso não se aplica — BUSINESS_RULES §4.3).

### 4.4 Progresso e Pedidos históricos [F2]

- **Progresso:** importa `lesson_progress` (status/percentual/`completed_at`) para preservar histórico do
  aluno na migração. Anti-seek **não** se aplica a dados históricos (são fatos passados; importados como
  `completed` quando a origem indicar). Marcado **F2** por exigir mapeamento fino e ser raro nas origens.
- **Pedidos:** importa `orders` históricos como `status='paid'` **sem** acionar cobrança nem webhooks de
  saída (flag `historical=true`). Útil para relatórios/LTV. **F2**; cuidado para **não** disparar máquina de
  matrícula/comissão (importação é fato consumado, não evento novo).

---

## 5. Importação de vídeos (lote → Bunny via TUS) [F2]

> Vídeo é o ativo mais pesado e o maior risco de migração. **F2.** Reusa integralmente o pipeline de vídeo
> existente (ARCHITECTURE §7): 1 Video Library por tenant, upload TUS resumable, webhook "vídeo pronto"
> (HMAC) → job idempotente que atualiza `lessons.video_guid`/`video_status` (BUSINESS_RULES §3).

### 5.1 Duas estratégias de ingestão

1. **Fetch-from-URL (preferida quando há URL pública/assinada):** o backend instrui a **Bunny a baixar o
   vídeo da URL de origem** (mesmo mecanismo já previsto para o VOD de lives — ARCHITECTURE §7, ADR-0015:
   "Bunny fetch-from-URL"). Sem trafegar bytes pelo nosso backend/browser. Ideal para lotes grandes.
2. **Upload em lote (TUS):** quando só há arquivos locais (ZIP/pasta), o admin sobe arquivos que o front
   envia **direto ao edge da Bunny via TUS** (resumable), aula a aula, reusando o fluxo de
   USER_FLOWS §4.2. A API key da Bunny **nunca** vai ao front.

### 5.2 Fluxo (lote por fetch-from-URL)

```
[Import vídeos] → CSV com (lesson_ref, video_source_url[, checksum]) → preview/validação de URLs
   → (Job: video.import.batch, por chunk) para cada item:
        1. withTenant → resolve bunny_library_id do tenant
        2. cria objeto de vídeo na Library + dispara fetch-from-URL (idempotente por (lesson_id|source_url))
        3. lessons.video_status = 'queued'/'processing'
   → (Webhook Bunny "vídeo pronto" HMAC) → (Job) atualiza video_guid/duration → 'ready'
   → relatório por item (ok | falha de download | formato inválido | timeout)
```

- **Idempotência:** dedupe por `(lesson_id, source_url)`; reenfileirar não recria vídeos.
- **Limites:** respeita quota de **storage/bandwidth** do plano (lida via API Bunny — DATA_MODEL §6.12);
  excedente bloqueia novos itens com aviso. **Concorrência limitada** (poucos itens simultâneos) para não
  estourar rate limits da Bunny.
- **Falhas:** download falho/formato inválido → item `failed` no relatório; retry manual por item.
- **Origens sem URL acessível (DRM/embed fechado):** **não importáveis automaticamente** — ver §6.

---

## 6. Viabilidade por provedor de origem (limitações)

> Tabela de referência comercial/operacional. O que é **viável automaticamente** depende do que o provedor
> de origem **deixa exportar**. Vídeo hospedado com DRM/streaming fechado normalmente **não** é exportável
> sem reupload do arquivo-fonte pelo produtor.

| Provedor | Alunos | Matrículas | Estrutura de curso | Vídeo | Observações |
|---|---|---|---|---|---|
| **Hotmart** | CSV de compradores (export nativo) ✅ | Derivável de compras/produto ⚠️ | Manual/limitado ⚠️ | **Não exporta** mídia ❌ | Vídeo hospedado fechado; produtor precisa do arquivo-fonte. API/relatórios variam por plano. |
| **Kiwify** | CSV de clientes ✅ | Por produto ⚠️ | Limitado ⚠️ | **Não exporta** ❌ | Foco em vendas; conteúdo/membros área fechada. |
| **Eduzz** | CSV de clientes ✅ | Por produto ⚠️ | Limitado ⚠️ | **Não exporta** ❌ | Similar a Hotmart. |
| **Teachable** | CSV de students + progress ✅ | CSV de enrollments ✅ | Export de curso parcial ⚠️ | **Não exporta** vídeo ❌ (URLs internas) | Mais "LMS": exporta progresso/enrollments; mídia continua hospedada lá. |

Legenda: ✅ viável via CSV/export nativo · ⚠️ viável com mapeamento/parcial · ❌ requer arquivo-fonte do produtor.

**Conclusões operacionais:**
- **Sempre viável (MVP):** importar **alunos** e **matrículas** via CSV (todos os provedores exportam ao
  menos a base de compradores).
- **Conteúdo de vídeo:** em **todos** os casos, a mídia precisa do **arquivo-fonte** do produtor (ou URL
  assinada que ele consiga gerar). Por isso a importação de vídeo é **F2** e tratada como reupload/fetch
  (§5), não "cópia" do provedor.
- **Conectores por API** (em vez de CSV manual) são **F2/F3** e dependem da disponibilidade/estabilidade da
  API de cada provedor; começamos por **perfis de mapeamento de CSV** (cobrem a maioria dos casos com baixo
  custo).
- **Progresso:** só Teachable costuma exportar com fidelidade; por isso §4.4 (progresso) é **F2** e
  best-effort.

---

## 7. Exportação — portabilidade do tenant (anti-lock-in)

> **Promessa anti-lock-in:** o tenant pode **sair levando seus dados** a qualquer momento. Disponível mesmo
> com o tenant em `cancelled` durante a janela de retenção (somente leitura — BUSINESS_RULES §1.3).

### 7.1 Escopo do export do tenant [MVP]

Pacote ZIP com um arquivo por entidade (CSV) **+** um `manifest.json` (versão do schema, data, contadores):

- `users.csv` (alunos/equipe — sem `password_hash`; hashes **nunca** saem),
- `courses.csv`, `modules.csv`, `lessons.csv` (estrutura + metadados; `video_guid` como referência),
- `enrollments.csv`, `lesson_progress.csv`,
- `orders.csv`, `subscriptions.csv`, `coupons.csv`, `affiliate_commissions.csv` (financeiro do tenant),
- `certificates.csv` (metadados + URL de verificação pública),
- `comments.csv` (comunidade) — opcional/togglável.
- **Conteúdo de vídeo:** o ZIP traz **referências** (`video_guid` + nome). A **mídia em si** sai da própria
  Bunny Library do tenant (o tenant tem/recebe acesso à sua Library — isolamento por tenant já garante isso),
  ou via job de export de mídia **[F2]** que gera URLs assinadas em lote. PDFs/anexos do R2: incluídos no ZIP
  ou listados com URLs assinadas conforme tamanho.

### 7.2 Formatos

- **CSV** (interoperável, abre em planilha; default para o anti-lock-in) **e/ou** **JSON** (mais fiel para
  campos `jsonb` como `content`/`i18n`). O usuário escolhe (`export_jobs.format`).
- Encoding UTF-8, datas ISO-8601, monetário em centavos + `currency` (consistência com o modelo).

### 7.3 Geração assíncrona + R2 + expiração

```
[Admin → Configurações → Exportar dados] → escolhe escopo (tudo | seleção) + formato → "Gerar export"
   → (BE) cria export_jobs(status='queued') → enfileira (Job: tenant.export)
   → (Job) withTenant: lê entidades em streaming → escreve CSV/JSON → zipa → grava em R2
        <tenantId>/exports/<jobId>/export.zip → status='ready', expires_at = now()+TTL
   → (Notif) e-mail/in-app "seu export está pronto" com link assinado (TTL curto)
   → download via URL assinada → após expires_at: status='expired' + objeto removido (lifecycle)
```

- **Streaming/chunking** para tenants grandes (não carrega tudo em memória).
- **Link expira** (proposta: 24–72h — ver Dependências #5). Re-geração é livre (novo job).
- **Idempotência/segurança:** cada export é um job auditado; download exige sessão válida do tenant +
  escopo (RBAC Owner/Admin).

### 7.4 Auditoria

Exports de tenant geram entrada de auditoria (quem, quando, escopo). Para tenant `cancelled`, o acesso é
**somente leitura** e a janela é a de retenção (BUSINESS_RULES §1.3, §10).

---

## 8. Exportação/portabilidade do aluno (LGPD — acesso/portabilidade) [MVP]

> **Direito de acesso e portabilidade (LGPD Art. 18, II e V).** O **aluno** baixa **seus próprios dados**
> em formato legível e portável. **Self-service** na área do aluno; também acionável por Admin a pedido do titular.

### 8.1 Escopo (dados do titular naquele tenant)

- Perfil (`users`: nome, e-mail, telefone, consentimentos/timestamps),
- Matrículas e progresso (`enrollments`, `lesson_progress`),
- Pedidos/assinaturas do aluno (`orders`, `subscriptions` — sem dados de cartão; PCI fica no gateway),
- Certificados (metadados + link de verificação),
- Comentários/interações (`lesson_comments`),
- (F2) gamificação, notificações, presença em lives.

### 8.2 Fluxo

```
[Área do aluno → Privacidade → Baixar meus dados] → confirma identidade (sessão) → "Solicitar"
   → (BE) export_jobs(scope='student', subject_ref=user_id, status='queued') → (Job: student.export)
   → (Job) withTenant + filtro por user_id → gera JSON (legível) + CSVs → zip em R2 (<tenantId>/exports/...)
   → (Notif) "seus dados estão prontos" → download por URL assinada (TTL curto) → expira
```

- **Formato:** **JSON** (portável/legível) por padrão + CSVs auxiliares. Estruturado e documentado
  (`manifest.json`).
- **Isolamento:** o filtro por `user_id` roda dentro de `withTenant`; jamais inclui dados de outros alunos
  (ex.: threads de comentários trazem só os do titular; respostas de terceiros são omitidas/anonimizadas).
- **Prazo LGPD:** atender em prazo legal (proposta: imediato/assíncrono em minutos; SLA formal a confirmar
  com jurídico — ver Dependências).
- **Self-service vs ticket:** MVP entrega **self-service** (reduz custo operacional). Admin tem ação
  equivalente para solicitações por outros canais.

---

## 9. Direito ao esquecimento (LGPD — exclusão/anonimização) [MVP]

> **Direito de eliminação (LGPD Art. 18, VI).** Apaga/anonimiza dados pessoais do **aluno** dentro do
> schema do tenant (ARCHITECTURE §3.6: "deleção/anonimização dentro do schema do tenant").

### 9.1 Estratégia: anonimizar > apagar fisicamente

- **Anonimização** é preferível a `DELETE` físico porque preserva **integridade referencial** e **registros
  financeiros/fiscais** que a lei exige reter (pedidos pagos, notas) — desvinculando-os da pessoa.
- `users`: e-mail → `deleted+<hash>@anon.invalid`, nome → "Usuário removido", telefone/CPF → `null`,
  `password_hash` → `null`, `status='deleted'`, `deleted_at` setado (soft-delete onde fizer sentido —
  DATA_MODEL convenções).
- **Conteúdo gerado:** comentários do titular → corpo removido/anonimizado (`deleted_at`), mantendo a thread
  íntegra (já há `lesson_comments.deleted_at` — DATA_MODEL §6.3).
- **Progresso/matrícula:** mantidos de forma agregada/anônima para integridade de relatórios, ou removidos
  conforme política (a confirmar — Dependências).
- **Certificados:** emitidos podem ser **revogados** (`certificates.revoked=true`) e desvinculados do nome,
  conforme política do tenant.

### 9.2 Fluxo

```
[Aluno → Privacidade → Excluir minha conta]  (ou Admin a pedido do titular)
   → confirma (impacto: perde acesso; ação irreversível) → (BE) registra solicitação
   → (Job: student.erase) withTenant + user_id:
        anonimiza users; anonimiza/soft-delete conteúdo; revoga certificados (política);
        mantém registros fiscais desvinculados; encerra sessões
   → audit_log (platform): titular, tenant, ação, timestamp (sem PII no log)
   → (Notif) confirmação de exclusão
```

- **Retenção legal:** dados fiscais/financeiros podem ter retenção obrigatória; a anonimização os mantém
  **sem identificar a pessoa** (prazos = jurídico — Dependências #6, README #24).
- **Irreversível:** a UI exige confirmação forte; opcional período de carência curto antes de executar
  (a definir).
- **Idempotente:** reexecutar `student.erase` para um titular já anonimizado é no-op.

---

## 10. Saída do tenant (offboarding) e DROP SCHEMA [MVP — base / F2 — automação]

> Encerramento de tenant. Sequência **export-antes-de-apagar** para honrar anti-lock-in + LGPD.

```
[Super-Admin → Tenant detalhe] "Cancelar"  (USER_FLOWS §10.2)
   → tenants.status='cancelled' → tenant entra em SOMENTE LEITURA (janela de retenção)
   → tenant/owner pode gerar EXPORT completo (§7) durante a janela
   → (fim da retenção legal/contratual) → (Job controlado, auditado):
        DROP SCHEMA tenant_<slug> CASCADE  (ARCHITECTURE §3.6, BUSINESS_RULES §1.2)
        + remove artefatos do R2 (<tenantId>/*) + apaga/limpa Bunny Library do tenant
        + anonimiza referências no control plane (platform.tenants/audit mantêm registro mínimo)
   → status lógico [purgado]; audit_log registra a purga
```

- **`DROP SCHEMA ... CASCADE`** é o "botão forte" de esquecimento do **tenant inteiro** (vs. anonimização
  por aluno do §9). Só executa **após a janela de retenção** e com **export disponibilizado** antes.
- **Limpeza externa:** purga deve incluir **R2** (`<tenantId>/*`) e **Bunny Library** do tenant — senão a
  mídia continua existindo fora do Postgres. Esses passos são parte do job de purga (idempotente, auditado).
- **Reativação na janela:** se o schema ainda existe, `cancelled → provisioning` é possível
  (BUSINESS_RULES §1.2). Após purga, não.
- **Fase:** o `DROP SCHEMA` manual/controlado é **MVP** (operável pelo Super-Admin); a **automação completa**
  (lifecycle + purga R2/Bunny encadeada + relatório de conformidade) é **F2**.

---

## 11. Segurança, isolamento e auditoria

- **`withTenant` sempre.** Import/export rodam dentro de transação com `search_path` setado; jobs carregam
  `tenantId` (ARCHITECTURE §3.3/§9). **Teste de isolamento cross-tenant** obrigatório para cada novo job
  (gate de CI — CLAUDE.md).
- **Prefixo R2 por tenant** (`<tenantId>/...`) + **URLs assinadas TTL curto**; nunca URLs públicas para
  dados pessoais/export.
- **Sem segredos no export:** `password_hash`, keys Bunny/pagamento, recipients cifrados **nunca** saem
  (CLAUDE.md anti-padrões).
- **Sem PII em logs/relatórios persistidos além do necessário;** relatórios de erro ficam no R2 sob o tenant
  e expiram.
- **Auditoria:** export/erase/offboarding gravam em `platform.audit_log` (ator, tenant, ação, timestamp) —
  consistente com ações sensíveis (BUSINESS_RULES §0, USER_FLOWS §10).
- **RBAC (RBAC_MATRIX):** import/export-tenant = Owner/Admin; export/erase-aluno = titular ou Admin a pedido;
  `DROP SCHEMA` = Super-Admin (control plane). Instrutor/afiliado **não** importam/exportam base.
- **Rate limit / abuso:** export e import são caros — limitar nº de jobs concorrentes por tenant; um job
  ativo por escopo por vez (evita martelar a base/Bunny).

---

## 12. Máquinas de estado (job de import/export)

**Import (`import_jobs.status`):**
```
uploaded ──(mapear)──▶ validating ──(dry-run ok)──▶ preview_ready ──(confirmar)──▶ committing
   committing ──(todos chunks ok)──▶ done
   committing ──(alguns erros, política parcial)──▶ partially_done
   validating/committing ──(erro fatal)──▶ failed
   preview_ready/committing ──(cancelar)──▶ canceled
```

**Export (`export_jobs.status`):**
```
queued ──(worker pega)──▶ running ──(zip em R2)──▶ ready ──(expires_at)──▶ expired
   running ──(erro)──▶ failed
```

- Transições não listadas são **proibidas** (`InvalidStateTransitionError` — BUSINESS_RULES §0).
- `committing`/`running` são **retomáveis** por checkpoint (chunk/linha); reprocessar é idempotente.

---

## 13. Contratos (Zod) e formatos canônicos

> Fonte única em `packages/contracts` (ADR-0004, CLAUDE.md DRY). Esboço dos schemas de linha (os tipos reais
> serão derivados destes Zod; aqui em pseudocódigo para alinhamento):

```ts
// students.import.row
{ email: z.string().email(), name: z.string().min(1), phone?: string, cpf?: string,
  createdAt?: isoDate, tags?: string[] }

// enrollments.import.row
{ email: z.string().email(), courseSlug: z.string(), enrolledAt?: isoDate,
  expiresAt?: isoDate, status?: z.enum(['active','suspended','refunded','expired']).default('active') }

// courses.import (json aninhado) — course → modules[] → lessons[]
{ title, slug?, description?, pricingType: z.enum(['one_time','subscription','free']),
  priceCents?: int, modules: [{ title, position, lessons: [
     { title, position, type: z.enum(['video','text','pdf','quiz']), content?, videoSourceUrl? } ] }] }

// videos.import.row [F2]
{ lessonRef: string, videoSourceUrl: z.string().url(), checksum?: string }

// export manifest
{ schemaVersion: string, tenantId: string, scope: z.enum(['tenant','student']),
  generatedAt: isoDate, counts: Record<string, number> }
```

- Os **vocabulários de `status`** seguem o glossário canônico (README §2.3 / DATA_MODEL §6.0) — sem inventar
  novos valores.
- Erros de validação → `DomainError` mapeados a Problem Details (RFC 9457) na borda HTTP.

---

## Dependências e pontos para o coordenador

1. **Modelagem de import/export ausente no DATA_MODEL.** As tabelas `import_jobs`, `import_rows`,
   `export_jobs` (§2.1) **não existem** hoje. Decisão: adicionar ao DATA_MODEL §6 + migration + possível ADR
   (ex.: "ADR-00xx — Import/Export & Portabilidade"). Sem elas, não há retomada/idempotência/auditoria de lote.
2. **Fase do import de vídeo, progresso e pedidos.** Confirmar **F2** para vídeo em lote (§5),
   progresso/pedidos históricos (§4.4) e conectores por API (§6). O MVP fica em alunos + matrículas +
   estrutura de curso. Validar com produto/comercial se "trazer a base" sem vídeo é suficiente para o
   discurso de migração no lançamento.
3. **Senhas e onboarding do aluno importado (§3.3).** Confirmar que **não** importamos senhas e que o aluno
   define senha por link/social. Definir se o e-mail de convite é enviado em massa (opt-in) e como evitar
   spam/bloqueio de reputação de e-mail (Resend) em importações grandes.
4. **Política de update na dedupe (§3.3).** Default `skip` vs `update` para alunos já existentes; quais
   campos podem ser atualizados por import (nunca papel/senha). Confirmar.
5. **TTL de expiração de links de export (§7.3).** Proposto 24–72h. Confirmar prazo e política de
   re-geração; e lifecycle do bucket R2 para purga automática.
6. **Prazos e escopo LGPD (§8/§9) — jurídico.** SLA de atendimento (acesso/portabilidade), prazos de
   **retenção** de dados fiscais antes de anonimizar/apagar, e se progresso/matrícula são anonimizados ou
   removidos no esquecimento. Alinha com README #24 (retenção LGPD pendente de jurídico) e
   BUSINESS_RULES Dependências #9.
7. **Granularidade do esquecimento vs. retenção fiscal (§9).** Confirmar quais registros são **retidos
   desvinculados** (pedidos pagos/notas) e por quanto tempo. Impacta a função de anonimização.
8. **Purga de mídia externa no `DROP SCHEMA` (§10).** O offboarding precisa apagar **R2** e **Bunny Library**
   do tenant, não só o schema Postgres. Confirmar dono operacional desse job e a ordem (export → confirmação
   → purga) e a janela de retenção contratual antes da purga.
9. **Export da mídia de vídeo no anti-lock-in (§7.1).** Definir se entregamos URLs assinadas em lote (job F2)
   ou orientamos o tenant a usar o acesso à própria Bunny Library. Tem impacto comercial (promessa de
   portabilidade total).
10. **Perfis de mapeamento por provedor (§6).** Validar prioridade dos perfis de CSV (Hotmart/Kiwify/Eduzz/
    Teachable) para o assistente; e se algum conector por API entra antes (F2) por demanda comercial.
11. **Quota durante import (§3.4).** Confirmar comportamento ao exceder `max_students`/storage no meio do
    lote (bloquear excedente com relatório vs. permitir overage conforme MONETIZATION §A.4).
12. **Notificações (NOTIFICATIONS_MATRIX).** Registrar os eventos novos: `import_ready`/`import_done`,
    `export_ready`, `data_erasure_done` (canais/prioridade) para não ficarem fora da matriz.
```
