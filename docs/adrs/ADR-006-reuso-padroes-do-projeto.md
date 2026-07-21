# ADR-006 — Reúso máximo dos padrões e da infraestrutura já existentes no módulo de webhooks

- **Status:** Aceito
- **Data:** 2026-07-21 (registro) — decisão tomada na reunião técnica de design (`TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior — Plataforma), Bruno (Eng. Pleno — Pedidos), Marcos (PM), Sofia (Eng. Segurança)
- **Decisões relacionadas:** [ADR-002](./ADR-002-worker-separado-polling.md) (`PrismaClient` e worker separado), [ADR-003](./ADR-003-retry-backoff-dlq.md) (`requireRole` no replay de DLQ)

## Contexto

A feature de webhooks entra numa codebase que já tem convenções estabelecidas e em produção.
Cada domínio hoje é um módulo em `src/modules` com `controller`, `service`, `repository`,
`routes` e `schemas` — o módulo de pedidos em `src/modules/orders` é o exemplo canônico desse
padrão (`[09:27] Bruno`). O tratamento de erro é centralizado, o logging é padronizado e a
autorização por papel já existe.

A questão de design não é *se* dá para construir webhooks com essas peças, e sim se o módulo
deve seguir os padrões existentes ou introduzir stack própria. O time é pequeno, e a decisão
tem impacto direto na superfície de manutenção de longo prazo. Larissa fechou o bloco
pedindo **reúso máximo do que já existe**: `AppError`, Pino, error middleware, padrão de
módulos, padrão de schemas Zod e padrão de códigos de erro — "webhook fica como módulo igual
aos outros" (`[09:30] Larissa`).

## Decisão

Construir o módulo de webhooks **sobre os padrões e a infraestrutura já existentes**, sem
introduzir ferramentas ou padrões paralelos. Em concreto:

- **Estrutura de módulo:** uma pasta `src/modules/webhooks` seguindo a mesma composição de
  `controller`/`service`/`repository`/`routes`/`schemas` dos demais módulos (`[09:27]
  Bruno`), registrada no roteamento como os outros.
- **Worker:** `src/worker.ts` como entry-point separada, com a lógica de processamento num
  arquivo do próprio módulo (ex.: `webhook.worker.ts`/`webhook.processor.ts`) — a topologia
  de execução é tratada na [ADR-002](./ADR-002-worker-separado-polling.md) (`[09:28] Bruno`).
- **Erros:** reaproveitar a classe `AppError` e o padrão das classes específicas existentes
  (como `InsufficientStockError` e `InvalidStatusTransitionError`), com código em
  SCREAMING_SNAKE. Todos os códigos do módulo recebem o prefixo `WEBHOOK_` (ex.:
  `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) (`[09:28] Bruno`;
  `[09:29] Larissa`).
- **Middleware de erro:** nenhuma mudança. O middleware centralizado já trata `AppError`, Zod
  e Prisma, então os erros novos são atendidos sem alteração (`[09:29] Bruno`).
- **Logging:** o Pino já está no projeto inteiro; o módulo usa o logger existente, nada novo
  (`[09:29] Bruno`).
- **Validação:** as rotas usam o mesmo padrão de schemas Zod já adotado nos outros módulos
  (`[09:30] Larissa`).
- **Persistência:** o worker abre um `PrismaClient` próprio, porque o client é por processo;
  mesmo banco e mesma `DATABASE_URL` da API (`[09:30] Bruno`, ver
  [ADR-002](./ADR-002-worker-separado-polling.md)).
- **Autorização:** o endpoint de replay de DLQ reaproveita o `requireRole` existente,
  exigindo o papel `ADMIN` (`[09:36] Larissa`, detalhado na
  [ADR-003](./ADR-003-retry-backoff-dlq.md)).

## Alternativas Consideradas

### 1. Stack e padrões próprios para o módulo de webhooks — *descartada*

Introduzir, só para webhooks, uma estrutura de pastas diferente, outra biblioteca de log,
outro formato de erro ou convenções próprias. **Trade-off que motivou o descarte:** aumentaria
a superfície de manutenção e divergiria do restante da codebase sem ganho técnico — o módulo
resolve o problema inteiramente com as peças que já existem. Para um time pequeno, a
consistência vale mais que a autonomia local (`[09:30] Larissa`), no mesmo espírito da
rejeição a "subir infra à toa" registrada em `[09:07] Diego` ("overengineering").

## Consequências

**Positivas**

- **Zero ferramenta ou padrão novo:** o módulo herda `AppError`, Pino, o error middleware, o
  `requireRole` e o padrão de módulos/schemas já validados em produção (`[09:30]` e
  `[09:36] Larissa`).
- **Menor superfície de manutenção e curva de leitura:** quem conhece qualquer outro módulo
  (ex.: `src/modules/orders`) já entende o de webhooks — mesma forma, mesmos códigos de erro,
  mesmo logging.
- **Integração sem alterar peças compartilhadas:** o middleware de erro central absorve os
  erros novos sem modificação, o que reduz o risco de regressão no core (`[09:29] Bruno`).

**Negativas / trade-offs**

- **Acoplamento aos padrões atuais:** o módulo fica preso às convenções existentes; uma
  eventual evolução dessas convenções (ex.: troca de logger ou do formato de erro) passa a
  afetar também webhooks. É o custo aceito da consistência.
- **Duas conexões ao mesmo banco:** por o worker ser processo separado, abre-se um
  `PrismaClient` adicional sobre a mesma `DATABASE_URL`, somando ao pool total de conexões
  (consequência da topologia da [ADR-002](./ADR-002-worker-separado-polling.md)) (`[09:30]
  Bruno`).
- **Disciplina de convenção:** o prefixo `WEBHOOK_` e a aderência ao padrão de módulo
  dependem de revisão; não há mecanismo automático impedindo divergência.

## Referências de código

- `src/modules/orders` — exemplo do padrão de módulo (`controller`/`service`/`repository`/
  `routes`/`schemas`) que `src/modules/webhooks` deve espelhar.
- `src/shared/errors/app-error.ts` — classe `AppError` (com `statusCode`, `errorCode`,
  `details`) reaproveitada pelos erros do módulo.
- `src/shared/errors/http-errors.ts` — `InsufficientStockError` e
  `InvalidStatusTransitionError`; padrão de código em SCREAMING_SNAKE a ser seguido com
  prefixo `WEBHOOK_`.
- `src/middlewares/error.middleware.ts` — tratamento centralizado de `AppError`, Zod e Prisma,
  que absorve os erros novos sem alteração.
- `src/middlewares/validate.middleware.ts` — validação Zod nas rotas, mesmo padrão para os
  schemas do módulo.
- `src/middlewares/auth.middleware.ts` — `requireRole`, reaproveitado no replay de DLQ
  exigindo `ADMIN`.
- `src/shared/logger/index.ts` — logger Pino já configurado, usado sem adição de dependência.
- `src/app.ts` e `src/routes/index.ts` — composição e registro dos módulos, onde o módulo de
  webhooks é plugado como os demais.
- `prisma/schema.prisma` — padrão de modelos/tabelas que as novas tabelas do módulo devem
  seguir.
