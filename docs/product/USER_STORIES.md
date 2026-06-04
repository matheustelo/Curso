# Backlog de Produto — Épicos, User Stories e Critérios de Aceite

- **Versão:** 1.0
- **Data:** 2026-06-04
- **Autor:** Product Owner (sênior)
- **Status:** Para refinamento com o time
- **Documentos de referência:** [PRD.md](../PRD.md) · [ROADMAP.md](../ROADMAP.md) · [DATA_MODEL.md](../DATA_MODEL.md) · [ARCHITECTURE.md](../ARCHITECTURE.md) · [CLAUDE.md](../../CLAUDE.md)

---

## Como ler este documento

- **Formato das stories:** "Como [persona], quero [ação], para [benefício]."
- **Critérios de aceite:** Gherkin (Given/When/Then), cobrindo caminho feliz + exceções principais.
- **Prioridade:** `MVP` (Must) · `F2` (Should) · `F3` (Could), alinhada ao [ROADMAP.md](../ROADMAP.md).
- **Estimativa relativa:** `P` (pequena) · `M` (média) · `G` (grande). Não é tempo absoluto; é tamanho relativo.
- **RN:** referência à regra de negócio do PRD §4 (ou seção citada) que sustenta a story.
- **Personas:** Super-Admin · Admin do Tenant · Instrutor · Afiliado · Aluno.

> **Convenção crítica (todo o backlog):** Toda story que toca dados de tenant pressupõe acesso via
> `withTenant` + isolamento por schema (CLAUDE.md §1). O Épico 0 detalha as stories transversais de
> isolamento; demais épicos as referenciam por `[ISO]`.

---

## Índice por épico

