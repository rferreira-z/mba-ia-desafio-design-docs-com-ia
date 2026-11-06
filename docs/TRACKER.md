# Tracker de Rastreabilidade

Este tracker liga cada item registrado no pacote (requisito, decisão, restrição, trade-off, contrato, erro, risco) à sua **origem**:

- a fala da reunião em `TRANSCRICAO.md`, no formato `[hh:mm] Nome`; ou
- o arquivo do código-base que sustenta o item.

Se um item não tem linha aqui, ele não deveria estar nos documentos.

## Como ler

| Coluna | Significado |
|---|---|
| **ID** | Identificador do item, prefixado pelo documento: `PRD-FR-*`, `PRD-NFR-*`, `RFC-ALT-*`, `RFC-Q-*`, `ADR-00N-*`, `FDD-CONTRATO-*`, `FDD-ERRO-*`, `FDD-INT-*` etc. |
| **Documento** | O arquivo onde o item aparece. |
| **Tipo** | Requisito Funcional, Requisito Não Funcional, Decisão, Alternativa, Restrição, Trade-off, Contrato, Erro, Risco, Integração etc. |
| **Fonte** | `TRANSCRICAO` (fala da reunião) ou `CODIGO` (arquivo existente no repositório). |
| **Localização** | Para `TRANSCRICAO`, o timestamp e o falante, por exemplo `[09:17] Diego`. Quando a decisão foi proposta por uma pessoa e confirmada por outra, a localização aponta para a fala que **fecha** o item. Para `CODIGO`, o caminho do arquivo. |

**Correções registradas.** Quando a reunião se corrige, o tracker aponta para a fala corrigida e o resumo menciona a correção. Os casos são:

- `customer_id` do JWT: dito em [09:31] por Marcos, corrigido em [09:32] por Larissa;
- PrismaClient: "mesmo" em [09:11] por Bruno, "separado" em [09:30] por Bruno;
- débito de estoque: afirmado em [09:04] por Bruno e corrigido pelo código (`src/modules/orders/order.status.ts`).

## Resumo

| Métrica | Valor |
|---|---|
| Linhas | 318 |
| Fonte = TRANSCRICAO | 263 (82.7%) |
| Fonte = CODIGO | 55 (17.3%) |
| Linhas por documento | PRD: 85 · RFC: 36 · ADRs: 73 · FDD: 124 |

