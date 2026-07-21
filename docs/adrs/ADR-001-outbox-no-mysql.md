# ADR-001 — Padrão Outbox no MySQL para publicação de eventos de webhook

- **Status:** Aceito
- **Data:** 2026-07-21 (registro) — decisão tomada na reunião técnica de design (`TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior — Plataforma), Bruno (Eng. Pleno — Pedidos), Marcos (PM), Sofia (Eng. Segurança)
- **Decisões relacionadas:** [ADR-002](./ADR-002-worker-separado-polling.md) (worker/polling), [ADR-005](./ADR-005-at-least-once-x-event-id.md) (garantia de entrega), [ADR-007](./ADR-007-payload-snapshot-na-insercao.md) (snapshot do payload)

## Contexto

A feature de webhooks precisa notificar clientes B2B quando o status de um pedido muda. A
mudança de status hoje acontece em `src/modules/orders/order.service.ts`, no método
`changeStatus`, dentro de uma única transação (`prisma.$transaction`) que atualiza a tabela
`orders`, insere em `order_status_history` e debita/repõe estoque dos produtos do pedido.

Duas restrições saíram da reunião e moldam a decisão:

- **Não pode acoplar a latência do cliente à transação de negócio.** Se um `HTTP call` for
  disparado dentro do `changeStatus`, um cliente lento ou fora do ar trava a mudança de
  status de outros pedidos, e não há rollback sensato ("dá rollback na mudança de status?
  Não dá" — `[09:04] Bruno`).
- **A entrega precisa ser consistente com o commit do pedido.** Se a transação de status
  commitou, o evento tem de existir; se deu rollback, o evento tem de sumir junto — sem
  janela de inconsistência (`[09:06] Diego`).

Os clientes consideram "tempo real" qualquer latência abaixo de 10 segundos (`[09:02]
Marcos`), então não há necessidade de disparo imediato — há espaço para um mecanismo
assíncrono.

## Decisão

Adotar o **padrão Outbox sobre o MySQL já existente**. Na mudança de status, **dentro da
mesma transação SQL** que atualiza `orders` e `order_status_history`, inserir também uma
linha numa nova tabela `webhook_outbox` com o evento. Um processo separado (ver
[ADR-002](./ADR-002-worker-separado-polling.md)) lê essa tabela e dispara as chamadas HTTP.

A tabela terá índice por status do evento (`pendente`, `processando`, `falhou`, `entregue`)
e por `created_at`, para o worker ler apenas os pendentes mais antigos em batch pequeno
(`[09:08] Diego`). O arquivamento de linhas entregues (após ~30 dias) fica **fora do escopo**
desta feature (`[09:08] Diego`).

## Alternativas Consideradas

### 1. Disparo síncrono do HTTP dentro do `changeStatus` — *descartada*

Chamar o webhook direto na transação de mudança de status. **Trade-off que motivou o
descarte:** acopla a latência (e a disponibilidade) do cliente à transação de negócio,
travando mudanças de status de outros pedidos, e não oferece rollback aceitável se o cliente
estiver offline. "Síncrono está fora de questão" (`[09:06] Diego`; argumentado por Bruno em
`[09:04]`).

### 2. Fila/stream externo (Redis Streams) — *descartada*

Publicar os eventos numa fila dedicada como Redis Streams. **Trade-off que motivou o
descarte:** exigiria subir e operar infraestrutura nova (Redis Cluster) para um time
pequeno — "overengineering"; o Outbox no MySQL existente resolve sem custo operacional novo
(`[09:07] Diego` e Larissa).

## Consequências

**Positivas**

- **Atomicidade / consistência forte:** o evento existe se e somente se a mudança de status
  foi commitada — não há inconsistência possível (`[09:06] Diego`).
- **Zero infraestrutura nova:** reaproveita o MySQL e o Prisma já em produção, alinhado à
  decisão de reuso máximo (ver [ADR-006](./ADR-006-reuso-padroes-do-projeto.md)).
- **Desacopla a latência do cliente** do caminho crítico de mudança de status.

**Negativas / trade-offs**

- Introduz a necessidade de um **worker e de polling**, com latência mínima de entrega de
  ~2s no pior caso (detalhado na [ADR-002](./ADR-002-worker-separado-polling.md)); aceitável
  frente ao requisito de < 10s.
- A tabela `webhook_outbox` **cresce** e demanda uma rotina de arquivamento (reconhecida,
  mas fora do escopo desta feature — `[09:08]`).
- Acrescenta um `INSERT` à transação já pesada do `changeStatus`; e, por ser atômico, uma
  falha ao inserir na outbox **causa rollback da mudança de status** ("se a outbox falhar de
  inserir, rollback" — `[09:40] Bruno`). É o comportamento desejado, mas significa que um
  defeito no módulo de webhooks pode impactar o core de pedidos — a integração precisa ser
  cuidadosa (detalhada no FDD, seção "Integração com o sistema existente").

## Referências de código

- `src/modules/orders/order.service.ts` — método `changeStatus`, onde o `INSERT` na
  `webhook_outbox` entra na `prisma.$transaction` existente.
- `prisma/schema.prisma` — padrão para a nova tabela `webhook_outbox` (PK UUID
  `@db.Char(36)`, mapeamento `@@map`, índices), seguindo os modelos atuais.
