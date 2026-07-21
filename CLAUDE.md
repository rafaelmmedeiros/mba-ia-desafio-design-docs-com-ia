# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Natureza do projeto (leia primeiro)

Este é um **desafio de MBA cujo entregável é documentação, não código**. A tarefa é
transformar a transcrição de uma reunião técnica (`TRANSCRICAO.md`) + a base de código
existente em um pacote de _design docs_ para a feature **"Sistema de Webhooks de
Notificação de Pedidos"**. O enunciado completo e a checklist de aceite estão em
`DESAFIO.md` — trate-o como a especificação normativa do que precisa ser entregue.

**Regra absoluta:** nunca altere `src/`, `prisma/`, `tests/` nem configs (`package.json`,
`tsconfig*`, `.eslintrc*`, `docker-compose.yml`, etc). O código é **contexto read-only**.
Se um pedido soar como "implemente X", o resultado correto é _documentar como X seria
implementado_, não escrever código. A única árvore que você produz/edita é `docs/`,
`README.md` e este `CLAUDE.md`.

## Entregáveis e onde vivem

Todos os arquivos de `docs/` estão hoje como templates vazios (`<!-- ... -->`). Você os preenche.

| Arquivo | Papel / altura | Regra anti-duplicação |
|---|---|---|
| `docs/PRD.md` | Problema, público, escopo, métricas — *por que e o quê* | Nível produto; sem detalhe de implementação |
| `docs/RFC.md` | Proposta técnica p/ revisão: abordagem, alternativas, questões em aberto — *como pretendemos resolver* | Conciso (2–4 págs); **não** replica o detalhe do FDD; linka os ADRs |
| `docs/FDD.md` | Especificação de implementação: fluxos, contratos, erros — *como construir em detalhe* | Único doc com payloads, matriz de erros, seção "Integração com o sistema existente" |
| `docs/adrs/ADR-NNN-*.md` | Cada decisão isolada (MADR): Status, Contexto, Decisão, Alternativas, Consequências | 5–8 ADRs; cada decisão em UM lugar |
| `docs/TRACKER.md` | Rastreabilidade de cada item → `TRANSCRICAO` ou `CODIGO` | Tabela fixa (ver `DESAFIO.md` §5) |
| `README.md` | Documentação **do processo** (substitui o enunciado atual) | Sobre a jornada, não sobre a feature |

Se o mesmo conteúdo aparece em dois documentos, algo está na altura errada. Ordem de
produção sugerida pelo desafio: **ADRs → RFC → FDD → PRD → Tracker → README**.

## Disciplina de rastreabilidade (o coração do desafio)

Só existem **duas fontes válidas**: `TRANSCRICAO.md` e o código. Não invente requisitos,
decisões ou restrições. Regra prática: se você não consegue preencher a coluna
`Localização` do Tracker para uma afirmação (timestamp `[hh:mm] Nome` ou caminho de
arquivo real), ela provavelmente é alucinação — corrija ou remova.

Ao citar código, **use caminhos que existem** (a checklist de aceite reprova arquivo
inexistente). Ao citar a reunião, use o timestamp e o falante reais da transcrição.

## Decisões fechadas na reunião (o esqueleto)

As 6 decisões principais (o pacote de ADRs deve cobrir ≥5):
1. **Outbox no MySQL** — evento inserido na mesma transação do `changeStatus` (atômico).
2. **Retry com backoff + DLQ** — 5 tentativas `1m/5m/30m/2h/12h`, depois tabela `webhook_dead_letter`.
3. **HMAC-SHA256, secret por endpoint** — assinatura no header, rotação com _grace period_ 24h.
4. **At-least-once + `X-Event-Id`** — UUID por evento; dedupe é responsabilidade do cliente.
5. **Worker em processo separado, polling 2s** — não roda dentro da API; latência-alvo < 10s.
6. **Reuso dos padrões do projeto** — `AppError`, Pino, error middleware, módulo em `src/modules/`, prefixo `WEBHOOK_`.

