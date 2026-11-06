# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Status** | Pronto para implementação (aguarda a sessão de revisão Larissa + Bruno + Diego, [09:50] Larissa) |
| **Data** | 2026-11-06 |
| **Proposta e decisões** | [RFC](RFC.md) · [ADR-001…007](adrs/) |
| **Produto** | [PRD](PRD.md) |
| **Rastreabilidade** | [TRACKER](TRACKER.md) |

> **Convenções deste documento**
> - `[hh:mm] Nome` aponta para a fala de origem em `TRANSCRICAO.md`. Caminhos `src/...` apontam para o código atual.
> - Itens marcados como **(proposta de implementação)** são detalhes que a reunião não fixou, propostos aqui para tornar o documento acionável. Cada um está ligado à decisão de que deriva e pode ser ajustado na revisão sem reabrir os ADRs.

---

## 1. Contexto e motivação técnica

Os clientes B2B fazem polling em `GET /api/v1/orders` para descobrir mudanças de status ([09:00] Marcos). A feature troca isso por **webhooks outbound** com entrega em menos de 10s ([09:02] Marcos).

Hoje a mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`). É um `prisma.$transaction` que, em ordem:

1. lê o pedido com os itens;
2. valida a transição com `canTransition`, de `src/modules/orders/order.status.ts`;
3. debita estoque se a transição for `PENDING → PAID` (`shouldDebitStock`) ou devolve estoque se for `PAID|PROCESSING → CANCELLED` (`shouldReplenishStock`);
4. atualiza `orders.status`;
5. insere em `order_status_history`;
6. relê o pedido com as relações.

> **Correção em relação à reunião:** em [09:04], Bruno diz que a transação "decrementa stock_quantity dos produtos do pedido". Pelo código, o débito só acontece em `PENDING → PAID` e a devolução em cancelamentos vindos de `PAID`/`PROCESSING`. As outras transições só tocam pedido e histórico. Isso não muda o desenho, mas o FDD segue o código.

A aplicação não tem eventos, fila nem nenhum processo além do servidor HTTP (`src/server.ts`). A motivação técnica para cada peça está nos ADRs:

| Peça | Motivação |
|---|---|
| Outbox na mesma transação | Atomicidade entre status e evento ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)) |
| Worker separado com polling | Isolar a entrega do caminho HTTP ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)) |
| Retry e DLQ | Tolerar indisponibilidade do cliente ([ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)) |
| HMAC | Autenticidade e integridade dos envios ([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)) |
| `X-Event-Id` | Deduplicação no cliente ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)) |
| Reuso de padrões | Consistência com o resto do OMS ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)) |
| Snapshot e filtro na inserção | Evento fiel ao momento da mudança, outbox menor ([ADR-007](adrs/ADR-007-snapshot-e-filtro-na-insercao-do-outbox.md)) |

## 2. Objetivos técnicos

| # | Objetivo | Origem |
|---|---|---|
| OT-1 | Toda mudança de status com pelo menos um webhook ativo assinante gera um evento no outbox **na mesma transação**. Se a gravação falhar, a mudança de status também falha. | [09:40] Bruno, [09:41] Diego |
| OT-2 | O worker verifica o outbox a cada 2s. A espera introduzida pelo polling é de no máximo 2s e não depende de outros clientes (§4.2). | [09:09] Diego, [09:10] Larissa |
| OT-3 | Entrega fim a ponta (commit → resposta 2xx) com p95 abaixo de 10s para clientes saudáveis. | [09:02] Marcos |
| OT-4 | Nenhum evento perdido por falha do cliente: o envio original mais 5 retentativas (1m/5m/30m/2h/12h), depois DLQ. | [09:16] Diego, [09:17] Larissa, [09:18] Diego |
| OT-5 | Toda requisição de saída assinada com HMAC-SHA256 usando a secret do endpoint. | [09:22] Sofia |
| OT-6 | Nenhuma dependência nova de runtime: só `node:crypto`, `fetch` nativo (Node ≥ 20) e o `uuid` que o projeto já usa. | [09:29] Bruno, [09:30] Larissa; `package.json` |
| OT-7 | O worker roda como processo independente: `src/worker.ts` e `npm run worker`. | [09:11] Diego, [09:11] Larissa |

## 3. Escopo e exclusões

### 3.1 Dentro do escopo

- O módulo `src/modules/webhooks/`: CRUD de configuração, rotação de secret, histórico de entregas e replay da DLQ.
- A função `publishWebhookEvent(tx, order, fromStatus, toStatus)` e a chamada a ela em `changeStatus` ([09:41] Bruno).
- O worker (`src/worker.ts`, `webhook.worker.ts`): polling, assinatura, envio, retry e DLQ.
- A migration Prisma com 4 tabelas: `webhooks`, `webhook_outbox`, `webhook_deliveries` e `webhook_dead_letter`.
- Um único tipo de evento: `order.status_changed` ([09:43] Diego).

### 3.2 Fora do escopo

As exclusões de produto (e-mail de alerta, dashboard, rate limiting, webhooks inbound, arquivamento, exactly-once) estão no [PRD §5.2](PRD.md#52-fora-de-escopo). As exclusões técnicas que afetam a implementação são estas:

| Exclusão técnica | Consequência para quem implementa | Origem |
|---|---|---|
| Vários workers, particionamento ou lock pessimista | Uma instância só. Nada de `SELECT … FOR UPDATE SKIP LOCKED` nesta fase | [09:13] Diego |
| Evento na **criação** do pedido (`null → PENDING`) | `OrderService.create` não muda. `PENDING` não é aceito no filtro | [RFC Q8](RFC.md#5-questões-em-aberto) |
| Itens do pedido no payload | `buildPayload` não lê `order.items` | [09:43] Diego |
| Arquivamento do outbox | Sem job de limpeza. Só os índices de suporte | [09:08] Diego |

## 4. Fluxos detalhados

### 4.1 Criação do evento no outbox (dentro de `changeStatus`)

```mermaid
sequenceDiagram
  autonumber
  participant C as Cliente HTTP (operador)
  participant OC as OrderController
  participant OS as OrderService.changeStatus
  participant P as publishWebhookEvent(tx,…)
  participant DB as MySQL (tx)
  C->>OC: PATCH /api/v1/orders/:id/status {toStatus}
  OC->>OS: changeStatus(id, input, userId)
  OS->>DB: BEGIN; SELECT order + items
  OS->>OS: canTransition(from,to) / debit/replenish stock
  OS->>DB: UPDATE orders; INSERT order_status_history
  OS->>P: publishWebhookEvent(tx, order, from, to)
  P->>DB: SELECT webhooks WHERE customerId=? AND active=true
  P->>P: filtra os que têm `to` em events
  alt nenhum assinante
    P-->>OS: 0 eventos (nada inserido)
  else 1..N assinantes
    P->>DB: INSERT webhook_outbox (1 linha por webhook, payload snapshot)
  end
  OS->>DB: SELECT refreshed order; COMMIT
  Note over OS,DB: Qualquer exceção (inclusive no INSERT do outbox) faz ROLLBACK de tudo
  OS-->>OC: pedido atualizado (resposta inalterada)