## Tabela

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CTX-01 | docs/PRD.md | Contexto | Três clientes B2B (Atlas, MaxDistribuição, Nova Cargo) pedem notificação de mudança de status | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Restrição | Webhook somente outbound (plataforma → cliente) | TRANSCRICAO | [09:02] Marcos |
| PRD-PROB-01 | docs/PRD.md | Problema | Clientes fazem polling em GET /orders; integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-02 | docs/PRD.md | Risco de negócio | Atlas pode migrar para concorrente se não houver entrega | TRANSCRICAO | [09:00] Marcos |
| PRD-PROB-03 | docs/PRD.md | Requisito Não Funcional | "Tempo real" = abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-PUB-01 | docs/PRD.md | Público-alvo | Usuários da plataforma que representam o cliente configuram via API com JWT do sistema | TRANSCRICAO | [09:32] Marcos |
| PRD-PUB-02 | docs/PRD.md | Público-alvo | ADMIN reprocessa DLQ com auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-PUB-03 | docs/PRD.md | Público-alvo | Produto documenta integração no portal do desenvolvedor | TRANSCRICAO | [09:40] Marcos |
| PRD-CEN-01 | docs/PRD.md | Cenário (ilustrativo) | Cadastro de webhook só para SHIPPED e DELIVERED (exemplo genérico de Marcos) | TRANSCRICAO | [09:33] Marcos |
| PRD-CEN-02 | docs/PRD.md | Cenário | Recebimento <10s, validação HMAC e dedup | TRANSCRICAO | [09:25] Diego |
| PRD-CEN-03 | docs/PRD.md | Cenário (ilustrativo) | Indisponibilidade de 2h por manutenção (cliente não nomeado) coberta por retry | TRANSCRICAO | [09:16] Diego |
| PRD-CEN-04 | docs/PRD.md | Cenário | Secret vazada em log do cliente → rotação com 24h | TRANSCRICAO | [09:22] Diego |
| PRD-CEN-05 | docs/PRD.md | Cenário | Cliente consulta histórico de entregas para investigar | TRANSCRICAO | [09:34] Marcos |
| PRD-CEN-06 | docs/PRD.md | Cenário | Cliente fora além da janela de ≈14h36 → DLQ → replay por ADMIN | TRANSCRICAO | [09:18] Diego |
| PRD-OBJ-01 | docs/PRD.md | Métrica | Latência p95 mudança→entrega < 10s | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Métrica | 100% das mudanças assinadas geram evento (atomicidade) | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-03 | docs/PRD.md | Métrica | 3 de 3 clientes ativos; Atlas até fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-04 | docs/PRD.md | Métrica | Janela de retry ≈14h36min aceita pelo produto | TRANSCRICAO | [09:17] Marcos |
| PRD-OBJ-05 | docs/PRD.md | Métrica | Redução de polling medida por userId dos logs http_request (vínculo usuário→cliente manual) | CODIGO | src/middlewares/request-logger.middleware.ts |
| PRD-OBJ-06 | docs/PRD.md | Indicador | Volume de DLQ como insumo para decidir e-mail na fase 2 | TRANSCRICAO | [09:37] Larissa |
| PRD-OBJ-07 | docs/PRD.md | Indicador | Eventos/cliente/minuto como insumo para rate limiting | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-01 | docs/PRD.md | Fora de escopo (adiado) | Aviso por e-mail quando webhook falha | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de escopo (descartado) | Dashboard visual para o cliente; projeto do time de frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de escopo (adiado) | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de escopo | Webhooks inbound | TRANSCRICAO | [09:02] Marcos |
| PRD-OOS-05 | docs/PRD.md | Fora de escopo | Arquivamento de linhas entregues (~30 dias) | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-06 | docs/PRD.md | Fora de escopo | Ordem global e múltiplos workers | TRANSCRICAO | [09:13] Larissa |
| PRD-OOS-07 | docs/PRD.md | Fora de escopo | Exactly-once | TRANSCRICAO | [09:25] Diego |
| PRD-OOS-08 | docs/PRD.md | Fora de escopo | Itens do pedido no payload | TRANSCRICAO | [09:43] Diego |
| PRD-OOS-09 | docs/PRD.md | Restrição (correção) | customer_id NÃO vem do JWT (corrige [09:31] Marcos) | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook (customerId, url, status); secret gerada e devolvida | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Editar webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remover webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listar webhooks de um cliente (GET) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status, aplicado na geração do evento | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Notificação automática a cada mudança de status assinada | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 do corpo em X-Signature | TRANSCRICAO | [09:22] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Rotação de secret com 24h de validade da anterior | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas (resultado, payload, resposta, tempo) | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | 5 retentativas (1m/5m/30m/2h/12h) e DLQ | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Replay de DLQ por ADMIN com registro do autor | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Identificador estável do evento, do cadastro e horário de envio (contrato no FDD) | TRANSCRICAO | [09:44] Diego |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Notificação enxuta: identificação do pedido e transição, sem itens | TRANSCRICAO | [09:43] Diego |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Entrega <10s; espera de polling ≤2s aceita | TRANSCRICAO | [09:10] Larissa |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s no envio | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Somente URLs https; http recusado com erro de validação | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Limite de 64KB; erro em vez de truncar | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Garantia at-least-once | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Sem status sem evento nem evento de mudança desfeita | TRANSCRICAO | [09:06] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Cliente lento/offline não bloqueia nem desfaz mudanças de status | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-08 | docs/PRD.md | Restrição | Ordem só por pedido e com worker único (limitação) | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Secret única por endpoint | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-09b | docs/PRD.md | Proposta (derivada) | Secret exibida só na criação/rotação; fora de listagens e logs (redact atual não cobre secret) | CODIGO | src/shared/logger/index.ts |
| PRD-NFR-09c | docs/PRD.md | Requisito Não Funcional | Secret devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Entrega em processo separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-11 | docs/PRD.md | Restrição | Nenhuma infraestrutura nova | TRANSCRICAO | [09:07] Diego |
| PRD-NFR-12 | docs/PRD.md | Requisito Não Funcional | Replay restrito a ADMIN e auditado | TRANSCRICAO | [09:36] Sofia |
| PRD-DEC-01 | docs/PRD.md | Trade-off | Até 2s de espera aceito pelo produto | TRANSCRICAO | [09:10] Marcos |
| PRD-DEC-02 | docs/PRD.md | Trade-off | Cliente fora >15h exige reprocessamento manual — aceito | TRANSCRICAO | [09:17] Marcos |
| PRD-DEC-03 | docs/PRD.md | Trade-off | Cliente precisa deduplicar; produto documenta em destaque | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança ≥2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Sessão de revisão do design antes de codar | TRANSCRICAO | [09:50] Larissa |
| PRD-DEP-03 | docs/PRD.md | Dependência | Documentação no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-04 | docs/PRD.md | Dependência | Confirmação do prazo com a Atlas | TRANSCRICAO | [09:47] Marcos |
| PRD-DEP-05 | docs/PRD.md | Dependência | Novo processo a operar (npm run worker) | TRANSCRICAO | [09:11] Larissa |
| PRD-RISK-01 | docs/PRD.md | Risco | Prazo de 3 sprints vs. fim de novembro; duração de sprint não informada (prob. Alta) | TRANSCRICAO | [09:46] Larissa |
| PRD-RISK-02 | docs/PRD.md | Risco | Vazamento de secret pelo cliente (já ocorreu) | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Cliente não deduplica eventos | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-04 | docs/PRD.md | Risco | Cliente fora >15h sem aviso proativo | TRANSCRICAO | [09:37] Larissa |
| PRD-RISK-05 | docs/PRD.md | Risco | Rajada de eventos sem rate limiting | TRANSCRICAO | [09:38] Diego |
| PRD-RISK-06 | docs/PRD.md | Risco | Processo de entrega parado | TRANSCRICAO | [09:11] Diego |
| PRD-RISK-07 | docs/PRD.md | Risco | Eventos fora de ordem em retentativa; mitigado por clientes não exigirem ordem global | TRANSCRICAO | [09:14] Marcos |
| PRD-RISK-08 | docs/PRD.md | Risco | CRUD aberto a qualquer autenticado nesta fase | TRANSCRICAO | [09:37] Sofia |
| PRD-CA-01 | docs/PRD.md | Critério de aceitação | Notificação <10s com headers e assinatura válidos | TRANSCRICAO | [09:44] Diego |
| PRD-CA-02 | docs/PRD.md | Critério de aceitação | Webhook não recebe status não assinado | TRANSCRICAO | [09:33] Marcos |
| PRD-CA-03 | docs/PRD.md | Critério de aceitação | URL http recusada | TRANSCRICAO | [09:23] Sofia |
| PRD-CA-04 | docs/PRD.md | Critério de aceitação | Secret só na resposta de criação | TRANSCRICAO | [09:31] Marcos |
| PRD-CA-05 | docs/PRD.md | Critério de aceitação | 5 retentativas e DLQ | TRANSCRICAO | [09:17] Larissa |
| PRD-CA-06 | docs/PRD.md | Critério de aceitação | Replay ADMIN mantém X-Event-Id; OPERATOR negado | TRANSCRICAO | [09:36] Larissa |
| PRD-CA-07 | docs/PRD.md | Critério de aceitação | Validação com secret antiga ou nova nas 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-CA-08 | docs/PRD.md | Critério de aceitação | Histórico com até 100 entregas por página | TRANSCRICAO | [09:34] Marcos |
| PRD-CA-09 | docs/PRD.md | Critério de aceitação | Falha ao registrar evento desfaz a mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-CA-10 | docs/PRD.md | Critério de aceitação | Endpoint lento não afeta outros pedidos | TRANSCRICAO | [09:04] Bruno |
| PRD-CA-11 | docs/PRD.md | Critério de aceitação | Revisão de segurança aprovada antes do deploy | TRANSCRICAO | [09:49] Sofia |
| PRD-TEST-01 | docs/PRD.md | Estratégia de teste | Testes automatizados com a suíte existente do projeto (detalhe no FDD) | CODIGO | tests/orders.test.ts |
| RFC-META-01 | docs/RFC.md | Metadado | Autoria do documento de design por Larissa; revisão com Bruno e Diego | TRANSCRICAO | [09:50] Larissa |
| RFC-TLDR-01 | docs/RFC.md | Proposta | Outbox + worker + HMAC + at-least-once + retry/DLQ (resumo da reunião) | TRANSCRICAO | [09:48] Larissa |
| RFC-CTX-01 | docs/RFC.md | Contexto | Atlas quer entrega até fim de novembro | TRANSCRICAO | [09:45] Marcos |
| RFC-CTX-02 | docs/RFC.md | Contexto | Outbound only (Sofia confirma) | TRANSCRICAO | [09:03] Sofia |
| RFC-CTX-03 | docs/RFC.md | Restrição | changeStatus já é transação com várias escritas | CODIGO | src/modules/orders/order.service.ts |
| RFC-PROP-01 | docs/RFC.md | Proposta | Captura transacional via publishWebhookEvent | TRANSCRICAO | [09:41] Bruno |
| RFC-PROP-02 | docs/RFC.md | Proposta | Worker separado, polling 2s, instância única | TRANSCRICAO | [09:09] Diego |
| RFC-PROP-03 | docs/RFC.md | Proposta | Timeout 10s, 5 retentativas, DLQ, replay ADMIN | TRANSCRICAO | [09:18] Diego |
| RFC-PROP-04 | docs/RFC.md | Proposta | HMAC-SHA256, secret por endpoint, rotação 24h | TRANSCRICAO | [09:22] Sofia |
| RFC-PROP-05 | docs/RFC.md | Proposta | HTTPS e 64KB são RNF, não ADR | TRANSCRICAO | [09:24] Larissa |
| RFC-PROP-06 | docs/RFC.md | Proposta | Headers de entrega (X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id) | TRANSCRICAO | [09:44] Sofia |
| RFC-PROP-07 | docs/RFC.md | Proposta | customer_id explícito no body/path, não do JWT | TRANSCRICAO | [09:32] Bruno |
| RFC-PROP-08 | docs/RFC.md | Proposta | Módulo padrão + AppError WEBHOOK_ + Pino | TRANSCRICAO | [09:30] Larissa |
| RFC-PROP-09 | docs/RFC.md | Estimativa | 3 sprints com revisão de segurança incluída | TRANSCRICAO | [09:47] Larissa |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | HTTP síncrono em changeStatus | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams / broker | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger do MySQL | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Retry indefinido ou 3 tentativas | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Secret global | TRANSCRICAO | [09:21] Sofia |
| RFC-ALT-06 | docs/RFC.md | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-07 | docs/RFC.md | Alternativa descartada | Renderizar payload no envio | TRANSCRICAO | [09:52] Larissa |
| RFC-Q-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída | TRANSCRICAO | [09:39] Diego |
| RFC-Q-02 | docs/RFC.md | Questão em aberto | Aviso por e-mail (próxima fase) | TRANSCRICAO | [09:37] Larissa |
| RFC-Q-03 | docs/RFC.md | Questão em aberto | Escala para múltiplos workers (partição/lock) | TRANSCRICAO | [09:13] Diego |
| RFC-Q-04 | docs/RFC.md | Questão em aberto | Arquivamento do outbox | TRANSCRICAO | [09:08] Diego |
| RFC-Q-05 | docs/RFC.md | Questão em aberto | Endurecer permissões do CRUD | TRANSCRICAO | [09:37] Sofia |
| RFC-Q-05b | docs/RFC.md | Restrição | Não há vínculo usuário↔cliente (UserRole só ADMIN/OPERATOR) | CODIGO | prisma/schema.prisma |
| RFC-Q-06 | docs/RFC.md | Questão em aberto (derivada) | Ordem por pedido durante retry: evento posterior pode ultrapassar um em backoff | TRANSCRICAO | [09:12] Diego |
| RFC-Q-07 | docs/RFC.md | Questão em aberto | Pontos para a revisão de segurança: assinatura na carência, X-Timestamp fora do HMAC, secret em repouso | TRANSCRICAO | [09:46] Sofia |
| RFC-Q-08 | docs/RFC.md | Questão em aberto | Evento na criação do pedido não discutido (create grava null→PENDING) | CODIGO | src/modules/orders/order.service.ts |
| RFC-IMP-01 | docs/RFC.md | Impacto | changeStatus ganha leitura + N inserts; falha faz rollback | TRANSCRICAO | [09:40] Bruno |
| RFC-IMP-02 | docs/RFC.md | Impacto | Novo processo com conexão Prisma própria | TRANSCRICAO | [09:30] Bruno |
| RFC-IMP-03 | docs/RFC.md | Impacto | Clientes precisam deduplicar e verificar HMAC | TRANSCRICAO | [09:26] Marcos |
| RFC-RISK-01 | docs/RFC.md | Risco | Ponto único de falha na entrega (instância única) | TRANSCRICAO | [09:12] Diego |
| RFC-RISK-02 | docs/RFC.md | Risco | Crescimento do outbox sem arquivamento nesta fase | TRANSCRICAO | [09:08] Diego |
| RFC-RISK-03 | docs/RFC.md | Risco | Sequência fora de ordem durante retry | TRANSCRICAO | [09:13] Larissa |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox no MySQL existente | TRANSCRICAO | [09:08] Larissa |
| ADR-001-D02 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Insert no outbox na mesma transação; falha → rollback | TRANSCRICAO | [09:40] Bruno |
| ADR-001-D03 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Índices em status e created_at; estados pendente/processando/falhou/entregue | TRANSCRICAO | [09:08] Diego |
| ADR-001-D04 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Ids UUID (decidido após a saída de Marcos e Sofia) | TRANSCRICAO | [09:51] Larissa |
| ADR-001-ALT1 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa | Síncrono: cliente lento trava mudanças; rollback não faz sentido | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT2 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa | Redis Cluster = overengineering para time pequeno | TRANSCRICAO | [09:07] Diego |
| ADR-001-C01 | docs/adrs/ADR-001-outbox-no-mysql.md | Contexto (código) | Transação de changeStatus: orders, history, estoque | CODIGO | src/modules/orders/order.service.ts |
| ADR-001-C02 | docs/adrs/ADR-001-outbox-no-mysql.md | Contexto (código) | Estoque só debita em PENDING→PAID (corrige [09:04] Bruno) | CODIGO | src/modules/orders/order.status.ts |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Polling a cada 2s | TRANSCRICAO | [09:10] Larissa |
| ADR-002-D02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Processo separado da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-D03 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | src/worker.ts + npm run worker | TRANSCRICAO | [09:11] Larissa |
| ADR-002-D04 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | PrismaClient separado, mesma DATABASE_URL (corrige [09:11] Bruno) | TRANSCRICAO | [09:30] Bruno |
| ADR-002-D05 | docs/adrs/ADR-002-worker-separado-em-polling.md | Restrição | Instância única; ordem só por order_id (limitação) | TRANSCRICAO | [09:13] Larissa |
| ADR-002-ALT1 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa | Trigger MySQL não notifica processo externo | TRANSCRICAO | [09:09] Diego |
| ADR-002-ALT2 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa | Worker no processo da API morre no restart | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT3 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa | Múltiplos workers perdem ordem; futuro | TRANSCRICAO | [09:12] Diego |
| ADR-002-C01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Contexto (código) | server.ts é o único entry-point; padrão de shutdown | CODIGO | src/server.ts |
| ADR-002-C02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Contexto (código) | Worker usa o singleton prisma de database.ts: uma instância por processo | CODIGO | src/config/database.ts |
| ADR-002-D06 | docs/adrs/ADR-002-worker-separado-em-polling.md | Proposta (derivada) | Filas lógicas por webhook para que cliente lento não estoure a meta de 10s | TRANSCRICAO | [09:02] Marcos |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | 5 retentativas com 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| ADR-003-D02 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Progressão de backoff ≈15h | TRANSCRICAO | [09:17] Diego |
| ADR-003-D03 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | DLQ em tabela webhook_dead_letter (payload, motivo, timestamp) | TRANSCRICAO | [09:18] Diego |
| ADR-003-D04 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Replay POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:35] Diego |
| ADR-003-D05 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Replay exige ADMIN via requireRole existente | TRANSCRICAO | [09:36] Larissa |
| ADR-003-D06 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Logar quem fez o replay | TRANSCRICAO | [09:36] Sofia |
| ADR-003-D07 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | Timeout de 10s conta como falha | TRANSCRICAO | [09:42] Diego |
| ADR-003-D08 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Interpretação | "Tentativa" = retentativa: "retentaria três vezes em 30 minutos" = 1m+5m+30m | TRANSCRICAO | [09:16] Diego |
| ADR-003-D09 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Restrição | Backoff "exponencial" é na prática tabela fixa; sem jitter | TRANSCRICAO | [09:17] Diego |
| ADR-003-ALT1 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa | Retry indefinido deixa evento pendurado | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT2 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa | 3 tentativas (proposta de Bruno) | TRANSCRICAO | [09:16] Bruno |
| ADR-003-ALT2b | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Trade-off | 3 é pouco: cliente já ficou 2h fora | TRANSCRICAO | [09:16] Diego |
| ADR-003-ALT3 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa | Flag failed no próprio outbox (sem tabela) | TRANSCRICAO | [09:18] Diego |
| ADR-003-C01 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Contexto (código) | requireRole('ADMIN') já existe | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo | TRANSCRICAO | [09:22] Sofia |
| ADR-004-D02 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Secret única por endpoint | TRANSCRICAO | [09:21] Sofia |
| ADR-004-D03 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Config guarda url, secret, customer_id, ativo | TRANSCRICAO | [09:21] Bruno |
| ADR-004-D04 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Rotação via API com 24h de carência | TRANSCRICAO | [09:21] Sofia |
| ADR-004-D05 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Requisito Não Funcional | HTTPS obrigatório via Zod | TRANSCRICAO | [09:23] Sofia |
| ADR-004-ALT1 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Alternativa | Secret global: vaza uma, vaza tudo | TRANSCRICAO | [09:21] Sofia |
| ADR-004-ALT2 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Alternativa | SHA-256 é o padrão de mercado | TRANSCRICAO | [09:20] Sofia |
| ADR-004-ALT3 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Alternativa | Secret fixa sem rotação: vazamento já ocorreu | TRANSCRICAO | [09:22] Diego |
| ADR-004-C01 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Contexto (código) | redactPaths do Pino não cobre secret | CODIGO | src/shared/logger/index.ts |
| ADR-004-C02 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Trade-off (derivado) | X-Timestamp não coberto pelo HMAC do corpo | TRANSCRICAO | [09:44] Diego |
| ADR-004-C03 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Proposta (derivada) | Secret fora de listagens e com redact no Pino (incidente era no log do cliente) | CODIGO | src/shared/logger/index.ts |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| ADR-005-D02 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | X-Event-Id UUID gerado na entrada no outbox | TRANSCRICAO | [09:25] Diego |
| ADR-005-D03 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Interpretação proposta | Unidade de evento = mudança × webhook (não discutido); X-Webhook-Id identifica o cadastro | TRANSCRICAO | [09:44] Sofia |
| ADR-005-D04 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | Documentar em destaque no portal | TRANSCRICAO | [09:26] Marcos |
| ADR-005-ALT1 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Alternativa | Exactly-once exige coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| ADR-005-ALT2 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Alternativa (plausível) | At-most-once sem retry: incompatível com a pergunta "cliente offline, o que fazemos?" | TRANSCRICAO | [09:14] Larissa |
| ADR-005-T01 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Trade-off | Joga responsabilidade de dedup para o cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Reuso máximo dos padrões existentes | TRANSCRICAO | [09:30] Larissa |
| ADR-006-D02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Módulo src/modules/webhooks com controller/service/repository/routes/schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006-D03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Processamento em src/modules/webhooks/webhook.worker.ts | TRANSCRICAO | [09:28] Bruno |
| ADR-006-D04 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Prefixo WEBHOOK_ em todos os códigos | TRANSCRICAO | [09:29] Larissa |
| ADR-006-D05 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Mesmo logger Pino; error middleware sem alteração | TRANSCRICAO | [09:29] Bruno |
| ADR-006-D06 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | publishWebhookEvent(tx, order, fromStatus, toStatus) | TRANSCRICAO | [09:41] Bruno |
| ADR-006-ALT1 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Alternativa | Injetar repository inteiro no OrderService | TRANSCRICAO | [09:41] Diego |
| ADR-006-ALT2 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Alternativa | Bibliotecas/padrões novos: "não vamos botar nada novo" | TRANSCRICAO | [09:29] Bruno |
| ADR-006-ALT3 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Alternativa | Ids auto-incrementais descartados em favor de UUID | TRANSCRICAO | [09:51] Larissa |
| ADR-006-C01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Padrão (código) | Classe base AppError (statusCode, errorCode) | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-C02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Padrão (código) | InvalidStatusTransitionError, InsufficientStockError; NotFoundError fixa NOT_FOUND | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-C03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Padrão (código) | errorMiddleware trata AppError, ZodError, Prisma; não cobre o worker | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-C04 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Restrição (código) | validate() converte todo ZodError em VALIDATION_ERROR | CODIGO | src/middlewares/validate.middleware.ts |
| ADR-006-C05 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Padrão (código) | Estrutura de módulo de referência | CODIGO | src/modules/orders/order.routes.ts |
| ADR-006-C06 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Padrão (código) | Ids uuid() Char(36) em todos os modelos | CODIGO | prisma/schema.prisma |
| ADR-006-T01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão (derivada) | WEBHOOK_SECRET_REQUIRED avaliado e não adotado: secret é gerada pelo servidor | TRANSCRICAO | [09:31] Marcos |
| ADR-007 | docs/adrs/ADR-007-snapshot-e-filtro-na-insercao-do-outbox.md | Decisão | Filtrar na inserção; sem assinante não insere | TRANSCRICAO | [09:34] Bruno |
| ADR-007-D02 | docs/adrs/ADR-007-snapshot-e-filtro-na-insercao-do-outbox.md | Decisão | Payload snapshot na inserção (decidido após saída de Marcos e Sofia) | TRANSCRICAO | [09:52] Bruno |
| ADR-007-D03 | docs/adrs/ADR-007-snapshot-e-filtro-na-insercao-do-outbox.md | Decisão | Payload enxuto sem itens | TRANSCRICAO | [09:44] Bruno |
| ADR-007-ALT1 | docs/adrs/ADR-007-snapshot-e-filtro-na-insercao-do-outbox.md | Alternativa | Filtrar no envio (pergunta de Diego) | TRANSCRICAO | [09:34] Diego |
| ADR-007-ALT2 | docs/adrs/ADR-007-snapshot-e-filtro-na-insercao-do-outbox.md | Alternativa | Guardar só order_id e renderizar no envio | TRANSCRICAO | [09:51] Bruno |
| ADR-007-ALT3 | docs/adrs/ADR-007-snapshot-e-filtro-na-insercao-do-outbox.md | Alternativa | Payload com itens | TRANSCRICAO | [09:43] Diego |
| FDD-CTX-01 | docs/FDD.md | Contexto (código) | Sequência da transação de changeStatus | CODIGO | src/modules/orders/order.service.ts |
| FDD-CTX-02 | docs/FDD.md | Correção | Débito de estoque só em PENDING→PAID; devolução em cancelamento | CODIGO | src/modules/orders/order.status.ts |
| FDD-CTX-03 | docs/FDD.md | Correção | Afirmação da reunião sobre estoque (corrigida pelo código) | TRANSCRICAO | [09:04] Bruno |
| FDD-OBJ-01 | docs/FDD.md | Objetivo técnico | Evento na mesma transação ou falha | TRANSCRICAO | [09:41] Diego |
| FDD-OBJ-02 | docs/FDD.md | Objetivo técnico | Tick a cada 2s; espera de polling ≤2s independente de outros clientes | TRANSCRICAO | [09:09] Diego |
| FDD-OBJ-03 | docs/FDD.md | Objetivo técnico | p95 <10s | TRANSCRICAO | [09:02] Marcos |
| FDD-OBJ-04 | docs/FDD.md | Objetivo técnico | Envio original + 5 retentativas e DLQ | TRANSCRICAO | [09:17] Larissa |
| FDD-OBJ-05 | docs/FDD.md | Objetivo técnico | Toda saída assinada | TRANSCRICAO | [09:22] Sofia |
| FDD-OBJ-06 | docs/FDD.md | Restrição | Nenhuma dependência nova (Node ≥20, uuid existente) | CODIGO | package.json |
| FDD-OBJ-07 | docs/FDD.md | Objetivo técnico | Worker como processo independente | TRANSCRICAO | [09:11] Larissa |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | publishWebhookEvent após history.create, antes do findUnique final | TRANSCRICAO | [09:40] Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Filtro de webhooks ativos que assinam toStatus | TRANSCRICAO | [09:34] Bruno |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | timestamp do payload = changedAt do histórico; X-Timestamp = envio | TRANSCRICAO | [09:44] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Proposta (derivada) | 64KB verificado no worker: validar na inserção desfaria status por problema de entrega | TRANSCRICAO | [09:04] Bruno |
| FDD-FLUXO-05 | docs/FDD.md | Fluxo | Tick a cada 2s, lote pequeno (50) e até 20 lanes (valores propostos) | TRANSCRICAO | [09:08] Diego |
| FDD-FLUXO-06 | docs/FDD.md | Fluxo | Lane por webhook processa em ordem de createdAt; preserva ordem por pedido | TRANSCRICAO | [09:12] Diego |
| FDD-FLUXO-07 | docs/FDD.md | Fluxo | Mesma string assinada e enviada | TRANSCRICAO | [09:22] Sofia |
| FDD-FLUXO-08 | docs/FDD.md | Fluxo | 64KB → DLQ sem retry (erro, não trunca) | TRANSCRICAO | [09:23] Sofia |
| FDD-FLUXO-09 | docs/FDD.md | Proposta (derivada) | Redirect manual: 3xx = falha, preserva HTTPS | TRANSCRICAO | [09:23] Sofia |
| FDD-FLUXO-10 | docs/FDD.md | Proposta (derivada) | Reabertura de PROCESSING no start (at-least-once) | TRANSCRICAO | [09:24] Diego |
| FDD-FLUXO-11 | docs/FDD.md | Fluxo | Shutdown gracioso espelhando server.ts | CODIGO | src/server.ts |
| FDD-FLUXO-12 | docs/FDD.md | Fluxo | Tabela de retry 1m/5m/30m/2h/12h = 14h36 | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-13 | docs/FDD.md | Fluxo | DLQ com payload, motivo, timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-14 | docs/FDD.md | Fluxo | Replay recoloca na outbox como pendente, mesmo eventId | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-15 | docs/FDD.md | Fluxo | Replay auditado (log + replayedById) | TRANSCRICAO | [09:36] Sofia |
| FDD-FLUXO-16 | docs/FDD.md | Fluxo | Rotação: anterior válida 24h em paralelo | TRANSCRICAO | [09:21] Sofia |
| FDD-FLUXO-17 | docs/FDD.md | Proposta (derivada) | Lanes por webhook: tick não espera lanes; cliente lento não atrasa os demais | TRANSCRICAO | [09:42] Diego |
| FDD-FLUXO-18 | docs/FDD.md | Proposta de implementação | Webhook inativo = pausado; eventos aguardam reativação | TRANSCRICAO | [09:21] Bruno |
| FDD-FLUXO-19 | docs/FDD.md | Proposta de implementação | Replay zera attempts; nova linha na DLQ se refalhar | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | E1 POST /webhooks (url, events, secret gerada) | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | customerId no body, não do JWT | TRANSCRICAO | [09:32] Larissa |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | E2 GET /customers/:customerId/webhooks (customer_id no path) | TRANSCRICAO | [09:32] Larissa |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | E3 PATCH /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | E4 DELETE /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | E5 rotate-secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | E6 GET /webhooks/:id/deliveries (últimos 100) | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | E7 POST /admin/webhooks/dead-letter/:id/replay (ADMIN) | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | CRUD para qualquer papel autenticado | TRANSCRICAO | [09:37] Sofia |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | Headers X-Event-Id, X-Signature, X-Timestamp, Content-Type | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-11 | docs/FDD.md | Contrato | Header X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| FDD-CONTRATO-12 | docs/FDD.md | Contrato | Payload: event_id, event_type, timestamp, order_*, status, customer_id, total_cents | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-13 | docs/FDD.md | Contrato | 2xx em ≤10s = entregue; demais = falha | TRANSCRICAO | [09:42] Diego |
| FDD-CONTRATO-14 | docs/FDD.md | Contrato | Prefixo /api/v1 nas rotas | CODIGO | src/app.ts |
| FDD-CONTRATO-15 | docs/FDD.md | Contrato | Respostas paginadas via paginated() | CODIGO | src/shared/http/response.ts |
| FDD-CONTRATO-16 | docs/FDD.md | Contrato | Filtro de eventos sem PENDING (nenhuma transição leva a PENDING) | CODIGO | src/modules/orders/order.status.ts |
| FDD-CONTRATO-17 | docs/FDD.md | Contrato | Formato order_number ORD-000000 | CODIGO | src/modules/orders/order.service.ts |
| FDD-CONTRATO-18 | docs/FDD.md | Contrato | pageSize máx. 100 como em listOrdersQuerySchema | CODIGO | src/modules/orders/order.schemas.ts |
| FDD-CONTRATO-19 | docs/FDD.md | Proposta de implementação | Path rotate-secret, DELETE em cascata e 202 no replay | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-20 | docs/FDD.md | Proposta de implementação | Prefixo sha256= e duas assinaturas na carência (sujeito à revisão) | TRANSCRICAO | [09:46] Sofia |
| FDD-ERRO-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL (em details de VALIDATION_ERROR) | TRANSCRICAO | [09:23] Sofia |
| FDD-ERRO-03 | docs/FDD.md | Erro (proposto, derivado da regra citada) | WEBHOOK_INVALID_EVENT_FILTER | TRANSCRICAO | [09:33] Marcos |
| FDD-ERRO-04 | docs/FDD.md | Erro (proposto, derivado da regra citada) | WEBHOOK_DEAD_LETTER_NOT_FOUND | TRANSCRICAO | [09:18] Diego |
| FDD-ERRO-05 | docs/FDD.md | Erro (proposto, derivado da regra citada) | WEBHOOK_SECRET_ROTATION_IN_PROGRESS | TRANSCRICAO | [09:21] Sofia |
| FDD-ERRO-06 | docs/FDD.md | Erro (proposto, derivado da regra citada) | WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED | TRANSCRICAO | [09:18] Diego |
| FDD-ERRO-07 | docs/FDD.md | Erro (proposto, derivado da regra citada) | WEBHOOK_INACTIVE no replay de webhook inativo/removido | TRANSCRICAO | [09:21] Bruno |
| FDD-ERRO-08 | docs/FDD.md | Erro (proposto, derivado da regra citada) | WEBHOOK_OUTBOX_INSERT_FAILED → rollback | TRANSCRICAO | [09:40] Bruno |
| FDD-ERRO-09 | docs/FDD.md | Erro (proposto, derivado da regra citada) | WEBHOOK_DELIVERY_TIMEOUT | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-10 | docs/FDD.md | Erro (proposto, derivado da regra citada) | WEBHOOK_DELIVERY_HTTP_ERROR / NETWORK_ERROR (cliente offline) | TRANSCRICAO | [09:14] Larissa |
| FDD-ERRO-11 | docs/FDD.md | Erro (proposto, derivado da regra citada) | WEBHOOK_PAYLOAD_TOO_LARGE | TRANSCRICAO | [09:24] Larissa |
| FDD-ERRO-12 | docs/FDD.md | Decisão (derivada) | WEBHOOK_SECRET_REQUIRED (exemplo do Bruno) não adotado: nunca dispararia | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-13 | docs/FDD.md | Erro (proposto, derivado da regra citada) | WEBHOOK_MAX_ATTEMPTS_EXCEEDED | TRANSCRICAO | [09:15] Diego |
| FDD-ERRO-14 | docs/FDD.md | Erro | Classes estendem AppError/ConflictError; NotFoundError não aceita código | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERRO-15 | docs/FDD.md | Erro | Códigos de validação aparecem em details por causa do validate() | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-RES-01 | docs/FDD.md | Resiliência | Fallback por e-mail adiado; DLQ + replay | TRANSCRICAO | [09:37] Larissa |
| FDD-RES-02 | docs/FDD.md | Resiliência | Queda da API não afeta worker | TRANSCRICAO | [09:11] Diego |
| FDD-RES-03 | docs/FDD.md | Resiliência | Cliente lento não bloqueia outros: lanes por webhook (proposta) | TRANSCRICAO | [09:02] Marcos |
| FDD-RES-04 | docs/FDD.md | Resiliência | Worker trata os próprios erros (errorMiddleware só cobre Express) | CODIGO | src/middlewares/error.middleware.ts |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Sem lib nova: logs Pino como base de métricas | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Métrica de latência p95 vs. meta de 10s | TRANSCRICAO | [09:02] Marcos |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Métrica de eventos/webhook/minuto para decidir rate limiting | TRANSCRICAO | [09:39] Larissa |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | Log de replay com autor | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-05 | docs/FDD.md | Observabilidade | redactPaths acrescido de secret/previousSecret/signature | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-06 | docs/FDD.md | Observabilidade | Correlação com requestId/http_request existente | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-OBS-07 | docs/FDD.md | Proposta (derivada) | eventId como trace id ponta a ponta | TRANSCRICAO | [09:25] Diego |
| FDD-OBS-08 | docs/FDD.md | Proposta de implementação | Limiares de alerta sugeridos (idade >10s por 5 min; nova entrada na DLQ) | TRANSCRICAO | [09:02] Marcos |
| FDD-DADOS-01 | docs/FDD.md | Modelo de dados | Convenções: uuid Char(36), camelCase, @@map snake_case | CODIGO | prisma/schema.prisma |
| FDD-DADOS-02 | docs/FDD.md | Modelo de dados | Tabela webhook_outbox | TRANSCRICAO | [09:06] Diego |
| FDD-DADOS-03 | docs/FDD.md | Modelo de dados | Índices status/created_at | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-04 | docs/FDD.md | Modelo de dados | Tabela de configuração url+secret+customer+ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-DADOS-05 | docs/FDD.md | Modelo de dados | webhook_deliveries guarda resultado, resposta, tempo | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-06 | docs/FDD.md | Modelo de dados | webhook_dead_letter | TRANSCRICAO | [09:18] Diego |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus chama publishWebhookEvent(tx, …) | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Filtro válido derivado das transições | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | WebhookNotFoundError estende AppError | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-04 | docs/FDD.md | Integração | ConflictError com código WEBHOOK_* | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-05 | docs/FDD.md | Integração | errorMiddleware sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | validate() com superRefine | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração | authenticate + requireRole('ADMIN') | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Montagem de /customers/:customerId/webhooks (mergeParams), /webhooks e /admin/webhooks | CODIGO | src/routes/index.ts |
| FDD-INT-09 | docs/FDD.md | Integração | buildControllers instancia WebhookController e WebhookAdminController | CODIGO | src/app.ts |
| FDD-INT-10 | docs/FDD.md | Integração | src/worker.ts replica bootstrap de server.ts | CODIGO | src/server.ts |
| FDD-INT-11 | docs/FDD.md | Integração | Worker importa o singleton prisma (instância própria por processo) | CODIGO | src/config/database.ts |
| FDD-INT-12 | docs/FDD.md | Integração | env.ts exige JWT_SECRET também no worker | CODIGO | src/config/env.ts |
| FDD-INT-13 | docs/FDD.md | Integração | logger.child + redactPaths | CODIGO | src/shared/logger/index.ts |
| FDD-INT-14 | docs/FDD.md | Integração | Novos modelos + relação reversa em Customer | CODIGO | prisma/schema.prisma |
| FDD-INT-15 | docs/FDD.md | Integração | Scripts worker/start:worker espelhando dev/start | CODIGO | package.json |
| FDD-INT-16 | docs/FDD.md | Integração | Limpeza das novas tabelas no beforeEach (ordem de FK) | CODIGO | tests/setup.ts |
| FDD-INT-17 | docs/FDD.md | Integração | Factory createTestWebhook | CODIGO | tests/helpers/factories.ts |
| FDD-DEP-01 | docs/FDD.md | Dependência | MySQL 8 com colunas Json | CODIGO | docker-compose.yml |
| FDD-DEP-02 | docs/FDD.md | Dependência | Revisão de segurança antes do deploy | TRANSCRICAO | [09:46] Sofia |
| FDD-DEP-03 | docs/FDD.md | Dependência | Portal documenta HMAC e dedup | TRANSCRICAO | [09:40] Marcos |
| FDD-CA-01 | docs/FDD.md | Critério de aceite técnico | 1 linha PENDING por webhook assinante, mesma transação | TRANSCRICAO | [09:06] Diego |
| FDD-CA-02 | docs/FDD.md | Critério de aceite técnico | Sem assinante → sem linha | TRANSCRICAO | [09:34] Bruno |
| FDD-CA-03 | docs/FDD.md | Critério de aceite técnico | Falha no INSERT → rollback completo | TRANSCRICAO | [09:41] Diego |
| FDD-CA-04 | docs/FDD.md | Critério de aceite técnico | Payload sem items | TRANSCRICAO | [09:43] Diego |
| FDD-CA-05 | docs/FDD.md | Critério de aceite técnico | Envio ≤2s com 5 headers | TRANSCRICAO | [09:44] Diego |
| FDD-CA-06 | docs/FDD.md | Critério de aceite técnico | X-Signature confere; duas assinaturas na carência | TRANSCRICAO | [09:22] Sofia |
| FDD-CA-07 | docs/FDD.md | Critério de aceite técnico | Retry 1m/5m/30m/2h/12h e DLQ | TRANSCRICAO | [09:17] Diego |
| FDD-CA-08 | docs/FDD.md | Critério de aceite técnico | >64KB → DLQ sem envio; webhook inativo pausado | TRANSCRICAO | [09:24] Diego |
| FDD-CA-09 | docs/FDD.md | Critério de aceite técnico | Replay ADMIN 202, OPERATOR 403, repetido 409 | TRANSCRICAO | [09:36] Sofia |
| FDD-CA-10 | docs/FDD.md | Critério de aceite técnico | http → 400 WEBHOOK_INVALID_URL | TRANSCRICAO | [09:23] Sofia |
| FDD-CA-11 | docs/FDD.md | Critério de aceite técnico (proposta derivada) | Secret só em E1/E5, nunca em log | CODIGO | src/shared/logger/index.ts |
| FDD-CA-12 | docs/FDD.md | Critério de aceite técnico | Anterior válida 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-CA-13 | docs/FDD.md | Critério de aceite técnico | Deliveries paginadas até 100 | TRANSCRICAO | [09:34] Marcos |
| FDD-CA-14 | docs/FDD.md | Critério de aceite técnico | Reabertura de PROCESSING, SIGTERM sem perda, uma lane por webhook | CODIGO | src/server.ts |
| FDD-CA-15 | docs/FDD.md | Critério de aceite técnico | Lint/test/migration passam; sem dependência nova | CODIGO | package.json |
| FDD-CA-16 | docs/FDD.md | Critério de aceite técnico | Revisão da Sofia aprovada | TRANSCRICAO | [09:46] Sofia |
| FDD-RISK-01 | docs/FDD.md | Risco | Muitos clientes lentos esgotam as lanes | TRANSCRICAO | [09:42] Diego |
| FDD-RISK-02 | docs/FDD.md | Risco | Ordem quebrada durante retry | TRANSCRICAO | [09:12] Diego |
| FDD-RISK-03 | docs/FDD.md | Risco | Transação de changeStatus mais longa | TRANSCRICAO | [09:04] Bruno |
| FDD-RISK-04 | docs/FDD.md | Risco | Secret vazando em log (histórico de vazamento citado) | TRANSCRICAO | [09:22] Diego |
| FDD-RISK-05 | docs/FDD.md | Risco | Crescimento das tabelas; arquivamento futuro | TRANSCRICAO | [09:08] Diego |
| FDD-RISK-06 | docs/FDD.md | Risco | CRUD aberto a qualquer autenticado | TRANSCRICAO | [09:37] Sofia |
