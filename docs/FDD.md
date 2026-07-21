# FDD — Sistema de Webhooks de Notificação de Pedidos

> Documento de design de implementação. Operacionaliza a proposta do [RFC](./RFC.md) e as
> decisões dos [ADRs 001–007](./adrs/). Nível: acionável para um desenvolvedor começar a codar.
> Base path da API: `/api/v1` (ver `src/app.ts`). Rastreabilidade em [TRACKER](./TRACKER.md).

## 1. Contexto e motivação técnica

O OMS controla o ciclo de vida do pedido por uma máquina de estados
(`src/modules/orders/order.status.ts`) e muda status numa transação única
(`src/modules/orders/order.service.ts`, `changeStatus`) que atualiza `orders`, grava
`order_status_history` e ajusta estoque. Não há hoje **nenhum** mecanismo de notificação
externa. Clientes B2B fazem _polling_ no `GET /orders`, o que é lento e caro (`[09:00] Marcos`).

A feature adiciona **webhooks _outbound_**: cada mudança de status vira um evento entregue por
HTTP ao endpoint do cliente, com latência-alvo abaixo de 10 segundos (`[09:02] Marcos`), sem
acoplar a entrega à transação de negócio nem introduzir infraestrutura nova.

## 2. Objetivos técnicos

- Publicar um evento por mudança de status **atomicamente** com a transação do `changeStatus`
  (padrão Outbox — [ADR-001](./adrs/ADR-001-outbox-no-mysql.md)).
- Entregar via **worker em processo separado**, _polling_ de 2s, com latência p95 < 10s
  ([ADR-002](./adrs/ADR-002-worker-separado-polling.md)).
- Garantir resiliência: **retry com backoff + DLQ**
  ([ADR-003](./adrs/ADR-003-retry-backoff-dlq.md)) e **at-least-once** com idempotência via
  `X-Event-Id` ([ADR-005](./adrs/ADR-005-at-least-once-x-event-id.md)).
- Autenticar cada envio com **HMAC-SHA256**, secret por endpoint, rotação com _grace period_
  ([ADR-004](./adrs/ADR-004-hmac-secret-por-endpoint.md)).
- **Reusar a stack existente** sem novas dependências de runtime
  ([ADR-006](./adrs/ADR-006-reuso-padroes-do-projeto.md)).

## 3. Escopo e exclusões

**No escopo:** modelagem de outbox/DLQ/config/deliveries; worker de entrega; CRUD de
configuração de webhook; rotação de secret; histórico de entregas; replay administrativo de
DLQ; assinatura HMAC; integração no `changeStatus`.

**Fora de escopo** (rastreado à reunião): e-mail de alerta ao cliente (`[09:37]` próxima
fase); rate limiting de saída (`[09:39]` observar); dashboard/painel visual (`[09:40]` outro
time); webhooks _inbound_ (`[09:03]`); arquivamento de eventos entregues após ~30 dias
(`[09:08]`); ordenação global entre pedidos (`[09:13]` limitação conhecida); escala
multi-worker (`[09:13]` futuro).

## 4. Modelo de dados (novas tabelas)

Seguem o padrão de `prisma/schema.prisma`: PK UUID `@db.Char(36)` (`[09:51] Larissa`),
`@@map` em snake_case, índices explícitos.

- **`webhook_endpoint`** — configuração por cliente: `id`, `customer_id`, `url`, `secret`,
  `previous_secret`, `previous_secret_expires_at`, `subscribed_statuses` (JSON com valores de
  `OrderStatus`), `active`, timestamps. Índice em `customer_id` e `active` (`[09:21] Bruno`).
- **`webhook_outbox`** — fila de eventos: `id`, `webhook_endpoint_id`, `order_id`,
  `event_type`, `payload` (snapshot JSON — [ADR-007](./adrs/ADR-007-payload-snapshot-na-insercao.md)),
  `status` (`pendente`/`processando`/`falhou`/`entregue`), `attempts`, `next_attempt_at`,
  `created_at`. Índice em (`status`, `next_attempt_at`) e `created_at` (`[09:08] Diego`).
- **`webhook_delivery`** — histórico de tentativas: `id`, `webhook_outbox_id`,
  `webhook_endpoint_id`, `attempt`, `request_url`, `response_status`, `response_time_ms`,
  `error_reason`, `created_at`. Índice em `webhook_endpoint_id` (`[09:34] Marcos`).