```

#### Regras

1. A chamada acontece **depois** de `tx.orderStatusHistory.create(...)` e **antes** do `findUnique` final. Assim o evento só existe se toda a mudança passou ([09:40] Bruno).
2. O filtro usa os webhooks **ativos** do `order.customerId` cujo `events` contém `toStatus` ([09:33] Marcos, [09:34] Bruno). O filtro roda em memória: um cliente tem poucos endpoints. Isso evita depender de operadores JSON do MySQL no Prisma **(proposta de implementação)**.
3. Cada webhook que passa no filtro gera uma linha com:
   - um `eventId` novo (UUID, [09:25] Diego);
   - `status = PENDING`, `attempts = 0`, `nextAttemptAt = now()`;
   - o `payload` já montado (snapshot, [09:52] Larissa), no formato da [§5.1](#51-requisição-enviada-ao-cliente-outbound).
4. O `timestamp` do payload é o momento da mudança de status, ou seja, o `changedAt` da linha de histórico recém-criada. Não é o momento do envio, que vai no header `X-Timestamp` ([09:44] Diego).
5. **Não** se valida o tamanho de 64KB aqui **(proposta de implementação)**. Um erro na inserção desfaria a mudança de status por um problema de entrega, o que vai contra o princípio de isolamento usado para descartar o envio síncrono ([09:04] Bruno). O limite é verificado no worker ([§4.2](#42-processamento-pelo-worker)).
6. Se o INSERT falhar, a função registra `webhook_outbox_insert_failed` em log e **relança** o erro original. A transação faz rollback e o status não muda ([09:40] Bruno). O `errorMiddleware` responde 500 `INTERNAL_SERVER_ERROR`, ou 409/404 se o erro for um Prisma P2002/P2025.

#### Esboço (proposta de implementação)

```ts
// src/modules/webhooks/webhook.publisher.ts
export async function publishWebhookEvent(
  tx: Prisma.TransactionClient,
  order: Pick<Order, 'id' | 'orderNumber' | 'customerId' | 'totalCents'>,
  fromStatus: OrderStatus,
  toStatus: OrderStatus,
  occurredAt: Date,
): Promise<number> {
  const hooks = await tx.webhook.findMany({ where: { customerId: order.customerId, active: true } });
  const targets = hooks.filter((h) => (h.events as OrderStatus[]).includes(toStatus));
  for (const hook of targets) {
    const eventId = uuidv4();
    await tx.webhookOutbox.create({
      data: {
        eventId,
        eventType: 'order.status_changed',
        webhookId: hook.id,
        orderId: order.id,
        payload: buildPayload(eventId, order, fromStatus, toStatus, occurredAt),
      },
    });
  }
  return targets.length;
}
```

> A assinatura acrescenta `occurredAt` aos parâmetros citados em [09:41] Bruno (`tx, order, fromStatus, toStatus`). É o `changedAt` do histórico, e garante que o `timestamp` do payload reflita o momento da transição.

### 4.2 Processamento pelo worker

O worker é **um único processo** ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)). Dentro dele, os eventos são organizados em **filas lógicas por webhook** ("lanes"). Isso concilia três requisitos da reunião:

- ordem por pedido com instância única ([09:12] Diego);
- espera de polling de no máximo 2s ([09:10] Larissa);
- latência abaixo de 10s para cada cliente ([09:02] Marcos), sem que um cliente lento (até 10s de timeout, [09:42] Diego) atrase os outros.

As lanes são uma **proposta de implementação**.

```mermaid
sequenceDiagram
  autonumber
  participant T as Tick (a cada 2s)
  participant L as Lane do webhook W (assíncrona)
  participant DB as MySQL
  participant X as Endpoint do cliente W
  T->>DB: SELECT pendentes vencidos de webhooks ATIVOS<br/>cujo webhookId NÃO está em processamento,<br/>ORDER BY createdAt LIMIT 50
  T->>T: agrupa por webhookId
  T-->>L: inicia 1 lane por webhook (não espera terminar)
  loop eventos do webhook W, em ordem de createdAt
    L->>DB: UPDATE status='PROCESSING'
    L->>L: body = JSON.stringify(payload); checa 64KB; assina HMAC
    L->>X: POST url (timeout 10s, redirect: manual)
    alt 2xx
      L->>DB: INSERT webhook_deliveries(SUCCESS); UPDATE outbox DELIVERED
    else falha (timeout / rede / não-2xx)
      L->>DB: INSERT webhook_deliveries(FAILURE); aplica §4.3 e encerra a lane
    end
  end
  L-->>T: libera o webhookId
```

#### Regras

1. **Tick a cada 2s** ([09:09] Diego). Cada tick busca os eventos `PENDING` com `nextAttemptAt <= now()` de webhooks **ativos** que **não** têm lane em andamento. Ele inicia as lanes e **não espera** que elas terminem. Assim, a espera de um cliente saudável é sempre de no máximo 2s, independentemente de clientes lentos **(proposta de implementação)**.
2. **Lote pequeno** ([09:08] Diego): no máximo 50 eventos por tick e no máximo 20 lanes simultâneas, em constantes **(proposta de implementação)**.
3. **Ordem:** dentro de uma lane, os eventos do webhook seguem `createdAt` e são processados **um de cada vez**. Um webhook nunca tem duas lanes ao mesmo tempo. Isso preserva a ordem por pedido para aquele webhook, como Diego descreveu para o worker único ([09:12] Diego). Se um evento falha, a lane **encerra** e os eventos seguintes daquele webhook esperam o próximo tick. A ultrapassagem durante o backoff é a questão [RFC Q6](RFC.md#5-questões-em-aberto).
4. **Corpo:** `JSON.stringify(payload)` é calculado **uma vez**. A mesma string é assinada e enviada, para que o HMAC bata byte a byte com o que o cliente recebe ([09:22] Sofia).
5. **Limite de 64KB** ([09:24] Diego, [09:24] Larissa): se `Buffer.byteLength(body) > 65536`, o evento **não é enviado**. Ele vai direto para a DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`, sem retry. Repetir não muda o tamanho, e passar desse limite indica bug: "se chegou nesse tamanho, tem algo errado" ([09:23] Sofia).
6. **Webhook inativo = pausado.** Eventos de webhooks com `active = false` não são selecionados. Eles ficam `PENDING` e voltam a fluir se o webhook for reativado. A remoção (E4) descarta esses eventos **(proposta de implementação)**.
7. **Sucesso:** qualquer status 2xx recebido em até 10s ([09:42] Diego). O corpo da resposta é gravado truncado (4 KB) em `webhook_deliveries` e não é interpretado **(proposta de implementação)**.
8. **Redirecionamentos (3xx) contam como falha.** O fetch usa `redirect: 'manual'`, porque seguir um redirect poderia levar o envio para um destino `http://` e furar o HTTPS obrigatório ([09:23] Sofia) **(proposta de implementação)**.
9. **Inicialização:** eventos presos em `PROCESSING`, deixados por uma queda do worker no meio de um envio, voltam para `PENDING`. Isso pode gerar reenvio, o que é aceito pela garantia at-least-once ([09:24] Diego) **(proposta de implementação)**.
10. **Encerramento** (`SIGINT`/`SIGTERM`): o worker para de agendar ticks, espera as lanes em andamento terminarem o envio atual (no máximo 10s) e chama `prisma.$disconnect()`, como `src/server.ts` já faz.

### 4.3 Retry

Cada falha (timeout, erro de rede, resposta não-2xx ou 3xx) grava uma linha em `webhook_deliveries` com `result = FAILURE` e um `errorCode`. Depois disso:

```text
attempts += 1
se attempts <= 5:   status = PENDING, nextAttemptAt = now() + BACKOFF[attempts - 1]
senão:              status = FAILED  → §4.4 (DLQ)
BACKOFF = [1 min, 5 min, 30 min, 2 h, 12 h]
```

| Envio | Quando | Intervalo desde o envio anterior | Tempo acumulado desde a 1ª falha |
|---|---|---|---|
| 1 (original) | próximo tick, ≤ 2s após o commit | — | — |
| 2 (retry 1) | | 1 min | 1 min |
| 3 (retry 2) | | 5 min | 6 min |
| 4 (retry 3) | | 30 min | 36 min |
| 5 (retry 4) | | 2 h | 2 h 36 min |
| 6 (retry 5, última) | | 12 h | **14 h 36 min** ("quase 15 horas", [09:17] Diego) |

