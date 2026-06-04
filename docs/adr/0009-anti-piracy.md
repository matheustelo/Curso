# ADR-0009 — Anti-pirataria / proteção de conteúdo: estratégia faseada

- **Status:** Aceito · **Data:** 2026-06-04 · **Confiança:** 85%

## Contexto
Proteção de vídeo é dor real de infoprodutores BR. O stakeholder não tem preferência definida
("não sei"), então a decisão coube ao time técnico. Há um trade-off: **não dá para ter DRM Enterprise
e watermark dinâmico por aluno simultaneamente** sem esforço grande (DRM funciona out-of-the-box no
player iframe da Bunny; watermark por aluno exige player próprio).

## Opções consideradas
1. **Token + MediaCage Basic (player iframe Bunny):** rápido de entregar; sem watermark por aluno;
não impede screen recording.
2. **Watermark dinâmico por aluno (player Vidstack):** overlay com e-mail/CPF; forte deterrente contra
compartilhamento; exige player próprio; quebra o plug-and-play do DRM.
3. **DRM Enterprise (Widevine/FairPlay):** proteção "Hollywood" + anti screen-grab; +US$99/mês +
licenças; mais complexo; sem watermark por aluno.

## Decisão (faseada)
- **MVP:** Token Authentication + **MediaCage Basic (grátis)** + MP4 progressivo **desabilitado** +
restrição de **referer** + expiração curta. (Opção 1 — entrega rápida, cobre a maioria.)
- **Fase 2:** player próprio **Vidstack** com **watermark dinâmico por aluno** (Opção 2) — melhor
custo-benefício como deterrente.
- **Sob demanda:** **DRM Enterprise** (Opção 3) apenas se um cliente premium exigir.
- **`VideoProvider`** abstraído (SOLID) mantém todas as portas abertas sem retrabalho de domínio.

## Justificativa
Equilibra velocidade de entrega (MVP) com o deterrente mais eficaz e barato (watermark) na F2,
preservando a opção de DRM "Hollywood" para contratos que justifiquem o custo.

## Consequências
- A troca do player iframe → Vidstack na F2 é localizada (atrás da port `VideoProvider`/componente de
player). Decisão revisável quando surgir cliente premium (ver OPEN_QUESTIONS #4).

## Fontes
docs.bunny.net (Token Auth, MediaCage Basic/Enterprise, security options); relatório de streaming.
