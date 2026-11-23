# FDD — Feature Design Document

> **Convenção de rastreabilidade:** `[hh:mm Nome]` indica origem na transcrição; `(CODIGO: caminho)` indica origem no código; **(Proposto)** indica detalhe de desenho deste FDD, necessário para implementar mas **não discutido na reunião**. Itens "Proposto" devem ser validados na revisão.

## 1. Contexto e motivação técnica

Clientes B2B precisam ser notificados em menos de 10 s quando o status de um pedido muda, hoje consultando `GET /orders` [09:00–09:02 Marcos]. A mudança de status ocorre em `OrderService.changeStatus` dentro de um `prisma.$transaction` que atualiza `orders`, insere em `order_status_history` e ajusta o estoque (CODIGO: `src/modules/orders/order.service.ts`). A proposta, a visão geral e as alternativas estão no [RFC](RFC.md); aqui está o "como construir". A aplicação monta rotas sob `/api/v1` (CODIGO: `src/app.ts`).

## 2. Objetivos técnicos

- Registrar o evento de forma atômica com a mudança de status [09:40 Bruno].
- Entregar em menos de 10 s no caso normal (polling de 2 s) [09:09–09:10].
- Entrega at-least-once, assinada e com retry/DLQ ([ADR-002](adrs/ADR-002-retry-backoff-e-dlq.md), [ADR-003](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md), [ADR-004](adrs/ADR-004-at-least-once-com-x-event-id.md)).
- Reutilizar os padrões do projeto ([ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)).

## 3. Escopo e exclusões

**Dentro:** CRUD de configuração de webhook, rotação de secret, outbox, worker, retry, DLQ, replay ADMIN, histórico de entregas.

**Fora (descartado ou adiado):** e-mail de alerta [09:37 Larissa]; rate limiting de saída, a observar [09:39 Larissa]; dashboard visual [09:40 Larissa]; arquivamento da outbox após 30 dias [09:08 Diego]; múltiplos workers e ordering global [09:13 Larissa]; webhooks de entrada (apenas saída) [09:02 Marcos].

## 4. Modelo de dados (Prisma)

Todas as tabelas usam UUID `@db.Char(36)`, como o restante de `prisma/schema.prisma` [09:51 Larissa]. Nomes de colunas além dos citados na transcrição são **(Proposto)**.

| Tabela | Colunas principais |
|---|---|
| `webhook_endpoints` | `id`, `customer_id` (FK `customers`), `url`, `secret`, `previous_secret` (nullable), `previous_secret_expires_at` (nullable), `events` (JSON: lista de `OrderStatus`), `active`, `created_at`, `updated_at` [09:21 Sofia, 09:31–09:33] |
| `webhook_outbox` | `id` (= `event_id`, UUID), `webhook_id` (FK), `order_id`, `payload` (JSON snapshot), `status` (`PENDING`, `PROCESSING`, `FAILED`, `DELIVERED`), `attempts`, `next_attempt_at`, `created_at`, `updated_at`; índices em `status` e `created_at` [09:08 Diego, 09:25 Diego, 09:52 Larissa] |
| `webhook_dead_letter` | `id`, `outbox_id`, `webhook_id`, `payload`, `failure_reason`, `failed_at` [09:18 Diego] |
| `webhook_deliveries` **(Proposto)** | `id`, `outbox_id`, `webhook_id`, `attempt`, `success`, `http_status`, `response_body` (truncado), `duration_ms`, `created_at`; sustenta `GET /webhooks/:id/deliveries`, que expõe sucesso/falha, payload, response e tempo de resposta [09:34 Marcos] |

A coluna `next_attempt_at` e a escolha de uma linha de outbox **por endpoint interessado** (e não por evento de pedido) são **(Proposto)**: a entrega, o retry e o `X-Webhook-Id` são por endpoint.

A secret precisa ser legível para assinar o corpo; como será protegida em repouso é **ponto aberto para a revisão de segurança da Sofia** [09:46 Sofia]. Por padrão, a secret nunca é logada (o logger tem lista de `redact`, CODIGO: `src/shared/logger/index.ts`, e deve ganhar os campos `secret` e `previous_secret` **(Proposto)**).

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox

1. `PATCH /orders/:id/status` chega em `OrderController.changeStatus` e chama `OrderService.changeStatus` (CODIGO: `src/modules/orders/order.routes.ts`).
2. Dentro do `$transaction`, depois de `tx.order.update` e `tx.orderStatusHistory.create`, chama-se `publishWebhookEvent(tx, order, from, to)` [09:41 Bruno].
3. `publishWebhookEvent` busca em `webhook_endpoints` os registros `active` do `customer_id` do pedido cujo `events` contém `to` [09:34 Bruno]. Se nenhum, **não insere nada**.
4. Para cada endpoint: monta o payload snapshot (seção 6.3), valida que tem no máximo **64 KB** (erro `WEBHOOK_PAYLOAD_TOO_LARGE` e rollback, "erra, não trunca") [09:23–09:24] e insere em `webhook_outbox` com `status = PENDING`, `attempts = 0`, `next_attempt_at = now()`; o `id` gerado é o `X-Event-Id`.
5. Qualquer falha de inserção lança exceção e a transação inteira sofre rollback [09:40 Bruno].

### 5.2 Processamento pelo worker

Entry-point `src/worker.ts`, script `npm run worker`, `PrismaClient` próprio [09:11, 09:30]. Laço a cada 2 s [09:09]:

1. Seleciona um lote pequeno de eventos `PENDING` com `next_attempt_at <= now()`, ordenados por `created_at` [09:08, 09:12]. Tamanho do lote **(Proposto)**: 10, configurável.
2. Marca cada evento como `PROCESSING`.
3. Carrega o endpoint; monta os headers (seção 6.4); assina o corpo com HMAC-SHA256 usando a secret atual.
4. Faz `POST` na `url` com **timeout de 10 s** [09:42 Diego].
5. **Sucesso** **(Proposto: qualquer resposta 2xx)**: grava em `webhook_deliveries`, marca `DELIVERED`.
6. **Falha** (não 2xx, timeout ou erro de rede): grava em `webhook_deliveries` e segue para 5.3.
7. Eventos `PROCESSING` presos após reinício do worker são devolvidos a `PENDING` na inicialização **(Proposto)**; isso é compatível com at-least-once e pode gerar duplicata.
8. Em `SIGINT`/`SIGTERM`, o worker termina o lote corrente e chama `prisma.$disconnect()`, como em `src/server.ts` **(Proposto)**.

### 5.3 Retry

Falhas incrementam `attempts` e reagendam `next_attempt_at` conforme a tabela, voltando o evento a `PENDING` [09:17 Larissa/Diego]:

| Falha nº | Espera até a próxima tentativa |
|---|---|
| 1 | 1 minuto |
| 2 | 5 minutos |
| 3 | 30 minutos |
| 4 | 2 horas |
| 5 | 12 horas |

São 5 tentativas no total [09:15–09:17]. A interpretação exata de "5 tentativas" (1 envio inicial mais 4 retries contra 5 retries) não foi detalhada na reunião; esta tabela assume **5 esperas após a primeira falha**, o que dá cerca de 15 h entre a primeira falha e a última tentativa, como citado [09:17 Diego]. **(Proposto; validar com Diego.)**

### 5.4 DLQ e replay

- Esgotado o limite, o evento é copiado para `webhook_dead_letter` com payload, motivo e timestamp e removido da fila ativa (`FAILED` na outbox) [09:18 Diego].
- `POST /admin/webhooks/dead-letter/:id/replay` (role `ADMIN`) recoloca o evento na outbox como `PENDING`, com `attempts = 0`, e registra em log o `userId` que executou o replay [09:18, 09:36 Sofia].

### 5.5 Rotação de secret

`POST /webhooks/:id/rotate-secret` gera nova secret, guarda a atual em `previous_secret` com `previous_secret_expires_at = now() + 24h` e devolve a nova uma única vez [09:21–09:22 Sofia]. Durante o grace period, a assinatura usa a secret nova. Como o cliente valida a assinatura, ele pode aceitar qualquer uma das duas nesse período; o worker **não** envia duas assinaturas **(Proposto)**. Depois de 24 h, `previous_secret` é descartada pelo worker ou na leitura **(Proposto)**.

## 6. Contratos públicos

Todas as rotas ficam sob `/api/v1`, exigem `authenticate` e usam o formato de erro `{ error: { code, message, details? } }` (CODIGO: `src/middlewares/error.middleware.ts`). Os caminhos seguem a transcrição, e os nomes de campos de request/response são **(Proposto)**. O `customer_id` é informado no body ou na query, **não** vem do JWT [09:32 Larissa].

### 6.1 `POST /api/v1/webhooks`: cadastrar webhook

Requer qualquer role autenticada [09:36–09:37].