Secundárias que podem virar ADR extra ou só FDD: payload snapshot na inserção, timeout 10s,
headers (`X-Signature`/`X-Timestamp`/`X-Webhook-Id`), limite 64KB, UUID como PK.

## O que **não** entra (evita as alucinações mais prováveis)

A reunião descartou/adiou explicitamente — se aparecer como requisito, está errado:
- **E-mail de alerta** ao cliente após falhas → "próxima fase" `[09:37] Larissa`.
- **Rate limiting de saída** → "observar e decidir depois" `[09:39]`.
- **Dashboard/painel visual** → fora de escopo, é projeto do time de frontend `[09:40]`.
- **Webhooks inbound** → só _outbound_ `[09:03] Sofia`.
- **Ordering global** → só por `order_id` enquanto single-worker; limitação conhecida `[09:13]`.
- **TLS obrigatório e limite 64KB** são **NFR/validação Zod**, não decisão arquitetural (não vire ADR) `[09:24]`.

## Âncoras de código (o que a documentação referencia)

Base: Node 20 + TypeScript (ESM), Express, Prisma + MySQL, Zod, Pino. Padrão de módulo:
`src/modules/<dominio>/` com `controller` / `service` / `repository` / `routes` / `schemas`.

- `src/modules/orders/order.service.ts` → **`changeStatus()`**: a transação-alvo. Hoje faz
  `update` da order + insert em `order_status_history` + débito/reposição de estoque, tudo
  em `prisma.$transaction`. O outbox insere aqui dentro (a reunião propõe
  `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebendo o `tx`).
- `src/modules/orders/order.status.ts` → máquina de estados (`canTransition`,
  `shouldDebitStock`, `shouldReplenishStock`) e enum `OrderStatus`
  (`PENDING→PAID→PROCESSING→SHIPPED→DELIVERED`, `CANCELLED`). Fonte dos `event_type`.
- `src/shared/errors/` → `app-error.ts` (base `AppError` com `statusCode`/`errorCode`/`details`),
  `http-errors.ts` (subclasses ex. `InvalidStatusTransitionError`, `InsufficientStockError`;
  códigos `SCREAMING_SNAKE`). Erros do módulo seguem o prefixo **`WEBHOOK_`**.
- `src/middlewares/error.middleware.ts` → trata `AppError`/`ZodError`/Prisma de forma
  centralizada; pega os erros novos sem alteração.
- `src/middlewares/auth.middleware.ts` → `authenticate` + **`requireRole('ADMIN')`** (o replay
  de DLQ exige ADMIN); CRUD de webhook é rota autenticada comum.
- `src/shared/logger/index.ts` → Pino singleton com `redact` (segredos precisam entrar na lista).
- `src/config/database.ts` / `src/config/env.ts` → `prisma` e `env` (Zod) singletons. O worker,
  por ser **processo separado**, abre o próprio `PrismaClient`.
- `src/server.ts` → entrypoint da API; o worker proposto é um `src/worker.ts` análogo (`npm run worker`).
- `src/app.ts` (DI manual em `buildControllers`) + `src/routes/index.ts` → onde o módulo
  `webhooks` se registraria.
- `prisma/schema.prisma` → MySQL; PKs UUID `@db.Char(36)`, `@@map` snake_case. Novas tabelas
  discutidas: config de webhook, `webhook_outbox`, `webhook_dead_letter`, deliveries.

## Comandos (apenas contexto — o entregável não roda código)

App de referência (útil para validar que caminhos/nomes citados existem):

```bash
npm run dev            # API em watch (tsx)
npm run build          # tsc -> dist/
npm test               # vitest run  (single: npx vitest run tests/orders.test.ts | -t "nome do teste")
npm run lint           # eslint . --ext .ts
npm run db:migrate     # prisma migrate dev   (MySQL via docker-compose.yml)
npm run db:seed
```

## Convenção de commits

O desafio valoriza um bom histórico. Faça commits pequenos e temáticos (idealmente um por
documento/marco), em português, no estilo `docs:` — a mensagem deve explicar **o porquê** da
decisão de documentação, não só "adiciona arquivo". Ex.: `docs: adicionar ADR-002 (retry+DLQ)`.
