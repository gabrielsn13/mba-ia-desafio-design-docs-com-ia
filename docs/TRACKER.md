# Tracker

Rastreabilidade dos itens de `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` e `docs/adrs/` até a origem. Formato de **Localização**: `[hh:mm] Falante` para TRANSCRICAO; caminho do arquivo para CODIGO.

Itens de desenho **sem origem identificável** (marcados "Proposto" nos documentos) ficam na seção 2, fora da tabela principal, para não serem confundidos com decisões da reunião.

## 1. Itens com origem

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CTX-01 | docs/PRD.md | Contexto | Três clientes B2B (Atlas, MaxDistribuição, Nova Cargo) pediram notificação de mudança de status | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Contexto | Clientes fazem polling em GET /orders, o que é lento e caro | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Restrição | Atlas pode migrar para concorrente sem entrega até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-04 | docs/PRD.md | Requisito Não Funcional | "Tempo real" para o cliente = menos de 10 s | TRANSCRICAO | [09:02] Marcos |
| PRD-CTX-05 | docs/PRD.md | Escopo | Somente webhooks de saída (cliente não envia para nós) | TRANSCRICAO | [09:02] Marcos |
| PRD-CTX-06 | docs/PRD.md | Contexto | A aplicação não tem nenhum mecanismo de notificação externa; mudança de status concentrada em changeStatus | CODIGO | src/modules/orders/order.service.ts |
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Meta: tempo de notificação menor que 10 s (polling 2 s) | TRANSCRICAO | [09:10] Larissa |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Prazo de 3 sprints, incluindo revisão de segurança | TRANSCRICAO | [09:47] Larissa |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Não degradar mudanças de status (sem chamada externa na transação) | TRANSCRICAO | [09:04] Bruno |
| PRD-USO-01 | docs/PRD.md | Cenário | Usuários do sistema que representam o cliente usam a API com JWT | TRANSCRICAO | [09:32] Marcos |
| PRD-USO-02 | docs/PRD.md | Cenário | Cliente consulta histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-USO-03 | docs/PRD.md | Cenário | Marcos documenta a integração no portal de desenvolvedor | TRANSCRICAO | [09:40] Marcos |
| PRD-OOS-01 | docs/PRD.md | Fora de escopo | E-mail de alerta de falha adiado para próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de escopo | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de escopo | Dashboard visual descartado nesta fase | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de escopo | Arquivamento da outbox após cerca de 30 dias fora do escopo | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-05 | docs/PRD.md | Fora de escopo | Multi-worker e ordering global fora do escopo | TRANSCRICAO | [09:13] Larissa |
| PRD-OOS-06 | docs/PRD.md | Fora de escopo | Exactly-once descartado | TRANSCRICAO | [09:25] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook (url, status de interesse); secret gerada e devolvida | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | customer_id informado na requisição, não vem do JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Editar (PATCH) e remover (DELETE) webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de status por webhook, aplicado na inserção | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Evento consistente com a mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | 5 tentativas com backoff e depois DLQ | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Assinar envios com HMAC-SHA256, secret por webhook | TRANSCRICAO | [09:22] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace de 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Id único por evento (X-Event-Id) para dedup | TRANSCRICAO | [09:25] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Replay de DLQ restrito a ADMIN, com log de quem executou | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Apenas URLs https | TRANSCRICAO | [09:23] Sofia |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Payload enxuto, sem items | TRANSCRICAO | [09:43] Diego |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Garantia at-least-once | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 s por chamada | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Limite de 64 KB, erro sem truncar | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Isolamento: worker em processo separado | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Ordering só por pedido e single-worker | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Sem infraestrutura nova | TRANSCRICAO | [09:07] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Revisão de segurança de 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança da Sofia | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Cliente precisa deduplicar por X-Event-Id | TRANSCRICAO | [09:25] Sofia |
| PRD-DEP-03 | docs/PRD.md | Dependência | Dashboard é projeto separado do time de frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-RSK-01 | docs/PRD.md | Risco | Prazo da Atlas (fim de novembro) frente a 3 sprints | TRANSCRICAO | [09:45] Marcos |
| PRD-RSK-02 | docs/PRD.md | Risco | Vazamento de secret (já ocorreu com cliente) | TRANSCRICAO | [09:22] Diego |
| PRD-RSK-03 | docs/PRD.md | Risco | Cliente indisponível por longos períodos (já houve 2 h) | TRANSCRICAO | [09:16] Diego |
| PRD-RSK-04 | docs/PRD.md | Risco | Rajada de 50 eventos por minuto para um cliente | TRANSCRICAO | [09:38] Diego |
| PRD-RSK-05 | docs/PRD.md | Risco | Falha na outbox causa rollback da mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-TST-01 | docs/PRD.md | Teste | Testes ponta a ponta da integração no order.service | TRANSCRICAO | [09:46] Larissa |
| PRD-TST-02 | docs/PRD.md | Teste | Testes existentes como referência (Vitest) | CODIGO | tests/orders.test.ts |
| RFC-META-01 | docs/RFC.md | Metadado | Autora Larissa; abre o documento de design | TRANSCRICAO | [09:50] Larissa |
| RFC-META-02 | docs/RFC.md | Metadado | Bruno e Diego revisam com Larissa; Sofia e Marcos como demais participantes | TRANSCRICAO | [09:50] Larissa |
| RFC-CTX-01 | docs/RFC.md | Contexto | Transação de changeStatus já pesada (orders, history, estoque) | TRANSCRICAO | [09:04] Bruno |
| RFC-CTX-02 | docs/RFC.md | Contexto | changeStatus com $transaction | CODIGO | src/modules/orders/order.service.ts |
| RFC-PROP-01 | docs/RFC.md | Decisão | Outbox na mesma transação de changeStatus | TRANSCRICAO | [09:06] Diego |
| RFC-PROP-02 | docs/RFC.md | Decisão | Worker separado com polling de 2 s | TRANSCRICAO | [09:10] Larissa |
| RFC-PROP-03 | docs/RFC.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| RFC-PROP-04 | docs/RFC.md | Decisão | Retry 5x com backoff e DLQ com replay | TRANSCRICAO | [09:17] Larissa |
| RFC-PROP-05 | docs/RFC.md | Decisão | HMAC-SHA256, secret por endpoint, grace de 24 h | TRANSCRICAO | [09:22] Sofia |
| RFC-PROP-06 | docs/RFC.md | Decisão | Reuso de AppError, Pino e padrão de módulos | TRANSCRICAO | [09:30] Larissa |
| RFC-ALT-01 | docs/RFC.md | Alternativa | Chamada HTTP síncrona descartada (trava transação, rollback) | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa | Redis Streams descartado (infra nova) | TRANSCRICAO | [09:07] Larissa |
| RFC-ALT-03 | docs/RFC.md | Alternativa | Trigger de banco descartado (não notifica processo externo) | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa | Retry indefinido descartado (evento pendurado) | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa | 3 tentativas descartado (cobre só 30 min) | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa | Exactly-once descartado (coordenação complexa) | TRANSCRICAO | [09:25] Diego |
| RFC-OPEN-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída: observar e decidir | TRANSCRICAO | [09:39] Larissa |
| RFC-OPEN-02 | docs/RFC.md | Questão em aberto | Escalar para múltiplos workers (particionar ou lock) | TRANSCRICAO | [09:13] Diego |
| RFC-OPEN-03 | docs/RFC.md | Questão em aberto | E-mail de alerta ao cliente na próxima fase | TRANSCRICAO | [09:37] Larissa |
| RFC-OPEN-04 | docs/RFC.md | Questão em aberto | Arquivamento de eventos entregues sem política definida | TRANSCRICAO | [09:08] Diego |
| RFC-OPEN-05 | docs/RFC.md | Questão em aberto | HMAC e geração de secret a rever com a Sofia | TRANSCRICAO | [09:46] Sofia |
| RFC-IMP-01 | docs/RFC.md | Impacto | Alteração em módulo existente: order.service.ts | CODIGO | src/modules/orders/order.service.ts |
| RFC-IMP-02 | docs/RFC.md | Impacto | Registro de rotas em buildApiRouter | CODIGO | src/routes/index.ts |
| RFC-IMP-03 | docs/RFC.md | Impacto | Novo entry-point src/worker.ts e npm run worker | TRANSCRICAO | [09:11] Larissa |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox no MySQL, atômica com a mudança de status | TRANSCRICAO | [09:08] Larissa |
| ADR-001-A | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Índices em status e created_at; estados pendente/processando/falhou/entregue | TRANSCRICAO | [09:08] Diego |
| ADR-001-B | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | UUID como chave primária, como no resto do projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-001-C | docs/adrs/ADR-001-outbox-no-mysql.md | Contexto | IDs em schema.prisma usam uuid() Char(36) | CODIGO | prisma/schema.prisma |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-e-dlq.md | Decisão | 5 tentativas, backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| ADR-002-A | docs/adrs/ADR-002-retry-backoff-e-dlq.md | Decisão | DLQ em tabela separada webhook_dead_letter | TRANSCRICAO | [09:18] Diego |
| ADR-002-B | docs/adrs/ADR-002-retry-backoff-e-dlq.md | Decisão | Replay manual via POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18] Diego |
| ADR-002-C | docs/adrs/ADR-002-retry-backoff-e-dlq.md | Decisão | Replay exige ADMIN via requireRole e registra quem executou | TRANSCRICAO | [09:36] Larissa |
| ADR-002-D | docs/adrs/ADR-002-retry-backoff-e-dlq.md | Decisão | Timeout de 10 s conta como falha | TRANSCRICAO | [09:42] Diego |
| ADR-002-E | docs/adrs/ADR-002-retry-backoff-e-dlq.md | Contexto | requireRole existente | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, grace de 24 h | TRANSCRICAO | [09:22] Sofia |
| ADR-003-A | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Decisão | Assinatura no header X-Signature | TRANSCRICAO | [09:20] Sofia |
| ADR-003-B | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Alternativa | Secret global descartada (vaza uma, vaza tudo) | TRANSCRICAO | [09:21] Sofia |
| ADR-003-C | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Decisão | URL https validada no schema Zod | TRANSCRICAO | [09:23] Sofia |
| ADR-003-D | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Decisão | Headers X-Timestamp e X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| ADR-004 | docs/adrs/ADR-004-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id (UUID gerado na outbox) | TRANSCRICAO | [09:26] Larissa |
| ADR-004-A | docs/adrs/ADR-004-at-least-once-com-x-event-id.md | Alternativa | Exactly-once descartado | TRANSCRICAO | [09:25] Diego |
| ADR-004-B | docs/adrs/ADR-004-at-least-once-com-x-event-id.md | Restrição | Ordering só por order_id e com single-worker | TRANSCRICAO | [09:13] Larissa |
| ADR-004-C | docs/adrs/ADR-004-at-least-once-com-x-event-id.md | Trade-off | Responsabilidade de deduplicar fica com o cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-005 | docs/adrs/ADR-005-worker-separado-em-polling.md | Decisão | Polling a cada 2 s | TRANSCRICAO | [09:09] Diego |
| ADR-005-A | docs/adrs/ADR-005-worker-separado-em-polling.md | Decisão | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| ADR-005-B | docs/adrs/ADR-005-worker-separado-em-polling.md | Decisão | Entry-point src/worker.ts e lógica em webhook.worker.ts ou webhook.processor.ts | TRANSCRICAO | [09:28] Bruno |
| ADR-005-C | docs/adrs/ADR-005-worker-separado-em-polling.md | Decisão | PrismaClient próprio no worker, mesma DATABASE_URL | TRANSCRICAO | [09:30] Bruno |
| ADR-005-D | docs/adrs/ADR-005-worker-separado-em-polling.md | Alternativa | Trigger de banco descartado | TRANSCRICAO | [09:09] Diego |
| ADR-005-E | docs/adrs/ADR-005-worker-separado-em-polling.md | Contexto | server.ts como modelo de entry-point | CODIGO | src/server.ts |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Reuso máximo: AppError, Pino, error middleware, módulos, Zod | TRANSCRICAO | [09:30] Larissa |
| ADR-006-A | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Módulo src/modules/webhooks com controller, service, repository, routes e schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006-B | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisão | Prefixo WEBHOOK_ nos códigos de erro | TRANSCRICAO | [09:29] Larissa |
| ADR-006-C | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Contexto | AppError e subclasses | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-D | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Contexto | Subclasses HTTP (NotFoundError, ConflictError etc.) | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-E | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Contexto | Middleware de erro trata AppError, Zod e Prisma | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-F | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Contexto | Logger Pino compartilhado | CODIGO | src/shared/logger/index.ts |
| ADR-006-G | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Restrição | CRUD com qualquer role autenticada; endurecer depois | TRANSCRICAO | [09:37] Sofia |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-e-publish-webhook-event.md | Decisão | publishWebhookEvent(tx, order, from, to) recebe o tx | TRANSCRICAO | [09:41] Bruno |
| ADR-007-A | docs/adrs/ADR-007-snapshot-do-payload-e-publish-webhook-event.md | Decisão | Payload renderizado (snapshot) na inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-007-B | docs/adrs/ADR-007-snapshot-do-payload-e-publish-webhook-event.md | Decisão | Campos do payload e sem items | TRANSCRICAO | [09:43] Diego |
| ADR-007-C | docs/adrs/ADR-007-snapshot-do-payload-e-publish-webhook-event.md | Restrição | Limite de 64 KB com erro | TRANSCRICAO | [09:24] Larissa |
| ADR-007-D | docs/adrs/ADR-007-snapshot-do-payload-e-publish-webhook-event.md | Contexto | TxClient já definido em order.service.ts | CODIGO | src/modules/orders/order.service.ts |
| FDD-FLX-01 | docs/FDD.md | Fluxo | Inserção na outbox dentro do $transaction após update e history | TRANSCRICAO | [09:40] Bruno |
| FDD-FLX-02 | docs/FDD.md | Fluxo | Filtro de status na inserção; sem interessados, não insere | TRANSCRICAO | [09:34] Bruno |
| FDD-FLX-03 | docs/FDD.md | Fluxo | Rollback se a outbox falhar | TRANSCRICAO | [09:40] Bruno |
| FDD-FLX-04 | docs/FDD.md | Fluxo | Worker lê pendentes mais antigos em batch pequeno | TRANSCRICAO | [09:08] Diego |
| FDD-FLX-05 | docs/FDD.md | Fluxo | Timeout de 10 s no envio | TRANSCRICAO | [09:42] Diego |
| FDD-FLX-06 | docs/FDD.md | Fluxo | Tabela de backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-FLX-07 | docs/FDD.md | Fluxo | Esgotadas as tentativas, vai para webhook_dead_letter (payload, motivo, timestamp) | TRANSCRICAO | [09:18] Diego |
| FDD-FLX-08 | docs/FDD.md | Fluxo | Replay recoloca na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-FLX-09 | docs/FDD.md | Fluxo | Rotação: antiga válida 24 h em paralelo | TRANSCRICAO | [09:21] Sofia |
| FDD-DAT-01 | docs/FDD.md | Modelo de dados | webhook_endpoints guarda url, secret, customer_id e estado ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-DAT-02 | docs/FDD.md | Modelo de dados | webhook_outbox com índices em status e created_at | TRANSCRICAO | [09:08] Diego |
| FDD-DAT-03 | docs/FDD.md | Modelo de dados | Tabelas com UUID Char(36) | CODIGO | prisma/schema.prisma |
| FDD-DAT-04 | docs/FDD.md | Modelo de dados | Lista de status por webhook (filtro de eventos) | TRANSCRICAO | [09:33] Marcos |
| FDD-CTR-01 | docs/FDD.md | Contrato | POST /webhooks (cadastro, secret devolvida na criação) | TRANSCRICAO | [09:31] Marcos |
| FDD-CTR-02 | docs/FDD.md | Contrato | GET /webhooks listagem por customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CTR-03 | docs/FDD.md | Contrato | PATCH /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CTR-04 | docs/FDD.md | Contrato | DELETE /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CTR-05 | docs/FDD.md | Contrato | Endpoint de rotação de secret pela API | TRANSCRICAO | [09:21] Sofia |
| FDD-CTR-06 | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries (últimos 100) | TRANSCRICAO | [09:34] Marcos |
| FDD-CTR-07 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay, ADMIN | TRANSCRICAO | [09:35] Diego |
| FDD-CTR-08 | docs/FDD.md | Contrato | Rotas montadas sob /api/v1 | CODIGO | src/app.ts |
| FDD-CTR-09 | docs/FDD.md | Contrato | DELETE responde 204 sem corpo | CODIGO | src/modules/customers/customer.controller.ts |
| FDD-CTR-10 | docs/FDD.md | Contrato | Formato de erro { error: { code, message, details } } | CODIGO | src/middlewares/error.middleware.ts |
| FDD-CTR-11 | docs/FDD.md | Contrato | Listagem paginada no formato paginated() | CODIGO | src/shared/http/response.ts |
| FDD-CTR-12 | docs/FDD.md | Contrato | Headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id, Content-Type | TRANSCRICAO | [09:44] Diego |
| FDD-CTR-13 | docs/FDD.md | Contrato | Corpo: event_id, event_type, timestamp, order_id, order_number, from/to_status, customer_id, total_cents | TRANSCRICAO | [09:43] Diego |
| FDD-ERR-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL (https obrigatório) | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-03 | docs/FDD.md | Erro | WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-04 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE (acima de 64 KB) | TRANSCRICAO | [09:24] Larissa |
| FDD-ERR-05 | docs/FDD.md | Erro | Erros estendem AppError, ConflictError etc. | CODIGO | src/shared/errors/http-errors.ts |
| FDD-RES-01 | docs/FDD.md | Resiliência | Timeout 10 s | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | Retry/backoff de 5 tentativas | TRANSCRICAO | [09:17] Larissa |
| FDD-RES-03 | docs/FDD.md | Resiliência | Isolamento: worker separado | TRANSCRICAO | [09:11] Diego |
| FDD-RES-04 | docs/FDD.md | Resiliência | Fallback por DLQ e replay; e-mail fora de escopo | TRANSCRICAO | [09:18] Diego |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs com Pino do projeto, sem logger novo | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Log de auditoria de quem executou o replay | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Lista de redact do logger | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | requestId atribuído em req.id | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-01 | docs/FDD.md | Integração | publishWebhookEvent chamada em changeStatus | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Novos models em schema.prisma e migration | CODIGO | prisma/schema.prisma |
| FDD-INT-03 | docs/FDD.md | Integração | Novas classes de erro WEBHOOK_* | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-04 | docs/FDD.md | Integração | authenticate e requireRole('ADMIN') | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | validate({ body, params, query }) com schemas Zod | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Registro do router em buildApiRouter e Controllers | CODIGO | src/routes/index.ts |
| FDD-INT-07 | docs/FDD.md | Integração | Instanciação em buildControllers | CODIGO | src/app.ts |
| FDD-INT-08 | docs/FDD.md | Integração | server.ts como modelo do worker | CODIGO | src/server.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Env com Zod; mesma DATABASE_URL | CODIGO | src/config/env.ts |
| FDD-INT-10 | docs/FDD.md | Integração | Node >= 20 e script worker | CODIGO | package.json |
| FDD-DEP-01 | docs/FDD.md | Dependência | Sem Redis ou broker | TRANSCRICAO | [09:07] Diego |
| FDD-ACE-01 | docs/FDD.md | Critério de aceite | Entrega em menos de 10 s | TRANSCRICAO | [09:02] Marcos |
| FDD-ACE-02 | docs/FDD.md | Critério de aceite | Revisão de segurança antes do deploy | TRANSCRICAO | [09:46] Sofia |
| FDD-ACE-03 | docs/FDD.md | Critério de aceite | Reiniciar a API não interrompe o worker | TRANSCRICAO | [09:11] Diego |
| FDD-RSK-01 | docs/FDD.md | Risco | Ordering ao escalar workers | TRANSCRICAO | [09:13] Diego |
| FDD-RSK-02 | docs/FDD.md | Risco | Duplicatas de evento | TRANSCRICAO | [09:25] Diego |
| FDD-RSK-03 | docs/FDD.md | Risco | Crescimento da outbox e carga de polling | TRANSCRICAO | [09:07] Bruno |

