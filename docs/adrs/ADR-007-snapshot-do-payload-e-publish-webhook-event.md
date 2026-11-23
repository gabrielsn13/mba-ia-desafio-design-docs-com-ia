# ADR-007: Snapshot do payload na inserção e `publishWebhookEvent(tx, ...)`

- **Status:** Aceito
- **Origem:** TRANSCRICAO [09:41], [09:43], [09:51]–[09:52]; CODIGO `src/modules/orders/order.service.ts`

## Contexto

`OrderService.changeStatus` precisa enfileirar o evento dentro da transação existente ([ADR-001](ADR-001-outbox-no-mysql.md)). Era preciso decidir como o `OrderService` acessa a outbox e quando o payload é montado.

## Decisão

- Criar a função **`publishWebhookEvent(tx, order, fromStatus, toStatus)`**, que recebe o client da transação atual (`Prisma.TransactionClient`, já usado como `TxClient` em `order.service.ts`). `changeStatus` a chama após atualizar `orders` e `order_status_history`. É função simples, sem injetar o repository inteiro no `OrderService` [09:41 Bruno/Diego].
- O **payload é renderizado na inserção (snapshot)**, não no envio, para refletir o estado do pedido no momento da mudança [09:52 Larissa/Diego].
- Conteúdo: `event_id`, `event_type` (`order.status_changed`), timestamp ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos como `total_cents`. **Sem `items`**; o cliente consulta `GET /orders/:id` se precisar [09:43 Diego].
- Limite de **64 KB** por payload; acima disso, erro em vez de truncar [09:23–09:24 Sofia/Diego/Larissa].

## Alternativas Consideradas

1. **Guardar só `order_id` e renderizar no envio**: descartada; o evento poderia refletir um estado posterior e gerar casos esquisitos [09:52 Larissa].
2. **Injetar um repository de webhooks no `OrderService`**: descartada em favor da função com `tx` [09:41 Diego].
3. **Incluir `items` no payload**: descartada para manter o payload enxuto [09:43–09:44].

## Consequências

**Positivas**
- O evento é imutável e determinístico; reenvios e replays enviam o mesmo conteúdo.
- Acoplamento mínimo entre `OrderService` e o módulo de webhooks.

**Negativas / trade-offs**
- Mais bytes armazenados por evento na outbox.
- O cliente pode ver um payload "antigo" em relação ao estado atual do pedido (comportamento intencional).
- Uma falha em `publishWebhookEvent` derruba a mudança de status (rollback), o que é desejado [09:40 Bruno], mas amplia a superfície de falha de `changeStatus`.
