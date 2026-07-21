# Architectural Decision Records (ADRs)

Este diretório contém os ADRs da feature **Sistema de Webhooks de Notificação de Pedidos**, no
formato MADR. Cada arquivo registra uma decisão arquitetural isolada, com Status, Contexto,
Decisão, Alternativas Consideradas e Consequências, e é rastreável à transcrição da reunião
(`../../TRANSCRICAO.md`) ou ao código.

| ADR | Decisão |
|-----|---------|
| [ADR-001](./ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL |
| [ADR-002](./ADR-002-worker-separado-polling.md) | Worker em processo separado, polling de 2s |
| [ADR-003](./ADR-003-retry-backoff-dlq.md) | Retry com backoff exponencial e Dead Letter Queue |
| [ADR-004](./ADR-004-hmac-secret-por-endpoint.md) | HMAC-SHA256 com secret única por endpoint |
| [ADR-005](./ADR-005-at-least-once-x-event-id.md) | Entrega at-least-once com `X-Event-Id` |
| [ADR-006](./ADR-006-reuso-padroes-do-projeto.md) | Reúso dos padrões e da infraestrutura do projeto |
| [ADR-007](./ADR-007-payload-snapshot-na-insercao.md) | Snapshot do payload na inserção da outbox |

Visão consolidada da proposta no [RFC](../RFC.md); rastreabilidade completa no
[Tracker](../TRACKER.md).