- **`webhook_dead_letter`** — eventos esgotados: `id`, `webhook_endpoint_id`, `order_id`,
  `payload`, `failure_reason`, `created_at` (`[09:18] Diego`).

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox (síncrona, dentro do `changeStatus`)

1. `OrderService.changeStatus` abre a `prisma.$transaction` já existente e executa as operações
   atuais: valida transição, ajusta estoque, `tx.order.update`, `tx.orderStatusHistory.create`.
2. **Nova etapa, dentro da mesma `tx`:** chama `publishWebhookEvent(tx, order, fromStatus,
   toStatus)` (`[09:41] Bruno`). A função:
   - Busca os `webhook_endpoint` ativos do `customer_id` que assinaram `toStatus`. Se **nenhum**
     assinou aquele status, **não insere nada** (filtro na inserção — `[09:34] Bruno`).
   - Para cada endpoint assinante, **renderiza o snapshot** do payload (seção 6.1) e insere uma
     linha em `webhook_outbox` com `status = pendente` e um `event_id` UUID
     ([ADR-005](./adrs/ADR-005-at-least-once-x-event-id.md)).
3. Se qualquer insert falhar, a transação inteira faz **rollback** — a mudança de status não
   acontece sem o evento registrado (`[09:40] Bruno`; [ADR-001](./adrs/ADR-001-outbox-no-mysql.md)).

### 5.2 Processamento pelo worker (assíncrono, processo separado)

1. Loop de _polling_ a cada **2s** (`[09:09] Diego`). Busca em lote pequeno os eventos com
   `status = pendente` e `next_attempt_at <= agora`, ordenados por `created_at`.
2. Marca o lote como `processando` (claim), para não reprocessar no próximo ciclo.
3. Para cada evento: assina o `payload` (HMAC-SHA256, seção 6.1), monta os headers e faz o
   `POST` na `url` do endpoint com **timeout de 10s** (`[09:42] Diego`).
4. Grava um registro em `webhook_delivery` (status HTTP, tempo de resposta, motivo da falha).
5. **Sucesso** (resposta 2xx): marca o evento como `entregue`.
6. **Falha** (não-2xx, timeout ou erro de conexão): segue o fluxo de retry (5.3).

### 5.3 Retry com backoff

1. Incrementa `attempts`. Se `attempts < 5`, agenda a próxima tentativa definindo
   `next_attempt_at = agora + backoff[attempts]`, com `backoff = [1m, 5m, 30m, 2h, 12h]`, e volta
   o `status` para `pendente` (`[09:17] Diego`; [ADR-003](./adrs/ADR-003-retry-backoff-dlq.md)).
2. O worker naturalmente reprocessa quando `next_attempt_at` vencer.

### 5.4 Dead Letter Queue

1. Ao falhar a **5ª** tentativa, o evento é movido para `webhook_dead_letter` com `payload`,
   `failure_reason` e timestamp; a linha correspondente na outbox é marcada como `falhou`
   (`[09:18] Diego`).

### 5.5 Replay administrativo da DLQ

1. `POST /api/v1/admin/webhooks/dead-letter/:id/replay` (role `ADMIN`) recria o evento na
   `webhook_outbox` como `pendente`, zerando `attempts` (`[09:18] Diego`).
2. A ação registra em log **quem** executou o replay, para auditoria (`[09:36] Sofia`).

## 6. Contratos públicos

### 6.1 Contrato de saída — o webhook que enviamos ao cliente

Requisição `POST` na `url` cadastrada. Headers (`[09:44] Diego`/`Sofia`):

```
POST https://cliente.example.com/webhooks/oms
Content-Type: application/json
X-Event-Id: 4f9d2c1a-...              # UUID único do evento (dedup do lado do cliente)
X-Webhook-Id: 9a1c7b33-...           # id do endpoint cadastrado
X-Signature: sha256=3b8e...          # HMAC-SHA256(hex) do corpo cru com a secret do endpoint
X-Timestamp: 2026-07-21T12:00:03Z    # instante do envio (detecção de replay pelo cliente)
```

Corpo (snapshot renderizado na inserção — [ADR-007](./adrs/ADR-007-payload-snapshot-na-insercao.md);
campos de `[09:43] Diego`, sem `items` para não inflar):

```json
{
  "event_id": "4f9d2c1a-...",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-21T12:00:03Z",
  "data": {
    "order_id": "b1c2...",
    "order_number": "ORD-000123",
    "from_status": "PAID",
    "to_status": "PROCESSING",
    "customer_id": "a0f1...",
    "total_cents": 154900
  }
}
```

