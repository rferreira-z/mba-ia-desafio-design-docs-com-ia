# Da Reunião ao Documento — Design Docs do Sistema de Webhooks

Pacote de design docs do **Sistema de Webhooks de Notificação de Pedidos** do OMS. O pacote foi produzido com IA a partir da transcrição da reunião técnica (`TRANSCRICAO.md`) e do código-base. O enunciado original está no [repositório base do desafio](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia#readme).

## Sobre o desafio

O ponto de partida é uma reunião de cerca de 55 minutos entre tech lead, PM, dois engenheiros e uma engenheira de segurança. Nela foi decidido como o OMS (Node.js + TypeScript + Prisma/MySQL) vai notificar clientes B2B sobre mudanças de status de pedidos. A única coisa registrada foi a transcrição literal. A tarefa é transformar essa conversa, cruzada com o código que já existe, num pacote de documentação acionável:

- **PRD:** por quê e o quê;
- **RFC:** a proposta técnica, aberta para revisão;
- **ADRs:** cada decisão fechada, isolada;
- **FDD:** como construir;
- **Tracker:** de onde veio cada item.

A restrição central é a **rastreabilidade**. Nenhum requisito, decisão ou restrição pode existir sem uma fala da reunião ou um arquivo do código que o sustente. Isso exige filtrar o que **não** entra (o que foi descartado, adiado ou corrigido durante a própria conversa) tanto quanto o que entra. O código da aplicação não foi alterado: a entrega é só documental.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|---|---|
| **Claude Code** (extensão VS Code, modelo Claude Opus 5.5) | Agente principal. Leu o enunciado, a transcrição e o código. Montou o plano, escreveu todos os documentos e aplicou as correções de cada ciclo de revisão. |
| **Subagente de extração** (Claude, contexto isolado) | Fez uma extração independente e classificada da transcrição, sem ver o inventário do agente principal. Serviu de *cross-check* contra viés de confirmação. |
| **Subagente revisor adversarial** (Claude, contexto isolado) | Revisou o pacote pronto procurando contradições, invenções, citações erradas, vazamento de detalhe entre documentos e itens do checklist não atendidos. |
| **Script de validação** (`check_docs.py`, Python, escrito pela IA e mantido fora do repositório) | Verificação mecânica de que: <br>• toda citação `[hh:mm] Nome` existe na transcrição com aquele falante; <br>• todo caminho de código citado existe (ou está explicitamente marcado como arquivo novo); <br>• links e âncoras entre documentos resolvem; <br>• os nomes e as seções dos ADRs estão no formato; <br>• as estatísticas do tracker atingem o mínimo. |

## Workflow adotado

1. **Planejamento antes de gerar.** O primeiro pedido foi um plano, não documentos: *"read challenge.txt file and create a plan to complete this challenge in this repository."* Para montar o plano, a IA leu o enunciado, a transcrição inteira e os arquivos-chave do código (`order.service.ts`, `order.status.ts`, `schema.prisma`, `auth.middleware.ts`, erros, scripts). Com isso, as armadilhas foram listadas **antes** de qualquer documento existir (ver "Iterações e ajustes").
2. **Setup.** O repositório base foi clonado e foi configurada uma identidade git pessoal. O `docs/adrs/README.md` original foi removido, porque prescrevia nomes `0001-...` que conflitam com o formato exigido (`ADR-NNN-...`) e acrescentava à pasta um arquivo que não é ADR.
3. **Extração em paralelo.** Enquanto o agente principal escrevia os ADRs, um subagente fez a extração independente da transcrição (Prompt 2). As divergências entre os dois inventários viraram correções.
4. **Ordem de produção**, seguindo a sugestão do enunciado:
   - **ADRs:** o esqueleto das decisões;
   - **RFC:** a proposta, as alternativas e as questões em aberto, com link para os ADRs;
   - **FDD:** fluxos, contratos, erros e integração;
   - **PRD:** a consolidação no nível de produto;
   - **Tracker:** gerado a partir de uma planilha TSV de itens, com estatísticas calculadas por script, sem contagem manual.
5. **Fronteira entre documentos.** Antes de escrever cada documento, a IA definiu o que ficaria fora dele:
   - o **RFC** não tem payloads, endpoints detalhados nem modelo de dados;
   - o **PRD** fala em capacidades, não em tabelas;
   - os **ADRs** registram a decisão e o trade-off, e apontam para o FDD quando o detalhe é de implementação;
   - HTTPS e o limite de 64KB ficaram como requisito não funcional e detalhe do FDD, **não** como ADR, porque a própria reunião disse que não eram decisões arquiteturais ([09:23] Sofia, [09:24] Larissa).
6. **Verificação mecânica** com o script, incluindo um teste negativo: erros foram injetados de propósito numa cópia do pacote para confirmar que o script os detectava.
7. **Revisão adversarial** por um subagente que não conhecia o raciocínio do agente principal (Prompt 3), seguida de uma rodada de correções.
8. **Checklist final** dos critérios de aceite, item por item.

## Prompts customizados

### Prompt 1: planejamento (escrito pelo autor)

```text
read challenge.txt file and create a plan to complete this challenge in this repository.
```

O pedido foi deliberadamente de **plano**, não de documento. O plano resultante (8 fases) listou as armadilhas da transcrição e as decisões de fronteira entre documentos. Ele foi aprovado com `implement all` antes de qualquer geração.

### Prompt 2: extração classificada da transcrição (subagente)

```text
Você é um analista de requisitos fazendo a extração de uma transcrição de reunião técnica.
Seja literal e cético: nada entra sem timestamp.

Arquivo: TRANSCRICAO.md (formato `[hh:mm] Nome: fala`). Leia o arquivo inteiro.
Código de contexto (somente leitura): src/ e prisma/schema.prisma. NÃO edite nenhum arquivo.

Categorias (uma tabela por categoria, colunas: Item | Fonte `[hh:mm] Nome` | Observação):
1. DECISÃO FECHADA — só o que foi explicitamente confirmado ("decidido", "anotado",
   "tá fechado", concordância sem objeção). Cite o timestamp da confirmação, não só da proposta.
2. REQUISITO FUNCIONAL — capacidades que o sistema deve oferecer.
3. REQUISITO NÃO FUNCIONAL / RESTRIÇÃO — números (latência, timeout, tamanho, prazos),
   segurança, operação.
4. ALTERNATIVA DESCARTADA — o que foi proposto/considerado e rejeitado, com o motivo
   (trade-off) dito na reunião.
5. FORA DE ESCOPO / ADIADO — explicitamente adiado ou excluído.
6. QUESTÃO EM ABERTO — levantado e não decidido, ou "observar e decidir depois".
7. CORREÇÕES E ARMADILHAS — falas que foram corrigidas depois por outro participante,
   afirmações da reunião que você suspeita não baterem com o código (verifique no código e
   cite o arquivo), ambiguidades numéricas (ex.: contagem de tentativas vs. intervalos de
   backoff), e inconsistências internas da transcrição.
8. GANCHOS COM O CÓDIGO — cada menção a arquivo, classe, função, tabela ou padrão existente,
   com o caminho real no repositório (confirme que existe) ou "NÃO EXISTE" se for algo a ser criado.

Regras: não invente; se algo não tem fala correspondente, não liste. Não resuma decisões com
palavras mais fortes do que as usadas (ex.: "limitação conhecida" não é "garantia").
Ao final, liste em 5–10 bullets as armadilhas que um gerador de documentos com IA
provavelmente erraria.
```

Por que funciona:

- **A categoria 7 obriga a IA a procurar o que normalmente passaria despercebido:** correções feitas no meio da reunião, divergências com o código e ambiguidades numéricas.
- **A regra "não resuma com palavras mais fortes" ataca um defeito comum de LLMs.** Eles tendem a transformar uma "limitação conhecida" numa "garantia".

### Prompt 3: revisão adversarial do pacote (subagente)

```text
Você é um revisor técnico cético avaliando um pacote de design docs gerado com IA.
Seu trabalho é ENCONTRAR PROBLEMAS, não elogiar. Não edite nenhum arquivo; apenas reporte.

Fonte de verdade 1: TRANSCRICAO.md. Fonte de verdade 2: o código em src/, prisma/, tests/,
package.json. Documentos sob revisão: docs/PRD.md, docs/RFC.md, docs/FDD.md, docs/adrs/*.md,
docs/TRACKER.md.

Procure especificamente, com evidência (arquivo:linha + timestamp ou caminho de código):
1. CONTRADIÇÕES com a transcrição ou o código (inclua números: 2s, 10s, 64KB, 24h,
   5 tentativas, backoff, 100 entregas; quem decidiu; descartado/adiado virando requisito).
2. INVENÇÕES: afirmações apresentadas como discutidas na reunião sem fala correspondente
   (itens marcados "proposta de implementação" são permitidos, mas aponte detalhes
   inventados que NÃO estão marcados).
3. CITAÇÕES ERRADAS: `[hh:mm] Nome` que existe mas não sustenta o que o doc afirma
   (amostre pelo menos 25 citações, priorizando PRD e ADRs).
4. FRONTEIRAS ENTRE DOCS: RFC descendo a detalhe de FDD; duplicação desnecessária.
5. CÓDIGO: afirmações erradas sobre funções, classes, middleware, schema, scripts.
6. CHECKLIST do enunciado: quais critérios de aceite NÃO estão atendidos.
7. Inconsistências internas entre os documentos.

Formato: lista numerada de achados com severidade (ALTA/MÉDIA/BAIXA), local, evidência e
correção sugerida. No máximo 25, ordenados por severidade.
```

Por que funciona:

- **O revisor roda em contexto isolado.** Ele não herda o raciocínio de quem escreveu, então não tende a "confirmar" as próprias escolhas.
- **A exigência de evidência com arquivo:linha elimina críticas genéricas.**
- **A amostragem de citações** pega o caso mais traiçoeiro: o timestamp existe, mas a fala não diz o que o documento afirma. Esse caso passa pelo script mecânico.

## Iterações e ajustes

Foram **5 ciclos principais** de geração, revisão e correção.

### Iteração 1: armadilhas encontradas no planejamento, antes de gerar

A leitura cruzada da transcrição com o código apontou pontos em que um documento gerado "de primeira" erraria:

- **`customer_id` vindo do JWT.** Marcos afirma isso em [09:31] e Larissa corrige em [09:32]: o JWT é do operador. O schema confirma que não existe vínculo entre usuário e cliente (`UserRole` só tem `ADMIN`/`OPERATOR`).
- **Estoque.** Bruno diz em [09:04] que a mudança de status "decrementa stock_quantity". O código (`order.status.ts`) só debita em `PENDING → PAID` e devolve em cancelamentos. O FDD segue o código e registra a correção.
- **"5 tentativas".** Larissa ([09:48]) fala em "5 tentativas", mas são 5 intervalos com cerca de 15h "entre a primeira falha e a última tentativa" ([09:17] Diego). A única leitura coerente é 1 envio + 5 retentativas. Isso foi documentado como interpretação e como questão em aberto, em vez de ser escolhido em silêncio.
- **Ordem.** Ela é só por `order_id` e com um único worker. Foi registrada como **limitação**, não como garantia ([09:13] Larissa).
- **Itens descartados ou adiados que não podem virar requisito:** e-mail, dashboard, rate limiting, arquivamento, exactly-once, Redis, trigger.

### Iteração 2: o código contradiz o que a reunião assumiu

A leitura completa do código mudou o FDD em pontos concretos:

- **`WEBHOOK_INVALID_URL`.** Bruno quer o código `WEBHOOK_INVALID_URL` ([09:28]) e Sofia quer a validação de HTTPS no schema Zod ([09:23]). Mas `validate.middleware.ts` converte **todo** `ZodError` em `VALIDATION_ERROR`. A matriz de erros passou a dizer explicitamente que os códigos de validação aparecem em `error.details[]`.
- **`NotFoundError`** não aceita código customizado. Por isso `WebhookNotFoundError` estende `AppError` diretamente.
- **`PENDING`.** Nenhuma transição em `order.status.ts` leva a `PENDING`, então o filtro de eventos recusa `PENDING`. O evento na criação do pedido virou questão em aberto, em vez de ser inventado como requisito.
- **Prefixo `/api/v1`.** As rotas são montadas com esse prefixo em `src/app.ts`, então os endpoints citados na reunião ganharam o prefixo.
- **Testes e logs.** `tests/setup.ts` limpa as tabelas manualmente, então as tabelas novas precisam entrar lá, na ordem das FKs. O `redact` do Pino não cobre `secret`, o que é relevante porque um cliente já vazou secret em log ([09:22] Diego).

### Iteração 3: extração independente contra o inventário do agente principal

O subagente do Prompt 2 confirmou as armadilhas da Iteração 1 e trouxe pontos que ainda não estavam nos documentos. Todos foram incorporados:

- **Bruno se contradiz sobre o Prisma Client:** "mesmo Prisma client" em [09:11], "Separado" em [09:30]. Os ADRs usam a versão final.
- **`WEBHOOK_SECRET_REQUIRED`**, exemplo do Bruno, não faz sentido na API, porque a secret é gerada pelo servidor ([09:31] Marcos). Nesta iteração ele foi **ressignificado** como erro de integridade do worker. Na Iteração 5 o revisor mostrou que o caso é impossível (a coluna é obrigatória), e o código acabou **não adotado**, com a justificativa no ADR-006.
- **O `errorMiddleware` não cobre o worker**, que não é Express. O FDD exige que o worker trate os próprios erros.
- **O backoff "exponencial" é uma tabela fixa** (×5, ×6, ×4, ×6). A nota foi para o ADR-003.
- **O `X-Event-Id` é único por evento ou por (evento × webhook)?** Foi decidido por (evento × webhook), com justificativa no ADR-005.
- **UUID e snapshot foram decididos depois que Sofia e Marcos saíram da call** ([09:50]). Isso está registrado no tracker.

### Iteração 4: problemas de design encontrados ao detalhar o FDD

Ao descer para o nível de implementação, apareceram pontos que a reunião não cobriu. Eles foram registrados como **questões em aberto** ou como **propostas de implementação** marcadas, nunca como decisões:

- **Ordem durante o retry.** Mesmo com um único worker, um evento posterior pode ultrapassar um que está esperando retentativa (RFC Q7).
- **`X-Timestamp` fora da assinatura.** O HMAC cobre só o corpo ([09:22] Sofia), então o `X-Timestamp` ([09:44] Diego) pode ser adulterado. Foi para a revisão da Sofia (RFC Q8).
- **Onde validar os 64KB.** Validar na inserção faria rollback da mudança de status, o que contradiz [09:04] Bruno. A validação foi para o worker, com ida direta para a DLQ.
- **Redirects.** Seguir um redirect poderia furar o HTTPS obrigatório, então o fetch usa `redirect: 'manual'`.

### Iteração 5: verificação mecânica e revisão adversarial

- **O script de validação** foi testado com erros injetados de propósito: timestamp inexistente, falante errado, âncora quebrada e caminho de código inexistente. Ele detectou todos.
- **No pacote real**, uma citação fraca do tracker (`[09:45] Diego` para "5 headers") foi trocada pela fala que de fato define os headers (`[09:44] Diego`).
- **A revisão adversarial (Prompt 3)** trouxe os achados abaixo, corrigidos antes da entrega:

O revisor apontou 21 achados e o checklist com 25 de 27 critérios atendidos. Todos os achados foram tratados. Os mais importantes:

| Achado do revisor | Severidade | Correção aplicada |
|---|---|---|
| **O desenho do worker estourava a meta de 10s.** Com processamento sequencial em lote e timeout de 10s, um único cliente lento atrasaria os outros em até cerca de 100s. O ADR-002 chegava a dizer que "2s + 10s cabe na meta" | ALTA | O FDD §4.2 foi redesenhado com **filas lógicas por webhook** dentro do worker único: o tick a cada 2s não espera as lanes, e cada webhook é sequencial. A ordem por pedido continua preservada. O ADR-002 foi corrigido. |
| **A contagem de tentativas aparecia ao mesmo tempo como "aberta" (RFC) e "decidida" (ADR).** A melhor evidência não estava citada: "retentaria três vezes em 30 minutos" ([09:16] Diego) = 1m + 5m + 30m, o que prova que "tentativa" = retentativa | ALTA | A questão saiu do RFC. O ADR-003 agora cita [09:16] Diego, e "6 envios" foi trocado por "envio original + 5 retentativas" em todos os documentos. |
| **`GET /webhooks?customerId=`** contrariava "no body ou no path" ([09:32] Larissa) | MÉDIA | A rota virou `GET /api/v1/customers/:customerId/webhooks`, com a nota de montagem (`mergeParams`, ordem em `routes/index.ts`). |
| **Só 1 endpoint tinha request e response completos** (critério "≥ 4 endpoints com payload de exemplo") | MÉDIA | Os 7 endpoints e o outbound agora têm um bloco `http` de request (linha, headers, body) e um JSON de response. |
| **O tracker atribuía à transcrição itens que eram derivação.** Exemplos: os códigos de erro propostos, o redirect manual, a "unidade de evento" | MÉDIA | As linhas passaram a ter o tipo "Proposta (derivada)" / "Erro (proposto, derivado da regra citada)". A matriz de erros ganhou a coluna **Origem**, que separa os 2 códigos citados na reunião dos propostos. |
| **"A secret nunca aparece em listagens nem em logs" citava [09:22] Diego.** Mas o vazamento de que Diego falou foi no log **do cliente** | MÉDIA | O item virou proposta derivada, ancorada no código (`redactPaths` não cobre `secret`). |
| **Detalhes inventados sem a marca de proposta:** 2048 caracteres, `sha256=`, DELETE em cascata, replay 202, limiares de alerta, "sem jitter" com justificativa errada | MÉDIA | Todos foram marcados. "Webhook inativo vai para a DLQ" foi trocado por "webhook inativo = pausado", que evita falsos alarmes na DLQ. |
| **RFC e PRD desciam a detalhe de FDD:** formato de assinatura, `PENDING` no filtro, nomes de headers no PRD, arquivos de teste no PRD | MÉDIA | Esse conteúdo foi movido para o FDD, com links. O "fora de escopo" do FDD agora só lista exclusões técnicas e aponta para o PRD. |
| **Outros:** OBJ-5 não era mensurável (o log tem `userId`, não o cliente); o instanciamento do PrismaClient criaria dois clientes; nomes divergentes (`secretRotationExpiresAt`); falha no INSERT nem sempre vira 500 (P2002 → 409); o Status dos ADRs não era seção; decisores inflados; cenários atribuíam fatos a clientes nomeados; risco de prazo subestimado | BAIXA | Tudo corrigido. Os ADRs ganharam `## Status`. Os cenários foram marcados como ilustrativos. R1 subiu para probabilidade Alta. |

Depois das correções, o script de validação rodou de novo sem problemas: 318 linhas no tracker, 82,7% com fonte na transcrição.

## Como navegar a entrega

| Ordem | Documento | Pergunta que responde |
|---|---|---|
| 1 | [`docs/PRD.md`](docs/PRD.md) | Por que fazer, para quem, o que entra e o que fica fora, como medir |
| 2 | [`docs/RFC.md`](docs/RFC.md) | Qual a proposta, que alternativas foram descartadas, o que ainda está em aberto |
| 3 | [`docs/adrs/`](docs/adrs/) | Por que cada decisão foi tomada exatamente assim (7 ADRs) |
| 4 | [`docs/FDD.md`](docs/FDD.md) | Como construir: fluxos, contratos, erros, observabilidade, integração com o código |
| 5 | [`docs/TRACKER.md`](docs/TRACKER.md) | De onde veio cada item (transcrição ou código) |

### ADRs

- [ADR-001 — Outbox no MySQL](docs/adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker separado em polling](docs/adrs/ADR-002-worker-separado-em-polling.md)
- [ADR-003 — Retry com backoff e DLQ](docs/adrs/ADR-003-retry-com-backoff-e-dlq.md)
- [ADR-004 — HMAC-SHA256 com secret por endpoint](docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005 — At-least-once com X-Event-Id](docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões existentes](docs/adrs/ADR-006-reuso-dos-padroes-existentes.md)
- [ADR-007 — Snapshot e filtro na inserção do outbox](docs/adrs/ADR-007-snapshot-e-filtro-na-insercao-do-outbox.md)

### Leitura rápida, por perfil

- **Produto:** PRD.
- **Revisor de arquitetura:** RFC, depois os ADRs.
- **Quem vai implementar:** FDD §4 (fluxos), §5 (contratos), §6 (matriz de erros) e §10 (integração com o código existente).
- **Auditoria:** use o Tracker a partir de qualquer ID citado.