Request:
```json
{
  "customerId": "5b0f7c0e-2d5e-4a41-9d2c-6a1f0e9a7c11",
  "url": "https://api.atlas.example.com/hooks/orders",
  "events": ["SHIPPED", "DELIVERED"]
}
```
Response `201`:
```json
{
  "id": "c1b2e0a4-7f3a-4b4e-9a8d-1d2c3e4f5a6b",
  "customerId": "5b0f7c0e-2d5e-4a41-9d2c-6a1f0e9a7c11",
  "url": "https://api.atlas.example.com/hooks/orders",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_example_returned_only_on_creation",
  "createdAt": "2026-11-23T18:00:00.000Z"
}
```
Status: `201` criado; `400` `VALIDATION_ERROR`/`WEBHOOK_INVALID_URL`; `401`; `404` customer inexistente.

### 6.2 `GET /api/v1/webhooks?customerId=...`: listar webhooks do customer

Response `200` (paginado, como `paginated()` em `src/shared/http/response.ts`):
```json
{
  "data": [
    {
      "id": "c1b2e0a4-7f3a-4b4e-9a8d-1d2c3e4f5a6b",
      "customerId": "5b0f7c0e-2d5e-4a41-9d2c-6a1f0e9a7c11",
      "url": "https://api.atlas.example.com/hooks/orders",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```
A secret **não** é devolvida na listagem **(Proposto)**. Status: `200`, `400`, `401`.

### 6.3 `PATCH /api/v1/webhooks/:id`: editar webhook

Request:
```json
{ "url": "https://api.atlas.example.com/hooks/v2", "events": ["PAID", "SHIPPED"], "active": true }
```
Response `200`: o objeto do webhook (como em 6.2, sem secret). Status: `200`; `400` `WEBHOOK_INVALID_URL`/`VALIDATION_ERROR`; `401`; `404` `WEBHOOK_NOT_FOUND`.

### 6.4 `DELETE /api/v1/webhooks/:id`: remover webhook

Response `204` sem corpo (mesmo padrão de `src/modules/customers/customer.controller.ts`). Status: `204`, `401`, `404` `WEBHOOK_NOT_FOUND`.

### 6.5 `POST /api/v1/webhooks/:id/rotate-secret`: rotacionar secret

Response `200`:
```json
{
  "id": "c1b2e0a4-7f3a-4b4e-9a8d-1d2c3e4f5a6b",
  "secret": "whsec_example_new_secret",
  "previousSecretExpiresAt": "2026-11-24T18:00:00.000Z"
}
```
Status: `200`, `401`, `404` `WEBHOOK_NOT_FOUND`.

### 6.6 `GET /api/v1/webhooks/:id/deliveries`: histórico de entregas

Retorna as últimas 100 entregas [09:34 Marcos]. Response `200`:
```json
{
  "data": [
    {
      "eventId": "9f1c2a30-3b4d-4e5f-8a6b-7c8d9e0f1a2b",
      "attempt": 1,
      "success": false,
      "httpStatus": 503,
      "responseBody": "Service Unavailable",
      "durationMs": 1240,
      "payload": { "event_type": "order.status_changed" },
      "createdAt": "2026-11-23T18:01:02.000Z"
    }
  ]
}
```
Status: `200`, `401`, `404` `WEBHOOK_NOT_FOUND`.

### 6.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay`: replay da DLQ

Exige `requireRole('ADMIN')` [09:36 Larissa]. Sem body. Response `202`:
```json
{ "id": "d3e4f5a6-1111-4222-8333-444455556666", "status": "PENDING", "replayedBy": "a1b2c3d4-0000-4000-8000-000000000001" }
```
Status: `202`, `401`, `403` (OPERATOR), `404` `WEBHOOK_DEAD_LETTER_NOT_FOUND`.

### 6.8 Requisição enviada ao cliente (webhook de saída)

`POST {url}` com `Content-Type: application/json` e os headers [09:44–09:45]:

| Header | Valor |
|---|---|
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID do evento (id da outbox) |
| `X-Signature` | HMAC-SHA256 do corpo (formato do valor, por exemplo `sha256=<hex>`, **(Proposto)**) |
| `X-Timestamp` | timestamp do envio |
| `X-Webhook-Id` | id do endpoint webhook |

