# ADR-004: Autenticação dos envios com HMAC-SHA256 e secret por endpoint, com rotação e 24h de carência

- **Data da decisão:** reunião técnica de webhooks ([09:22] Sofia)
- **Decisores:** Sofia (Segurança), Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Relacionados:** [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)

## Status

Aceito

## Contexto

Vamos enviar dados de pedidos para endpoints fora da nossa infraestrutura. O cliente precisa conseguir verificar duas coisas: que a requisição veio de nós (autenticidade) e que o payload não foi adulterado no caminho (integridade) ([09:19] Sofia).

Já houve um cliente que vazou uma secret no log da própria aplicação ([09:22] Diego). O logger do projeto (`src/shared/logger/index.ts`) mascara `authorization`, `password`, `passwordHash`, `token` e `accessToken`, mas **não** mascara um campo `secret`.

## Decisão

1. **HMAC-SHA256 sobre o corpo da requisição.** A assinatura vai no header `X-Signature` ([09:20] Sofia, [09:22] Sofia, [09:44] Diego). Ela é calculada sobre exatamente os mesmos bytes que são enviados.
2. **Uma secret única por endpoint de webhook**, não uma secret global da plataforma ([09:21] Sofia). A configuração guarda URL, secret, `customer_id` e estado ativo ([09:21] Bruno, [09:21] Sofia).
3. **A secret é gerada pela plataforma** e devolvida ao cliente na criação ([09:31] Marcos). Ela não vem do cliente.
4. **A secret é rotacionável** por um endpoint da API. Depois da rotação, a secret anterior continua válida **por 24 horas em paralelo** e depois é descartada ([09:21] Sofia, [09:22] Sofia).
5. **HTTPS obrigatório:** URLs `http://` são recusadas com erro de validação ([09:23] Sofia). Como foi dito na reunião, isso é validação de schema Zod, não decisão arquitetural. O detalhe está no [FDD](../FDD.md).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Uma secret global da plataforma** | Se ela vaza, todos os clientes ficam expostos ([09:21] Sofia). |
| **Outros algoritmos de HMAC** | SHA-256 é o padrão de mercado e todo cliente sério tem biblioteca para ele ([09:20] Sofia). |
| **Secret fixa, sem rotação** | O vazamento de secret já aconteceu na prática ([09:22] Diego). Sem rotação, a única saída seria recriar o webhook. |

## Consequências

### Positivas

- O vazamento de uma secret afeta um único endpoint.
- O cliente rotaciona sozinho pela API, sem janela de indisponibilidade, graças às 24h de carência.
- A verificação do lado do cliente é trivial, com bibliotecas padrão.

### Negativas / pontos de atenção

- O worker precisa da secret em claro para assinar. Ela **não pode ser armazenada como hash**, como acontece com `passwordHash`. A proteção da secret em repouso não foi discutida e fica para a revisão de segurança da Sofia ([09:46] Sofia).
- O incidente citado em [09:22] Diego foi um vazamento no log **do próprio cliente**. Mesmo assim, ele mostra que secrets vazam por logs. Como **proposta derivada**, a reunião não discutiu exposição do nosso lado:
  - a secret só é devolvida na criação ([09:31] Marcos) e na rotação, nunca em listagens;
  - `*.secret` entra no `redact` do Pino, que hoje não cobre esse campo (`src/shared/logger/index.ts`).

  O detalhe está no [FDD](../FDD.md).
- Durante as 24h de carência, a forma de expor duas assinaturas válidas não foi definida na reunião. O FDD propõe um formato, que fica aberto para validação ([RFC Q7](../RFC.md#5-questões-em-aberto)).
- Como a decisão é assinar só o **corpo**, o `X-Timestamp` ([09:44] Diego) não fica coberto pela assinatura. Isso enfraquece a detecção de replay pelo cliente. É uma consequência derivada da decisão, registrada como questão em aberto ([RFC Q7](../RFC.md#5-questões-em-aberto)).
- O código de HMAC e de geração de secret depende de revisão de segurança dedicada (2 dias úteis) antes do deploy ([09:46] Sofia).

**Trade-off explícito:** gerenciar N secrets e um ciclo de rotação custa mais do que uma secret única. Em troca, o raio de dano de um vazamento fica limitado a um endpoint.
