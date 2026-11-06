# ADR-006: Reuso máximo dos padrões existentes do projeto no módulo de webhooks

- **Data da decisão:** reunião técnica de webhooks ([09:30] Larissa)
- **Decisores:** Larissa (Tech Lead), Bruno (Pedidos), Diego (Plataforma)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md), [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md)

## Status

Aceito

## Contexto

O OMS já tem convenções consolidadas ([09:27] Bruno):

- **Módulos por domínio** em `src/modules/<dominio>/`, com `controller`, `service`, `repository`, `routes` e `schemas`. Exemplos: `src/modules/orders/` e `src/modules/customers/`.
- **Erros tipados:** a classe base `AppError` (`src/shared/errors/app-error.ts`) carrega `statusCode` e `errorCode`. As subclasses ficam em `src/shared/errors/http-errors.ts`, como `InvalidStatusTransitionError` com o código `INVALID_STATUS_TRANSITION` e `InsufficientStockError` com `INSUFFICIENT_STOCK` ([09:28] Bruno).
- **Tratamento centralizado de erros:** `src/middlewares/error.middleware.ts` já trata `AppError`, `ZodError` e erros conhecidos do Prisma (`P2002`, `P2025`) ([09:29] Bruno).
- **Logger Pino** compartilhado em `src/shared/logger/index.ts` ([09:29] Bruno).
- **Validação Zod** por rota, com `validate()` em `src/middlewares/validate.middleware.ts`.
- **Autorização** por `authenticate` e `requireRole(...)` em `src/middlewares/auth.middleware.ts`.
- **Ids UUID** em todas as tabelas (`@default(uuid()) @db.Char(36)` em `prisma/schema.prisma`) ([09:51] Larissa).

O time é pequeno ([09:07] Diego) e a feature tem prazo apertado ([09:45] Marcos).

## Decisão

O webhook vira **um módulo igual aos outros** e reaproveita tudo o que já existe ([09:30] Larissa):

- Pasta `src/modules/webhooks/` com `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts` e `webhook.schemas.ts` ([09:27] Bruno). A lógica de processamento do worker fica em `src/modules/webhooks/webhook.worker.ts` ([09:28] Bruno).
- Os erros do módulo são subclasses de `AppError`, com o **prefixo `WEBHOOK_`** em todos os códigos. Bruno citou como exemplos `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED` ([09:28] Bruno, [09:29] Larissa).
- O logger é o mesmo Pino, sem biblioteca nova de log ([09:29] Bruno).
- O `error.middleware.ts` passa a capturar os erros novos **sem modificação** ([09:29] Bruno).
- O replay da DLQ usa o `requireRole('ADMIN')` que já existe ([09:36] Larissa).
- A integração com pedidos é feita pela função `publishWebhookEvent(tx, order, fromStatus, toStatus)`. Ela recebe o `Prisma.TransactionClient` da transação corrente, em vez de injetar um repository inteiro no `OrderService` ([09:41] Bruno, [09:41] Diego).
- O worker usa uma instância própria de `PrismaClient`, com a mesma `DATABASE_URL` ([09:30] Bruno). É o singleton `prisma` de `src/config/database.ts`, que é criado uma vez por processo.
- Os ids são UUID ([09:51] Larissa).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Injetar um `WebhookRepository` no construtor de `OrderService`** | Uma função que recebe o `tx` resolve com acoplamento menor e "não precisa injetar repository inteiro" ([09:41] Diego). |
| **Introduzir bibliotecas ou padrões novos** (outro logger, outra estrutura de erros) | Contraria a decisão de reuso máximo ([09:30] Larissa) e o "não vamos botar nada novo" ([09:29] Bruno). |
| **Ids auto-incrementais no outbox** | Descartados em favor do UUID, "segue o padrão do resto do projeto" ([09:51] Larissa, respondendo a [09:51] Diego). |

## Consequências

### Positivas

- Menos código novo e menos decisões novas. Quem conhece um módulo do OMS entende o de webhooks.
- O formato de erro HTTP (`{ error: { code, message, details } }`) fica igual em toda a API.
- O worker reaproveita o `prisma` de `src/config/database.ts` e o mesmo logger, com o mesmo `redact`.

### Negativas / pontos de atenção

- O `validate()` converte **qualquer** `ZodError` em `ValidationError`, com o código fixo `VALIDATION_ERROR`. As regras que ficam no schema Zod, como a de HTTPS ([09:23] Sofia), portanto não aparecem como `error.code = WEBHOOK_*`. O [FDD](../FDD.md) define como esses códigos aparecem em `details`.
- O `NotFoundError` atual fixa o código `NOT_FOUND`: o construtor só recebe o nome do recurso. Para emitir `WEBHOOK_NOT_FOUND`, a classe nova estende `AppError` diretamente. Já `ConflictError` e `BadRequestError` aceitam código customizado.
- `WEBHOOK_SECRET_REQUIRED`, citado como exemplo por Bruno ([09:28] Bruno), foi avaliado e **não foi adotado**. A secret é gerada pela plataforma e devolvida na criação ([09:31] Marcos), e a coluna é obrigatória. Não existe cenário, nem na API nem no worker, em que ela esteja ausente. Manter o código só para reaproveitar o exemplo criaria um erro impossível de disparar.
- O `errorMiddleware` só atua em requisições Express. O worker não passa por ele e precisa capturar e registrar os próprios erros.
- O `redact` atual do Pino não cobre `secret`. Reusar o logger exige estender a lista de caminhos mascarados.
- O projeto não tem biblioteca de métricas nem de tracing. A observabilidade do worker vai se apoiar em logs estruturados do Pino, com o `event_id` como id de correlação. Isso é coerente com "não vamos botar nada novo" ([09:29] Bruno), mas limita os dashboards.

**Trade-off explícito:** priorizamos consistência e velocidade de entrega, ao custo de herdar as limitações dos padrões atuais: código de erro fixo na validação Zod e ausência de stack de métricas e tracing.
