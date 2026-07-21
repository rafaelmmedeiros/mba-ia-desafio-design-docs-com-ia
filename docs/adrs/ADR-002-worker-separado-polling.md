# ADR-002 — Worker de webhooks como processo separado consumindo a outbox por polling

- **Status:** Aceito
- **Data:** 2026-07-21 (registro) — decisão tomada na reunião técnica de design (`TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior — Plataforma), Bruno (Eng. Pleno — Pedidos), Marcos (PM), Sofia (Eng. Segurança)
- **Decisões relacionadas:** [ADR-001](./ADR-001-outbox-no-mysql.md) (fonte dos eventos), [ADR-003](./ADR-003-retry-backoff-dlq.md) (o que o worker faz quando o envio falha), [ADR-006](./ADR-006-reuso-padroes-do-projeto.md) (PrismaClient e stack reaproveitados)

## Contexto

O padrão Outbox (ver [ADR-001](./ADR-001-outbox-no-mysql.md)) grava o evento na tabela
`webhook_outbox` dentro da transação de negócio, mas quem efetivamente **lê a outbox e
dispara as chamadas HTTP** para os clientes é outra peça, deliberadamente **fora da transação
de mudança de status**. Este ADR decide o formato dessa peça: como ela consome a outbox e
onde ela roda.

Duas restrições da reunião moldam a decisão:

- **Latência de entrega tem folga.** Os clientes consideram "tempo real" qualquer coisa
  abaixo de 10 segundos (`[09:02] Marcos`), então não é preciso disparo imediato — há espaço
  para um mecanismo assíncrono de baixa frequência.
- **O envio não pode depender do ciclo de vida da API.** Um reinício da API não pode derrubar
  o mecanismo de entrega junto (`[09:11] Diego`).

## Decisão

O worker de webhooks roda como **processo separado da API** e consome a `webhook_outbox` por
**polling a cada 2 segundos**: a cada ciclo busca os eventos pendentes mais antigos, processa
e marca o resultado (`[09:09] Diego`). O polling de 2s atende o requisito de "< 10 segundos"
com folga; a contrapartida é uma **latência mínima de ~2s no pior caso**, explicitamente
aceita (`[09:10] Larissa`).

O worker **não** roda dentro da instância da API — se a API reiniciar, não pode levar o
worker junto (`[09:11] Diego`). Ele entra como uma **entry-point nova, `src/worker.ts`**,
análoga à `src/server.ts`, acionada por um script `npm run worker` (`[09:11] Larissa`); a
lógica de processamento fica no módulo de webhooks, em um `webhook.processor.ts`
(`[09:28] Bruno`).

O worker usa o **mesmo banco e a mesma `DATABASE_URL`**, porém com um **`PrismaClient`
próprio**, porque o `PrismaClient` é por processo — sendo outro processo Node, precisa da sua
própria instância (`[09:11] Diego`; `[09:30] Bruno`). Isso reaproveita a stack existente sem
compartilhar o pool de conexões da API (ver [ADR-006](./ADR-006-reuso-padroes-do-projeto.md)).

Enquanto houver **um único worker**, o processamento segue a ordem de `created_at` da outbox,
de modo que o cliente recebe os eventos de um mesmo pedido em ordem — uma **ordenação
implícita por `order_id`**, não global (`[09:12] Diego`; `[09:13] Larissa`). Essa é uma
**limitação conhecida**, aceita porque os clientes nunca pediram ordering global
(`[09:14] Marcos`).

## Alternativas Consideradas

### 1. Trigger no banco (MySQL) para reagir à inserção — *descartada*

Usar uma trigger de banco para acionar o envio de forma mais reativa, em vez de polling.
**Trade-off que motivou o descarte:** o MySQL não tem listener nativo como o `NOTIFY`/`LISTEN`
do Postgres; a trigger só executa SQL, não notifica um processo externo. Para avisar o worker
seria preciso improvisar algo (escrever em arquivo, bater num endpoint), o que fica esquisito
e frágil — e o polling de 2s já atende o requisito de "< 10 segundos" tranquilamente
(`[09:09] Bruno` pergunta; `[09:09] Diego` responde).

### 2. Worker dentro do mesmo processo/instância da API — *descartada*

Rodar a lógica de consumo da outbox dentro do próprio processo da API. **Trade-off que
motivou o descarte:** acopla o ciclo de vida do worker ao da API — se a API reinicia, o worker
cai junto e para de entregar eventos (`[09:11] Diego`).

## Consequências

**Positivas**

- **Resiliência ao ciclo de vida da API:** worker e API sobem, caem e reiniciam de forma
  independente; um deploy ou restart da API não interrompe a entrega de webhooks
  (`[09:11] Diego`).
- **Simplicidade operacional:** polling em loop, sem depender de recurso de banco inexistente
  no MySQL nem de infraestrutura de mensageria nova; reaproveita a stack e o Prisma já em uso
  (ver [ADR-006](./ADR-006-reuso-padroes-do-projeto.md)).
- **Ordenação por pedido "de graça":** com um único worker, o cliente recebe os eventos de um
  mesmo `order_id` na ordem em que os status mudaram, sem coordenação extra
  (`[09:12] Diego`).

**Negativas / trade-offs**

- **Latência mínima de ~2s** entre o commit do evento e o envio, por conta do intervalo de
  polling — aceita frente ao requisito de < 10s (`[09:10] Larissa`).
- **Ordenação não é global.** A garantia de ordem vale apenas por `order_id` e apenas
  enquanto for **single-worker**; escalar para múltiplos workers em paralelo perde essa
  garantia — limitação conhecida e documentada (`[09:13] Larissa`).
- **Novo artefato de processo** para operar e observar (a `src/worker.ts` e seu `npm run
  worker`), separado do da API.
- **Caminho de escala conhecido, não resolvido agora:** se um dia for preciso rodar múltiplos
  workers, será necessário particionar por `order_id` ou usar lock pessimista para preservar
  a ordem — tratado como problema do futuro, não como decisão desta feature
  (`[09:13] Diego`).

## Referências de código

- `src/server.ts` — template do entry-point a ser espelhado pela nova `src/worker.ts`
  (bootstrap, `graceful shutdown`, `$disconnect` do Prisma).
- `src/config/database.ts` — `createPrismaClient`; o worker instancia o seu próprio
  `PrismaClient` a partir daqui, por ser outro processo.
- `package.json` — onde entra o script `npm run worker`, ao lado dos scripts de `dev`/`start`
  já existentes.
- Arquivos novos **propostos:** `src/worker.ts` (entry-point) e
  `src/modules/webhooks/webhook.processor.ts` (lógica de consumo da outbox e disparo dos
  envios).
