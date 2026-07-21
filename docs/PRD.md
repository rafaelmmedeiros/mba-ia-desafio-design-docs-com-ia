# PRD — Sistema de Webhooks de Notificação de Pedidos

> Documento de produto (o "porquê e o quê"). O "como" técnico está no [RFC](./RFC.md) e no
> [FDD](./FDD.md); as decisões isoladas, nos [ADRs](./adrs/). Rastreabilidade em
> [TRACKER](./TRACKER.md).

## 1. Resumo e contexto da feature

O Order Management System (OMS) opera em produção com pedidos que percorrem uma máquina de
estados (`PENDING → PAID → PROCESSING → SHIPPED → DELIVERED`, ou `CANCELLED`). Hoje **não há
nenhuma forma de notificar sistemas externos** quando um pedido muda de status. Esta feature
adiciona **webhooks _outbound_**: sempre que o status de um pedido muda, a plataforma envia um
evento HTTP, assinado e resiliente, para o endpoint que o cliente cadastrou — em tempo quase
real (`[09:00]`–`[09:02] Marcos`).

## 2. Problema e motivação

Três clientes B2B — **Atlas Comercial, MaxDistribuição e Nova Cargo** — pediram formalmente para
serem notificados quando o status dos pedidos deles muda (`[09:00] Marcos`). Hoje eles fazem
_polling_ repetido no `GET /orders` para descobrir mudanças, o que torna a integração **lenta e
cara** e os obriga a "ficar atualizando manualmente" (`[09:00]`–`[09:02] Marcos`).

Há motivação comercial direta: a **Atlas sinalizou que pode migrar para um concorrente** se a
entrega não sair até o fim do trimestre (`[09:00] Marcos`). Resolver isso protege receita e
melhora a experiência de integração dos clientes B2B.

## 3. Público-alvo e cenários de uso

**Público-alvo:** clientes B2B integradores que consomem a API do OMS e precisam reagir a
mudanças de status de pedido nos próprios sistemas (ERPs, WMS, dashboards internos deles).

**Cenários de uso:**

- **Notificação de expedição.** Um cliente assina os status `SHIPPED` e `DELIVERED`; quando um
  pedido é despachado, o sistema dele recebe o evento e dispara o aviso ao cliente final, sem
  _polling_ (`[09:33] Marcos`).
- **Reconciliação de pagamento.** Um cliente assina `PAID` para liberar automaticamente a
  separação do pedido no WMS dele assim que o pagamento é confirmado.
- **Diagnóstico e recuperação.** O cliente consulta o histórico de entregas do webhook dele — os
  últimos envios, com sucesso/falha, resposta e tempo de resposta — para investigar o que recebeu
  (`[09:34] Marcos`); internamente, um **ADMIN** reprocessa eventos que falharam
  (`[09:35]`–`[09:36]`).

## 4. Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta |
|---|---|---|
| Notificar em tempo quase real | Latência entre a mudança de status e a entrega ao cliente | **< 10 segundos** (p95) (`[09:02] Marcos`) |
| Eliminar o _polling_ dos clientes B2B | Nº de clientes migrados de _polling_ para webhook | **3 clientes** (Atlas, MaxDistribuição, Nova Cargo) (`[09:00]`) |
| Entregar dentro da janela comercial | Data de disponibilização | **Fim de novembro** / ~3 sprints (`[09:45]`–`[09:47]`) |
| Não perder eventos por indisponibilidade temporária do cliente | Cobertura da janela de retry | Reentrega por **até ~15h** antes da DLQ (`[09:17]`) |

## 5. Escopo

**Incluso:** CRUD de configuração de webhook por cliente; assinatura de quais status receber;
rotação de secret; envio assinado (HMAC) na mudança de status; retry com DLQ; histórico de
entregas; replay administrativo.

**Fora de escopo** (explicitamente descartado ou adiado na reunião):

- **E-mail de alerta ao cliente** quando o webhook dele falha repetidamente — adiado para uma
  próxima fase, "depois que a gente medir o impacto" (`[09:37] Larissa`).
- **Rate limiting de saída** (limitar a taxa de chamadas a um cliente) — decidido "observar e
  implementar se virar problema" (`[09:39] Diego`/`Larissa`).
