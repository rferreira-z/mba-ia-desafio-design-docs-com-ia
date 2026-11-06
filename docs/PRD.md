# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Product Manager** | Marcos |
| **Tech Lead** | Larissa |
| **Status** | Aprovado em reunião técnica; documentação em revisão |
| **Data** | 2026-11-06 |
| **Prazo-alvo** | Fim de novembro, pedido da Atlas ([09:45] Marcos). Estimativa de 3 sprints, já com a revisão de segurança ([09:47] Larissa) |
| **Documentos relacionados** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

---

## 1. Resumo e contexto da feature

O OMS vai **notificar automaticamente os clientes B2B, via webhook HTTPS, sempre que o status de um pedido deles mudar**. O cliente cadastra uma URL, escolhe quais status quer receber e passa a receber um evento assinado poucos segundos depois de cada mudança. Isso substitui a consulta periódica que esses clientes fazem hoje.

Hoje o OMS não tem nenhum mecanismo de notificação externa. A feature é **somente de saída**: a plataforma envia e o cliente recebe ([09:02] Marcos, [09:03] Sofia).

## 2. Problema e motivação

- **Pedido formal de três clientes B2B.** Atlas Comercial, MaxDistribuição e Nova Cargo querem ser avisados em tempo real quando o status dos seus pedidos muda ([09:00] Marcos).
- **A integração atual é lenta e cara para eles.** Os clientes ficam chamando `GET /orders` de tempos em tempos para descobrir se algo mudou ([09:00] Marcos).
- **Risco comercial.** A Atlas sinalizou que pode migrar para um concorrente se a feature não sair até o fim do trimestre ([09:00] Marcos). Depois, pediu a entrega até o fim de novembro ([09:45] Marcos).
- **Expectativa de "tempo real".** Para os clientes, qualquer coisa abaixo de 10 segundos já conta, desde que eles não precisem ficar atualizando manualmente ([09:02] Marcos).

## 3. Público-alvo e cenários de uso

### 3.1 Público-alvo

| Público | Necessidade | Origem |
|---|---|---|
| **Times de integração dos clientes B2B** (Atlas, MaxDistribuição, Nova Cargo) | Receber mudanças de status sem polling, com garantia de autenticidade | [09:00] Marcos, [09:19] Sofia |
| **Usuários da plataforma que representam o cliente** | Configurar webhooks pela API do OMS, autenticados com o JWT do sistema | [09:32] Marcos |
| **Administradores do OMS** (papel `ADMIN`) | Reprocessar eventos que falharam de vez, com trilha de auditoria | [09:36] Sofia |
| **Produto / portal do desenvolvedor** | Documentar a integração, a assinatura e a deduplicação | [09:26] Marcos, [09:40] Marcos |

### 3.2 Cenários de uso

Os cenários são **ilustrativos**. Eles combinam situações citadas na reunião com os clientes que fizeram o pedido. A reunião não atribuiu cada situação a um cliente específico.

| # | Cenário | Origem |
|---|---|---|
| C1 | Um usuário que representa um cliente (ex.: a Atlas) cadastra um webhook HTTPS para receber só `SHIPPED` e `DELIVERED`, e recebe a secret na resposta. | [09:31] Marcos, [09:33] Marcos |
| C2 | Um pedido da Atlas passa para `SHIPPED`. Em poucos segundos, o sistema da Atlas recebe o evento, valida o HMAC e descarta duplicatas pelo `X-Event-Id`. | [09:02] Marcos, [09:20] Sofia, [09:25] Diego |
| C3 | O endpoint de um cliente fica 2 horas fora do ar por manutenção planejada, como já aconteceu com um cliente. As retentativas automáticas entregam os eventos quando ele volta. | [09:16] Diego |
| C4 | Um cliente percebe que vazou a secret num log. Ele pede uma nova pela API e tem 24h para atualizar os sistemas, sem perder eventos. | [09:21] Sofia, [09:22] Diego |
| C5 | Um cliente diz que "não recebeu" um evento. Ele mesmo consulta o histórico de entregas: sucesso ou falha, payload, resposta e tempo de resposta. | [09:34] Marcos |
| C6 | O endpoint de um cliente ficou fora por mais tempo do que a janela de retentativas (≈ 14h36min) e o evento foi para a DLQ. Um `ADMIN` reprocessa o evento depois que o cliente se recupera. | [09:17] Marcos, [09:18] Diego |