## 2. Itens propostos sem origem na reunião

Detalhes de desenho do FDD e do PRD necessários para implementar, **não discutidos na reunião nem presentes no código**. Devem ser validados na revisão do RFC.

| ID | Documento | Item proposto | Observação |
|---|---|---|---|
| FDD-PROP-01 | docs/FDD.md | Tabela `webhook_deliveries` | Necessária para sustentar o histórico pedido em [09:34] Marcos |
| FDD-PROP-02 | docs/FDD.md | Uma linha de outbox por endpoint interessado | Deriva da entrega por endpoint e do `X-Webhook-Id` |
| FDD-PROP-03 | docs/FDD.md | Colunas `next_attempt_at`, `previous_secret`, `previous_secret_expires_at` | Suporte ao backoff e ao grace de 24 h |
| FDD-PROP-04 | docs/FDD.md | Sucesso = resposta 2xx | Critério não definido na reunião |
| FDD-PROP-05 | docs/FDD.md | Tamanho de lote 10 | Reunião diz apenas "batch pequeno" [09:08] Diego |
| FDD-PROP-06 | docs/FDD.md | Recuperação de eventos `PROCESSING` no reinício | Comportamento de robustez |
| FDD-PROP-07 | docs/FDD.md | Interpretação de "5 tentativas" como 5 esperas | Ambiguidade a confirmar com Diego |
| FDD-PROP-08 | docs/FDD.md | Nomes de campos de request/response e do endpoint `rotate-secret` | Reunião define o conceito, não o caminho |
| FDD-PROP-09 | docs/FDD.md | Códigos WEBHOOK_INVALID_EVENTS, DEAD_LETTER_NOT_FOUND, REPLAY_FORBIDDEN, DELIVERY_FAILED, DELIVERY_TIMEOUT | Além dos citados em [09:28] Bruno |
| FDD-PROP-10 | docs/FDD.md | Métricas e tracing | Não discutidos |
| FDD-PROP-11 | docs/FDD.md | Formato `sha256=<hex>` de X-Signature | Formato não definido |
| FDD-PROP-12 | docs/FDD.md | Proteção da secret em repouso | Pendente de revisão de segurança |
| PRD-PROP-01 | docs/PRD.md | Metas "0 eventos perdidos" e redução de polling a definir | Não numeradas na reunião |
| PRD-PROP-02 | docs/PRD.md | Probabilidade e impacto dos riscos | Avaliação do PRD |
| PRD-PROP-03 | docs/PRD.md | Piloto com os 3 clientes e testes unitários e de integração específicos | Estratégia de testes sugerida |
| ADR-PROP-01 | docs/adrs/ADR-003-hmac-sha256-secret-por-endpoint.md | Secret precisa ser recuperável (não hash) | Consequência técnica de HMAC |
| ADR-PROP-02 | docs/adrs/ADR-001, ADR-004, ADR-006 | Alternativas plausíveis não discutidas (at-most-once, BullMQ, sem assinatura) | Marcadas como plausíveis nos ADRs |
