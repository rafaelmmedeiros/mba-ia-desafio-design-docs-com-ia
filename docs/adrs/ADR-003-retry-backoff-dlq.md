# ADR-003 — Reentrega por backoff exponencial e Dead Letter Queue para eventos de webhook

- **Status:** Aceito
- **Data:** 2026-07-21 (registro) — decisão tomada na reunião técnica de design (`TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior — Plataforma), Bruno (Eng. Pleno — Pedidos), Marcos (PM), Sofia (Eng. Segurança)
- **Decisões relacionadas:** [ADR-002](./ADR-002-worker-separado-polling.md) (quem executa o retry), [ADR-001](./ADR-001-outbox-no-mysql.md) (origem dos eventos), [ADR-006](./ADR-006-reuso-padroes-do-projeto.md) (o `requireRole` reaproveitado no replay ADMIN)

## Contexto

Os eventos de webhook são disparados por um worker que lê a outbox e faz a chamada HTTP para
o endpoint do cliente (ver [ADR-001](./ADR-001-outbox-no-mysql.md) e
[ADR-002](./ADR-002-worker-separado-polling.md)). O endpoint é infraestrutura de terceiros,
fora do nosso controle: o cliente pode estar temporariamente fora do ar, lento ou em
manutenção. Precisamos de uma política clara para "o que fazer quando a entrega falha".

Duas restrições saíram da reunião e moldam a decisão:

- **Uma falha não é definitiva por si só.** Se o cliente está offline, reentregamos depois,
  aumentando o intervalo entre as tentativas; só depois de um teto de tentativas é que
  consideramos falha permanente (`[09:15] Diego`).
- **Cliente que não responde a tempo conta como falha.** O `HTTP call` do worker tem timeout
  de 10 segundos; um cliente que não responde nesse prazo é tratado como falha e marcado para
  retry (`[09:42] Diego` e Sofia), o que alimenta diretamente a política de reentrega.

O objetivo é absorver indisponibilidades temporárias sem manter eventos "pendurados" para
sempre, e sem perder a evidência de eventos que realmente não puderam ser entregues.

## Decisão

Adotar **reentrega por backoff exponencial com 5 tentativas** e, esgotadas as tentativas,
**mover o evento para uma Dead Letter Queue (DLQ)** persistida em **tabela separada**.

- **Progressão do backoff:** 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas — cinco
  tentativas, com quase 15 horas entre a primeira falha e a última tentativa (`[09:17]
  Diego`). O total foi aceito por Marcos e por Larissa (`[09:17]`).
- **DLQ em tabela própria (`webhook_dead_letter`):** ao esgotar as tentativas, o evento é
  movido para uma tabela separada que guarda a payload, o motivo da falha e o timestamp,
  servindo de evidência para debug e reprocessamento (`[09:18] Diego`).
- **Reprocessamento manual:** o replay é feito por um endpoint administrativo
  (`POST /admin/webhooks/dead-letter/:id/replay`) que recoloca o evento na outbox como
  pendente (`[09:18] Diego`). O endpoint exige role `ADMIN` e registra quem executou o
  replay, para auditoria (`[09:36] Sofia` e Larissa) — reaproveitando o `requireRole` já
  existente (ver [ADR-006](./ADR-006-reuso-padroes-do-projeto.md)).

A lógica de contagem de tentativas e de agendamento da próxima reentrega vive no worker/
processor (ver [ADR-002](./ADR-002-worker-separado-polling.md)).

## Alternativas Consideradas

### 1. Retry indefinido com backoff — *descartada*

Continuar reentregando para sempre, apenas aumentando o intervalo, sem teto de tentativas.
**Trade-off que motivou o descarte:** se o cliente sumiu de vez, o evento fica pendurado para
sempre, sem nunca virar falha permanente nem gerar evidência acionável (`[09:15] Diego`).

### 2. 3 tentativas, mais agressivo — *descartada*

Encerrar mais cedo, com apenas 3 tentativas. **Trade-off que motivou o descarte:** 3 é pouco
— retentaria três vezes em cerca de 30 minutos e mataria o evento; já houve cliente com
indisponibilidade de duas horas em manutenção planejada, que seria perdido nessa janela
(`[09:16] Bruno` propõe 3; Diego rebate). Cinco tentativas cobrem janelas de indisponibilidade
bem maiores.

### 3. Marcar como "failed" na própria `webhook_outbox` — *descartada*

Não criar tabela nova e apenas marcar o evento esgotado como `failed` na outbox principal.
**Trade-off que motivou o descarte:** uma tabela DLQ separada deixa a leitura da outbox
principal mais limpa e concentra a evidência das falhas num único lugar, facilitando debug e
reprocessamento (`[09:18] Diego`).

## Consequências

**Positivas**

- **Absorve indisponibilidades temporárias:** a janela de quase 15 horas cobre quedas e
  manutenções planejadas de clientes sem perder o evento (`[09:17] Diego`).
- **Não deixa eventos pendurados:** há um teto explícito de tentativas; o que não entrega vira
  falha permanente e evidência na DLQ, em vez de reentrega infinita.
- **Evidência e recuperação:** a `webhook_dead_letter` preserva payload, motivo e timestamp,
  permitindo debug e replay manual sem poluir a outbox (`[09:18] Diego`).
- **Replay auditável e restrito:** o replay exige role `ADMIN` e registra o autor,
  reaproveitando o `requireRole` já existente (`[09:36] Sofia` e Larissa).

**Negativas / trade-offs**

- **Latência longa em caso de falha:** um evento pode levar quase 15 horas até esgotar as
  tentativas e cair na DLQ. Aceito porque, se o cliente ficou 15 horas fora, "ele já está com
  um problema sério dele" (`[09:17] Marcos`).
- **Recuperação não é automática:** eventos na DLQ só voltam a ser entregues por replay manual
  via endpoint admin (`[09:18] Diego`); não há reprocessamento automático nesta fase.
- **Superfície adicional:** introduz uma tabela nova (`webhook_dead_letter`) e um endpoint
  administrativo privilegiado, que precisa de controle de acesso e auditoria próprios
  (`[09:36] Sofia`).

## Referências de código

- `prisma/schema.prisma` — padrão para a nova tabela `webhook_dead_letter`, seguindo os
  modelos atuais (mesma convenção de PK, `@@map` e índices já usada nas demais tabelas).
- `src/middlewares/auth.middleware.ts` — `requireRole`, reaproveitado para exigir a role
  `ADMIN` no endpoint de replay da DLQ.
- A lógica de retry (contagem de tentativas e agendamento do próximo backoff) vive no worker/
  processor — ver [ADR-002](./ADR-002-worker-separado-polling.md).