Corpo (sem `items`) [09:43 Diego]:
```json
{
  "event_id": "9f1c2a30-3b4d-4e5f-8a6b-7c8d9e0f1a2b",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-23T18:00:30.000Z",
  "order_id": "7a6b5c4d-3e2f-4a1b-8c9d-0e1f2a3b4c5d",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "5b0f7c0e-2d5e-4a41-9d2c-6a1f0e9a7c11",
  "total_cents": 45900
}
```
Semântica: at-least-once; o cliente deduplica por `X-Event-Id`; ordenação apenas por pedido [09:24–09:26].

## 7. Matriz de erros

Erros estendem `AppError`/`ConflictError`/`ValidationError` etc. (CODIGO: `src/shared/errors/http-errors.ts`) e são tratados pelo `errorMiddleware` sem alterações [09:28–09:29 Bruno]. Os três primeiros códigos vêm da transcrição ("WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED, etc."); os demais são **(Proposto)**.

| Código | HTTP | Quando |
|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | Webhook inexistente |
| `WEBHOOK_INVALID_URL` | 400 | URL ausente ou não `https` [09:23 Sofia] |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Operação que exige secret sem ela |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Payload maior que 64 KB [09:23–09:24] |
| `WEBHOOK_INVALID_EVENTS` | 400 | Lista de status vazia ou com valor fora de `OrderStatus` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Item de DLQ inexistente no replay |
| `WEBHOOK_REPLAY_FORBIDDEN` | 403 | Replay sem role ADMIN (na prática, `ForbiddenError` de `requireRole`) |
| `WEBHOOK_DELIVERY_FAILED` | (interno) | Falha de entrega registrada na DLQ como motivo |
| `WEBHOOK_DELIVERY_TIMEOUT` | (interno) | Timeout de 10 s [09:42] |

Os erros "internos" nunca viram resposta HTTP; são motivos gravados em `webhook_deliveries` e `webhook_dead_letter`.

## 8. Estratégias de resiliência

- **Timeout:** 10 s por chamada [09:42 Diego].
- **Retry/backoff:** 5 tentativas, 1 min, 5 min, 30 min, 2 h, 12 h [09:17].
- **Fallback:** DLQ persistida e replay manual ADMIN [09:18]; e-mail ao cliente fora de escopo [09:37].
- **Atomicidade:** outbox na transação de `changeStatus`; falha de outbox é rollback [09:40].
- **Isolamento:** worker em processo separado; cliente lento não afeta a API [09:11].
- **Falha do worker:** eventos `PROCESSING` retornam a `PENDING` no reinício **(Proposto)**.
- **Limitação conhecida:** ordenação por `order_id` só com um worker [09:13 Larissa].

## 9. Observabilidade

A transcrição define apenas que o logger é o Pino do projeto [09:29 Bruno]; métricas e tracing não foram discutidos. O que segue é **(Proposto)**:

- **Logs (Pino, `src/shared/logger/index.ts`):** eventos estruturados `webhook_event_enqueued`, `webhook_delivery_succeeded`, `webhook_delivery_failed`, `webhook_moved_to_dlq` e `webhook_replayed` (com `userId` do operador, exigência de auditoria [09:36 Sofia]), contendo `eventId`, `webhookId`, `orderId`, `attempt`, `httpStatus`, `durationMs`. Nunca registrar secret nem assinatura.
- **Métricas:** contadores de eventos enfileirados, entregues, falhos e movidos para DLQ; histograma de duração da chamada HTTP; gauge de tamanho da outbox pendente e da idade do evento pendente mais antigo (indica atraso fora do alvo de 10 s). Não há biblioteca de métricas no `package.json`; a escolha da ferramenta fica em aberto, e inicialmente podem ser emitidas como logs estruturados.
- **Tracing:** propagar o `requestId` atribuído a `req.id` em `src/middlewares/request-logger.middleware.ts` para o evento da outbox e usar o `eventId` como identificador de correlação entre enfileiramento, tentativas e replay. Tracing distribuído não é pré-requisito.

## 10. Integração com o sistema existente

