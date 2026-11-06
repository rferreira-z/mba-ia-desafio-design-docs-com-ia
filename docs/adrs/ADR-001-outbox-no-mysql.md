# ADR-001: Padrão Outbox no MySQL existente para eventos de webhook

- **Data da decisão:** reunião técnica de webhooks ([09:08] Larissa)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-007](ADR-007-snapshot-e-filtro-na-insercao-do-outbox.md)

## Status

Aceito

## Contexto

Três clientes B2B querem ser notificados quando o status dos seus pedidos muda ([09:00] Marcos). A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de um `prisma.$transaction` que:

- atualiza `orders.status`;
- insere uma linha em `order_status_history`;
- conforme a transição, debita estoque (só `PENDING → PAID`) ou devolve estoque (`CANCELLED` vindo de `PAID` ou `PROCESSING`). As regras estão em `shouldDebitStock` e `shouldReplenishStock`, em `src/modules/orders/order.status.ts`.

A aplicação não tem hoje nenhum mecanismo de eventos, fila ou notificação externa. Precisamos de um jeito de emitir o evento que:

1. não deixe a mudança de status dependente da disponibilidade do cliente ([09:04] Bruno);
2. não perca eventos nem gere eventos para mudanças que foram desfeitas ([09:06] Diego).

## Decisão

Adotar o **padrão Transactional Outbox no MySQL já existente** ([09:06] Diego, [09:08] Larissa):

- Dentro da **mesma transação** de `changeStatus`, inserir uma linha por evento na tabela `webhook_outbox` ([09:06] Diego, [09:40] Bruno).
- Se a inserção no outbox falhar, a transação inteira faz rollback. Não pode existir status alterado sem evento ([09:40] Bruno, [09:41] Diego).
- A tabela tem índice por status (`PENDING`, `PROCESSING`, `FAILED`, `DELIVERED`) e por `created_at` ([09:08] Diego).
- Um processo separado consome a tabela e faz as chamadas HTTP ([ADR-002](ADR-002-worker-separado-em-polling.md)).
- Os ids seguem o padrão UUID do projeto ([09:51] Larissa).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Disparo HTTP síncrono dentro de `changeStatus`** | A transação já é pesada. Um cliente lento travaria a mudança de status de outros pedidos ([09:04] Bruno). Se o cliente estiver fora do ar, não faz sentido dar rollback na mudança de status ([09:04] Bruno). "Síncrono está fora de questão" ([09:06] Diego). |
| **Redis Streams (ou broker equivalente)** | Exige subir e operar infraestrutura nova ([09:07] Larissa). Para um time pequeno, subir um Redis Cluster para isso é overengineering ([09:07] Diego). |

## Consequências

### Positivas

- O evento tem atomicidade com a mudança de status. Commit grava os dois, rollback descarta os dois. Não há estado inconsistente ([09:06] Diego).
- Nenhuma infraestrutura nova: usa o MySQL e o Prisma que já existem (`prisma/schema.prisma`, `src/config/database.ts`).
- A falha ou lentidão do cliente fica isolada do fluxo transacional de pedidos.

### Negativas

- A transação de `changeStatus` ganha mais escritas: uma consulta de webhooks ativos do cliente e N inserts no outbox.
- A tabela cresce sem parar. O arquivamento de linhas entregues (citado como "depois de 30 dias ou assim") ficou **fora do escopo** desta feature ([09:08] Diego).
- A latência mínima de entrega passa a depender do intervalo de polling do worker ([ADR-002](ADR-002-worker-separado-em-polling.md)).

**Trade-off explícito:** aceitamos o custo de uma transação um pouco mais pesada e de uma tabela que cresce, em troca de consistência forte entre status e evento sem operar infraestrutura nova.