**Semântica:** resposta **2xx** = entregue; qualquer outra coisa (não-2xx, timeout > 10s, erro
de conexão) = falha → retry. **Verificação pelo cliente:** recalcular `HMAC-SHA256` sobre o
corpo cru e comparar com `X-Signature`; durante o _grace period_, aceitar assinatura da secret
vigente **ou** da anterior ([ADR-004](./adrs/ADR-004-hmac-secret-por-endpoint.md)).

### 6.2 `POST /api/v1/webhooks` — criar configuração

Autenticado (qualquer role — `[09:37] Sofia`). `customer_id` vem do corpo, **não** do JWT
(`[09:32]`–`[09:33]`).

Request:
```json
{
  "customerId": "a0f1...",
  "url": "https://cliente.example.com/webhooks/oms",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"]
}
```
Response `201 Created` — a `secret` é **gerada pela plataforma e devolvida só nesta resposta**
(`[09:31] Marcos`):
```json
{
  "data": {
    "id": "9a1c7b33-...",
    "customerId": "a0f1...",
    "url": "https://cliente.example.com/webhooks/oms",
    "subscribedStatuses": ["SHIPPED", "DELIVERED"],
    "active": true,
    "secret": "whsec_2f8a...",
    "createdAt": "2026-07-21T12:00:00Z"
  }
}
```
Erros: `422 WEBHOOK_URL_NOT_HTTPS`, `422 WEBHOOK_INVALID_STATUS_FILTER`, `404 NOT_FOUND`
(customer), `400 VALIDATION_ERROR`.

### 6.3 `POST /api/v1/webhooks/:id/rotate-secret` — rotacionar secret

Autenticado. Gera nova secret e mantém a anterior válida por **24h** (`[09:21] Sofia`).
Response `200 OK`:
```json
{
  "data": {
    "secret": "whsec_9d3c...",
    "previousSecretValidUntil": "2026-07-22T12:00:00Z"
  }
}
```
Erros: `404 WEBHOOK_NOT_FOUND`.

### 6.4 `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas

Autenticado. Paginado, no envelope padrão de `src/shared/http/response.ts` (`[09:34] Marcos`).
Response `200 OK`:
```json
{
  "data": [
    {
      "id": "d10a...",
      "eventId": "4f9d2c1a-...",
      "attempt": 1,
      "status": "success",
      "responseStatus": 200,
      "responseTimeMs": 142,
      "errorReason": null,
      "createdAt": "2026-07-21T12:00:03Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```
Erros: `404 WEBHOOK_NOT_FOUND`.

### 6.5 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — replay da DLQ

Role **`ADMIN`** obrigatória, via `requireRole` existente (`[09:36] Larissa`). Response
`202 Accepted`:
```json
{
  "data": {
    "deadLetterId": "e77f...",
    "requeuedOutboxId": "c33b...",
    "status": "pendente"
  }
}
```
Erros: `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`, `401 UNAUTHORIZED`, `403 FORBIDDEN` (role
insuficiente).

### 6.6 Demais endpoints de configuração (resumo)

- `GET /api/v1/webhooks?customerId=&page=&pageSize=` → `200` lista paginada (a `secret` **não**
  é retornada) (`[09:33] Bruno`).
- `PATCH /api/v1/webhooks/:id` → `200` edita `url`, `subscribedStatuses`, `active`
  (`[09:33] Bruno`).
- `DELETE /api/v1/webhooks/:id` → `204 No Content` (`[09:33] Bruno`).

## 7. Matriz de erros (`WEBHOOK_*`)

Todos os erros estendem `AppError` e são serializados pelo error middleware existente
(`src/middlewares/error.middleware.ts`) no formato `{ "error": { "code", "message", "details"? } }`.

| Código | HTTP | Quando ocorre | Origem |
|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | configuração de webhook inexistente | `[09:28] Bruno` |
| `WEBHOOK_INVALID_URL` | 422 | URL malformada | `[09:28] Bruno` |
| `WEBHOOK_URL_NOT_HTTPS` | 422 | URL não é `https` (TLS obrigatório) | `[09:23] Sofia` |
| `WEBHOOK_SECRET_REQUIRED` | 400 | operação que exige secret sem secret | `[09:28] Bruno` |
| `WEBHOOK_INVALID_STATUS_FILTER` | 422 | status assinado fora do enum `OrderStatus` | `order.status.ts` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | payload renderizado excede 64KB | `[09:23]`–`[09:24]` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | replay de item de DLQ inexistente | `[09:18]`/`[09:35]` |