| # | Épico | Foco | Prioridade dominante |
|---|-------|------|----------------------|
| 0 | [Isolamento Multitenant (transversal)](#épico-0--isolamento-multitenant-transversal) | Anti-vazamento, gate de CI | MVP |
| 1 | [Multitenant / Onboarding / Provisionamento](#épico-1--multitenant--onboarding--provisionamento) | Saga de provisionamento, branding, subdomínio | MVP |
| 2 | [Identidade, Auth e RBAC](#épico-2--identidade-auth-e-rbac) | Login, papéis, sessão por tenant | MVP |
| 3 | [Catálogo de Conteúdo](#épico-3--catálogo-de-conteúdo) | Curso/Módulo/Aula, drip, publicação | MVP |
| 4 | [Vídeo e Player](#épico-4--vídeo-e-player) | Upload TUS, token, progresso, anti-pirataria | MVP |
| 5 | [Avaliações e Certificados](#épico-5--avaliações-e-certificados) | Quiz, certificado, verificação pública | MVP |
| 6 | [Matrícula e Estados de Acesso](#épico-6--matrícula-e-estados-de-acesso) | Máquina de estados pagamento↔acesso | MVP |
| 7 | [Checkout e Pagamentos](#épico-7--checkout-e-pagamentos) | Pix/boleto/cartão, cupom, webhooks idempotentes | MVP |
| 8 | [Afiliados e Split](#épico-8--afiliados-e-split) | Links, comissões, split Pagar.me | MVP |
| 9 | [Comunidade](#épico-9--comunidade) | Comentários por aula (MVP), fórum/lives (F2) | MVP/F2 |
| 10 | [Gamificação](#épico-10--gamificação-f2f3) | Pontos, badges, ranking | F2/F3 |
| 11 | [Analytics e Relatórios](#épico-11--analytics-e-relatórios) | Conclusão, receita; coorte/MRR (F2) | MVP/F2 |
| 12 | [Super-Admin e Billing SaaS](#épico-12--super-admin-e-billing-saas) | Gestão de tenants, planos, Stripe Billing | MVP |
| 13 | [LGPD, Acessibilidade e Compliance](#épico-13--lgpd-acessibilidade-e-compliance) | Consentimento, export/delete, WCAG AA | MVP |
| 14 | [Marketing e Integrações](#épico-14--marketing-e-integrações) | Landing/SEO/pixels, webhooks de saída | MVP/F2 |
| 15 | [Mobile, i18n e White-label avançado](#épico-15--mobile-i18n-e-white-label-avançado-f2f3) | PWA, domínio próprio, i18n | F2/F3 |
| 16 | [IA na plataforma](#épico-16--ia-na-plataforma-f2f3) | Transcrição, geração de quiz, tutor IA | F2/F3 |

---

# Épico 0 — Isolamento Multitenant (transversal)

> **Por que é um épico próprio:** vazar dados entre tenants é o bug mais grave possível (CLAUDE.md §1,
> PRD §4.1). Estas stories são **gate de CI** e pré-condição de qualquer feature de dados.

### US-0.1 — Acesso a dados sempre via `withTenant`
**Como** time de engenharia, **quero** que todo acesso a dados de tenant passe por `withTenant(tenantId, fn)`,
**para** garantir o `SET LOCAL search_path` por transação e impossibilitar vazamento cross-tenant.
**Prioridade:** MVP · **Estimativa:** G · **RN:** PRD §4.1; ARCHITECTURE §3.3.

```gherkin
Given um repositório que consulta tabelas do schema do tenant
When o repositório executa qualquer query
Then a query roda dentro de uma transação iniciada com "SET LOCAL search_path TO tenant_<slug>, public"
And nenhuma tabela é referenciada por nome não-qualificado fora dessa transação

Given uma conexão reaproveitada pelo PgBouncer (transaction pooling)
When uma nova requisição usa essa conexão
Then o search_path não vaza da transação anterior (SET LOCAL é transacional)
And o tenant da nova requisição é aplicado do zero
```

### US-0.2 — `tenantId` derivado da sessão/JWT, nunca de header não autenticado
**Como** plataforma, **quero** que o `tenant_id` efetivo venha do claim assinado da sessão,
**para** impedir que um usuário troque de tenant manipulando subdomínio/header.
**Prioridade:** MVP · **Estimativa:** M · **RN:** ARCHITECTURE §3.2.

```gherkin
Given uma requisição cujo subdomínio é "acme.app.com"
And cuja sessão tem claim tenant_id = "acme"
When o hook onRequest resolve o tenant
Then request.tenant = acme

Given uma requisição com subdomínio "acme.app.com"
And cuja sessão tem claim tenant_id = "escola2"
When o hook onRequest resolve o tenant
Then a requisição é rejeitada com 403 (mismatch tenant)
And o evento é registrado em audit_log

Given uma requisição com header "X-Tenant-Id" forjado e sem sessão válida
When o backend processa a requisição
Then o header é ignorado e a requisição recebe 401
```

### US-0.3 — Teste automatizado de isolamento cross-tenant (gate de CI)
**Como** time, **quero** um teste provando que o tenant A nunca enxerga dados do tenant B,
**para** que nenhum recurso novo possa vazar dados (gate obrigatório).
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §4.1; ARCHITECTURE §11; CLAUDE.md "Testes".

```gherkin
Given dois schemas de tenant efêmeros A e B com dados distintos
When uma requisição autenticada como usuário do tenant A lista/lê qualquer recurso
Then somente dados do tenant A são retornados
And nenhum id/registro do tenant B aparece na resposta

Given um PR que adiciona um novo recurso de dados de tenant
When o pipeline de CI roda
Then existe um teste de isolamento cobrindo o novo recurso
And o build falha se o teste de isolamento estiver ausente ou vermelho
```

### US-0.4 — Jobs de fila carregam e respeitam o tenant
**Como** plataforma, **quero** que todo job no worker carregue `tenantId` no payload e resolva o schema igual ao request,
**para** que processamento assíncrono não vaze dados entre tenants.
**Prioridade:** MVP · **Estimativa:** M · **RN:** ARCHITECTURE §9.

```gherkin
Given um job enfileirado com payload contendo tenantId = "acme"
When o worker processa o job
Then o worker executa dentro de withTenant("acme", ...) antes de tocar dados
And falha explicitamente se tenantId estiver ausente no payload
```

---

# Épico 1 — Multitenant / Onboarding / Provisionamento

### US-1.1 — Provisionamento automatizado de tenant (saga idempotente)
**Como** Super-Admin, **quero** que ao criar um tenant a plataforma execute uma saga (schema + migrations + seed + Bunny Library + roteamento),
**para** entregar um ambiente isolado e funcional sem passos manuais.
**Prioridade:** MVP · **Estimativa:** G · **RN:** PRD §4.6, §5.1; ARCHITECTURE §3.4.

```gherkin
Given uma nova assinatura de tenant criada
When a saga de provisionamento inicia
Then é feito INSERT em platform.tenants com status "provisioning" e idempotency_key
And é criado o schema "tenant_<slug>" (CREATE SCHEMA IF NOT EXISTS)
And as migrations são aplicadas no schema
And o seed cria papéis + admin do tenant + curso de exemplo
And é criada a Bunny Video Library do tenant com keys cifradas em platform.tenants
And ao final o status passa a "active" e um smoke test pós-provisionamento é executado
And o admin do tenant recebe e-mail de boas-vindas

Given uma saga que falhou no passo "criar Bunny Library"
When a saga é reexecutada com a mesma idempotency_key
Then os passos já concluídos não são repetidos (IF NOT EXISTS / migrations versionadas)
And a saga retoma do passo que falhou
And attempts e last_error são registrados em platform.provisioning_jobs

Given o smoke test pós-provisionamento falha
When a saga finaliza
Then o tenant NÃO é marcado "active"
And o incidente é registrado para o Super-Admin
```

### US-1.2 — Branding por tenant (logo, cores, subdomínio)
**Como** Admin do Tenant, **quero** configurar logo, cores e usar meu subdomínio,
**para** entregar uma marca própria aos meus alunos (white-label).
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.11.

```gherkin
Given que sou Admin do Tenant autenticado
When eu defino logo, cor primária e cor secundária
Then o tema é aplicado via CSS variables em toda a área do tenant
And o subdomínio "acme.app.com" carrega o branding configurado

Given um arquivo de logo acima do limite de tamanho ou com formato inválido
When tento salvar
Then recebo erro de validação e o branding anterior é mantido
```

### US-1.3 — Resolução de tenant por subdomínio no frontend
**Como** Aluno, **quero** acessar a escola pelo subdomínio dela,
**para** ter uma experiência coesa de marca.
**Prioridade:** MVP · **Estimativa:** M · **RN:** ARCHITECTURE §3.2, §5.

```gherkin
Given que acesso "acme.app.com"
When o middleware do Next.js resolve o tenant pelo subdomínio
Then o contexto de tenant e branding são injetados nas páginas
And as chamadas à API enviam credenciais de sessão do tenant correto

Given um subdomínio inexistente "naoexiste.app.com"
When acesso a URL
Then vejo uma página de "tenant não encontrado" (404) sem vazar lista de tenants
```

### US-1.4 — Domínio próprio com SSL automático
**Como** Admin do Tenant, **quero** usar meu domínio próprio com SSL,
**para** remover totalmente a marca da plataforma.
**Prioridade:** F2 · **Estimativa:** G · **RN:** PRD §3.11.

```gherkin
Given que configurei um domínio próprio e apontei o DNS
When a verificação de DNS conclui
Then o SSL é emitido automaticamente e o domínio serve o tenant
```

---

# Épico 2 — Identidade, Auth e RBAC

### US-2.1 — Cadastro e login de usuário por tenant
**Como** Aluno, **quero** me cadastrar e logar na escola,
**para** acessar os cursos que comprei.
**Prioridade:** MVP · **Estimativa:** M · **RN:** ARCHITECTURE §6; DATA_MODEL §2.1.

```gherkin
Given que estou no subdomínio do tenant acme
When me cadastro com e-mail e senha
Then um usuário é criado no schema tenant_acme com role "student"
And recebo sessão via cookie HttpOnly + Secure

Given que o mesmo e-mail já existe no tenant acme
When tento cadastrar novamente
Then recebo erro de e-mail já em uso (unicidade dentro do schema)

Given que o mesmo e-mail existe no tenant acme mas não no tenant escola2
When me cadastro em escola2 com esse e-mail
Then o cadastro é permitido (e-mail único apenas dentro do schema do tenant)
```

### US-2.2 — RBAC por tenant aplicado em guards e use-cases
**Como** plataforma, **quero** papéis (owner/admin/instructor/affiliate/student) aplicados em guards e revalidados nos use-cases,
**para** garantir autorização consistente (defense in depth).
**Prioridade:** MVP · **Estimativa:** M · **RN:** ARCHITECTURE §6.

```gherkin
Given um usuário com role "student"
When tenta acessar a rota de criação de curso
Then recebe 403 no guard de rota
And o use-case também rejeitaria caso fosse chamado diretamente

Given um usuário com role "instructor"
When cria um curso do qual é instrutor
Then a ação é permitida
```

### US-2.3 — Gestão de equipe do tenant (convites de papéis)
**Como** Admin do Tenant, **quero** convidar membros e atribuir papéis,
**para** montar minha equipe (instrutores, admins, afiliados).
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD Personas; DATA_MODEL §2.1.

```gherkin
Given que sou Admin do Tenant
When convido um e-mail com role "instructor"
Then um convite é enviado e, ao aceitar, o usuário recebe a role no tenant

Given um convite com role inválida ou acima do meu nível
When tento enviar
Then recebo erro de validação/autorização
```

### US-2.4 — Recuperação de senha
**Como** Aluno, **quero** redefinir minha senha por e-mail,
**para** recuperar o acesso quando esquecer.
**Prioridade:** MVP · **Estimativa:** P · **RN:** ARCHITECTURE §6.

```gherkin
Given que esqueci a senha
When solicito redefinição informando meu e-mail
Then recebo e-mail com link de uso único e expiração curta
And ao redefinir, sessões antigas são invalidadas

Given um e-mail não cadastrado no tenant
When solicito redefinição
Then a resposta é genérica (sem revelar se o e-mail existe)
```

---

# Épico 3 — Catálogo de Conteúdo

### US-3.1 — Criar curso com hierarquia Curso → Módulo → Aula
**Como** Instrutor, **quero** criar um curso com módulos e aulas,
**para** organizar meu conteúdo.
**Prioridade:** MVP · **Estimativa:** G · **RN:** PRD §3.1; DATA_MODEL §2.2.

```gherkin
Given que sou Instrutor
When crio um curso com título, slug e descrição
Then o curso é criado com status "draft"
And posso adicionar módulos e, dentro deles, aulas

Given um slug de curso já existente no tenant
When tento criar o curso
Then recebo erro de slug duplicado (unique por schema)
```

### US-3.2 — Reordenar módulos e aulas (drag-and-drop)
**Como** Instrutor, **quero** reordenar módulos e aulas arrastando,
**para** ajustar a sequência didática.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.1.

```gherkin
Given um curso com módulos e aulas
When arrasto uma aula para outra posição
Then o campo "position" é persistido e a ordem reflete a nova sequência
And a reordenação respeita o módulo de destino
```

### US-3.3 — Tipos de conteúdo: vídeo, texto rico, PDF/anexo
**Como** Instrutor, **quero** aulas em vídeo, texto rico e PDFs/anexos para download,
**para** variar os formatos de ensino.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.1; DATA_MODEL §2.2.

```gherkin
Given uma aula do tipo "text"
When salvo o conteúdo rico
Then o conteúdo é persistido em content (jsonb)

Given uma aula do tipo "pdf"
When anexo um arquivo
Then o anexo é armazenado no R2 e referenciado em lesson_assets
And o aluno matriculado pode baixá-lo por URL com acesso controlado
```

### US-3.4 — Rascunho e publicação de curso/aula
**Como** Instrutor, **quero** manter conteúdo em rascunho e publicar quando pronto,
**para** editar sem expor aos alunos.
**Prioridade:** MVP · **Estimativa:** P · **RN:** PRD §3.1.

```gherkin
Given um curso/aula com status "draft"
When um aluno tenta acessar
Then o conteúdo não aparece para ele

Given que publico o curso/aula
When um aluno matriculado acessa
Then o conteúdo publicado fica visível
```

### US-3.5 — Drip por data fixa e por dias após a matrícula
**Como** Instrutor, **quero** programar a liberação de aulas por data fixa ou por dias após a matrícula,
**para** controlar o ritmo do curso.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.1; DATA_MODEL §2.2 (drip_release_at, drip_days_after_enroll).

```gherkin
Given uma aula com drip_release_at no futuro
When um aluno acessa antes da data
Then a aula aparece bloqueada com a data de liberação

Given uma aula com drip_days_after_enroll = 7
And um aluno matriculado há 3 dias
When ele acessa a aula
Then a aula está bloqueada e indica liberação em 4 dias

Given um aluno matriculado há 10 dias e drip_days_after_enroll = 7
When ele acessa a aula
Then a aula está liberada
```

### US-3.6 — Pré-requisitos / bloqueio sequencial de aulas
**Como** Instrutor, **quero** liberar uma aula só após a anterior ser concluída,
**para** garantir progressão sequencial.
**Prioridade:** F2 · **Estimativa:** M · **RN:** PRD §3.1.

```gherkin
Given uma aula B com pré-requisito de concluir a aula A
When o aluno não concluiu A
Then B permanece bloqueada
When o aluno conclui A (regra de conclusão ≥90% + anti-seek)
Then B é liberada
```

### US-3.7 — Biblioteca de mídia reutilizável; quiz como aula; aula ao vivo; áudio
**Como** Instrutor, **quero** reutilizar mídias entre cursos e usar novos tipos de aula (quiz, ao vivo, áudio),
**para** ampliar formatos sem retrabalho.
**Prioridade:** F2 · **Estimativa:** G · **RN:** PRD §3.1.

```gherkin
Given uma mídia já enviada
When a referencio em outro curso
Then ela é reutilizada sem novo upload
```

### US-3.8 — Importação SCORM/xAPI e versionamento de conteúdo
**Como** Admin do Tenant (B2B), **quero** importar pacotes SCORM/xAPI e versionar conteúdo,
**para** atender mercado corporativo/educacional.
**Prioridade:** F3 · **Estimativa:** G · **RN:** PRD §3.1.

```gherkin
Given um pacote SCORM válido
When o importo
Then ele é convertido/registrado como conteúdo navegável e rastreável
```

---

# Épico 4 — Vídeo e Player

### US-4.1 — Upload de vídeo direto ao Bunny via TUS (resumable)
**Como** Instrutor, **quero** subir vídeos direto ao Bunny com upload retomável,
**para** enviar arquivos grandes sem consumir banda do backend e sem expor a API key.
**Prioridade:** MVP · **Estimativa:** G · **RN:** PRD §5.2; ARCHITECTURE §7.

```gherkin
Given que inicio o upload de um vídeo numa aula
When o backend cria o vídeo na library do tenant e gera assinatura
Then o upload ocorre direto do browser ao edge da Bunny via TUS
And a API key da Bunny nunca é exposta ao frontend

Given que o upload é interrompido
When eu retomo
Then o upload continua do ponto onde parou (resumable)
```

### US-4.2 — Webhook "vídeo pronto" com HMAC e idempotência
**Como** plataforma, **quero** processar o webhook de encoding concluído com verificação HMAC e idempotência,
**para** atualizar o status do vídeo de forma segura e confiável.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.15, §5.2; ARCHITECTURE §7.

```gherkin
Given um webhook "vídeo pronto" da Bunny
When o endpoint verifica o HMAC sobre os bytes brutos em tempo constante
Then responde 200 e enfileira um job
And o worker mapeia VideoLibraryId → tenant e atualiza o status da aula

Given um webhook com assinatura inválida
When recebido
Then é rejeitado com 401 e nada é processado

Given o mesmo evento de webhook recebido duas vezes
When processado
Then o efeito é aplicado uma única vez (idempotência)
```

### US-4.3 — Reprodução segura com validação de entitlement e token de TTL curto
**Como** Aluno, **quero** assistir aulas apenas se tenho acesso válido,
**para** que o conteúdo fique protegido e disponível só a quem comprou.
**Prioridade:** MVP · **Estimativa:** G · **RN:** PRD §4.2; ARCHITECTURE §7.

```gherkin
Given que solicito reproduzir uma aula em vídeo
When o backend valida que tenho matrícula/pagamento ativo
Then gera um Embed Token (SHA256(key+videoId+expires)) com TTL curto (1–12h)
And o player renderiza com a URL assinada

Given um aluno sem matrícula ativa (suspenso/reembolsado/expirado)
When solicita reprodução
Then o backend NÃO gera a URL assinada e retorna 403

Given um token expirado
When o player tenta carregar o vídeo
Then a reprodução falha e o front solicita novo token
```

### US-4.4 — Player Bunny com HLS, velocidade, retomar e legendas
**Como** Aluno, **quero** player com HLS adaptativo, velocidade 0.5x–2x, retomar de onde parei e legendas,
**para** uma experiência de aprendizado confortável e acessível.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.2, §3.14.

```gherkin
Given que assisti parte de uma aula
When retorno à aula depois
Then o player retoma da última posição registrada

Given que ativo a velocidade 1.5x e as legendas
Then a reprodução respeita a velocidade e exibe legendas
```

### US-4.5 — Tracking de progresso por heartbeats
**Como** plataforma, **quero** registrar posição e % assistido via heartbeats,
**para** suportar conclusão, gamificação e certificado.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.2, §4.4; ARCHITECTURE §7.

```gherkin
Given que estou assistindo uma aula
When o player emite timeupdate (throttle 5–15s)
Then POST /progress atualiza position_seconds e watched_pct em lesson_progress

Given que paro de assistir
When retomo depois
Then o progresso anterior é preservado
```

### US-4.6 — Conclusão de aula com ≥90% assistido + anti-seek
**Como** plataforma, **quero** marcar a aula como concluída só com ≥90% assistido e checagem anti-seek,
**para** evitar fraude de progresso e sustentar a emissão de certificado.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §4.4 (≥90% + tempo real ≥ duração × 0.8).

```gherkin
Given uma aula em que assisti 95% do conteúdo
And o tempo real assistido é ≥ duração × 0.8 (anti-seek)
When o progresso é avaliado
Then a aula é marcada como "completed" com completed_at preenchido

Given que pulei (seek) para o fim atingindo 95% de posição
But o tempo real assistido é < duração × 0.8
When o progresso é avaliado
Then a aula NÃO é marcada como concluída
```

### US-4.7 — Anti-pirataria fase 1 (Token + MediaCage Basic + MP4 off + referer)
**Como** Instrutor, **quero** proteção básica contra pirataria de vídeo,
**para** dificultar o roubo do meu conteúdo.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.16; ARCHITECTURE §7.

```gherkin
Given um vídeo do tenant
When configurado
Then Token Authentication está ativo, MediaCage Basic ligado, MP4 progressivo desabilitado
And o referer está restrito aos domínios do tenant

Given uma tentativa de acessar o vídeo fora do domínio permitido
When a requisição chega
Then o acesso é negado pela restrição de referer
```

### US-4.8 — Player próprio (Vidstack) com watermark dinâmico por aluno
**Como** Instrutor, **quero** watermark com e-mail/CPF do aluno sobre o vídeo,
**para** rastrear vazamentos de conteúdo.
**Prioridade:** F2 · **Estimativa:** G · **RN:** PRD §3.16, §3.2.

```gherkin
Given um aluno assistindo no player Vidstack
Then um overlay com o identificador do aluno é exibido dinamicamente sobre o vídeo
```

### US-4.9 — Anotações, marcadores, transcrição pesquisável; DRM Enterprise
**Como** Aluno, **quero** anotar e marcar timestamps e buscar na transcrição (e DRM quando premium),
**para** estudar melhor e ter proteção máxima quando necessário.
**Prioridade:** F2/F3 · **Estimativa:** G · **RN:** PRD §3.2, §3.16.

```gherkin
Given uma aula com transcrição
When busco um termo
Then os trechos com o termo são listados com timestamp clicável
```

---

# Épico 5 — Avaliações e Certificados

### US-5.1 — Quiz builder (múltipla escolha e V/F) com correção automática
**Como** Instrutor, **quero** criar quizzes de múltipla escolha e verdadeiro/falso,
**para** avaliar o aluno com correção automática.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.3; DATA_MODEL §2.4.

```gherkin
Given um quiz com questões de múltipla escolha e V/F
When o aluno submete respostas
Then a correção é automática e a nota é calculada e registrada em quiz_attempts

Given um quiz com pass_score definido
When o aluno atinge a nota mínima
Then o resultado indica aprovação
```

### US-5.2 — Certificado de conclusão automático (PDF)
**Como** Aluno, **quero** receber um certificado em PDF ao concluir o curso,
**para** comprovar minha qualificação.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §4.5; DATA_MODEL §2.4.

```gherkin
Given que concluí 100% do curso
And atingi a nota mínima exigida (quando houver quiz com pass_score)
When o sistema avalia a emissão
Then um certificado é gerado (PDF no R2) com uuid_public e hash
And recebo notificação para baixá-lo

Given que ainda não concluí 100% do curso
When tento emitir o certificado
Then a emissão é negada
```

### US-5.3 — Verificação pública de certificado (UUID + QR + hash)
**Como** terceiro (ex.: empregador), **quero** verificar a autenticidade de um certificado,
**para** confiar na qualificação apresentada.
**Prioridade:** MVP · **Estimativa:** P · **RN:** PRD §3.9; DATA_MODEL §2.4.

```gherkin
Given um uuid_public de certificado válido
When acesso a página pública de verificação (ou leio o QR)
Then vejo aluno, curso e data de emissão, com hash conferido

Given um certificado revogado (revoked = true)
When verifico
Then a página indica que o certificado foi revogado

Given um uuid_public inexistente
When verifico
Then a página indica certificado não encontrado, sem vazar dados de outros tenants
```

### US-5.4 — Provas avançadas, gradebook e tipos de questão adicionais
**Como** Instrutor, **quero** tentativas limitadas, nota mínima, tempo, embaralhamento, novos tipos de questão e gradebook,
**para** avaliações mais ricas e acompanhamento por turma.
**Prioridade:** F2 · **Estimativa:** G · **RN:** PRD §3.3.

```gherkin
Given um quiz com max_attempts = 2 e time_limit_sec definido
When o aluno excede as tentativas ou o tempo
Then novas submissões são bloqueadas e a melhor/última nota vale conforme regra configurada
```

### US-5.5 — Tarefas com upload, correção manual/por pares; templates de certificado
**Como** Instrutor, **quero** tarefas com upload e correção manual/por pares e templates de certificado customizáveis,
**para** avaliações abertas e identidade visual própria.
**Prioridade:** F3 · **Estimativa:** G · **RN:** PRD §3.3.

```gherkin
Given uma tarefa com upload
When o aluno envia o arquivo
Then o instrutor (ou par) pode atribuir nota e feedback
```

---

# Épico 6 — Matrícula e Estados de Acesso

> **Núcleo crítico:** "pagamento governa acesso" (PRD §4.3). A máquina de estados liga pagamento ↔ acesso.

### US-6.1 — Máquina de estados de acesso (ativo/suspenso/reembolsado/expirado)
**Como** plataforma, **quero** uma máquina de estados de acesso atrelada ao status de pagamento,
**para** liberar/suspender o acesso de forma consistente.
**Prioridade:** MVP · **Estimativa:** G · **RN:** PRD §3.6, §4.3; DATA_MODEL §2.3 (enrollments.status).

```gherkin
Given um pedido aprovado para um curso
When o webhook de pagamento confirma
Then a matrícula passa/permanece "active" e o acesso é liberado

Given uma matrícula "active"
When ocorre reembolso/chargeback/cancelamento
Then a matrícula passa para "refunded"/"suspended" e o acesso é bloqueado (sem novas URLs assinadas)

Given uma matrícula com expires_at no passado
When o acesso é avaliado
Then a matrícula é tratada como "expired" e o acesso negado

Given uma transição inválida (ex.: "refunded" → "active" sem novo pagamento)
When tentada
Then é rejeitada e registrada
```

### US-6.2 — Matrícula manual, por compra, em massa (CSV) e auto-enroll
**Como** Admin do Tenant, **quero** matricular alunos manualmente, por compra, via CSV ou por regra de auto-enroll,
**para** dar acesso flexível ao conteúdo.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.6; DATA_MODEL §2.3 (enrollments.source).

```gherkin
Given um arquivo CSV de alunos válido
When faço a importação em massa
Then matrículas são criadas com source "bulk" e alunos inexistentes são criados/convidados
And linhas inválidas são reportadas sem abortar as válidas

Given uma matrícula manual de um aluno já matriculado no curso
When tento criar
Then a operação é idempotente (unique user_id+course_id), sem duplicar
```

### US-6.3 — Reconciliação periódica com o gateway
**Como** plataforma, **quero** reconciliar periodicamente os estados com o gateway,
**para** corrigir divergências entre pagamento e acesso.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.6, §4.3; ARCHITECTURE §8.

```gherkin
Given um pagamento marcado pago no gateway mas matrícula não liberada (webhook perdido)
When o job de reconciliação roda
Then a matrícula é liberada e a divergência registrada

Given um reembolso no gateway não refletido localmente
When a reconciliação roda
Then o acesso é suspenso
```

### US-6.4 — Dashboard de progresso do aluno e do instrutor
**Como** Aluno e Instrutor, **quero** ver o progresso (meu / dos meus alunos),
**para** acompanhar o avanço no curso.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.6.

```gherkin
Given que sou Aluno
When abro meu dashboard
Then vejo % de progresso por curso e próximas aulas

Given que sou Instrutor
When abro o dashboard do curso
Then vejo progresso agregado e por aluno (somente do meu tenant)
```

### US-6.5 — Cohorts/turmas e relatórios de conclusão/drop-off
**Como** Admin do Tenant, **quero** turmas com cronograma e relatórios de conclusão/drop-off,
**para** gerir turmas e identificar evasão.
**Prioridade:** F2 · **Estimativa:** G · **RN:** PRD §3.6.

```gherkin
Given uma turma com cronograma compartilhado
When acompanho o relatório
Then vejo conclusão por aluno e pontos de drop-off
```

---

# Épico 7 — Checkout e Pagamentos

> **Núcleo crítico:** estados de pagamento + idempotência de webhook (PRD §4.3, §3.7).

### US-7.1 — Checkout BR (Pix, boleto, cartão com parcelamento)
**Como** Aluno, **quero** pagar por Pix, boleto ou cartão (com parcelamento),
**para** comprar o curso pelo meio que eu preferir.
**Prioridade:** MVP · **Estimativa:** G · **RN:** PRD §3.7; ARCHITECTURE §8.

```gherkin
Given um curso à venda (one_time)
When escolho Pix no checkout
Then um pedido "pending" é criado e recebo o QR/código Pix
When o pagamento Pix é confirmado por webhook
Then o pedido passa a "paid" e a matrícula é liberada

Given pagamento por cartão recusado
When submeto
Then o pedido permanece "pending"/falho e vejo mensagem de recusa, sem liberar acesso

Given pagamento por boleto
When gero o boleto
Then o pedido fica "pending" até a compensação confirmada por webhook
```

### US-7.2 — Assinatura/recorrência (mensal/anual/trial)
**Como** Aluno, **quero** assinar conteúdo recorrente (mensal/anual com trial),
**para** acesso contínuo enquanto pago.
**Prioridade:** MVP · **Estimativa:** G · **RN:** PRD §3.7; DATA_MODEL §2.5 (subscriptions).

```gherkin
Given uma assinatura ativa
When o ciclo é renovado com sucesso
Then current_period_end avança e o acesso continua

Given uma renovação que falha (inadimplência)
When o webhook informa
Then a assinatura entra em estado de suspensão e o acesso é bloqueado conforme política
```

### US-7.3 — Cupons de desconto
**Como** Aluno, **quero** aplicar um cupom válido no checkout,
**para** obter desconto.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.7; DATA_MODEL §2.5 (coupons).

```gherkin
Given um cupom válido dentro do período e com usos disponíveis
When aplico no checkout
Then o desconto é refletido no valor

Given um cupom expirado ou esgotado (uses >= max_uses)
When aplico
Then recebo erro e o valor original é mantido
```

### US-7.4 — Webhooks de pagamento idempotentes ↔ estados de acesso
**Como** plataforma, **quero** processar webhooks de pagamento de forma idempotente e dirigir a máquina de estados de acesso,
**para** que pagamento governe acesso sem efeitos duplicados.
**Prioridade:** MVP · **Estimativa:** G · **RN:** PRD §3.7, §4.3; DATA_MODEL §2.5 (payment_events.event_id unique).

```gherkin
Given um webhook de pagamento com assinatura válida
When recebido
Then a assinatura é verificada e o event_id é registrado em payment_events
And o estado do pedido/matrícula é atualizado conforme o evento

Given o mesmo event_id recebido novamente
When processado
Then o efeito não é reaplicado (idempotência via unique event_id)

Given um webhook com assinatura inválida
When recebido
Then é rejeitado e nada é processado
```

### US-7.5 — Order bump, upsell/downsell, bundles, recuperação de carrinho
**Como** Admin do Tenant, **quero** order bump, upsell/downsell one-click, bundles e recuperação de carrinho,
**para** aumentar ticket médio e conversão.
**Prioridade:** F2 · **Estimativa:** G · **RN:** PRD §3.7.

```gherkin
Given um checkout com order bump configurado
When o aluno aceita o bump
Then o item é adicionado ao pedido sem reentrar dados de pagamento
```

### US-7.6 — Emissão de nota fiscal / integração fiscal
**Como** Admin do Tenant, **quero** emitir nota fiscal,
**para** cumprir obrigações fiscais.
**Prioridade:** F3 · **Estimativa:** G · **RN:** PRD §3.7.

```gherkin
Given um pedido pago
When a integração fiscal está configurada
Then a nota fiscal é emitida e disponibilizada
```

---

# Épico 8 — Afiliados e Split

### US-8.1 — Link de afiliado e atribuição de venda
**Como** Afiliado, **quero** um link único de divulgação,
**para** receber comissão pelas vendas que eu gerar.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.7; DATA_MODEL §2.6 (affiliates.code).

```gherkin
Given que sou Afiliado aprovado com code próprio
When um aluno compra via meu link
Then o pedido registra affiliate_id e uma comissão "pending" é criada conforme commission_pct

Given um pedido sem código de afiliado
When concluído
Then nenhuma comissão de afiliado é gerada
```

### US-8.2 — Painel de comissões do afiliado
**Como** Afiliado, **quero** ver minhas comissões e seus status,
**para** acompanhar meus ganhos.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.7; DATA_MODEL §2.6 (affiliate_commissions.status).

```gherkin
Given comissões geradas pelas minhas vendas
When acesso o painel
Then vejo comissões com status pending/paid/reversed e valores em centavos

Given um pedido associado reembolsado/chargeback
When o estado é processado
Then a comissão correspondente passa a "reversed"
```

### US-8.3 — Split de pagamento (Pagar.me)
**Como** plataforma, **quero** dividir o pagamento entre produtor e afiliado no momento da transação,
**para** repassar comissões automaticamente.
**Prioridade:** MVP · **Estimativa:** G · **RN:** PRD §3.7; ARCHITECTURE §8; DATA_MODEL §2.6 (splits).

```gherkin
Given uma regra de split (course/recipient/percent) e venda via afiliado
When o pagamento é aprovado
Then o split é aplicado conforme os percentuais configurados
And a soma dos percentuais é validada (não excede 100%)

Given percentuais de split somando mais de 100%
When tento salvar a regra
Then recebo erro de validação
```

### US-8.4 — Co-produção (split entre produtores) e materiais de divulgação
**Como** Admin do Tenant, **quero** split entre coprodutores e materiais para afiliados,
**para** parcerias e melhor divulgação.
**Prioridade:** F2 · **Estimativa:** M · **RN:** PRD §3.7.

```gherkin
Given um curso com coprodutores e percentuais definidos
When uma venda é aprovada
Then o split distribui os valores entre os coprodutores
```

### US-8.5 — Marketplace de afiliados (descoberta cross-tenant)
**Como** Afiliado, **quero** descobrir cursos de vários tenants para promover,
**para** ampliar minhas oportunidades.
**Prioridade:** F3 · **Estimativa:** G · **RN:** PRD §3.7 (só se virar estratégia).

> **Atenção:** feature cross-tenant — requer decisão arquitetural específica (ver Dependências).

---

# Épico 9 — Comunidade

### US-9.1 — Comentários por aula
**Como** Aluno, **quero** comentar dúvidas no contexto da aula,
**para** tirar dúvidas e interagir.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.5; DATA_MODEL §2.7 (lesson_comments).

```gherkin
Given uma aula publicada e estou matriculado
When publico um comentário (com ou sem parent_id para resposta)
Then o comentário aparece na aula

Given que sou Instrutor do curso
When respondo ou modero comentários
Then a ação é permitida; comentários só são visíveis dentro do meu tenant
```

### US-9.2 — Fórum/feed, grupos e lives nativas
**Como** Aluno, **quero** fórum/feed por curso e participar de lives,
**para** engajar com a comunidade.
**Prioridade:** F2 · **Estimativa:** G · **RN:** PRD §3.5.

```gherkin
Given um fórum por curso
When crio um tópico
Then ele aparece no feed do curso com possibilidade de respostas
```

### US-9.3 — Chat/DMs, segmentação e resumo de feed por IA
**Como** Aluno, **quero** chat/DMs e resumos de feed,
**para** comunicação direta e digestão de conteúdo.
**Prioridade:** F3 · **Estimativa:** G · **RN:** PRD §3.5.

---

# Épico 10 — Gamificação (F2/F3)

### US-10.1 — Pontos/XP por ação
**Como** Aluno, **quero** ganhar pontos por concluir aulas/quizzes e login diário,
**para** me sentir recompensado.
**Prioridade:** F2 · **Estimativa:** M · **RN:** PRD §3.4.

```gherkin
Given que concluo uma aula (regra de conclusão válida)
When o evento é processado
Then recebo os pontos configurados para a ação
```

### US-10.2 — Badges/conquistas e ranking/leaderboard
**Como** Aluno, **quero** badges e ranking por turma/período,
**para** competir e me motivar.
**Prioridade:** F2 · **Estimativa:** M · **RN:** PRD §3.4.

```gherkin
Given que atinjo um marco
Then recebo o badge correspondente e minha posição no ranking é atualizada
```

### US-10.3 — Níveis, streaks e trilhas
**Como** Aluno, **quero** níveis com desbloqueio, streaks e trilhas,
**para** uma jornada gamificada de longo prazo.
**Prioridade:** F3 · **Estimativa:** G · **RN:** PRD §3.4.

---

# Épico 11 — Analytics e Relatórios

### US-11.1 — Métricas básicas (conclusão por curso, receita por curso)
**Como** Admin do Tenant, **quero** ver aulas assistidas, progresso, conclusão e receita por curso,
**para** entender desempenho e faturamento.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.13.

```gherkin
Given cursos com matrículas, progresso e pedidos pagos
When abro o painel de analytics
Then vejo conclusão por curso e receita por curso (somente do meu tenant)
```

### US-11.2 — Analytics avançado e exportação CSV
**Como** Admin do Tenant, **quero** coorte/retenção, MRR/churn, funil, heatmap, watch-time e export CSV,
**para** decisões de produto e marketing aprofundadas.
**Prioridade:** F2 · **Estimativa:** G · **RN:** PRD §3.13; métricas North Star (PRD §6).

```gherkin
Given dados de engajamento e receita
When solicito o relatório de coorte/MRR
Then vejo as métricas no período e posso exportar em CSV
```

---

# Épico 12 — Super-Admin e Billing SaaS

### US-12.1 — Painel Super-Admin: listar e gerir tenants
**Como** Super-Admin, **quero** listar e gerir tenants e seus status,
**para** operar a plataforma.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.12; DATA_MODEL §1.

```gherkin
Given que sou Super-Admin autenticado (identidade no schema platform)
When listo os tenants
Then vejo slug, status (provisioning/active/suspended/cancelled), plano e métricas básicas
```

### US-12.2 — Suspender/ativar tenant
**Como** Super-Admin, **quero** suspender ou ativar um tenant,
**para** controlar acesso conforme situação comercial/operacional.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.12, §1.3.

```gherkin
Given um tenant "active"
When o suspendo
Then o status vira "suspended" e o acesso ao tenant é bloqueado
And a ação é registrada em audit_log

Given um tenant suspenso por inadimplência
When o pagamento do SaaS é regularizado
Then ele pode ser reativado para "active"
```

### US-12.3 — Impersonação auditada para suporte
**Como** Super-Admin, **quero** impersonar um usuário de tenant para suporte,
**para** diagnosticar problemas — sempre com registro.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §4.7; ARCHITECTURE §6.

```gherkin
Given que preciso dar suporte a um tenant
When inicio uma sessão de impersonação
Then a sessão é registrada em audit_log (ator, tenant, ação, timestamp)
And não há acesso implícito a dados de tenant sem esse registro

Given o fim da impersonação
Then a sessão é encerrada e o evento de término é auditado
```

### US-12.4 — Planos do SaaS + quotas
**Como** Super-Admin, **quero** definir planos com limites/quotas (alunos, storage, features),
**para** monetizar por tier.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.12, §1.3; DATA_MODEL §1 (platform_plans.limits).

```gherkin
Given um plano com max_students e storage_gb
When um tenant excede a quota
Then a plataforma bloqueia/avisa conforme política de quota

Given uma feature não incluída no plano do tenant
When o tenant tenta usá-la
Then ela é bloqueada com mensagem de upgrade
```

### US-12.5 — Billing do SaaS (Stripe Billing) controla o tenant
**Como** plataforma, **quero** que o status da assinatura Stripe ative/suspenda o tenant,
**para** que o pagamento do SaaS governe a disponibilidade do tenant.
**Prioridade:** MVP · **Estimativa:** G · **RN:** PRD §1.3, §3.12; ARCHITECTURE §8; DATA_MODEL §1 (platform_subscriptions).

```gherkin
Given uma assinatura Stripe do tenant que entra em "past_due"/"canceled"
When o webhook do Stripe é processado
Then o tenant é suspenso conforme política e o evento é registrado

Given uma assinatura regularizada
When o webhook confirma "active"
Then o tenant é reativado

Given um webhook Stripe duplicado
When processado
Then o efeito é idempotente
```

---

# Épico 13 — LGPD, Acessibilidade e Compliance

### US-13.1 — Consentimento de dados (LGPD)
**Como** Aluno, **quero** dar/registrar meu consentimento de uso de dados,
**para** ter transparência e controle.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.9.

```gherkin
Given que acesso a plataforma pela primeira vez
When concedo consentimento
Then o consentimento é registrado com data/versão dos termos
```

### US-13.2 — Exportação de dados do aluno
**Como** Aluno, **quero** exportar meus dados,
**para** exercer meu direito de portabilidade.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.9; ARCHITECTURE §3.6.

```gherkin
Given que solicito exportação dos meus dados
When o job processa
Then recebo um arquivo com meus dados pessoais e de uso do tenant atual
```

### US-13.3 — Deleção/anonimização de dados do aluno (direito ao esquecimento)
**Como** Aluno, **quero** solicitar a deleção dos meus dados,
**para** exercer o direito ao esquecimento.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.9; ARCHITECTURE §3.6.

```gherkin
Given que solicito deleção
When a requisição é processada
Then meus dados pessoais são deletados/anonimizados dentro do schema do tenant
And dados que devem ser retidos por obrigação legal (ex.: registros fiscais) seguem política definida
```

### US-13.4 — Deleção de tenant via DROP SCHEMA
**Como** Super-Admin, **quero** excluir um tenant com `DROP SCHEMA CASCADE`,
**para** remover totalmente os dados ao encerrar contrato.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.9; ARCHITECTURE §3.6.

```gherkin
Given um tenant "cancelled" com confirmação explícita
When executo a deleção
Then o schema tenant_<slug> é removido (DROP SCHEMA CASCADE)
And a ação é registrada em audit_log
And opcionalmente um export final é gerado antes da deleção (anti lock-in)
```

### US-13.5 — Baseline de acessibilidade WCAG 2.1 AA
**Como** Aluno com deficiência, **quero** navegar por teclado, com bom contraste, labels e legendas,
**para** usar a plataforma de forma acessível.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.14, §4 (tratado desde o início).

```gherkin
Given qualquer tela da área logada/pública
When navego apenas por teclado
Then todos os controles interativos são alcançáveis e operáveis
And contraste, labels de formulário e foco visível atendem WCAG 2.1 AA
And vídeos suportam legendas
```

### US-13.6 — Trilha de auditoria e compliance training
**Como** Admin do Tenant, **quero** logs de acesso e (B2B) validade/recertificação,
**para** auditoria e conformidade corporativa.
**Prioridade:** F2/F3 · **Estimativa:** M · **RN:** PRD §3.9.

```gherkin
Given ações sensíveis dentro do tenant
When ocorrem
Then são registradas em uma trilha de auditoria consultável pelo Admin do Tenant
```

---

# Épico 14 — Marketing e Integrações

### US-14.1 — Landing page por curso com SEO básico e pixels/UTM
**Como** Admin do Tenant, **quero** uma página de vendas por curso com SEO e pixels (Meta, GA4) + UTM,
**para** atrair e converter alunos.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.8.

```gherkin
Given um curso publicado
When configuro a landing com SEO e pixels
Then a página é indexável (SSG/ISR), dispara os pixels e preserva UTMs até o checkout
```

### US-14.2 — Webhooks de saída (compra, reembolso, matrícula, conclusão)
**Como** Admin do Tenant, **quero** receber webhooks de eventos chave,
**para** integrar meu CRM/automação.
**Prioridade:** MVP · **Estimativa:** M · **RN:** PRD §3.15.

```gherkin
Given um endpoint de webhook de saída configurado pelo tenant
When ocorre compra aprovada/reembolso/matrícula/conclusão
Then o evento é entregue com assinatura
And há retries em caso de falha de entrega
```

### US-14.3 — E-mail marketing/automação e integração RD/ActiveCampaign
**Como** Admin do Tenant, **quero** sequências de e-mail e integração com RD/ActiveCampaign,
**para** nutrir e recuperar alunos.
**Prioridade:** F2 · **Estimativa:** G · **RN:** PRD §3.8.

```gherkin
Given uma sequência de boas-vindas configurada
When um aluno se matricula
Then a sequência é disparada via event bus
```

### US-14.4 — API REST pública (OpenAPI) com chaves por tenant
**Como** Admin do Tenant (técnico), **quero** uma API REST documentada com chave por tenant,
**para** integrações customizadas.
**Prioridade:** F2 · **Estimativa:** G · **RN:** PRD §3.15; ARCHITECTURE (OpenAPI gerado do Zod).

```gherkin
Given uma chave de API válida do meu tenant
When chamo a API pública
Then só acesso dados do meu tenant e a documentação OpenAPI reflete os contratos Zod
```

### US-14.5 — Construtor de landing/funil, blog; Zapier/Make, SSO, LTI
**Como** Admin do Tenant, **quero** construtor de funil/blog e integrações avançadas (Zapier/SSO/LTI),
**para** marketing e integração enterprise.
**Prioridade:** F3 · **Estimativa:** G · **RN:** PRD §3.8, §3.15.

---

# Épico 15 — Mobile, i18n e White-label avançado (F2/F3)

### US-15.1 — PWA instalável com push
**Como** Aluno, **quero** instalar a plataforma como app (PWA) e receber push,
**para** acesso rápido e notificações.
**Prioridade:** F2 · **Estimativa:** G · **RN:** PRD §3.10.

```gherkin
Given um dispositivo compatível
When instalo o PWA
Then a plataforma abre como app e posso optar por receber push
```

### US-15.2 — i18n da interface (PT-BR no lançamento; pronto para ES/EN)
**Como** Aluno, **quero** a interface no meu idioma,
**para** usar a plataforma confortavelmente.
**Prioridade:** F2 · **Estimativa:** M · **RN:** PRD §3.11 (arquitetura i18n desde o início).

```gherkin
Given a interface com next-intl
When troco o idioma (quando disponível)
Then os textos são traduzidos sem quebrar o layout
```

### US-15.3 — App white-label nas lojas; download offline seguro
**Como** Admin do Tenant, **quero** app próprio nas lojas e download offline seguro,
**para** experiência premium e consumo offline.
**Prioridade:** F3 · **Estimativa:** G · **RN:** PRD §3.10, §3.2.

---

# Épico 16 — IA na plataforma (F2/F3)

### US-16.1 — Transcrição/legenda automática
**Como** Instrutor, **quero** transcrição/legenda automática das aulas,
**para** acessibilidade, busca e SEO.
**Prioridade:** F2 · **Estimativa:** M · **RN:** PRD §3.17; DATA_MODEL §2.8 (lesson_transcripts).

```gherkin
Given um vídeo processado
When a transcrição automática roda
Then a legenda (VTT) é gerada e disponibilizada no player e para busca
```

### US-16.2 — Geração de quizzes a partir da transcrição
**Como** Instrutor, **quero** gerar quizzes a partir da transcrição (com revisão humana),
**para** acelerar a criação de avaliações.
**Prioridade:** F2 · **Estimativa:** M · **RN:** PRD §3.17.

```gherkin
Given uma transcrição disponível
When solicito geração de quiz
Then um rascunho de quiz é criado para revisão/edição antes de publicar
```

### US-16.3 — Tutor IA (RAG) e recomendação
**Como** Aluno, **quero** um tutor IA que responda com base no conteúdo e recomende próximos passos,
**para** aprender com apoio personalizado.
**Prioridade:** F3 · **Estimativa:** G · **RN:** PRD §3.17; DATA_MODEL §2.8 (lesson_embeddings, pgvector).

```gherkin
Given embeddings das transcrições do curso
When pergunto ao tutor IA
Then a resposta cita trechos do conteúdo do meu tenant (sem vazar de outros tenants)
```

---

## Dependências e pontos para o coordenador

> Pontos de coordenação cross-épico, conflitos de escopo e stories que dependem de decisões de
> **pricing / RBAC / estados** ainda não totalmente fechadas.

### A. Estados de pagamento e acesso (núcleo de risco)
- **US-6.1 (máquina de estados) é dependência dura** de US-4.3 (token só com entitlement), US-7.1/7.2/7.4
  (checkout e webhooks) e US-8.2 (reversão de comissão). Definir formalmente **o diagrama de estados**
  e as transições permitidas/negadas antes de implementar checkout.
- **Conflito a resolver:** política de suspensão em **inadimplência de assinatura do aluno** (US-7.2)
  vs. **período de carência** (grace period). O PRD cita estados (ativo/suspenso/reembolsado/expirado),
  mas não define carência — **decisão de produto pendente**.
- **Chargeback vs. reembolso:** ambos suspendem acesso e revertem comissão (US-8.2), mas têm SLAs e
  fluxos de disputa diferentes. Confirmar se há tratamento distinto no MVP.
- **Webhooks perdidos:** US-6.3 (reconciliação) é a rede de segurança de US-7.4. Definir periodicidade.

### B. RBAC e permissões
- **US-2.2 / US-2.3:** matriz de permissões por papel (owner/admin/instructor/affiliate/student) **não
  está detalhada** no PRD. Necessário um **mapa RBAC** (quem pode o quê) antes do MVP — afeta quase todos
  os épicos. Em especial: pode o `instructor` ver dados financeiros? Afiliado vê dados de alunos?
- **Super-Admin (US-12.x)** vive no `platform`; impersonação (US-12.3) precisa de política clara de
  escopo (o que pode/não pode fazer impersonando) e retenção dos logs de auditoria.

### C. Pricing / planos / quotas
- **US-12.4 (quotas) e US-12.5 (Stripe Billing):** os **tiers, limites e features por plano** ainda não
  foram especificados (PRD aponta a estrutura, não os valores). Bloqueia enforcement de quota e gating de
  features. Decisão comercial pendente.
- **Comportamento ao exceder quota** (bloquear vs. cobrar excedente vs. avisar) precisa ser definido.

### D. Isolamento multitenant (transversal)
- **Épico 0** é pré-requisito de **todos** os épicos de dados e é **gate de CI**. Nenhuma story de dados
  deve ser dada como "pronta" sem o teste de isolamento (US-0.3).
- **US-8.5 (marketplace cross-tenant)** e **US-16.3 (tutor IA com RAG)** são os dois pontos onde dados
  podem legitimamente cruzar/serem agregados — exigem **ADR específico** antes de qualquer
  implementação, pois conflitam com o princípio de isolamento absoluto (PRD §4.1).

### E. Conclusão e certificado (regras encadeadas)
- **US-4.6 (≥90% + anti-seek)** alimenta **US-5.2 (certificado)** e **US-10.1 (pontos)**. A definição de
  "tempo real assistido" (US-4.6) precisa de acordo técnico com o heartbeat (US-4.5) — granularidade do
  throttle (5–15s) impacta a precisão do anti-seek.
- **US-5.2:** quando há quiz com `pass_score`, a emissão depende da nota mínima. Confirmar regra para
  cursos **sem** quiz (basta 100% de progresso).

### F. Vídeo e provisionamento
- **US-4.1/4.2/4.3** dependem de **US-1.1 (provisionamento)** ter criado a Bunny Library e guardado as
  keys cifradas. Sequenciar: provisionamento → catálogo → vídeo.
- **Custo de egress** (risco do PRD §7) não é uma story funcional, mas impacta US-4.x — manter como item
  de NFR/observabilidade (cap de resolução, alertas).

### G. Sequenciamento sugerido (alinhado ao ROADMAP §"Sequência")
1. Épico 0 + PoC de tenant/auth (Épicos 2 parcial) → 2. Épico 1 (provisionamento) + Épico 12 mínimo →
3. Épico 3 (catálogo) + Épico 4 (vídeo) → 4. Épico 7 (checkout) + Épico 6 (acesso) + Épico 8 (afiliados)
→ 5. Épico 5 (quiz/certificado) + Épico 11 (analytics básico) + Épico 13 (LGPD/WCAG) → hardening.

### H. Pontos abertos a confirmar com stakeholder
- Grace period de inadimplência (aluno e SaaS).
- Matriz RBAC detalhada.
- Tabela de planos/quotas/preços do SaaS.
- Tratamento fiscal/nota fiscal (F3) — confirmar se sai do escopo no MVP (atualmente F3).
- Decisão sobre marketplace cross-tenant (F3) e tutor IA (F3) frente ao isolamento.
```