## 4. Objetivos e métricas de sucesso

| # | Objetivo | Métrica | Meta | Origem |
|---|---|---|---|---|
| OBJ-1 | Notificação em "tempo real" | Latência entre a mudança de status e a entrega bem-sucedida (p95), para clientes disponíveis | **< 10 segundos** | [09:02] Marcos |
| OBJ-2 | Não perder mudanças de status | % de mudanças de status com webhook assinante que geram evento | **100%** (garantido pela atomicidade) | [09:40] Bruno, [09:41] Diego |
| OBJ-3 | Atender os clientes que pediram a feature | Clientes solicitantes com webhook ativo em produção | **3 de 3** (Atlas, MaxDistribuição, Nova Cargo), começando pela Atlas até o fim de novembro | [09:00] Marcos, [09:45] Marcos |
| OBJ-4 | Tolerar indisponibilidade do cliente | Janela coberta por retentativas automáticas antes da DLQ | **≈ 14h36min** (5 retentativas: 1m/5m/30m/2h/12h) | [09:17] Diego, [09:17] Marcos |
| OBJ-5 | Reduzir o polling dos clientes integrados | Chamadas a `GET /api/v1/orders` feitas pelos usuários que representam cada cliente ([09:32] Marcos), medidas pelo `userId` dos logs `http_request` | Tendência de queda. O vínculo de usuário para cliente é mapeado manualmente pelo PM, porque o sistema não tem essa relação. A linha de base e a meta numérica ficam com o PM, porque a reunião não fixou um número | [09:00] Marcos; `src/middlewares/request-logger.middleware.ts` |

Indicadores de acompanhamento, sem meta definida, que vão informar a próxima fase:

- volume de eventos na DLQ, que serve de insumo para decidir sobre o aviso por e-mail, "depois que a gente medir o impacto" ([09:37] Larissa);
- eventos por cliente por minuto, que serve de insumo para decidir sobre rate limiting ([09:39] Larissa).

## 5. Escopo

### 5.1 Incluso

- Cadastro, listagem, edição e remoção de webhooks por cliente, via API autenticada.
- Filtro de eventos por status de destino.
- Notificação automática a cada mudança de status assinada.
- Assinatura HMAC-SHA256, com secret por endpoint e rotação com 24h de carência.
- Retentativas automáticas, DLQ e replay manual por `ADMIN`.
- Histórico de entregas consultável pelo cliente.
- Documentação no portal do desenvolvedor, conduzida pelo produto ([09:26] Marcos, [09:40] Marcos).

### 5.2 Fora de escopo