Os intervalos vêm de [09:17] Diego e [09:17] Larissa. A contagem como retentativas segue o [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md), sustentada por [09:16] Diego. O único erro não retentável, payload maior que 64KB (§4.2, item 5), vai direto para a DLQ.

### 4.4 DLQ e replay

**Ida para a DLQ.** Uma única transação faz duas coisas:

- `UPDATE webhook_outbox SET status='FAILED', lastErrorCode=...`;
- `INSERT webhook_dead_letter`, com `outboxId`, `webhookId`, `eventId`, `payload`, `reason` (código `WEBHOOK_*`), `lastError` (mensagem, resumida), `attempts` e `failedAt` ([09:18] Diego).

O worker registra `webhook_dead_lettered` em log, no nível `warn`.

**Replay**: `POST /api/v1/admin/webhooks/dead-letter/:id/replay`, só para ADMIN ([09:18] Diego, [09:36] Sofia).

```mermaid
sequenceDiagram
  participant A as ADMIN
  participant R as webhook admin routes
  participant S as WebhookService.replayDeadLetter
  participant DB as MySQL
  A->>R: POST /admin/webhooks/dead-letter/:id/replay
  R->>R: authenticate → requireRole('ADMIN')
  R->>S: replayDeadLetter(id, user)
  S->>DB: BEGIN; SELECT dead_letter, outbox, webhook
  alt já reprocessado / webhook removido ou inativo (proposta)
    S-->>A: 409 WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED / WEBHOOK_INACTIVE
  else
    S->>DB: UPDATE outbox SET status=PENDING, attempts=0, nextAttemptAt=now()
    S->>DB: UPDATE dead_letter SET replayedAt=now(), replayedById=user.id
    S->>DB: COMMIT
    S->>S: logger.info('webhook_dead_letter_replayed', {deadLetterId, eventId, userId})
    S-->>A: 202
  end
```

- O replay **reaproveita a mesma linha do outbox**, que volta a ficar pendente ([09:18] Diego). `attempts` volta a 0, o que dá um novo ciclo completo de retentativas **(proposta de implementação)**. O `eventId` e o payload (snapshot) continuam os mesmos, para que o cliente consiga deduplicar ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)). A assinatura é refeita com a secret **atual**.
- A auditoria de quem fez o replay fica em dois lugares: no log estruturado e na coluna `replayedById` ([09:36] Sofia).
- Se o evento falhar de novo depois do replay, uma **nova** linha é criada na DLQ. A anterior fica como histórico **(proposta de implementação)**.

### 4.5 Rotação de secret

1. `POST /api/v1/webhooks/:id/rotate-secret` ([09:21] Sofia).
2. Na mesma transação:
   - `previousSecret = secret`;
   - `previousSecretExpiresAt = now() + 24h`;
   - `secret = nova` (32 bytes aleatórios de `crypto.randomBytes`, em hex) **(proposta de implementação)**.
