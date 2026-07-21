# ADR-007 — Snapshot do payload na inserção da outbox

- **Status:** Aceito
- **Data:** 2026-07-21 (registro) — decisão tomada na reunião técnica de design (`TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior — Plataforma), Bruno (Eng. Pleno — Pedidos)
- **Decisões relacionadas:** [ADR-001](./ADR-001-outbox-no-mysql.md) (a coluna de payload faz parte da modelagem da outbox), [ADR-005](./ADR-005-at-least-once-x-event-id.md) (o `X-Event-Id` acompanha o payload snapshot)

## Contexto

Definido o padrão Outbox no MySQL (ver [ADR-001](./ADR-001-outbox-no-mysql.md)), sobrou uma
dúvida de modelagem no fim da reunião: cada linha da `webhook_outbox` deve guardar o **payload
já renderizado** do evento, ou apenas uma **referência** (o `order_id`) para o worker renderizar
o corpo na hora do envio? (`[09:51] Bruno`).

A distinção importa porque há um intervalo de tempo entre a inserção do evento (dentro da
transação de mudança de status) e o disparo HTTP feito pelo worker. Nesse intervalo o pedido
pode sofrer novas mudanças. Se o corpo for montado no momento do envio, ele passa a refletir o
estado **corrente** da order, e não o estado de quando o status realmente mudou — o que
produziria um webhook inconsistente com o fato que ele deveria notificar (`[09:52] Larissa`).

A mesma conversa fixou que a PK da `webhook_outbox` é UUID, seguindo o padrão do resto do
projeto (`[09:51] Larissa`); é uma nota de modelagem correlata, mas o foco desta decisão é o
conteúdo da coluna de payload.

## Decisão

A linha inserida na `webhook_outbox` guarda o **payload já renderizado (snapshot)** do evento,
montado **no momento da inserção**, dentro da mesma transação da mudança de status. O worker
não renderiza nada: ele apenas envia o que já está gravado.

O snapshot congela o estado do pedido no instante da transição. Se o pedido mudar depois, o
evento continua refletindo o estado de quando aquele status mudou (`[09:52] Larissa`); Diego
concordou com "snapshot na inserção" (`[09:52] Diego`) e Bruno confirmou a decisão
(`[09:52] Bruno`).

O conteúdo do snapshot segue o formato enxuto de payload definido no FDD — os campos básicos da
order, sem a lista de items para não inflar o corpo (`[09:43] Diego`; `[09:44] Bruno`).

## Alternativas Consideradas

### 1. Guardar só `order_id` e renderizar no envio — *descartada*

Persistir na outbox apenas uma referência ao pedido e montar o corpo do webhook no momento do
disparo, lendo a order corrente. **Trade-off que motivou o descarte:** o payload poderia
refletir um estado do pedido **posterior** ao momento da mudança de status, gerando
inconsistência entre o evento notificado ao cliente e o fato que de fato ocorreu — "se o pedido
mudar depois, o evento ainda reflete o estado de quando o status mudou. Senão tem caso esquisito"
(`[09:52] Larissa`).

## Consequências

**Positivas**

- **Consistência temporal:** o evento entregue representa exatamente o estado do pedido no
  instante da transição, independentemente de mudanças posteriores (`[09:52] Larissa`).
- **Worker mais simples:** o processo de envio não precisa consultar a order nem montar corpo —
  só transmite o que está gravado, o que reduz acoplamento com o módulo de pedidos e o custo por
  disparo (relevante sob at-least-once e retries, ver [ADR-005](./ADR-005-at-least-once-x-event-id.md)).

**Negativas / trade-offs**

- A coluna de payload da `webhook_outbox` **cresce**: cada evento carrega um snapshot completo
  em vez de uma referência curta. Trade-off aceito em troca da consistência temporal e da
  simplicidade do worker; o formato enxuto (sem items) mantém o corpo pequeno o suficiente
  (`[09:43] Diego`).
- O snapshot é **imutável** após gravado: se a estrutura do payload evoluir, eventos antigos
  ainda na outbox mantêm o formato do momento da inserção.

## Referências de código

- `src/modules/orders/order.service.ts` — método `changeStatus`, onde o snapshot do payload é
  montado e inserido na `webhook_outbox` dentro da `prisma.$transaction` existente.
- `prisma/schema.prisma` — coluna de payload da nova tabela `webhook_outbox`, seguindo os
  modelos atuais (PK UUID, `@@map`).
