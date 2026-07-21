# Da Reunião ao Documento — Pacote de Design Docs (processo)

Este repositório é a minha entrega do desafio de MBA **"Da Reunião ao Documento: Design Docs
Gerados por IA"**. O enunciado original está preservado em [`DESAFIO.md`](./DESAFIO.md); este
README documenta **o processo** que usei para produzir a entrega.

> **Onde está a entrega:** todos os documentos estão em [`docs/`](./docs/). Ordem de leitura
> sugerida na seção [Como navegar a entrega](#como-navegar-a-entrega).

## Sobre o desafio

O desafio parte de uma situação real de engenharia: uma reunião técnica de ~55 minutos decidiu
como construir um **Sistema de Webhooks de Notificação de Pedidos** para um Order Management
System (OMS) em produção, mas nada ficou registrado além da transcrição da call
([`TRANSCRICAO.md`](./TRANSCRICAO.md)). A tarefa é transformar essa transcrição — mais o código
existente da aplicação — num pacote completo de design docs (PRD, RFC, FDD, ADRs, Tracker) em
nível acionável para o time começar a implementar.

O que se avalia não é "gerar texto com IA", e sim o papel de **maestro**: decidir o que entra e
o que fica de fora, formular bons prompts, e revisar criticamente cada entrega da IA. A restrição
central é **rastreabilidade** — toda afirmação precisa ter origem identificável na transcrição
(timestamp + falante) ou no código (caminho de arquivo). Nada de requisito inventado. A entrega é
**puramente documental**: o código (`src/`, `prisma/`, `tests/`) serve só de contexto e não foi
alterado.

## Ferramentas de IA utilizadas

- **Claude Code (Opus 4.8), modo "ultracode"** — ferramenta principal. Leu o código e a
  transcrição, estruturou os documentos e gerou o conteúdo. Também **orquestrou subagentes** para
  paralelizar e verificar o trabalho.
- **Orquestração multi-agente (workflows do Claude Code)** — usada em duas frentes: gerar os 6
  ADRs em paralelo (um escritor por decisão) e montar o Tracker (um extrator por documento +
  um montador).
- **Subagentes verificadores adversariais** — o pilar de qualidade. Para cada documento, um
  agente com o papel de "revisor cético" abria a transcrição e o código e conferia **cada
  citação** e **cada caminho de arquivo**, com veredito `PASS`/`NEEDS_FIX`. Foi o que pegou as
  alucinações sutis.

## Workflow adotado

Segui a ordem sugerida pelo desafio, porque as decisões formam o esqueleto do resto:

1. **Contextualização** — a IA leu `DESAFIO.md`, `TRANSCRICAO.md` e os pontos-chave do código
   (o método `changeStatus`, a máquina de estados, as classes de erro, o `requireRole`, o error
   middleware, o logger Pino).
2. **ADRs primeiro** — escrevi o `ADR-001` sozinho, como **calibração**, e só depois de aprovar
   o formato gerei os demais (002–007) em paralelo, cada um verificado adversarialmente.
3. **RFC** — consolidou as decisões numa proposta de arquitetura, sem descer ao detalhe do FDD.
4. **FDD** — o "como construir": fluxos, contratos, matriz de erros `WEBHOOK_*`, e a seção
   obrigatória de integração com o código real.
5. **PRD** — o nível de produto; com RFC/FDD/ADRs prontos, virou consolidação.
6. **Tracker** — varredura de todos os documentos, amarrando cada item à sua origem.
7. **README** — por último, este documento.

Trabalhei em uma branch dedicada (`docs/design-docs-webhooks`), com **um commit por documento**,
para que o histórico do git conte a jornada. Cada documento passou pelo ciclo
**gerar → verificar adversarialmente → corrigir → commitar**.

## Prompts customizados

Dois prompts sustentaram a qualidade da entrega.

**1. Prompt do escritor de ADR** (com regra de rastreabilidade e de "altura"):

```text
Você é um engenheiro sênior documentando decisões arquiteturais (ADRs).
Fontes válidas: apenas TRANSCRICAO.md e o código existente. NÃO invente.
Toda afirmação precisa de origem rastreável: um [hh:mm] Nome que existe na
transcrição, ou um caminho de arquivo real.

ANTES de escrever, leia:
1. docs/adrs/ADR-001-outbox-no-mysql.md — o ADR de referência APROVADO.
   Copie EXATAMENTE a estrutura (metadados; Contexto; Decisão; Alternativas
   Consideradas, cada uma com o trade-off que motivou o descarte; Consequências
   positivas E negativas; Referências de código).
2. TRANSCRICAO.md — para pegar as citações [hh:mm] Nome EXATAS. Confirme cada
   timestamp antes de citar.

Regra de ALTURA: o ADR responde "por que decidimos assim". NÃO desça ao detalhe
de implementação do FDD (DDL, código, payload JSON completo).

Decisão a documentar: <fatos + timestamps da reunião>
```

**2. Prompt do verificador adversarial** (rodado contra cada documento antes do commit):

```text
Você é um revisor ADVERSARIAL. Seu trabalho é TENTAR REPROVAR o documento.
Seja cético; na dúvida, aponte o problema. Citação inventada é o erro mais grave.
NÃO confie na memória — leia os arquivos.

1. CITAÇÕES: para CADA [hh:mm] Nome, confirme na TRANSCRICAO.md que o
   timestamp+falante existe E sustenta a afirmação associada. Liste qualquer
   citação inexistente, atribuída ao falante errado, ou sem suporte.
2. CÓDIGO: confirme que cada caminho citado existe no repositório (ou está
   claramente marcado como proposto/novo).
3. SEÇÕES obrigatórias presentes.
4. ALTURA: aponte detalhe de implementação que pertence a outro documento
   (duplicação entre RFC e FDD, por exemplo).

Veredito: PASS ou NEEDS_FIX, com a lista de problemas e o número de linha.
```

## Iterações e ajustes

A IA **não** acertou tudo de primeira — e é justamente onde a revisão crítica entrou. Cada
documento passou por pelo menos um ciclo de verificação e correção. Os ajustes mais instrutivos:

- **Calibração antes de escalar.** Não deixei a IA gerar os 7 ADRs de uma vez. Escrevi o
  `ADR-001` primeiro, revisei formato, profundidade e voz, e só então usei ele como template
  para os demais. Isso evitou propagar um formato ruim por 7 arquivos.
- **Alucinação de atribuição (ADR-007).** A IA listou os 5 participantes da reunião como
  "decisores" da decisão de *snapshot do payload*. O verificador abriu a transcrição e percebeu
  que **Marcos e Sofia saíram da call em `[09:50]`**, antes dessa discussão (`[09:51]`–`[09:52]`).
  Corrigi para creditar só quem estava presente. Erro plausível, mas falso — o tipo que passa
  numa leitura rápida.
- **Citação entre aspas invertida (ADR-004).** A IA escreveu `"se vaza uma, não vaza tudo"`, mas
  a fala literal da Sofia é `"se vaza uma, vaza tudo"` (descrevendo o *risco* da secret global).
  Corrigi a aspa para bater com a transcrição.
- **Detalhe inventado no FDD.** Na tabela de riscos, a IA sugeriu "backoff com jitter" — algo que
  **não** foi decidido na reunião e que mexeria nos intervalos fixos acordados (1m/5m/30m/2h/12h).
  Removi, porque introduzia uma decisão sem origem.
- **Persona errada no PRD.** A IA colocou um "operador" consultando o histórico de entregas, mas
  a transcrição (`[09:34]`) diz que é **o cliente** que precisa vê-lo. Ajustei a persona.

No total, foram ~7 ciclos principais de geração→revisão→correção (um por documento/lote), além
da verificação programática final do Tracker, que cruzou os 215 itens contra a transcrição e o
código.

## Como navegar a entrega

Ordem de leitura sugerida (do "porquê" ao "como"):

1. [`docs/PRD.md`](./docs/PRD.md) — o problema, o público e o escopo (produto/negócio).
2. [`docs/RFC.md`](./docs/RFC.md) — a proposta técnica, alternativas e questões em aberto.
3. [`docs/adrs/`](./docs/adrs/) — as 7 decisões arquiteturais, cada uma isolada (`ADR-001` a
   `ADR-007`).
4. [`docs/FDD.md`](./docs/FDD.md) — o design de implementação: fluxos, contratos, erros,
   observabilidade e integração com o código.
5. [`docs/TRACKER.md`](./docs/TRACKER.md) — a rastreabilidade: cada item ligado à sua origem na
   transcrição ou no código.

Fontes: [`TRANSCRICAO.md`](./TRANSCRICAO.md) (a reunião) e o código em `src/`/`prisma/` (contexto,
não alterado). Guia interno para a IA: [`CLAUDE.md`](./CLAUDE.md).