3. Enquanto `now() < previousSecretExpiresAt`, o worker envia **duas** assinaturas no `X-Signature` ([§5.1](#51-requisição-enviada-ao-cliente-outbound)). Assim o cliente que ainda valida com a secret antiga continua funcionando ([09:21] Sofia).
4. Depois das 24h, a secret anterior é ignorada no envio e apagada na próxima rotação. Pedir outra rotação **durante** a carência retorna `409 WEBHOOK_SECRET_ROTATION_IN_PROGRESS`, para não existirem três secrets válidas ao mesmo tempo **(proposta de implementação)**.

## 5. Contratos públicos

Todas as rotas ficam sob o prefixo `/api/v1`, montado em `src/app.ts`. Todas exigem `Authorization: Bearer <JWT>` (`authenticate`).

Os erros seguem o formato do `errorMiddleware`:

```json
{ "error": { "code": "…", "message": "…", "details": [] } }
```

Respostas paginadas seguem `paginated()` de `src/shared/http/response.ts`.

**Secret (proposta derivada).** A secret é devolvida na criação ([09:31] Marcos) e na rotação, e **nunca** em listagens ou edições. A reunião não discutiu exposição do nosso lado. A regra deriva do incidente de vazamento citado em [09:22] Diego e do `redact` atual, que não cobre `secret` (`src/shared/logger/index.ts`).

**`customer_id`.** Ele chega no **body** (E1) ou no **path** (E2), nunca pelo JWT ([09:32] Larissa).

| # | Método e rota | Papel | Sucesso | Origem |
|---|---|---|---|---|
| E1 | `POST /api/v1/webhooks` | qualquer autenticado | 201 | [09:31] Marcos, [09:37] Sofia |
| E2 | `GET /api/v1/customers/:customerId/webhooks` | qualquer autenticado | 200 | [09:33] Bruno, [09:32] Larissa |
| E3 | `PATCH /api/v1/webhooks/:id` | qualquer autenticado | 200 | [09:33] Bruno |
| E4 | `DELETE /api/v1/webhooks/:id` | qualquer autenticado | 204 | [09:33] Bruno |
| E5 | `POST /api/v1/webhooks/:id/rotate-secret` | qualquer autenticado | 200 | [09:21] Sofia |
| E6 | `GET /api/v1/webhooks/:id/deliveries` | qualquer autenticado | 200 | [09:34] Marcos |
| E7 | `POST /api/v1/admin/webhooks/dead-letter/:id/replay` | **ADMIN** | 202 | [09:18] Diego, [09:36] Sofia |
| O1 | `POST <url do cliente>` (outbound) | — | 2xx esperado | [09:43] Diego, [09:44] Diego |

**Rotas e nomes definidos aqui.** A reunião nomeou explicitamente as rotas `/webhooks/:id/deliveries` e `/admin/webhooks/dead-letter/:id/replay`, e os verbos POST, PATCH, DELETE e GET. O caminho de E5 (`rotate-secret`) e o aninhamento de E2 sob `customers` são **propostas de implementação**. O aninhamento segue o "no body ou no path" de [09:32] Larissa.

### 5.1 Requisição enviada ao cliente (outbound)

#### Request

```http
POST https://hooks.atlas.example/oms
Content-Type: application/json
X-Event-Id: 8f0c2b3e-5a7d-4e8a-9a51-2f4b7c1d9e60
X-Webhook-Id: 5e1a9c7b-2d3f-4a6b-8c9d-0e1f2a3b4c5d
X-Timestamp: 2026-11-06T14:03:12.481Z
X-Signature: sha256=9c1f…e04a

{
  "event_id": "8f0c2b3e-5a7d-4e8a-9a51-2f4b7c1d9e60",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-06T14:03:10.902Z",
  "order_id": "3b8f6a12-0c4d-4f7e-8a1b-9d2e5c7f1a34",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f",
  "total_cents": 15900
}
```

| Header | Semântica | Origem |
|---|---|---|
| `Content-Type` | `application/json` | [09:44] Diego |
| `X-Event-Id` | UUID do evento. É estável entre retries e replay. Use para deduplicar ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)). | [09:25] Diego |
| `X-Webhook-Id` | UUID do cadastro do webhook. Identifica qual cadastro recebeu o envio, para clientes com vários endpoints. | [09:44] Sofia |
| `X-Timestamp` | ISO 8601 do **envio**. Serve para o cliente detectar replay. | [09:44] Diego |
| `X-Signature` | HMAC-SHA256 do corpo bruto com a secret do endpoint, em hex. O prefixo `sha256=` e o formato com duas assinaturas durante a carência (`sha256=<hex_nova>,sha256=<hex_anterior>`) são **proposta de implementação**, sujeita à revisão de segurança ([RFC Q7](RFC.md#5-questões-em-aberto)). | [09:22] Sofia |

**Corpo.** É o snapshot do momento da mudança, sem itens ([09:43] Diego). O formato de `order_number` (`ORD-000000`) é o de `reserveOrderNumber` em `order.service.ts`. Os valores de status são o enum `OrderStatus` de `prisma/schema.prisma`. Os campos são os listados por Diego, e `total_cents` é o único "campo básico" incluído **(proposta de implementação)**.

#### Response esperada do cliente

| Resposta | Resultado |
|---|---|
| Qualquer `2xx` em até 10s, com corpo livre | Entregue (`DELIVERED`). |
| `3xx`, `4xx` ou `5xx` | Falha (`WEBHOOK_DELIVERY_HTTP_ERROR`). Entra no retry. |
| Sem resposta em 10s | Falha (`WEBHOOK_DELIVERY_TIMEOUT`). Entra no retry ([09:42] Diego). |
| Erro de DNS, TLS ou conexão | Falha (`WEBHOOK_DELIVERY_NETWORK_ERROR`). Entra no retry. |

```http
HTTP/1.1 200 OK
Content-Type: application/json

{ "received": true }
```

**Verificação do lado do cliente** (exemplo para o portal do desenvolvedor, [09:26] Marcos)

```ts
import { createHmac, timingSafeEqual } from 'node:crypto';
const expected = 'sha256=' + createHmac('sha256', secret).update(rawBody).digest('hex');
const ok = req.header('X-Signature')!.split(',').some((sig) =>
  sig.length === expected.length && timingSafeEqual(Buffer.from(sig), Buffer.from(expected)));
// depois: deduplicar por X-Event-Id antes de processar
```

### 5.2 E1 — Criar webhook

#### Request

```http
POST /api/v1/webhooks
Authorization: Bearer <JWT>
Content-Type: application/json

{
  "customerId": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f",
  "url": "https://hooks.atlas.example/oms",
  "events": ["SHIPPED", "DELIVERED"]
}
```

**Validação** (`createWebhookSchema`, em Zod)

| Campo | Regra |
|---|---|
| `customerId` | `uuid` ([09:32] Larissa) |
| `url` | URL com esquema `https:` obrigatório ([09:23] Sofia). Máximo de 2048 caracteres **(proposta)** |
| `events` | Array não vazio, sem repetição, de `OrderStatus` **exceto `PENDING`** |

`PENDING` fica de fora porque nenhuma transição em `src/modules/orders/order.status.ts` leva a `PENDING` ([RFC Q8](RFC.md#5-questões-em-aberto)). O corpo **não** aceita `secret`, porque ela é gerada pelo servidor ([09:31] Marcos): 32 bytes de `crypto.randomBytes`, em hex **(proposta)**.

#### Response 201 Created

```json
{
  "id": "5e1a9c7b-2d3f-4a6b-8c9d-0e1f2a3b4c5d",
  "customerId": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f",
  "url": "https://hooks.atlas.example/oms",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "3f9a0c5e…(64 hex)…b71d",
  "createdAt": "2026-11-06T13:00:00.000Z",
  "updatedAt": "2026-11-06T13:00:00.000Z"
}
```

#### Erros

| Status | Código | Quando |
|---|---|---|
| 400 | `VALIDATION_ERROR` | `details[]` traz `WEBHOOK_INVALID_URL` ou `WEBHOOK_INVALID_EVENT_FILTER` ([§6](#6-matriz-de-erros)) |
| 401 | `UNAUTHORIZED` | — |
| 404 | `NOT_FOUND` | `Customer not found`, via `NotFoundError('Customer')` |

### 5.3 E2 — Listar webhooks de um cliente

#### Request

```http
GET /api/v1/customers/c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f/webhooks?page=1&pageSize=20
Authorization: Bearer <JWT>
```

A paginação é a mesma de `listOrdersQuerySchema`: `pageSize` vai de 1 a 100.

**Response 200 OK** (sem secret)

```json
{
  "data": [
    {
      "id": "5e1a9c7b-2d3f-4a6b-8c9d-0e1f2a3b4c5d",
      "customerId": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f",
      "url": "https://hooks.atlas.example/oms",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "previousSecretExpiresAt": null,
      "createdAt": "2026-11-06T13:00:00.000Z",
      "updatedAt": "2026-11-06T13:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

**Erros:** 400 `VALIDATION_ERROR` (`customerId` não é UUID), 401, 404 `NOT_FOUND` (`Customer`).

### 5.4 E3 — Editar webhook

#### Request

Todos os campos são opcionais, mas pelo menos um precisa vir. O `customerId` não pode ser alterado.

```http
PATCH /api/v1/webhooks/5e1a9c7b-2d3f-4a6b-8c9d-0e1f2a3b4c5d
Authorization: Bearer <JWT>
Content-Type: application/json

{ "events": ["PAID", "SHIPPED", "DELIVERED"], "active": false }
```

**Response 200 OK** (sem secret)

```json
{
  "id": "5e1a9c7b-2d3f-4a6b-8c9d-0e1f2a3b4c5d",
  "customerId": "c1d2e3f4-a5b6-4c7d-8e9f-0a1b2c3d4e5f",
  "url": "https://hooks.atlas.example/oms",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": false,
  "previousSecretExpiresAt": null,
  "createdAt": "2026-11-06T13:00:00.000Z",
  "updatedAt": "2026-11-06T15:20:00.000Z"
}
```

**Erros:** 400 `VALIDATION_ERROR` (mesmos `details` da E1, ou corpo vazio), 401, 404 `WEBHOOK_NOT_FOUND`.

#### Semântica

- Mudar `events` **não** afeta eventos já no outbox, por causa do snapshot e do filtro na inserção ([ADR-007](adrs/ADR-007-snapshot-e-filtro-na-insercao-do-outbox.md)).
- `active: false` **pausa** as entregas: os eventos pendentes aguardam até a reativação ([§4.2](#42-processamento-pelo-worker), item 6).

### 5.5 E4 — Remover webhook

#### Request

```http
DELETE /api/v1/webhooks/5e1a9c7b-2d3f-4a6b-8c9d-0e1f2a3b4c5d
Authorization: Bearer <JWT>
```

**Response:** `204 No Content`, sem corpo.

**Erros:** 401, 404 `WEBHOOK_NOT_FOUND`.

#### Semântica (proposta de implementação)

- Remove a configuração e, em cascata, as linhas de `webhook_outbox` e `webhook_deliveries` daquele webhook. Eventos pendentes **deixam de ser enviados**.
- As linhas de `webhook_dead_letter` ficam como evidência: `webhookId` vira `NULL` e o `payload` é preservado.
- Para uma pausa temporária, o recomendado é `PATCH { "active": false }`.

### 5.6 E5 — Rotacionar secret

#### Request

```http
POST /api/v1/webhooks/5e1a9c7b-2d3f-4a6b-8c9d-0e1f2a3b4c5d/rotate-secret
Authorization: Bearer <JWT>
```

O corpo é vazio.

#### Response 200 OK

```json
{
  "id": "5e1a9c7b-2d3f-4a6b-8c9d-0e1f2a3b4c5d",
  "secret": "a41c…(64 hex)…09fe",
  "previousSecretExpiresAt": "2026-11-07T13:00:00.000Z"
}
```

**Erros:** 401, 404 `WEBHOOK_NOT_FOUND`, 409 `WEBHOOK_SECRET_ROTATION_IN_PROGRESS` ([§4.5](#45-rotação-de-secret)).

### 5.7 E6 — Histórico de entregas

#### Request

```http
GET /api/v1/webhooks/5e1a9c7b-2d3f-4a6b-8c9d-0e1f2a3b4c5d/deliveries?page=1&pageSize=100
Authorization: Bearer <JWT>
```

- Ordenado por `createdAt desc`.
- `pageSize` vai de 1 a 100: são "os últimos 100 webhooks que vocês mandaram pra mim" ([09:34] Marcos). O padrão de 100 é **proposta**.
- Cada item traz sucesso ou falha, payload, resposta e tempo de resposta ([09:34] Marcos). O `payload` vem da linha correspondente do outbox.

#### Response 200 OK

```json
{
  "data": [
    {
      "id": "d7a1c3e5-…",
      "eventId": "8f0c2b3e-5a7d-4e8a-9a51-2f4b7c1d9e60",
      "eventType": "order.status_changed",
      "attempt": 2,
      "result": "SUCCESS",
      "payload": { "event_id": "8f0c2b3e-…", "event_type": "order.status_changed", "to_status": "SHIPPED", "…": "…" },
      "responseStatus": 200,
      "responseBody": "{\"received\":true}",
      "durationMs": 184,
      "errorCode": null,
      "createdAt": "2026-11-06T14:04:12.590Z"
    },
    {
      "id": "c2b9a0f1-…",
      "eventId": "8f0c2b3e-5a7d-4e8a-9a51-2f4b7c1d9e60",
      "eventType": "order.status_changed",
      "attempt": 1,
      "result": "FAILURE",
      "payload": { "…": "…" },
      "responseStatus": null,
      "responseBody": null,
      "durationMs": 10000,
      "errorCode": "WEBHOOK_DELIVERY_TIMEOUT",
      "createdAt": "2026-11-06T14:03:12.481Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 2, "totalPages": 1 }
}
```

**Erros:** 400 `VALIDATION_ERROR`, 401, 404 `WEBHOOK_NOT_FOUND`.

### 5.8 E7 — Replay de evento da DLQ (ADMIN)

#### Request

```http
POST /api/v1/admin/webhooks/dead-letter/9b3e7d21-4c6a-4f0e-b8d2-1a5c9e7f3b60/replay
Authorization: Bearer <JWT de usuário ADMIN>
```

O corpo é vazio. A rota usa `authenticate` e `requireRole('ADMIN')` ([09:36] Larissa).

**Response 202 Accepted (proposta de implementação).** O evento volta para a fila e será entregue no próximo ciclo do worker.

```json
{
  "deadLetterId": "9b3e7d21-4c6a-4f0e-b8d2-1a5c9e7f3b60",
  "outboxId": "61fd0a9e-…",
  "eventId": "8f0c2b3e-5a7d-4e8a-9a51-2f4b7c1d9e60",
  "status": "PENDING",
  "replayedAt": "2026-11-07T09:12:00.000Z",
  "replayedById": "a0b1c2d3-…"
}
```

#### Erros

| Status | Código | Quando |
|---|---|---|
| 401 | `UNAUTHORIZED` | — |
| 403 | `FORBIDDEN` | Usuário `OPERATOR`, via `requireRole` |
| 404 | `WEBHOOK_DEAD_LETTER_NOT_FOUND` | — |
| 409 | `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | — |
| 409 | `WEBHOOK_INACTIVE` | Webhook removido ou inativo |

## 6. Matriz de erros

Todos os códigos do módulo usam o prefixo `WEBHOOK_` ([09:28] Bruno, [09:29] Larissa). As classes ficam em `src/modules/webhooks/webhook.errors.ts` e estendem as classes de `src/shared/errors/` ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)).

A coluna **Origem** diferencia:

- os códigos **citados na reunião**: só `WEBHOOK_NOT_FOUND` e `WEBHOOK_INVALID_URL`, ambos em [09:28] Bruno;
- os **propostos** neste FDD para cobrir as regras decididas.

#### Como cada código chega ao cliente da API

- **Códigos HTTP** saem em `error.code`.
- **Códigos de validação** saem dentro de `error.details[].message` de um `VALIDATION_ERROR`. O motivo: `validate()` (`src/middlewares/validate.middleware.ts`) converte qualquer `ZodError` em `ValidationError` com código fixo, e a regra de HTTPS fica no schema Zod, como Sofia pediu ([09:23] Sofia). A resposta fica assim: `{"code":"VALIDATION_ERROR","details":[{"path":"url","message":"WEBHOOK_INVALID_URL: url must use https"}]}`.
- **Códigos do worker** não são respostas HTTP. Eles são gravados em `webhook_deliveries.errorCode` e `webhook_dead_letter.reason`, e emitidos nos logs.

| Código | HTTP | Canal | Classe / origem técnica | Gatilho | Retry? | Origem |
|---|---|---|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | `error.code` | `WebhookNotFoundError extends AppError`, porque `NotFoundError` não aceita código | `:id` inexistente em E3–E6 | — | [09:28] Bruno |
| `WEBHOOK_INVALID_URL` | 400 | `details[]` de `VALIDATION_ERROR` | `superRefine` em `webhook.schemas.ts` | URL inválida ou sem `https:` | — | [09:28] Bruno, [09:23] Sofia |
| `WEBHOOK_INVALID_EVENT_FILTER` | 400 | `details[]` de `VALIDATION_ERROR` | `superRefine` em `webhook.schemas.ts` | `events` vazio, repetido, fora de `OrderStatus` ou com `PENDING` | — | Proposto, para o filtro de [09:33] Marcos |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `error.code` | `WebhookDeadLetterNotFoundError extends AppError` | `:id` inexistente em E7 | — | Proposto, para o replay de [09:18] Diego |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | `error.code` | `ConflictError(msg, code)` | Replay repetido da mesma linha da DLQ | — | Proposto |
| `WEBHOOK_INACTIVE` | 409 | `error.code` | `ConflictError(msg, code)` | Replay para webhook inativo ou removido | — | Proposto |
| `WEBHOOK_SECRET_ROTATION_IN_PROGRESS` | 409 | `error.code` | `ConflictError(msg, code)` | Rotação durante a carência de 24h | — | Proposto, para a regra de [09:21] Sofia |
| `WEBHOOK_OUTBOX_INSERT_FAILED` | — | log | `publishWebhookEvent` loga e relança o erro original | Falha no INSERT do outbox: rollback de `changeStatus` ([09:40] Bruno). O cliente da API recebe o que o `errorMiddleware` der ao erro original, normalmente 500 `INTERNAL_SERVER_ERROR`, ou 409/404 nos casos Prisma P2002/P2025 | — | Proposto |
| `WEBHOOK_DELIVERY_TIMEOUT` | — | deliveries / log | worker | Sem resposta em 10s | Sim | Proposto, para o timeout de [09:42] Diego |
| `WEBHOOK_DELIVERY_HTTP_ERROR` | — | deliveries / log | worker | Resposta `3xx`, `4xx` ou `5xx` | Sim | Proposto |
| `WEBHOOK_DELIVERY_NETWORK_ERROR` | — | deliveries / log | worker | DNS, conexão recusada, TLS | Sim | Proposto |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | — | DLQ / log | worker | Corpo maior que 64KB | **Não** | Proposto, para o limite de [09:24] Larissa |
| `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | — | DLQ / log | worker | Falha na 5ª retentativa | — (encerra) | Proposto, para o teto de [09:17] Larissa |

**Códigos que já existem e são reutilizados:** `VALIDATION_ERROR`, `UNAUTHORIZED`, `FORBIDDEN` (de `requireRole`), `NOT_FOUND` (para `Customer`) e `INTERNAL_SERVER_ERROR`.

**`WEBHOOK_SECRET_REQUIRED` não foi adotado.** Bruno o citou como exemplo ([09:28]), mas a secret é gerada pelo servidor e a coluna é obrigatória, então o erro nunca dispararia ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)).

## 7. Estratégias de resiliência

| Aspecto | Estratégia | Origem |
|---|---|---|
| Timeout | `fetch(url, { signal: AbortSignal.timeout(10_000) })`. Estourou, conta como falha e entra no retry. | [09:42] Diego |
| Retries | 5 retentativas pela tabela fixa 1m/5m/30m/2h/12h (§4.3). Sem jitter: a reunião fixou intervalos exatos. O jitter pode ser avaliado se surgirem picos de retentativas simultâneas **(proposta)** | [09:17] Larissa |
| Backoff | `nextAttemptAt` persistido no outbox. Sobrevive a restarts do worker. | [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| Fallback | Depois da última falha, DLQ com replay manual por ADMIN. O fallback por **e-mail** fica para a próxima fase. | [09:18] Diego, [09:37] Larissa |
| Falha do cliente não afeta pedidos | O envio está fora da transação de `changeStatus`. Só o INSERT do outbox está dentro dela. | [09:04] Bruno |
| Queda do worker | Os eventos continuam `PENDING` no MySQL. Na volta, os que estavam em `PROCESSING` são reabertos (§4.2, item 9). | [09:11] Diego |
| Queda da API | Não afeta o worker, que é outro processo. | [09:11] Diego |
| Erro de banco no loop | Captura, `logger.error('webhook_worker_tick_failed')` e segue no próximo ciclo. O processo não morre por erro transitório. O `errorMiddleware` não cobre o worker. | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) |
| Duplicidade | Aceita (at-least-once). O cliente deduplica por `X-Event-Id`. | [09:24] Diego, [09:25] Diego |
| Cliente lento bloqueando os outros | Cada webhook tem sua lane, e o tick não espera as lanes (§4.2). Um cliente que leva 10s só atrasa os próprios eventos **(proposta de implementação)** | [09:02] Marcos, [09:42] Diego |
| Redirect para `http://` | `redirect: 'manual'`. Qualquer 3xx conta como falha. | [09:23] Sofia |

## 8. Observabilidade

O projeto não tem biblioteca de métricas nem de tracing, e a decisão é não trazer nada novo ([09:29] Bruno, [09:30] Larissa). A observabilidade se apoia em **logs estruturados do Pino** (`src/shared/logger/index.ts`) e em consultas às tabelas do módulo.

### 8.1 Métricas

São derivadas de campos numéricos nos logs e de consultas periódicas que o worker faz a cada ciclo.

| Métrica | Tipo | Fonte | Uso / alerta sugerido |
|---|---|---|---|
| `webhook_delivery_latency_ms` (commit → 2xx) | histograma | log `webhook_delivered` (`deliveredAt - createdAt`), só para entregas no 1º envio | p95 < 10 000 ms, a meta do PRD ([09:02] Marcos) |
| `webhook_outbox_pending_count` | gauge | `COUNT(*)` de `PENDING` com `nextAttemptAt <= now()`, logado em `webhook_worker_tick` | Crescimento contínuo indica worker parado ou lento |
| `webhook_outbox_oldest_pending_age_seconds` | gauge | `MIN(createdAt)` dos pendentes vencidos de webhooks ativos | Sugestão de alerta: acima de 10s por mais de 5 minutos **(proposta)** |
| `webhook_delivery_attempts_total{result,errorCode}` | contador | log `webhook_delivery_attempt` | Taxa de falha por cliente |
| `webhook_response_time_ms` | histograma | `durationMs` do envio | Cliente lento (perto de 10s) |
| `webhook_dead_letter_total{reason}` | contador | log `webhook_dead_lettered` | Sugestão: avisar o time a cada nova entrada **(proposta)** |
| `webhook_replay_total` | contador | log `webhook_dead_letter_replayed` | Auditoria ([09:36] Sofia) |
| `webhook_events_per_webhook_per_minute` | contador | log `webhook_delivery_attempt` agregado por `webhookId` | Insumo para decidir sobre rate limiting ([09:39] Larissa, [RFC Q1](RFC.md#5-questões-em-aberto)) |

### 8.2 Logs

Logs Pino em JSON, no mesmo padrão `logger.info({...}, 'evento')` de `src/server.ts` e de `request-logger.middleware.ts`.

| Evento | Nível | Campos |
|---|---|---|
| `webhook_event_enqueued` | info | `eventId`, `webhookId`, `orderId`, `fromStatus`, `toStatus` |
| `webhook_outbox_insert_failed` | error | `orderId`, `err`, `errorCode=WEBHOOK_OUTBOX_INSERT_FAILED` |
| `webhook_worker_started` / `webhook_worker_stopped` | info | `pollIntervalMs`, `batchSize`, `maxLanes`, `signal` |
| `webhook_worker_tick` | debug | `fetched`, `lanesStarted`, `lanesInFlight`, `pendingCount`, `oldestPendingAgeSeconds` |
| `webhook_delivery_attempt` | info / warn | `eventId`, `webhookId`, `orderId`, `attempt`, `result`, `responseStatus`, `durationMs`, `errorCode` |
| `webhook_delivered` | info | `eventId`, `webhookId`, `attempts`, `latencyMs` |
| `webhook_retry_scheduled` | warn | `eventId`, `attempt`, `nextAttemptAt` |
| `webhook_dead_lettered` | warn | `eventId`, `webhookId`, `reason`, `attempts` |
| `webhook_dead_letter_replayed` | info | `deadLetterId`, `eventId`, `userId`, `userEmail` ([09:36] Sofia) |
| `webhook_secret_rotated` | info | `webhookId`, `previousSecretExpiresAt`, `userId` |
| `webhook_worker_tick_failed` | error | `err` |

**Proteção de segredos (proposta derivada).** Acrescentar `'*.secret'`, `'*.previousSecret'` e `'*.signature'` em `redactPaths` (`src/shared/logger/index.ts`), que hoje não os cobre. Nunca logar o objeto `webhook` inteiro nem os headers de saída. A motivação é o histórico de vazamento de secret via log ([09:22] Diego).

### 8.3 Tracing

Não há stack de tracing distribuído. Adotar OpenTelemetry contraria o reuso decidido no [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md). A rastreabilidade de ponta a ponta usa **ids de correlação** presentes em todos os logs e tabelas:

```text
PATCH /orders/:id/status (requestId via X-Request-Id, request-logger)
  └─ orderId ──► webhook_event_enqueued {eventId, orderId}
                   └─ eventId ──► webhook_delivery_attempt × N ──► webhook_deliveries.eventId
                                   └─ X-Event-Id no cliente (o cliente correlaciona do lado dele)
                                   └─ webhook_dead_letter.eventId ──► webhook_dead_letter_replayed
```

- O `eventId` é o **trace id** do evento. Ele é o mesmo no outbox, nas entregas, na DLQ, nos logs e no header recebido pelo cliente. Com isso, um ticket de suporte do tipo "não recebi o evento X" se resolve com uma busca por `eventId` nos logs.
- O `orderId` liga o evento ao `http_request` que já é logado pelo `requestLogger`, pelo caminho `/orders/:id/status` e pelo horário.

## 9. Modelo de dados (migration Prisma)

Este é o modelo de dados proposto. Ele segue as convenções de `prisma/schema.prisma`: ids `uuid()` em `Char(36)` ([09:51] Larissa), colunas em camelCase e tabelas em snake_case via `@@map`.

```prisma
enum WebhookOutboxStatus {
  PENDING
  PROCESSING
  DELIVERED
  FAILED
}

enum WebhookDeliveryResult {
  SUCCESS
  FAILURE
}

model Webhook {
  id                      String    @id @default(uuid()) @db.Char(36)
  customerId              String    @db.Char(36)
  url                     String    @db.VarChar(2048)
  secret                  String    @db.VarChar(128)
  previousSecret          String?   @db.VarChar(128)
  previousSecretExpiresAt DateTime?
  events                  Json      // OrderStatus[] — filtro ([09:33] Marcos)
  active                  Boolean   @default(true)
  createdAt               DateTime  @default(now())
  updatedAt               DateTime  @updatedAt

  customer    Customer            @relation(fields: [customerId], references: [id])
  outbox      WebhookOutbox[]
  deliveries  WebhookDelivery[]
  deadLetters WebhookDeadLetter[]

  @@index([customerId, active])
  @@map("webhooks")
}

model WebhookOutbox {
  id            String              @id @default(uuid()) @db.Char(36)
  eventId       String              @unique @db.Char(36)
  eventType     String              @db.VarChar(64)
  webhookId     String              @db.Char(36)
  orderId       String              @db.Char(36)   // sem FK: pedidos CANCELLED podem ser apagados
  payload       Json                               // snapshot ([09:52] Larissa)
  status        WebhookOutboxStatus @default(PENDING)
  attempts      Int                 @default(0)
  nextAttemptAt DateTime            @default(now())
  lastErrorCode String?             @db.VarChar(64)
  deliveredAt   DateTime?
  createdAt     DateTime            @default(now())
  updatedAt     DateTime            @updatedAt

  webhook    Webhook           @relation(fields: [webhookId], references: [id], onDelete: Cascade)
  deliveries WebhookDelivery[]

  @@index([status, nextAttemptAt])
  @@index([createdAt])
  @@index([orderId])
  @@map("webhook_outbox")
}

model WebhookDelivery {
  id             String                @id @default(uuid()) @db.Char(36)
  outboxId       String                @db.Char(36)
  webhookId      String                @db.Char(36)
  eventId        String                @db.Char(36)
  attempt        Int
  result         WebhookDeliveryResult
  responseStatus Int?
  responseBody   String?               @db.Text     // truncado em 4 KB (proposta)
  durationMs     Int
  errorCode      String?               @db.VarChar(64)
  createdAt      DateTime              @default(now())

  outbox  WebhookOutbox @relation(fields: [outboxId], references: [id], onDelete: Cascade)
  webhook Webhook       @relation(fields: [webhookId], references: [id], onDelete: Cascade)

  @@index([webhookId, createdAt])
  @@index([eventId])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {
  id           String    @id @default(uuid()) @db.Char(36)
  outboxId     String    @db.Char(36)   // sem FK: a evidência sobrevive à remoção do webhook
  webhookId    String?   @db.Char(36)
  eventId      String    @db.Char(36)
  payload      Json
  reason       String    @db.VarChar(64)     // código WEBHOOK_* ([09:18] Diego: "motivo da falha")
  lastError    String?   @db.VarChar(1000)
  attempts     Int
  failedAt     DateTime  @default(now())
  replayedAt   DateTime?
  replayedById String?   @db.Char(36)

  webhook Webhook? @relation(fields: [webhookId], references: [id], onDelete: SetNull)

  @@index([failedAt])
  @@index([eventId])
  @@map("webhook_dead_letter")
}
```

Mudanças nos modelos existentes:

- `Customer` ganha só o campo de relação reversa `webhooks Webhook[]`, que o Prisma exige. **Nenhuma coluna nova** em `customers`.
- O índice `[status, nextAttemptAt]` atende a consulta do worker. O `[createdAt]` atende a ordenação e o futuro arquivamento ([09:08] Diego).

## 10. Integração com o sistema existente

| # | Arquivo existente | Como o módulo de webhooks se integra |
|---|---|---|
| 1 | `src/modules/orders/order.service.ts` | Em `changeStatus`, depois de `tx.orderStatusHistory.create(...)`, guardar o histórico criado e chamar `await publishWebhookEvent(tx, order, from, to, history.changedAt)`. O `tx` é o `Prisma.TransactionClient` (alias `TxClient`, que já existe no arquivo). O `order` já foi carregado no início da transação, com `orderNumber`, `customerId` e `totalCents`. O `OrderService` **não** recebe repository novo no construtor ([09:41] Bruno, [09:41] Diego). `create()` **não** muda ([RFC Q8](RFC.md#5-questões-em-aberto)). |
| 2 | `src/modules/orders/order.status.ts` | Fonte para o filtro de eventos válido. Como nenhuma transição leva a `PENDING`, o schema de `events` aceita `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED` e `CANCELLED`. Nada muda no arquivo. |
| 3 | `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | `webhook.errors.ts` cria `WebhookNotFoundError` e `WebhookDeadLetterNotFoundError` estendendo `AppError` (404, código `WEBHOOK_*`), porque `NotFoundError` fixa `NOT_FOUND`. Os conflitos usam `ConflictError(message, 'WEBHOOK_…')`, como `InvalidStatusTransitionError` já faz ([09:28] Bruno). |
| 4 | `src/middlewares/error.middleware.ts` | **Sem alteração.** Os erros novos são `AppError` e saem no formato padrão. Os `ZodError` já viram `VALIDATION_ERROR` ([09:29] Bruno). |
| 5 | `src/middlewares/validate.middleware.ts` | `webhook.routes.ts` usa `validate({ body, params, query })`. As regras de URL e de eventos usam `superRefine` com mensagens prefixadas por `WEBHOOK_*`, que aparecem em `details` (§6). |
| 6 | `src/middlewares/auth.middleware.ts` | E1–E6 usam `router.use(authenticate)`, como `order.routes.ts`. A E7 acrescenta `requireRole('ADMIN')`, como `user.routes.ts` ([09:36] Larissa). O `req.user` (`id`, `email`) alimenta `replayedById` e o log de auditoria. |
| 7 | `src/routes/index.ts` | Acrescentar `webhooks` e `adminWebhooks` ao tipo `Controllers`. Montar `router.use('/customers/:customerId/webhooks', buildCustomerWebhookRouter(...))` **antes** de `router.use('/customers', ...)`, com `Router({ mergeParams: true })` para ler `:customerId`. Montar também `router.use('/webhooks', buildWebhookRouter(...))` e `router.use('/admin/webhooks', buildAdminWebhookRouter(...))`. |
| 8 | `src/app.ts` | Em `buildControllers`, instanciar `WebhookRepository(prisma)`, `WebhookService`, `WebhookController` e `WebhookAdminController`, no mesmo padrão dos outros módulos, e devolver os dois controllers no objeto `Controllers`. |
| 9 | `src/server.ts` → **novo** `src/worker.ts` | O `src/worker.ts` replica o bootstrap de `server.ts`: importa `env` e `prisma`, chama `startWebhookWorker({ prisma, logger })` de `src/modules/webhooks/webhook.worker.ts` ([09:28] Bruno) e trata `SIGINT`/`SIGTERM` com `$disconnect()`. Não sobe Express. |
| 10 | `src/config/database.ts` | O worker importa o `prisma` exportado. O módulo cria o singleton ao ser carregado, então cada processo (API e worker) já tem a **própria** instância, com a mesma `DATABASE_URL` ([09:30] Bruno). Não há código novo de conexão. |
| 11 | `src/config/env.ts` | Reutilizado sem mudanças. Atenção: o schema exige `JWT_SECRET`, então o worker precisa do mesmo `.env` da API (`--env-file=.env`). Os parâmetros (2s, lote de 50, 20 lanes, 10s, 64KB, backoff) ficam em `webhook.constants.ts`, e não em variáveis de ambiente, nesta fase **(proposta de implementação)**. |
| 12 | `src/shared/logger/index.ts` | O worker usa `logger.child({ component: 'webhook-worker' })`. Acrescentar `'*.secret'`, `'*.previousSecret'` e `'*.signature'` em `redactPaths` ([09:22] Diego). |
| 13 | `src/shared/http/response.ts` | `paginated()` nas respostas de E2 e E6. |
| 14 | `prisma/schema.prisma` + `prisma/migrations/` | Modelos da §9, relação reversa em `Customer` e uma migration nova, gerada com `npm run db:migrate`. |
| 15 | `package.json` | Scripts novos: `"worker": "tsx --env-file=.env src/worker.ts"`, espelhando `dev`, e `"start:worker": "node --env-file=.env dist/worker.js"`, espelhando `start` ([09:11] Larissa). **Nenhuma dependência nova**: `uuid` já está lá, e `node:crypto` e `fetch` vêm do Node ≥ 20 (`engines`). |
| 16 | `tests/setup.ts` e `tests/helpers/factories.ts` | Em `beforeEach`, acrescentar `deleteMany` de `webhookDelivery`, `webhookDeadLetter`, `webhookOutbox` e `webhook` **antes** de `customer.deleteMany()`, por causa da FK. Nova factory `createTestWebhook()`. |

Estrutura final do módulo ([09:27] Bruno, [09:28] Bruno):

```text
src/modules/webhooks/
├── webhook.controller.ts     # E1–E6 (E2 via rota aninhada em customers)
├── webhook.admin.controller.ts  # E7
├── webhook.routes.ts         # /webhooks e /admin/webhooks
├── webhook.schemas.ts        # Zod (https, events)
├── webhook.service.ts        # CRUD, rotação, replay
├── webhook.repository.ts
├── webhook.publisher.ts      # publishWebhookEvent(tx, …)
├── webhook.worker.ts         # tick, lanes por webhook, retry, DLQ
├── webhook.signer.ts         # HMAC-SHA256, geração de secret
├── webhook.errors.ts         # classes WEBHOOK_*
└── webhook.constants.ts      # 2s, 50, 20 lanes, 10s, 64KB, BACKOFF
src/worker.ts                 # entry-point do worker
```

## 11. Dependências e compatibilidade

| Item | Situação |
|---|---|
| Node.js ≥ 20 (`package.json` → `engines`) | Fornece `fetch`, `AbortSignal.timeout` e `node:crypto` (`createHmac`, `randomBytes`, `timingSafeEqual`) |
| `@prisma/client` / `prisma` 5.22 com MySQL 8 (`docker-compose.yml`) | Colunas `Json` para `events` e `payload`. Transações interativas, já usadas em `changeStatus` |
| `uuid` 11 | Já é dependência (usado em `request-logger.middleware.ts`). Gera o `eventId` |
| `zod` 3.23, `pino` 9.5, `express` 4.21 | Já existem. Sem mudança de versão |
| **Dependências novas** | **Nenhuma** ([09:29] Bruno) |

#### Compatibilidade

- **Retrocompatível.** As rotas que já existem não mudam de contrato. `PATCH /orders/:id/status` responde igual, só que agora pode falhar com 500 se o outbox falhar, o que é desejado ([09:40] Bruno).
- **Migration aditiva.** Só tabelas novas e uma relação reversa em `Customer`. O rollback é dropar as tabelas novas.
- **Ordem de deploy:** (1) migration, (2) API, que já passa a gravar no outbox, (3) worker. Se a API subir antes do worker, os eventos ficam acumulados e são entregues quando o worker subir.
- **Dependência externa:** clientes com endpoint HTTPS válido. O portal do desenvolvedor documenta o HMAC e a deduplicação ([09:26] Marcos, [09:40] Marcos).
- **Dependência de processo:** a revisão de segurança da Sofia, com pelo menos 2 dias úteis antes do deploy, focada em HMAC e geração de secret ([09:46] Sofia).

## 12. Critérios de aceite técnicos

| # | Critério | Como verificar |
|---|---|---|
| CA-01 | Uma mudança de status com webhook assinante cria exatamente 1 linha `PENDING` por webhook assinante, na mesma transação | Teste de integração (supertest + Prisma) |
| CA-02 | Uma mudança de status sem webhook assinante (ou só com webhooks inativos) não cria linha no outbox | Teste de integração |
| CA-03 | Uma falha forçada no INSERT do outbox faz rollback completo: status, histórico e estoque inalterados, resposta 500 | Teste com mock de `tx.webhookOutbox.create` |
| CA-04 | O payload contém exatamente os campos da §5.1 e não contém `items` | Teste unitário de `buildPayload` |
| CA-05 | Um evento pendente de um webhook saudável é enviado no próximo tick (≤ 2s), com os 5 headers da §5.1, **mesmo havendo outro webhook que responde em 10s** | Teste do worker contra um servidor HTTP local |
| CA-06 | O `X-Signature` confere com `HMAC-SHA256(secret, corpo bruto)`. Durante a carência vêm as duas assinaturas | Teste unitário de `webhook.signer.ts` |
| CA-07 | Timeout de 10s, não-2xx ou 3xx agendam retry com 1m/5m/30m/2h/12h. A falha na 5ª retentativa cria uma linha em `webhook_dead_letter` com `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | Teste com relógio controlado |
| CA-08 | Payload acima de 64KB vai para a DLQ sem nenhum envio HTTP. Eventos de webhook inativo não são enviados e voltam a fluir quando ele é reativado | Teste do worker |
| CA-09 | O replay como ADMIN retorna 202, reabre o mesmo `eventId` e loga `userId`. Como OPERATOR retorna 403. O segundo replay retorna 409 | Teste de integração |
| CA-10 | Uma URL `http://` retorna 400 com `WEBHOOK_INVALID_URL` nos `details`. `events: ["PENDING"]` retorna 400 com `WEBHOOK_INVALID_EVENT_FILTER` | Teste de integração |
| CA-11 | A secret aparece só nas respostas de E1 e E5. E2 e E3 nunca a retornam. Nenhum log contém a secret | Teste de integração + inspeção de log |
| CA-12 | A rotação mantém a secret anterior válida por 24h. Uma nova rotação na janela retorna 409 | Teste de integração |
| CA-13 | E6 retorna as entregas paginadas, em ordem decrescente, com `pageSize` até 100 | Teste de integração |
| CA-14 | Ao reiniciar, o worker reabre eventos em `PROCESSING`. `SIGTERM` encerra sem perder evento. Um webhook nunca tem duas lanes ao mesmo tempo | Teste do worker |
| CA-15 | `npm run lint`, `npm test` e a migration passam. Nenhuma dependência nova em `package.json` | CI |
| CA-16 | A revisão de segurança da Sofia está aprovada antes do deploy | Checklist de release ([09:46] Sofia) |

## 13. Riscos e mitigação

São os riscos de implementação. Os riscos de produto, com probabilidade e impacto, estão no [PRD §10](PRD.md#10-riscos-e-mitigação).

| Risco técnico | Efeito | Mitigação |
|---|---|---|
| Muitos clientes lentos ao mesmo tempo esgotam as 20 lanes | Atraso para os demais | Limite de lanes configurável, métrica `lanesInFlight`, alerta de idade do evento pendente mais antigo |
| Evento posterior do mesmo pedido ultrapassa um em retry | O cliente vê a sequência fora de ordem | O contrato documenta a ordenação por `timestamp` e `to_status`. Decisão pendente em RFC Q6 |
| Transação de `changeStatus` mais longa | Mais contenção de locks em pedidos | Uma consulta indexada (`[customerId, active]`) e INSERTs simples. Nenhuma chamada externa dentro da transação |
| Eventos presos em `PROCESSING` depois de um crash | Atraso | Reabertura no início do worker (§4.2, item 9) |
| Secret vazando em log | Comprometimento do endpoint | `redactPaths`, sem log de headers, secret só em E1/E5 ([09:22] Diego) |
| O formato com duas assinaturas não ser aceito pela segurança | Retrabalho no signer | Isolado em `webhook.signer.ts`. Validar na revisão da Sofia (RFC Q7) |
| Crescimento de `webhook_outbox` e `webhook_deliveries` | Consultas mais lentas | Índices. O arquivamento entra numa fase futura ([09:08] Diego) |
| CRUD aberto a qualquer usuário autenticado, para qualquer `customerId` | Configuração indevida de webhooks de outro cliente | Aceito nesta fase ([09:37] Sofia). Endurecer depois (RFC Q5) |
