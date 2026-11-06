# ADR-003: Retry com backoff exponencial (1m/5m/30m/2h/12h) e DLQ em tabela separada com replay manual

- **Data da decisão:** reunião técnica de webhooks ([09:17] Larissa, [09:19] Larissa, [09:36] Larissa)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Marcos (PM), Sofia (Segurança)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)

## Status

Aceito

## Contexto

O endpoint do cliente pode estar fora do ar, lento ou responder com erro. Já houve cliente com duas horas de indisponibilidade em uma manutenção planejada ([09:16] Diego). Precisamos de uma política que:

- cubra indisponibilidades de horas;
- não deixe eventos pendurados para sempre;
- dê visibilidade e uma forma de reprocessar o que falhou de vez.

## Decisão

1. **Backoff exponencial com teto.** Depois da primeira entrega falha, o worker faz **5 retentativas** espaçadas por **1 min, 5 min, 30 min, 2 h e 12 h** ([09:15] Diego, [09:17] Diego, [09:17] Larissa).
   - São cerca de 15 horas entre a primeira falha e a última tentativa ([09:17] Diego), e isso foi aceito pelo produto ([09:17] Marcos).
   - Contagem: "5 tentativas" ([09:17] Larissa, [09:48] Larissa) são **5 retentativas depois do envio original**. A leitura é sustentada pela própria discussão. Ao recusar a proposta de 3, Diego diz que "retentaria três vezes em 30 minutos" ([09:16] Diego), o que corresponde a 1m + 5m + 30m, ou seja, "tentativa" = retentativa. Os 5 intervalos somam "quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego): exatamente 14h36min.
   - A reunião chamou o backoff de "exponencial". Na prática, a progressão combinada (×5, ×6, ×4, ×6) é uma **tabela fixa crescente**, não uma fórmula. A implementação usa a tabela literal, sem multiplicador nem jitter.
2. **Timeout é falha.** Se o cliente não responde em 10s, a tentativa conta como falha e entra no retry ([09:42] Diego).
3. **DLQ em tabela separada, `webhook_dead_letter`.** Esgotadas as retentativas, o evento vira falha permanente. Ele vai para a DLQ com o payload, o motivo da falha e o timestamp ([09:18] Diego).
4. **Replay manual** pelo endpoint `POST /admin/webhooks/dead-letter/:id/replay` ([09:18] Diego, [09:35] Diego):
   - devolve o evento ao outbox como pendente ([09:18] Diego);
   - exige o papel `ADMIN`, usando o `requireRole` que já existe em `src/middlewares/auth.middleware.ts` ([09:36] Sofia, [09:36] Larissa);
   - registra em log quem fez o replay, para auditoria ([09:36] Sofia).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Retry indefinido com backoff** | O evento fica pendurado para sempre se o cliente sumiu ([09:15] Diego). |
| **3 tentativas** (proposta de [09:16] Bruno) | É pouco. Três retentativas em cerca de 30 minutos não cobrem uma indisponibilidade de manhã nem uma manutenção de 2h ([09:16] Diego). |
| **Marcar como `failed` na própria tabela de outbox, sem tabela de DLQ** | A tabela separada deixa a leitura do outbox mais limpa e serve de evidência para debug e reprocessamento ([09:18] Diego). |

## Consequências

### Positivas

- Cobre indisponibilidades de até cerca de 14h36min sem intervenção humana.
- Falhas permanentes ficam isoladas, auditáveis e reprocessáveis.
- O replay é restrito a `ADMIN`, alinhado com a visão de segurança de que mexer em fila de entrega não é coisa de operador ([09:36] Sofia).

### Negativas

- Um evento pode chegar até cerca de 14h36min atrasado. Para eventos de status antigos, o cliente precisa tolerar a defasagem e consultar `GET /orders/:id` se quiser o estado atual.
- Retentativas aumentam a chance de entrega duplicada, o que reforça o [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md).
- Não há aviso proativo ao cliente quando o webhook dele está falhando. O e-mail de alerta foi **adiado para a próxima fase** ([09:37] Larissa).
- O replay é manual. Cada falha permanente exige ação de um `ADMIN`.

**Trade-off explícito:** um limite fixo de cerca de 14h36min troca uma garantia de entrega eventual "para sempre" por previsibilidade operacional (nada fica pendurado). O custo é intervenção manual nos casos que vão para a DLQ.