- **Dashboard/painel visual** para o cliente — é projeto separado do time de frontend
  (`[09:40] Larissa`).
- **Webhooks _inbound_** (cliente enviando para nós) — o escopo é só _outbound_
  (`[09:02]`–`[09:03] Sofia`/`Marcos`).
- **Arquivamento** de eventos entregues (após ~30 dias) — reconhecido, mas fora desta feature
  (`[09:08] Diego`).

## 6. Requisitos funcionais

| ID | Requisito | Fonte |
|---|---|---|
| PRD-RF-01 | Cadastrar um webhook (`POST`) com URL e lista de status assinados; a secret é gerada pela plataforma e devolvida na criação | `[09:31] Marcos` |
| PRD-RF-02 | Editar a configuração de um webhook (`PATCH`) | `[09:33] Bruno` |
| PRD-RF-03 | Remover um webhook (`DELETE`) | `[09:33] Bruno` |
| PRD-RF-04 | Listar os webhooks de um cliente (`GET`) | `[09:33] Bruno` |
| PRD-RF-05 | Assinar por endpoint quais status disparam notificação (filtro aplicado na inserção do evento) | `[09:33]`–`[09:34]` |
| PRD-RF-06 | Rotacionar a secret de um webhook, mantendo a anterior válida por 24h | `[09:21] Sofia` |
| PRD-RF-07 | Notificar o endpoint do cliente a cada mudança de status assinada, sem bloquear a operação de pedidos | `[09:04]`/`[09:06]` |
| PRD-RF-08 | Assinar cada envio com HMAC-SHA256 usando a secret do endpoint, em header | `[09:20]`–`[09:21] Sofia` |
| PRD-RF-09 | Enviar um identificador único por evento (`X-Event-Id`) para deduplicação pelo cliente | `[09:24]`–`[09:25] Diego` |
| PRD-RF-10 | Reentregar em caso de falha (backoff) e mover para uma DLQ após esgotar as tentativas | `[09:15]`–`[09:18]` |
| PRD-RF-11 | Consultar o histórico de entregas de um webhook (sucesso/falha, resposta, tempo) | `[09:34] Marcos` |
| PRD-RF-12 | Reprocessar (replay) um item da DLQ via endpoint administrativo restrito a `ADMIN` | `[09:18]`/`[09:35]`–`[09:36]` |

## 7. Requisitos não funcionais

| ID | Requisito | Fonte |
|---|---|---|
| PRD-RNF-01 | Latência de entrega abaixo de 10s (worker em _polling_ de 2s) | `[09:02]`/`[09:10]` |
| PRD-RNF-02 | A URL do webhook deve ser `https` (TLS obrigatório); URL `http` é recusada na validação | `[09:23] Sofia` |
| PRD-RNF-03 | Payload máximo de 64KB; acima disso, erro em vez de envio | `[09:23]`–`[09:24]` |
| PRD-RNF-04 | A secret nunca pode ser exposta em logs | `[09:22] Diego` |
| PRD-RNF-05 | Entrega **at-least-once** (não exactly-once); dedup é responsabilidade do cliente | `[09:24]`–`[09:25]` |
| PRD-RNF-06 | Ordenação garantida apenas por `order_id` enquanto single-worker; não há ordering global | `[09:13]` |
| PRD-RNF-07 | Timeout de 10s por tentativa de envio | `[09:42] Diego` |
| PRD-RNF-08 | Sem introduzir nova infraestrutura; reúso da stack existente | `[09:07]`/`[09:30]` |

## 8. Decisões e trade-offs principais

As decisões estão formalizadas nos ADRs; em resumo de produto:

- **Assíncrono via Outbox, não síncrono.** Garante que a mudança de status nunca trava por causa
  de um cliente lento, ao custo de uma latência mínima de ~2s
  ([ADR-001](./adrs/ADR-001-outbox-no-mysql.md), [ADR-002](./adrs/ADR-002-worker-separado-polling.md)).
- **At-least-once, não exactly-once.** Simplicidade e alinhamento com o mercado (Stripe/GitHub),
  ao custo de o cliente ter de deduplicar ([ADR-005](./adrs/ADR-005-at-least-once-x-event-id.md)).