Erros de **autenticação/autorização** reaproveitam os códigos existentes `UNAUTHORIZED` (401) e
`FORBIDDEN` (403), emitidos por `authenticate`/`requireRole`
(`src/middlewares/auth.middleware.ts`).

**Motivos de falha de entrega** (worker; gravados em `webhook_delivery.error_reason`, não são
erros HTTP da nossa API): `TIMEOUT` (> 10s, `[09:42]`), `CONNECTION_ERROR`, `HTTP_4XX`,
`HTTP_5XX`, `PAYLOAD_TOO_LARGE` (checado antes do envio, `[09:24]`).

## 8. Estratégias de resiliência

- **Timeout:** 10s por tentativa de envio; estouro conta como falha (`[09:42] Diego`).
- **Retry:** 5 tentativas com backoff `1m/5m/30m/2h/12h` (~15h no total)
  ([ADR-003](./adrs/ADR-003-retry-backoff-dlq.md)).
- **Backoff persistido:** `next_attempt_at` na outbox — o agendamento sobrevive a restart do
  worker (estado no banco, não em memória).
- **DLQ:** após a 5ª falha, evento vai para `webhook_dead_letter`; recuperação só por replay
  manual (`[09:18]`).
- **Idempotência / at-least-once:** o mesmo `X-Event-Id` viaja em todas as tentativas; o cliente
  deduplica ([ADR-005](./adrs/ADR-005-at-least-once-x-event-id.md)).
- **Ordenação:** por `created_at` e single-worker → ordem por `order_id`; não global
  (`[09:13]`).
- **Fallback:** **não há** e-mail de alerta nesta fase (`[09:37]`); a evidência de falha fica na
  DLQ e no histórico de entregas.

## 9. Observabilidade

- **Logs (Pino, `src/shared/logger/index.ts`):** o worker emite logs estruturados por tentativa
  — `webhook_delivery_attempt`, `webhook_delivered`, `webhook_failed`, `webhook_dead_lettered` —
  com campos `event_id`, `webhook_endpoint_id`, `order_id`, `attempt`, `response_status`,
  `response_time_ms`. A `secret` **precisa** entrar na lista de `redact` do Pino (hoje cobre
  `*.token`/`*.password`, ainda não `secret`) para nunca vazar (`[09:22] Diego`).
- **Métricas:** contadores/derivados dos logs estruturados (o projeto não tem stack de métricas
  hoje): `webhook_delivery_total{result}`, `webhook_delivery_duration_ms` (p50/p95),
  `webhook_retry_total`, `webhook_dead_letter_total` e um _gauge_ de profundidade da outbox
  (`pendentes`). O p95 alimenta o objetivo de latência < 10s.
- **Tracing/correlação:** reaproveita o `X-Request-Id` do `request-logger.middleware.ts` na API,
  e usa o `event_id` como **id de correlação** ponta a ponta (API → outbox → worker → entrega),
  amarrando o log da mudança de status ao log de cada tentativa de envio. Fica aberto o caminho
  para OpenTelemetry no futuro, sem alterar a modelagem.

## 10. Integração com o sistema existente

Esta seção nomeia os pontos reais do código base e como o módulo se integra a cada um.

1. **`src/modules/orders/order.service.ts` (`changeStatus`, linhas 126–179).** É a integração
   crítica. Dentro da `prisma.$transaction` existente, após `tx.order.update` (l.158) e
   `tx.orderStatusHistory.create` (l.159), adiciona-se a chamada
   `await publishWebhookEvent(tx, order, from, to)`. Recebe o **`tx` da transação atual** — função
   pura, sem injetar repositório inteiro (`[09:41] Diego`/Bruno) — preservando a atomicidade já
   usada para estoque/histórico.
2. **`src/modules/orders/order.status.ts`.** O enum `OrderStatus` e as transições válidas são a
   fonte de `from_status`/`to_status` e do filtro `subscribed_statuses`. O webhook dispara
   exatamente nas transições que a máquina de estados permite; `WEBHOOK_INVALID_STATUS_FILTER`
   valida contra esse enum.