| Caminho real | Como o módulo de webhooks se integra |
|---|---|
| `src/modules/orders/order.service.ts` | `changeStatus` passa a chamar `publishWebhookEvent(tx, order, from, to)` dentro do `$transaction`, após `tx.order.update` e `tx.orderStatusHistory.create`, usando o `TxClient` já definido no arquivo [09:41]. O `OrderService` não recebe repository de webhooks. |
| `prisma/schema.prisma` | Novos models `WebhookEndpoint`, `WebhookOutbox`, `WebhookDeadLetter` e `WebhookDelivery`, com UUID `@db.Char(36)` e `@@map` em snake_case; relação com `Customer`. Exige nova migration em `prisma/migrations/`. |
| `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | Novas classes `WebhookNotFoundError`, `WebhookInvalidUrlError` etc. estendem `NotFoundError`, `ValidationError`, `UnprocessableEntityError` ou `AppError` com códigos `WEBHOOK_*`; exportadas por `src/shared/errors/index.ts`. |
| `src/middlewares/auth.middleware.ts` | `authenticate` protege todas as rotas; `requireRole('ADMIN')` protege o replay [09:36]. O `req.user.id` alimenta o log de auditoria. |
| `src/middlewares/validate.middleware.ts` | Schemas Zod do módulo (`https` obrigatório, lista de status) são aplicados via `validate({ body, params, query })`. |
| `src/middlewares/error.middleware.ts` | Sem alterações: já responde a `AppError`, Zod e Prisma [09:29 Bruno]. |
| `src/routes/index.ts` e `src/app.ts` | `buildApiRouter` ganha `webhooks` em `Controllers`, e `buildControllers` instancia repository/service/controller de webhooks. O router de admin é montado em `/admin/webhooks`. |
| `src/server.ts` | Modelo para o novo `src/worker.ts` (bootstrap, `logger`, shutdown com `$disconnect`). |
| `src/config/env.ts` e `src/config/database.ts` | Mesma `DATABASE_URL`; o worker abre seu próprio `PrismaClient`. Variáveis de intervalo/lote/timeout, se existirem, entram no schema Zod de `env.ts` **(Proposto)**. |
| `package.json` | Novo script `"worker"` (por exemplo `tsx watch --env-file=.env src/worker.ts`, em paralelo a `dev`). `uuid`, `zod` e `pino` já existem; HMAC usa o `node:crypto` e o HTTP usa o `fetch` nativo do Node 20 (`engines.node >= 20`), sem dependência nova **(Proposto)**. |

## 11. Dependências e compatibilidade

- MySQL e Prisma 5.22 existentes; sem Redis nem broker [09:07].
- Node >= 20 (CODIGO: `package.json`); `node:crypto` e `fetch` nativos **(Proposto)**.
- **Compatibilidade:** nenhuma alteração nas rotas, nos contratos ou nos erros existentes. A única mudança de comportamento é o INSERT extra em `changeStatus`, e só quando há webhook interessado. A migration é aditiva.
- Clientes devem implementar deduplicação por `X-Event-Id` e, se quiserem, validar `X-Signature` e `X-Timestamp`.

## 12. Critérios de aceite técnicos

1. Mudança de status gera evento na outbox na mesma transação; rollback do pedido não deixa evento órfão [09:06, 09:40].
2. Se não há webhook interessado no status, nenhuma linha é criada [09:34].
3. Eventos pendentes são entregues em menos de 10 s em condições normais [09:02, 09:09].
4. Falha de entrega agenda retries 1 min, 5 min, 30 min, 2 h, 12 h e, depois, move para a DLQ [09:17–09:18].
5. A assinatura HMAC-SHA256 é verificável com a secret do endpoint; após rotação, a anterior vale por 24 h [09:22].
6. Toda requisição leva `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` [09:44–09:45].
7. URL `http` é rejeitada com `WEBHOOK_INVALID_URL`; payload acima de 64 KB falha com `WEBHOOK_PAYLOAD_TOO_LARGE` [09:23–09:24].
8. Replay retorna `403` para OPERATOR e registra o `userId` para ADMIN [09:36].
9. Reiniciar a API não interrompe o worker [09:11].
10. Revisão de segurança de HMAC e geração de secret concluída antes do deploy (2 dias úteis) [09:46 Sofia].

## 13. Riscos e mitigação

| Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|
| Outbox falha e bloqueia mudança de status | Baixa | Alto | Comportamento intencional [09:40]; testes ponta a ponta |
| Duplicatas de evento | Média | Médio | `X-Event-Id`; documentação destacada [09:26 Marcos] |
| Evento fora de ordem ao escalar workers | Baixa | Médio | Limitação documentada; single-worker [09:13] |
| Vazamento de secret | Média | Alto | Secret por endpoint e rotação [09:21–09:22]; redact em logs **(Proposto)** |
| Crescimento da outbox | Média | Médio | Índices e lote pequeno [09:08]; arquivamento em fase futura |
| Rajada de eventos para um cliente | Média | Médio | Fora do escopo: observar [09:39 Larissa] |
| Prazo (3 sprints) | Média | Alto | Estimativa de [09:46]; reserva da revisão de segurança |
