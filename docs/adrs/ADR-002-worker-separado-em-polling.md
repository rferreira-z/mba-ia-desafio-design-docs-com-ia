# ADR-002: Worker de entrega em processo separado, com polling de 2 segundos e instância única

- **Data da decisão:** reunião técnica de webhooks ([09:10] Larissa, [09:13] Larissa)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Marcos (PM)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md)

## Status

Aceito

## Contexto

Com o outbox ([ADR-001](ADR-001-outbox-no-mysql.md)), alguém precisa ler a tabela `webhook_outbox` e fazer as chamadas HTTP. Para os clientes, "tempo real" é qualquer coisa abaixo de 10 segundos ([09:02] Marcos).

Hoje o único entry-point do projeto é `src/server.ts`. Ele sobe o Express e trata `SIGINT`/`SIGTERM` com `prisma.$disconnect()`. O MySQL não tem mecanismo nativo de notificação de processos externos, como o `LISTEN/NOTIFY` do Postgres ([09:09] Diego).

## Decisão

1. **Polling em loop a cada 2 segundos.** O worker busca os eventos pendentes mais antigos em lotes pequenos, processa e marca o resultado ([09:08] Diego, [09:09] Diego, [09:10] Larissa). O pior caso de espera do polling é de 2s, e isso foi aceito ([09:10] Larissa, [09:10] Marcos).
2. **Processo separado da API.** O worker não roda na mesma instância da API, para que um restart da API não derrube a entrega ([09:11] Diego). Ele terá entry-point próprio, `src/worker.ts`, e o script `npm run worker` ([09:11] Larissa). A lógica fica no módulo, em `src/modules/webhooks/webhook.worker.ts` ([09:28] Bruno).
3. **Mesmo banco e mesma stack, outra instância de `PrismaClient`.** É um processo Node distinto, com a mesma `DATABASE_URL` ([09:11] Diego, [09:30] Bruno). O worker importa o `prisma` exportado por `src/config/database.ts`. Como o módulo instancia o singleton ao ser carregado, cada processo já ganha a própria instância, sem código novo.
4. **Uma única instância do worker nesta fase.** Ela processa em ordem de `created_at` do outbox. Por isso a ordem só é preservada **por `order_id` e enquanto houver um único worker**. Isso é uma **limitação conhecida**, não uma garantia de ordem global ([09:12] Diego, [09:13] Larissa). Os clientes nunca pediram ordem global ([09:14] Marcos).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Trigger no MySQL para reagir à inserção** | A trigger só executa SQL, não notifica processo externo. Avisar o worker exigiria improvisos, como escrever em arquivo ou chamar um endpoint ([09:09] Diego, em resposta a [09:09] Bruno). O polling de 2s já atende o requisito de menos de 10s. |
| **Worker dentro do processo da API** | Se a API reinicia, o worker morre junto ([09:11] Diego). |
| **Vários workers em paralelo** | Perde a garantia de ordem por pedido ([09:12] Diego). Ficou para o futuro, com particionamento por `order_id` ou lock pessimista ([09:13] Diego). |

## Consequências

### Positivas

- Simples de implementar e operar: um loop, sem dependências novas.
- A espera introduzida pelo polling é de no máximo 2s ([09:10] Larissa). O resto da latência é o tempo de resposta do próprio cliente, limitado pelo timeout de 10s ([09:42] Diego). A meta de menos de 10s ([09:02] Marcos) vale para clientes que respondem rápido. Por isso a instância única não pode deixar um cliente lento atrasar os outros. O FDD resolve isso com **filas lógicas por webhook** dentro do mesmo processo: sequencial por webhook, concorrente entre webhooks ([FDD §4.2](../FDD.md#42-processamento-pelo-worker)).
- O ciclo de vida do worker é independente do da API.

### Negativas

- Polling gera consultas constantes ao banco mesmo sem eventos. É mitigado pelos índices em status e `created_at` ([09:08] Diego).
- Uma única instância é um ponto único de falha da entrega. Se o worker cai, os eventos se acumulam (sem se perder) até ele voltar.
- **Limitação de ordem:** com retry com backoff ([ADR-003](ADR-003-retry-com-backoff-e-dlq.md)), um evento posterior do mesmo pedido pode ser entregue antes de outro que está esperando retentativa. A reunião não tratou desse caso. Ele está registrado como questão em aberto no [RFC](../RFC.md#5-questões-em-aberto).
- Escalar horizontalmente vai exigir um novo ADR (particionamento ou lock).

**Trade-off explícito:** trocamos reatividade e escala horizontal por simplicidade operacional. A espera do polling (≤2s) cabe na meta de 10s, e a ordem por pedido vale apenas enquanto houver um worker só.