3. **`src/shared/errors/http-errors.ts` e `src/shared/errors/app-error.ts`.** As classes de erro
   `WEBHOOK_*` estendem `AppError` (direta ou indiretamente) seguindo o padrão de `InsufficientStockError` e
   `InvalidStatusTransitionError` (código em SCREAMING_SNAKE). Nenhuma mudança é necessária no
   `src/middlewares/error.middleware.ts`, que já serializa qualquer `AppError` (`[09:29] Bruno`).
4. **`src/middlewares/auth.middleware.ts`.** `authenticate` protege todas as rotas de webhook;
   `requireRole('ADMIN')` protege o endpoint de replay da DLQ (`[09:36] Larissa`), sem código
   novo de autorização.
5. **`src/shared/logger/index.ts`.** Adicionar `secret`/`*.secret` à lista de `redact` do Pino,
   para que a secret nunca apareça em log (`[09:22] Diego`).
6. **`src/server.ts` + `src/config/database.ts`.** O worker (`src/worker.ts`, novo) espelha o
   _bootstrap_ e o _graceful shutdown_ de `server.ts` e instancia seu **próprio `PrismaClient`**
   via `createPrismaClient`, por ser outro processo (`[09:30] Bruno`;
   [ADR-002](./adrs/ADR-002-worker-separado-polling.md)).
7. **`src/app.ts` + `src/routes/index.ts`.** O módulo `webhooks` é montado em `buildControllers`
   e registrado em `buildApiRouter` sob `/api/v1/webhooks`, exatamente como os módulos atuais.
8. **`prisma/schema.prisma`.** As quatro novas tabelas (seção 4) seguem as convenções existentes
   (PK UUID `@db.Char(36)`, `@@map`, índices).

## 11. Dependências e compatibilidade

- **Sem novas dependências de runtime:** HMAC-SHA256 usa o módulo `crypto` nativo do Node; `uuid`,
  `@prisma/client`, `express`, `zod` e `pino` já estão no `package.json`
  ([ADR-006](./adrs/ADR-006-reuso-padroes-do-projeto.md)).
- **Mudanças aditivas e retrocompatíveis:** novas tabelas via migração Prisma; um `INSERT`
  adicional na transação do `changeStatus`; nenhum contrato existente da API de pedidos muda.
- **Novo artefato operacional:** o processo `worker` (`npm run worker`), a ser adicionado ao
  `package.json` e ao _deploy_.
- **Configuração:** novas variáveis de ambiente (ex.: intervalo de _polling_) validadas via o
  mesmo `src/config/env.ts` (Zod).

## 12. Critérios de aceite técnicos

- Mudança de status e inserção na outbox são atômicas (rollback conjunto verificado em teste).
- Nenhum evento é inserido para status sem webhook assinante (filtro na inserção).
- Worker entrega em < 10s (p95) em cenário nominal; latência mínima de ~2s aceita.
- Retry segue `1m/5m/30m/2h/12h`; após a 5ª falha o evento está na `webhook_dead_letter`.
- `X-Signature` é verificável pelo cliente com a secret; assinatura válida na secret vigente e na
  anterior durante o _grace period_.
- A `secret` nunca aparece em logs (verificado com `redact` ativo).
- Replay da DLQ exige `ADMIN` e registra o autor.
- `GET /webhooks/:id/deliveries` retorna o histórico paginado com status, tempo e motivo.
- Todos os erros do módulo usam o prefixo `WEBHOOK_` e o envelope `{ error: { code, message } }`.

## 13. Riscos e mitigação

| Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|
| Bug no `publishWebhookEvent` derruba mudança de status (rollback) | Média | Alto | Função pura recebendo `tx`; testes ponta a ponta; revisão focada (`[09:41]`) |
| Crescimento da `webhook_outbox` degrada o _polling_ | Média | Médio | Índice em (`status`,`next_attempt_at`); lote pequeno; arquivamento (fora de escopo, `[09:08]`) |
| Vazamento de secret (nosso lado) | Baixa | Alto | `redact` no Pino; rotação com _grace period_; revisão da Sofia antes do deploy (`[09:46]`) |
| _Thundering herd_ de retries simultâneos | Baixa | Médio | Falhas ocorrem em horários distintos, espalhando os `next_attempt_at`; lote limitado por ciclo |
| Worker como ponto único (single-worker) | Média | Médio | Estado no banco (não em memória); restart seguro; escala futura documentada (`[09:13]`) |
| Cliente lento consome o _budget_ do ciclo | Média | Médio | Timeout de 10s; rate limiting de saída em aberto (`[09:39]`) |
