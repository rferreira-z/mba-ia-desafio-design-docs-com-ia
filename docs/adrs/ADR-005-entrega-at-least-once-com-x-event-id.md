# ADR-005: Garantia de entrega at-least-once, com deduplicação no cliente via X-Event-Id

- **Data da decisão:** reunião técnica de webhooks ([09:26] Larissa)
- **Decisores:** Diego (Plataforma, proposta), Larissa (Tech Lead, decisão em [09:26]), Marcos (PM, documentação no portal)
- **Consultada:** Sofia (Segurança). Registrou a objeção de que a responsabilidade vai para o cliente ([09:25] Sofia)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md)

## Status

Aceito

## Contexto

Com outbox e retry ([ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md)), existem cenários em que o mesmo evento é enviado mais de uma vez. Exemplos:

- o cliente processa a requisição, mas a resposta não chega antes do timeout de 10s ([09:42] Diego);
- o worker cai depois de enviar e antes de marcar o evento como entregue;
- um `ADMIN` faz o replay de um evento da DLQ.

É preciso decidir que garantia oferecer e como o cliente lida com duplicatas.

## Decisão

1. A plataforma garante entrega **at-least-once**. O cliente pode receber o mesmo evento mais de uma vez e precisa estar preparado para isso ([09:24] Diego).
2. Todo evento leva um **`X-Event-Id`**: um UUID gerado quando o evento entra no outbox e único por evento. O mesmo valor aparece no campo `event_id` do payload ([09:25] Diego, [09:43] Diego). O cliente deduplica por esse id ([09:25] Diego).
3. O `X-Event-Id` **não muda** entre retentativas nem no replay da DLQ. Essa propriedade é o que torna a deduplicação possível.
4. **A unidade de evento é (mudança de status × webhook assinante).** Isto é uma **interpretação proposta**: a reunião não tratou do caso de vários webhooks do mesmo cliente. Pelo [ADR-007](ADR-007-snapshot-e-filtro-na-insercao-do-outbox.md), cada webhook que assina o status ganha a própria linha no outbox, com o próprio ciclo de retry e o próprio UUID. A reunião disse só que o id é "único por evento" ([09:25] Diego). Como retry e DLQ são por endpoint, o id também é. O `X-Webhook-Id` ([09:44] Sofia) diz ao cliente qual cadastro recebeu aquele envio.
5. O produto documenta essa responsabilidade **em destaque** no portal do desenvolvedor ([09:26] Marcos).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Exactly-once** | Exigiria coordenação dos dois lados e seria muito mais complexo ([09:25] Diego). |
| **At-most-once (sem retry)** | Incompatível com a política de retry decidida no [ADR-003](ADR-003-retry-com-backoff-e-dlq.md). Clientes com indisponibilidade perderiam eventos. É uma alternativa plausível, mas não foi defendida na reunião. |

## Consequências

### Positivas

- O modelo é o padrão de mercado, o mesmo de Stripe e GitHub ([09:25] Diego), e familiar para integradores.
- Resolve "99% dos casos" com uma implementação simples ([09:25] Diego).
- Combina naturalmente com o outbox: o UUID nasce na inserção transacional.

### Negativas

- Joga a responsabilidade da deduplicação para o cliente ([09:25] Sofia). Um cliente que ignora o `X-Event-Id` pode processar o mesmo evento em dobro.
- Exige documentação clara e exemplos no portal ([09:26] Marcos).

**Trade-off explícito:** aceitamos duplicatas eventuais, tratadas pelo cliente, em troca de um protocolo simples e de nunca perder um evento por causa de uma falha transitória.
