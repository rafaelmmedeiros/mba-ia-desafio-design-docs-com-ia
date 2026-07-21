# ADR-004 — HMAC-SHA256 com secret única por endpoint e rotação com grace period

- **Status:** Aceito
- **Data:** 2026-07-21 (registro) — decisão tomada na reunião técnica de design (`TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior — Plataforma), Bruno (Eng. Pleno — Pedidos), Marcos (PM), Sofia (Eng. Segurança)
- **Decisões relacionadas:** [ADR-005](./ADR-005-at-least-once-x-event-id.md) (headers que acompanham o request), [ADR-006](./ADR-006-reuso-padroes-do-projeto.md) (validações Zod e armazenamento seguindo os padrões do projeto)

## Contexto

O sistema de webhooks expõe eventos com dados de pedidos para um endpoint **fora da nossa
infra**. Isso levanta duas exigências de segurança do lado do cliente: ele precisa validar
que a requisição **veio realmente da gente** e que **ninguém adulterou o payload no meio**
(`[09:19] Sofia`). Como é tráfego que sai da nossa plataforma para um terceiro, não basta
confiar no canal — o próprio conteúdo precisa carregar prova de origem e integridade.

Fora do escopo desta decisão ficam dois requisitos que a própria reunião classificou como
**não arquiteturais**: o TLS obrigatório (a URL do webhook precisa ser `https`, tratado como
validação de schema Zod — `[09:23] Sofia`) e o limite de 64KB de payload (requisito não
funcional — `[09:24] Larissa`). São NFRs relacionados, não o objeto desta ADR, que trata de
autenticidade, integridade e gestão da secret.

## Decisão

Autenticar **cada requisição de webhook com HMAC-SHA256 sobre o corpo do request**. A
assinatura é calculada com uma secret compartilhada entre nós e o cliente e enviada num
header `X-Signature`; o cliente recalcula e compara do lado dele (`[09:20] Sofia`). O
algoritmo é SHA-256 por ser o padrão de mercado, com biblioteca disponível em qualquer
cliente sério (`[09:20] Sofia`).

A secret é **única por endpoint**, não uma secret global da plataforma (`[09:21] Sofia`). A
configuração de cada webhook guarda, entre outros campos, a `url`, a `secret`, o `customer_id`
e o estado ativo (`[09:21] Bruno`); o armazenamento segue os padrões do projeto (ver
[ADR-006](./ADR-006-reuso-padroes-do-projeto.md)).

A secret é **rotacionável**: existe um endpoint para o cliente pedir uma nova secret e, ao
rotacionar, a **secret antiga permanece válida em paralelo por 24h (grace period)** para dar
tempo de migrar os sistemas do cliente; depois desse prazo, a antiga morre (`[09:21] Sofia`;
decisão consolidada em `[09:22] Sofia`).

## Alternativas Consideradas

### 1. Secret global única da plataforma — *descartada*

Usar uma única secret compartilhada por todos os endpoints/clientes. **Trade-off que motivou
o descarte:** se essa secret vazar, compromete **todos** os clientes de uma vez — nas palavras
da Sofia, "se vaza uma, vaza tudo" (`[09:21] Sofia`). A secret por endpoint contém o vazamento a
um único cliente, e o risco é concreto: já houve cliente que vazou secret em log de aplicação
dele (`[09:22] Diego`).

### 2. Rotação sem grace period (troca imediata) — *descartada*

Invalidar a secret antiga no mesmo instante em que a nova é emitida. **Trade-off que motivou
o descarte:** a troca imediata quebraria a integração do cliente durante a janela de
migração, já que ele ainda estaria assinando/verificando com a secret anterior. Por isso
optou-se por manter as duas válidas em paralelo por 24h (`[09:21] Sofia`).

## Consequências

**Positivas**

- **Autenticidade e integridade verificáveis pelo cliente:** o HMAC-SHA256 sobre o corpo
  garante que o cliente detecta origem indevida e adulteração do payload (`[09:19]`,
  `[09:20] Sofia`), sem depender só do canal de transporte.
- **Raio de exposição contido:** secret por endpoint isola o impacto de um vazamento a um
  único cliente, em vez de comprometer a plataforma inteira (`[09:21] Sofia`; motivada pelo
  incidente citado em `[09:22] Diego`).
- **Rotação sem downtime de integração:** o grace period de 24h permite ao cliente migrar
  gradualmente sem quebrar a validação das assinaturas (`[09:21] Sofia`).
- **Padrão de mercado e baixo custo de adoção:** HMAC-SHA256 tem biblioteca em qualquer
  cliente sério, então a integração do lado deles é trivial (`[09:20] Sofia`).

**Negativas / trade-offs**

- A secret é um **segredo compartilhado** e precisa ser protegida dos dois lados; se o
  cliente a expuser (por exemplo, em log de aplicação — `[09:22] Diego`), a mitigação é
  rotacionar. Do nosso lado, a secret **nunca pode aparecer em log**, o que obriga incluí-la
  na lista de campos redigidos do Pino (ver Referências de código).
- Durante o grace period existem **duas secrets válidas em paralelo** para o mesmo endpoint,
  ampliando por 24h a superfície em que uma assinatura é aceita. É um trade-off assumido em
  favor de não quebrar a integração do cliente na migração (`[09:21] Sofia`).
- A rotação com convivência temporária adiciona **estado e ciclo de vida** à configuração do
  webhook (secret vigente, secret anterior e expiração do grace period), com o custo
  operacional de expirar a secret antiga após o prazo.

## Referências de código

- `src/shared/logger/index.ts` — `redact` do Pino: a `secret` do webhook precisa entrar na
  lista de campos redigidos (junto de `*.token`/`*.password` etc.) para não vazar em log.
- `src/middlewares/validate.middleware.ts` e os schemas de `src/modules` — padrão Zod usado
  para validar a entrada (por exemplo, exigir `https` na URL, NFR relacionado citado em
  `[09:23] Sofia`), a ser seguido pelo módulo de webhooks.
- `prisma/schema.prisma` — padrão a seguir para a nova tabela de configuração de webhook que
  guarda `url` + `secret` + `customer_id` + estado ativo (`[09:21] Bruno`).
