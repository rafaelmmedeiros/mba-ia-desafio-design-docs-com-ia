# ADR-005 — Entrega at-least-once com identificador de evento em `X-Event-Id`

- **Status:** Aceito
- **Data:** 2026-07-21 (registro) — decisão tomada na reunião técnica de design (`TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior — Plataforma), Bruno (Eng. Pleno — Pedidos), Marcos (PM), Sofia (Eng. Segurança)
- **Decisões relacionadas:** [ADR-001](./ADR-001-outbox-no-mysql.md) (o UUID é gerado na inserção do evento na outbox), [ADR-002](./ADR-002-worker-separado-polling.md) (reentregas do worker podem gerar duplicatas, daí o at-least-once)

## Contexto

A entrega dos webhooks é feita por um worker que lê a outbox e dispara chamadas HTTP com
retry (ver [ADR-002](./ADR-002-worker-separado-polling.md)). Nesse modelo, reentregas são
inevitáveis: uma falha de rede após o cliente já ter processado a chamada, um timeout ou uma
tentativa de retry podem fazer o mesmo evento chegar mais de uma vez. Assumir isso
explicitamente é mais honesto do que prometer uma garantia que a arquitetura não sustenta —
"a gente vai garantir at-least-once. Pode acontecer de o cliente receber o mesmo evento duas
vezes. Ele tem que estar preparado" (`[09:24] Diego`).

Para que o cliente consiga distinguir uma reentrega de um evento novo, é preciso um
identificador estável e único por evento, carregado em todas as tentativas de envio daquele
mesmo evento. Esse identificador nasce quando o evento é inserido na outbox e viaja no header
`X-Event-Id`; se o cliente recebe o mesmo evento duas vezes, ele deduplica pelo `event_id` do
lado dele (`[09:25] Diego`).

Há um trade-off reconhecido em reunião: a deduplicação passa a ser responsabilidade do
cliente (`[09:25] Sofia`). A decisão de aceitá-lo se apoia em ser o padrão de mercado — Stripe
e GitHub adotam o mesmo modelo (`[09:25] Diego`) — e no compromisso de tornar essa expectativa
explícita para os integradores, documentando-a em destaque no portal do desenvolvedor
(`[09:26] Marcos`).

## Decisão

Adotar entrega **at-least-once** e delegar a **deduplicação ao cliente**, expondo um
identificador único por evento no header `X-Event-Id`. Esse identificador é um UUID gerado no
momento em que o evento entra na outbox, permanecendo o mesmo em todas as reentregas daquele
evento — é o que o cliente usa como chave de idempotência do lado dele (`[09:25] Diego`).

A geração como UUID acompanha o padrão de identificadores do resto do projeto, em que a PK da
outbox também é UUID (`[09:51] Larissa`).

## Alternativas Consideradas

### 1. Garantia exactly-once — *descartada*

Garantir que cada evento chegue ao cliente exatamente uma vez, dispensando qualquer
deduplicação do lado dele. **Trade-off que motivou o descarte:** exigiria coordenação entre os
dois lados (produtor e consumidor) e tornaria o mecanismo muito mais complexo; a combinação de
at-least-once com `event_id` resolve 99% dos casos com uma fração do custo (`[09:25] Diego`).

## Consequências

**Positivas**

- **Simplicidade e alinhamento com o mercado:** o modelo é o mesmo de Stripe e GitHub, então
  clientes sérios já têm familiaridade e ferramentas para deduplicar (`[09:25] Diego`).
- **Robustez frente a reentregas:** o worker pode retentar sem medo de gerar efeito duplicado
  no cliente, já que o `X-Event-Id` permite descartar o que já foi processado
  (ver [ADR-002](./ADR-002-worker-separado-polling.md)).
- **Identificador consistente com o projeto:** o `event_id` é o mesmo UUID da linha na outbox,
  seguindo o padrão de identificadores adotado no restante do sistema (`[09:51] Larissa`).

**Negativas / trade-offs**

- **Transfere a deduplicação para o cliente** (`[09:25] Sofia`): quem não persistir os
  `event_id` já vistos pode processar o mesmo evento duas vezes. Mitiga-se documentando a
  expectativa em destaque no portal do desenvolvedor (`[09:26] Marcos`).
- **Não há garantia exactly-once:** cobre-se 99% dos casos, mas o 1% de duplicata residual é
  tratado fora da nossa fronteira, do lado do consumidor (`[09:25] Diego`).

## Referências de código

- `package.json` — a lib `uuid` (`uuid` 11.0.3) já é dependência do projeto e é reaproveitada
  para gerar o `event_id`.
- `src/modules/orders/order.service.ts` — método `changeStatus`, onde o `event_id` passará a ser
  gerado na inserção (proposta) do evento na outbox, junto da mudança de status.
