# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

- **Autor:** Time de Plataforma — Larissa (Tech Lead), condutora do design
- **Status:** Em revisão (Draft submetido à equipe)
- **Data:** 2026-07-21
- **Revisores:** Marcos (Product Manager), Bruno (Eng. Pleno — Pedidos), Diego (Eng. Sênior — Plataforma), Sofia (Eng. de Segurança)
- **Fonte:** reunião técnica de design (`TRANSCRICAO.md`) e código do Order Management System (OMS)

## Resumo executivo (TL;DR)

Propomos um sistema de **webhooks _outbound_** que notifica clientes B2B, em tempo quase real
(latência-alvo abaixo de 10 segundos), sempre que o **status de um pedido muda**. A abordagem é
um **padrão Outbox no MySQL existente**: a mudança de status grava o evento na mesma transação
de negócio, e um **worker em processo separado**, em _polling_, lê a fila e faz a entrega HTTP
com **assinatura HMAC-SHA256**, **retry com backoff e Dead Letter Queue**, e garantia
**at-least-once** (deduplicação pelo cliente via `X-Event-Id`). A solução **reaproveita a stack
atual** (Prisma/MySQL, `AppError`, Pino, error middleware, padrão de módulos) e **não introduz
infraestrutura nova**. As decisões estão formalizadas nos [ADRs 001–007](#decisões-relacionadas).

## Contexto e problema

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — pediram formalmente para
serem notificados quando o status dos pedidos deles muda. Hoje eles fazem _polling_ no
`GET /orders` de tempos em tempos, o que torna a integração **lenta e cara** e os obriga a
"ficar atualizando manualmente"; para eles, "tempo real" significa qualquer latência **abaixo
de 10 segundos** (`[09:00]`–`[09:02] Marcos`). Há pressão comercial: a Atlas sinalizou que pode
migrar para um concorrente se a entrega não sair até o fim do trimestre (`[09:00] Marcos`).

O OMS atual **não tem nenhum mecanismo de notificação externa, eventos, filas ou webhooks** — o
ciclo de vida do pedido é controlado por uma máquina de estados e a mudança de status é uma
transação que atualiza o pedido, grava histórico e ajusta estoque
(`src/modules/orders/order.service.ts`). É exatamente esse vácuo que a feature preenche. O
escopo é **apenas _outbound_** (nós → cliente); os clientes querem receber, não enviar
(`[09:02]`–`[09:03] Sofia`/`Marcos`).

## Proposta técnica (visão geral)

A solução se divide em três planos: **captura do evento**, **entrega** e **configuração/gestão**.
O detalhamento de contratos, payloads e códigos de erro fica no FDD; aqui fica a arquitetura.

**1. Captura do evento — Outbox transacional.** Quando o status de um pedido muda, o evento é
inserido numa tabela `webhook_outbox` **dentro da mesma transação** que já atualiza o pedido e
o histórico. Assim, o evento existe se e somente se a mudança de status foi commitada — sem
janela de inconsistência. A inserção é filtrada: só entram status que algum webhook do cliente
assinou. O evento guarda um **snapshot do payload** renderizado no momento da inserção (reflete
o estado de quando o status mudou). → [ADR-001](./adrs/ADR-001-outbox-no-mysql.md),
[ADR-007](./adrs/ADR-007-payload-snapshot-na-insercao.md).

**2. Entrega — worker separado, resiliente e seguro.** Um **worker em processo separado**
(`src/worker.ts`, `npm run worker`) faz _polling_ da outbox **a cada 2 segundos**, processa os
pendentes mais antigos e dispara as chamadas HTTP (timeout de 10s). Cada requisição carrega uma
**assinatura HMAC-SHA256** do corpo (secret única por endpoint) e um **`X-Event-Id`** (UUID por
evento) para o cliente deduplicar. Falhas entram em **retry com backoff exponencial** (5
tentativas: 1m/5m/30m/2h/12h) e, esgotadas, caem numa **Dead Letter Queue** em tabela separada,
reprocessável por um endpoint administrativo. →
[ADR-002](./adrs/ADR-002-worker-separado-polling.md),
[ADR-003](./adrs/ADR-003-retry-backoff-dlq.md),
[ADR-004](./adrs/ADR-004-hmac-secret-por-endpoint.md),
[ADR-005](./adrs/ADR-005-at-least-once-x-event-id.md).

**3. Configuração e gestão — módulo reaproveitando os padrões do projeto.** Um módulo
`src/modules/webhooks` (mesmo padrão `controller`/`service`/`repository`/`routes`/`schemas` dos
demais) expõe, em alto nível: **CRUD de configuração de webhook** por cliente (url, secret
gerada pela plataforma, lista de status assinados, estado ativo), **rotação de secret** com
_grace period_ de 24h, **histórico de entregas** por webhook, e um **endpoint administrativo de
_replay_ da DLQ** restrito à role `ADMIN`. Erros usam o prefixo `WEBHOOK_`, e nada de
infraestrutura nova é introduzido. → [ADR-006](./adrs/ADR-006-reuso-padroes-do-projeto.md).

A integração crítica com o código existente é a extensão do método `changeStatus`
(`src/modules/orders/order.service.ts`) para publicar o evento dentro da transação — detalhada
no FDD, seção "Integração com o sistema existente".

## Alternativas consideradas

**1. Disparo síncrono do HTTP dentro do `changeStatus` — descartada.** Chamar o webhook
diretamente na transação de mudança de status. **Trade-off que motivou o descarte:** acopla a
latência e a disponibilidade do cliente à transação de negócio, travando mudanças de status de
outros pedidos, e não há rollback aceitável se o cliente estiver offline (`[09:04] Bruno`;
"síncrono está fora de questão", `[09:06] Diego`). Formalizada na
[ADR-001](./adrs/ADR-001-outbox-no-mysql.md).

**2. Fila/stream externo, tipo Redis Streams — descartada.** Publicar os eventos numa fila
dedicada. **Trade-off que motivou o descarte:** exigiria subir e operar infraestrutura nova
(Redis Cluster) para um time pequeno — "overengineering"; o Outbox no MySQL existente resolve
sem custo operacional novo (`[09:07] Diego`). Formalizada na
[ADR-001](./adrs/ADR-001-outbox-no-mysql.md).

**3. Trigger de banco para reagir à inserção (em vez de _polling_) — descartada.** **Trade-off:**
o MySQL não tem listener nativo como o `NOTIFY`/`LISTEN` do Postgres; a trigger só executa SQL e
não notifica processo externo, e o _polling_ de 2s já atende o requisito de < 10s
(`[09:09] Diego`). Formalizada na [ADR-002](./adrs/ADR-002-worker-separado-polling.md).

**4. Garantia exactly-once — descartada.** **Trade-off:** exigiria coordenação entre produtor e
consumidor e tornaria o mecanismo muito mais complexo; at-least-once com `X-Event-Id` cobre 99%
dos casos (`[09:25] Diego`). Formalizada na
[ADR-005](./adrs/ADR-005-at-least-once-x-event-id.md).

## Questões em aberto

1. **Rate limiting de saída.** Se um cliente tiver muitos pedidos mudando de status em curto
   intervalo (ex.: 50 em um minuto), podemos bombardeá-lo com 50 chamadas. Diego levantou o
   ponto e o grupo decidiu **observar e implementar apenas se virar problema** — fica registrado
   como ponto em aberto, não resolvido nesta fase (`[09:38]`–`[09:39] Diego`/`Larissa`).

2. **Notificação ao cliente sobre webhook com falha (e-mail).** Marcos perguntou se dá para
   avisar o cliente por e-mail quando o webhook dele falha repetidamente. Ficou **fora de escopo
   desta fase**, adiado para "talvez a próxima fase, depois de medir o impacto"
   (`[09:37] Larissa`/`Marcos`).

3. **Endurecimento da autorização do CRUD de configuração.** Por ora, os endpoints de
   configuração de webhook aceitam qualquer role autenticada; apenas o _replay_ da DLQ exige
   `ADMIN`. Sofia sinalizou que "mais para frente a gente pode endurecer" — decisão adiada
   (`[09:37] Sofia`).

4. **Escala além de um worker.** A ordenação por `order_id` só vale enquanto for _single-worker_.
   Se for preciso escalar, será necessário particionar por `order_id` ou usar lock pessimista —
   tratado como problema do futuro (`[09:13] Diego`).

## Impacto e riscos

- **Acoplamento com o core de pedidos.** A publicação do evento entra na transação do
  `changeStatus`; por ser atômica, uma falha ao inserir na outbox causa rollback da mudança de
  status (`[09:40] Bruno`). É o comportamento desejado, mas significa que um defeito no módulo de
  webhooks pode impactar o fluxo de pedidos. **Mitigação:** integração via função pura recebendo
  o `tx` (`publishWebhookEvent(tx, ...)`), sem injetar repositório inteiro (`[09:41]`), e testes
  ponta a ponta.
- **Latência mínima de ~2s** por conta do _polling_; aceita frente ao requisito de < 10s
  (`[09:10] Larissa`).
- **Ordenação não global** (só por `order_id`, single-worker) — limitação conhecida e aceita, já
  que os clientes nunca pediram ordering global (`[09:13]`–`[09:14]`).
- **Gestão de segredos.** Secrets por endpoint precisam de rotação e **nunca podem vazar em log**
  (redaction no Pino). Risco concreto: já houve cliente que vazou secret em log (`[09:22] Diego`).
  **Mitigação:** rotação com _grace period_ e revisão de segurança da Sofia antes do deploy —
  reservados ao menos 2 dias úteis para revisar HMAC e geração de secret (`[09:46] Sofia`).
- **Prazo.** Estimativa de **3 sprints** (incluindo a revisão de segurança), com entrega prevista
  para o fim de novembro, conforme pedido da Atlas (`[09:45]`–`[09:47] Larissa`/`Marcos`).

## Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL](./adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker separado em polling](./adrs/ADR-002-worker-separado-polling.md)
- [ADR-003 — Retry com backoff e DLQ](./adrs/ADR-003-retry-backoff-dlq.md)
- [ADR-004 — HMAC-SHA256 e secret por endpoint](./adrs/ADR-004-hmac-secret-por-endpoint.md)
- [ADR-005 — At-least-once com `X-Event-Id`](./adrs/ADR-005-at-least-once-x-event-id.md)
- [ADR-006 — Reúso dos padrões do projeto](./adrs/ADR-006-reuso-padroes-do-projeto.md)
- [ADR-007 — Snapshot do payload na inserção](./adrs/ADR-007-payload-snapshot-na-insercao.md)