- **Retry longo (5 tentativas, ~15h) com DLQ.** Cobre indisponibilidades reais de clientes, ao
  custo de latência longa no pior caso ([ADR-003](./adrs/ADR-003-retry-backoff-dlq.md)).
- **Secret por endpoint com rotação.** Contém o raio de um vazamento, ao custo de gerir ciclo de
  vida de secret ([ADR-004](./adrs/ADR-004-hmac-secret-por-endpoint.md)).
- **Reúso máximo da stack.** Menor custo de manutenção para um time pequeno, ao custo de acoplar o
  módulo às convenções atuais ([ADR-006](./adrs/ADR-006-reuso-padroes-do-projeto.md)).

## 9. Dependências

- **Código existente do OMS:** a publicação do evento depende de estender o `changeStatus`
  (`src/modules/orders/order.service.ts`) e da máquina de estados
  (`src/modules/orders/order.status.ts`).
- **Banco MySQL / Prisma** já em produção (a outbox mora nele — sem infra nova).
- **Revisão de segurança** da Sofia antes do _deploy_: ao menos 2 dias úteis para HMAC e geração
  de secret (`[09:46] Sofia`).
- **Documentação para integradores:** o portal do desenvolvedor precisa descrever o modelo
  at-least-once e a verificação de assinatura, sob responsabilidade do PM
  (`[09:26]`/`[09:40] Marcos`).

## 10. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Atraso na entrega faz a Atlas migrar para o concorrente | Média | Alto | Escopo enxuto e priorizado; estimativa de 3 sprints com entrega para o fim de novembro (`[09:45]`–`[09:47]`) |
| Um defeito no módulo de webhooks derruba a mudança de status (rollback da transação) | Média | Alto | Integração via função pura recebendo o `tx`, sem injetar repositório; testes ponta a ponta (`[09:41]`) |
| Vazamento de secret compromete a autenticidade dos webhooks | Baixa | Alto | Secret por endpoint (contém o raio), rotação com _grace period_, redaction em log e revisão de segurança antes do deploy (`[09:22]`/`[09:46]`) |
| Cliente indisponível por muitas horas perde eventos | Baixa | Médio | Retry por até ~15h antes da DLQ e replay manual pelo ADMIN (`[09:17]`/`[09:18]`) |

## 11. Critérios de aceitação

- Ao mudar o status de um pedido cujo cliente tem webhook ativo assinando aquele status, o
  endpoint do cliente recebe a notificação em menos de 10 segundos no caso nominal.
- A notificação nunca bloqueia nem faz falhar a operação de mudança de status por causa do
  cliente (assíncrona; a transação só depende do registro do evento).
- O cliente consegue validar a autenticidade da requisição pela assinatura HMAC-SHA256.
- Eventos que falham são reentregues conforme a política e, esgotadas as tentativas, ficam
  visíveis na DLQ e recuperáveis por replay (ADMIN).
- É possível consultar o histórico de entregas de um webhook.
- Nenhum item listado como "fora de escopo" aparece implementado nesta fase.

## 12. Estratégia de testes e validação

- **Atomicidade:** teste que garante _rollback_ conjunto da mudança de status e da inserção na
  outbox quando o registro do evento falha (`[09:40]`).
- **Resiliência:** testes de retry cobrindo a progressão de backoff e a queda para DLQ após a 5ª
  falha; teste de replay recolocando o evento como pendente (`[09:15]`–`[09:18]`).
- **Segurança:** validação da assinatura HMAC-SHA256 por um consumidor de referência; teste da
  rotação com _grace period_ (assinatura antiga e nova válidas em paralelo por 24h); verificação
  de que a secret não aparece em logs (`[09:21]`/`[09:22]`). Revisão dedicada da Sofia antes do
  _deploy_ (`[09:46]`).
- **Contrato:** validação de que URLs `http` e payloads acima de 64KB são recusados
  (`[09:23]`–`[09:24]`).
- **Validação com clientes:** confirmação com Atlas, MaxDistribuição e Nova Cargo de que os
  eventos recebidos atendem à necessidade de "saber quando cada pedido deles muda" (`[09:14]`).
