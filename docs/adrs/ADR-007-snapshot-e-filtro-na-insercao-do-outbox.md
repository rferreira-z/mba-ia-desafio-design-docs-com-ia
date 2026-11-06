# ADR-007: Payload como snapshot e filtro de eventos aplicados na inserção do outbox

- **Data da decisão:** reunião técnica de webhooks ([09:34] Bruno, [09:52] Bruno)
- **Decisores:**
  - filtro: Marcos (PM), Bruno (Pedidos), Diego (Plataforma);
  - snapshot: Larissa (Tech Lead), Diego e Bruno, decidido depois que Marcos e Sofia saíram da call ([09:50]).
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)

## Status

Aceito

## Contexto

Cada endpoint de webhook escolhe quais status do pedido quer receber. Exemplo: "só quero saber quando vira SHIPPED e DELIVERED" ([09:33] Marcos). Isso levanta duas perguntas de modelagem do outbox ([ADR-001](ADR-001-outbox-no-mysql.md)):

1. **Onde filtrar:** na inserção do evento no outbox ou na hora do envio ([09:34] Diego)?
2. **O que guardar:** o payload já montado ou só o `order_id`, montando o payload na hora do envio ([09:51] Bruno)?

Entre a mudança de status e o envio podem passar até cerca de 14h36min, por causa do retry ([ADR-003](ADR-003-retry-com-backoff-e-dlq.md)). Nesse intervalo o pedido pode mudar de novo.

## Decisão

1. **Filtrar na inserção.** Dentro da transação de `changeStatus`, só se gravam eventos para os webhooks **ativos** do cliente que assinam o status de destino. Se nenhum assina, nada é inserido ([09:34] Bruno, [09:34] Diego).
2. **Guardar o payload como snapshot no momento da inserção.** O payload reflete o pedido no instante da mudança de status, mesmo que o pedido mude depois ([09:52] Larissa, [09:52] Diego, [09:52] Bruno).
3. **Payload enxuto**, em JSON, com:
   - `event_id`;
   - `event_type` (`"order.status_changed"`);
   - `timestamp` em ISO 8601;
   - `order_id`, `order_number`, `from_status`, `to_status` e `customer_id`;
   - campos básicos do pedido, como `total_cents`.

   Os **itens não vão** no payload, para não inflá-lo. Quem quiser detalhes consulta `GET /orders/:id` ([09:43] Diego, [09:44] Bruno).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Filtrar na hora do envio** | Grava linhas que nunca serão enviadas. Filtrar na inserção "economiza linha na tabela" ([09:34] Bruno). |
| **Guardar só `order_id` e montar o payload no envio** | Durante um retry, o evento passaria a refletir o estado *atual* do pedido, não o da mudança. Larissa chamou isso de "caso esquisito" ([09:52] Larissa). |
| **Payload completo, com itens** | Infla o payload sem necessidade. O detalhe está disponível em `GET /orders/:id` ([09:43] Diego). |

## Consequências

### Positivas

- O evento é fiel ao momento da transição. Isso vale também para eventos que vêm de retry ou de replay.
- O outbox só recebe o que vai ser enviado.
- O worker não consulta pedidos. Ele só lê o outbox e a configuração do webhook.
- Um payload enxuto fica muito longe do limite de 64KB ([09:24] Diego).

### Negativas

- Mudanças feitas na assinatura de eventos **depois** da inserção não afetam eventos já enfileirados. Por exemplo, um cliente que remove `SHIPPED` do filtro ainda recebe os eventos `SHIPPED` pendentes.
- A transação de `changeStatus` passa a ler a configuração de webhooks do cliente. É uma consulta a mais no caminho crítico.
- O cliente que precisar de mais dados faz uma chamada extra a `GET /orders/:id`.

**Trade-off explícito:** pagamos um pouco de custo na escrita (filtrar e montar o snapshot dentro da transação) para ter um evento imutável e fiel ao momento da mudança, e uma tabela de outbox menor.