| Item | Situação | Origem |
|---|---|---|
| **Aviso por e-mail quando o webhook do cliente falha repetidamente** | **Adiado** para a próxima fase, depois de medir o impacto | [09:37] Marcos, [09:37] Larissa, [09:38] Marcos |
| **Dashboard/painel visual para o cliente** | **Descartado** nesta entrega: só endpoints. É um projeto separado do time de frontend | [09:39] Marcos, [09:40] Larissa |
| **Rate limiting de envio por cliente** | **Adiado:** observar e decidir depois | [09:38] Diego, [09:39] Larissa |
| Webhooks de entrada (o cliente enviando para nós) | Descartado: os clientes só querem receber | [09:02] Marcos |
| Arquivamento de eventos antigos (~30 dias) | Fora do escopo desta feature | [09:08] Diego |
| Ordem global de eventos e escala para vários workers | Não é garantia. Fica para o futuro | [09:13] Larissa, [09:14] Marcos |
| Garantia exactly-once | Descartada em favor de at-least-once | [09:25] Diego |
| Itens do pedido no payload | Descartado: o cliente consulta `GET /orders/:id` | [09:43] Diego |
| `customer_id` derivado do JWT | Descartado: o JWT é do usuário operador, então o cliente é informado na requisição | [09:32] Bruno, [09:32] Larissa |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
|---|---|---|
| RF-01 | O sistema deve permitir **cadastrar um webhook** informando cliente (`customerId`, no body), URL e lista de status de interesse. A secret é **gerada pela plataforma** e devolvida na criação. | [09:31] Marcos, [09:32] Larissa |
| RF-02 | O sistema deve permitir **editar** um webhook (URL, status de interesse, ativo/inativo). | [09:33] Bruno |
| RF-03 | O sistema deve permitir **remover** um webhook. | [09:33] Bruno |
| RF-04 | O sistema deve permitir **listar os webhooks de um cliente**. | [09:33] Bruno |
| RF-05 | Cada webhook deve receber **apenas os status que assinou**. O filtro é aplicado no momento em que o evento é gerado. | [09:33] Marcos, [09:34] Bruno |
| RF-06 | A cada mudança de status de um pedido, o sistema deve **notificar automaticamente** todos os webhooks ativos do cliente que assinam o novo status. | [09:00] Marcos, [09:40] Bruno |
| RF-07 | Toda notificação deve levar uma **assinatura HMAC-SHA256** do corpo no header `X-Signature`, feita com a secret daquele endpoint. | [09:20] Sofia, [09:22] Sofia |
| RF-08 | O cliente deve conseguir **rotacionar a secret** pela API. A secret anterior continua válida por **24 horas**. | [09:21] Sofia |
| RF-09 | O cliente deve conseguir **consultar o histórico das últimas entregas** (até 100 por página) de um webhook, com sucesso ou falha, payload, resposta e tempo de resposta. | [09:34] Marcos |
| RF-10 | Notificações que falham devem ser **retentadas automaticamente** 5 vezes (1m, 5m, 30m, 2h, 12h). Depois disso, vão para uma **fila de falhas (DLQ)**. | [09:17] Larissa, [09:18] Diego |
| RF-11 | Um usuário **`ADMIN`** deve conseguir **reprocessar** um evento da DLQ. A ação fica registrada com o autor. | [09:18] Diego, [09:36] Sofia |
| RF-12 | Toda notificação deve levar um **identificador único e estável do evento**, para deduplicação, a **identificação do cadastro** que a recebeu e o **horário do envio**. O contrato está no [FDD §5.1](FDD.md#51-requisição-enviada-ao-cliente-outbound). | [09:25] Diego, [09:44] Diego, [09:44] Sofia |
| RF-13 | A notificação deve ser **enxuta**: dados de identificação do pedido e da transição de status, **sem os itens**. Quem precisar de detalhes consulta o pedido. | [09:43] Diego |

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
|---|---|---|
| RNF-01 | **Latência:** entrega em menos de 10s para clientes disponíveis. A espera do polling é de no máximo 2s. | [09:02] Marcos, [09:10] Larissa |
| RNF-02 | **Timeout:** o cliente tem 10s para responder. Depois disso, a tentativa conta como falha. | [09:42] Diego |
| RNF-03 | **Segurança de transporte:** só URLs `https://`. Uma URL `http://` é recusada com erro de validação. | [09:23] Sofia |
| RNF-04 | **Tamanho:** um evento com mais de 64KB não é enviado e conta como erro, sem truncamento. | [09:23] Sofia, [09:24] Diego, [09:24] Larissa |
| RNF-05 | **Garantia de entrega:** at-least-once. Duplicatas são possíveis e o cliente as identifica pelo `X-Event-Id`. | [09:24] Diego, [09:26] Larissa |
| RNF-06 | **Consistência:** não pode existir mudança de status sem o evento correspondente, nem evento de uma mudança que foi desfeita. | [09:06] Diego, [09:40] Bruno |
| RNF-07 | **Isolamento:** a lentidão ou a indisponibilidade de um cliente não pode bloquear nem desfazer mudanças de status. | [09:04] Bruno |
| RNF-08 | **Ordem:** a ordem é preservada apenas por pedido, enquanto houver um único processo de entrega. Isso é uma limitação conhecida. | [09:12] Diego, [09:13] Larissa |
| RNF-09 | **Segredos:** a secret é única por endpoint ([09:21] Sofia) e só é exibida ao cliente quando é gerada, na criação ([09:31] Marcos) ou na rotação. A regra de não aparecer em listagens nem em logs é uma **proposta derivada** do histórico de vazamento via log ([09:22] Diego). | [09:21] Sofia, [09:31] Marcos |
| RNF-10 | **Disponibilidade da entrega:** o envio roda em processo separado da API. Um restart da API não interrompe as entregas. | [09:11] Diego |
| RNF-11 | **Infraestrutura:** nenhuma infraestrutura nova. A feature usa o MySQL e a stack atuais. | [09:07] Diego, [09:30] Larissa |
| RNF-12 | **Auditoria:** o reprocessamento da DLQ é restrito a `ADMIN` e registra quem executou. | [09:36] Sofia |

## 8. Decisões e trade-offs principais

O detalhe de cada decisão está nos ADRs. Aqui fica só o impacto de produto.

| Decisão | Trade-off aceito pelo produto | ADR |
|---|---|---|
| Evento gravado junto com a mudança de status (outbox) | Nenhum evento perdido, em troca de uma escrita a mais por mudança de status | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Entrega por processo separado, verificando a cada 2s | Até 2s de espera, aceito por caber nos 10s ([09:10] Marcos) | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| 5 retentativas em ≈ 14h36min e depois DLQ | Cliente fora por mais tempo que isso exige reprocessamento manual. Aceito: "ele já tá com problema sério dele" ([09:17] Marcos) | [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| HMAC com secret por endpoint e rotação | Mais gestão de segredos, em troca de limitar o dano de um vazamento | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| At-least-once | O cliente precisa deduplicar. O produto documenta isso em destaque ([09:26] Marcos) | [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| Reuso dos padrões do OMS | Entrega mais rápida, sem stack nova de métricas | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) |
| Payload "foto" do momento da mudança, sem itens | O cliente consulta o pedido se precisar de mais detalhes | [ADR-007](adrs/ADR-007-snapshot-e-filtro-na-insercao-do-outbox.md) |

## 9. Dependências

| Dependência | Responsável | Origem |
|---|---|---|
| Revisão de segurança (HMAC e geração de secret), com **pelo menos 2 dias úteis** antes do deploy | Sofia | [09:46] Sofia, [09:49] Sofia |
| Sessão de revisão do design antes de codar | Larissa, Bruno, Diego | [09:50] Larissa |
| Documentação de integração no portal do desenvolvedor: HMAC, deduplicação, endpoints | Marcos | [09:26] Marcos, [09:40] Marcos |
| Confirmação do prazo com a Atlas e comunicação aos clientes | Marcos | [09:47] Marcos, [09:49] Marcos |
| Endpoint HTTPS disponível do lado de cada cliente | Clientes | [09:23] Sofia |
| Deploy e operação de um novo processo (`npm run worker`) além da API | Engenharia | [09:11] Larissa |

## 10. Riscos e mitigação

Escala: **Probabilidade** e **Impacto** em Baixo, Médio ou Alto.

| # | Risco | Prob. | Impacto | Mitigação | Origem |
|---|---|---|---|---|---|
| R1 | **Prazo:** a estimativa é de 3 sprints e a Atlas quer a feature no fim de novembro. A duração da sprint não foi informada. Se atrasar, a Atlas pode migrar para um concorrente | **Alta** | Alto | Escopo enxuto, sem e-mail, painel nem rate limiting. Revisão da Sofia agendada desde já. Marcos alinha o prazo com a Atlas | [09:00] Marcos, [09:45] Marcos, [09:46] Larissa, [09:47] Marcos |
| R2 | **Vazamento de secret pelo cliente**, que já aconteceu antes | Média | Alto | Secret por endpoint, rotação self-service com 24h de carência, secret fora de logs e listagens, revisão de segurança | [09:21] Sofia, [09:22] Diego |
| R3 | **Cliente não deduplica** e processa o mesmo evento duas vezes | Média | Médio | `X-Event-Id` estável, documentação em destaque no portal com exemplo de código | [09:25] Sofia, [09:26] Marcos |
| R4 | **Cliente fora por mais tempo que a janela de retentativas (≈ 14h36min)** e eventos na DLQ, sem aviso proativo (e-mail adiado) | Baixa | Médio | Histórico de entregas consultável, replay por `ADMIN`, monitoramento da DLQ. Reavaliar o e-mail na fase 2 | [09:16] Diego, [09:37] Larissa |
| R5 | **Rajada de eventos** (ex.: 50 em 1 minuto) sobrecarrega o endpoint do cliente | Baixa | Médio | Medir eventos por cliente por minuto e decidir sobre rate limiting com dados | [09:38] Diego, [09:39] Larissa |
| R6 | **Processo de entrega parado** atrasa todas as notificações | Baixa | Alto | Processo separado da API, eventos persistidos sem perda, alerta de evento pendente antigo | [09:11] Diego |
| R7 | **Eventos fora de ordem** para o mesmo pedido, quando um evento está em retentativa | Baixa | Baixo | Os clientes não exigem ordem global. O payload traz `timestamp` e `from_status`/`to_status` | [09:14] Marcos; [RFC Q6](RFC.md#5-questões-em-aberto) |
| R8 | **Configuração indevida:** qualquer usuário autenticado pode gerenciar webhooks de qualquer cliente nesta fase | Baixa | Médio | Aceito por ora. Endurecer as permissões numa fase seguinte | [09:37] Sofia |

## 11. Critérios de aceitação

| # | Critério |
|---|---|
| CA-01 | **Dado** um webhook ativo que assina `SHIPPED`, **quando** um pedido do cliente muda para `SHIPPED`, **então** o endpoint recebe uma notificação assinada e identificada (RF-07, RF-12) em menos de 10s, mesmo que outro cliente esteja lento. |
| CA-02 | **Dado** um webhook que assina só `DELIVERED`, **quando** o pedido muda para `SHIPPED`, **então** nenhuma notificação é enviada a esse webhook. |
| CA-03 | **Dado** um cadastro com URL `http://`, **então** a API recusa com erro de validação. |
| CA-04 | **Dado** um webhook criado, **então** a resposta da criação traz a secret e nenhuma listagem posterior a exibe. |
| CA-05 | **Dado** um endpoint fora do ar, **então** o sistema faz as 5 retentativas da política (RF-10). Se todas falharem, o evento vai para a DLQ. |
| CA-06 | **Dado** um evento na DLQ, **quando** um administrador pede o reprocessamento, **então** o evento é reenviado com o **mesmo** identificador e o autor fica registrado. **Quando** um usuário sem papel de administrador tenta, **então** recebe acesso negado. |
| CA-07 | **Dado** que o cliente rotacionou a secret, **então** nas 24h seguintes ele consegue validar as notificações com a secret anterior ou com a nova. |
| CA-08 | **Dado** um webhook com entregas, **então** o histórico mostra até 100 entregas por página, com resultado, payload, resposta e tempo de resposta. |
| CA-09 | **Dado** que o registro do evento falha, **então** a mudança de status do pedido também é desfeita. Nunca existe mudança de status sem evento. |
| CA-10 | **Dado** um endpoint lento ou fora do ar, **então** as mudanças de status de outros pedidos continuam sendo processadas normalmente. |
| CA-11 | A revisão de segurança da Sofia está aprovada antes do deploy. |

## 12. Estratégia de testes e validação

| Nível | O que valida | Responsável / como |
|---|---|---|
| Testes automatizados | Requisitos funcionais (§6) e critérios de aceitação (§11): cadastro, filtro, assinatura, retentativas, DLQ, replay, rotação, histórico | Engenharia, com a suíte de testes que o projeto já tem. O plano técnico, nos níveis unitário, integração e worker, está no [FDD §12](FDD.md#12-critérios-de-aceite-técnicos) |
| Segurança | Assinatura, geração e guarda da secret | Revisão da Sofia, com 2 dias úteis antes do deploy ([09:46] Sofia) |
| Revisão de design | Coerência entre a proposta, as decisões e o desenho | Sessão Larissa + Bruno + Diego antes de codar ([09:50] Larissa) |
| Validação de produto | Documentação de integração clara, com assinatura e deduplicação | Portal do desenvolvedor, conduzido por Marcos ([09:26] Marcos) |
| Pós-lançamento | Metas do §4 (latência p95, adoção pelos 3 clientes), volume de DLQ, eventos por minuto | Indicadores descritos no [FDD §8](FDD.md#8-observabilidade) |
