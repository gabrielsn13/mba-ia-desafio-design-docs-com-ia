# ADR-004: Garantia at-least-once com X-Event-Id para deduplicação

- **Status:** Aceito
- **Origem:** TRANSCRICAO [09:12]–[09:14], [09:24]–[09:26]

## Contexto

Com outbox, retry e worker que pode falhar entre o envio e a marcação de "entregue", o mesmo evento pode ser enviado mais de uma vez. É preciso definir a garantia de entrega e como o cliente lida com duplicatas.

## Decisão

- Garantia de entrega **at-least-once** [09:24 Diego].
- Cada evento recebe um **UUID gerado na inserção na outbox**, enviado no header `X-Event-Id`. O cliente deduplica por esse valor [09:25 Diego].
- **Ordering:** apenas implícito por `order_id` (ordem de `created_at` da outbox) e somente enquanto houver um único worker. Não há garantia de ordering global; isso é uma limitação conhecida [09:12–09:13 Diego/Larissa].

## Alternativas Consideradas

1. **Exactly-once**: descartada; exigiria coordenação entre os dois lados e é muito mais complexo [09:25 Diego].
2. **At-most-once (sem retry)**: incompatível com o requisito de retry/DLQ do [ADR-002](ADR-002-retry-backoff-e-dlq.md); não foi discutida como opção real na reunião.

## Consequências

**Positivas**
- Nenhum evento é perdido silenciosamente; padrão de mercado (Stripe, GitHub) [09:25 Diego].
- Implementação simples.

**Negativas / trade-offs**
- A responsabilidade de deduplicar é do cliente [09:25 Sofia]; Marcos documentará isso de forma destacada no portal de desenvolvedor [09:26].
- Escalar para múltiplos workers perde a garantia de ordem por pedido; seria necessário particionar por `order_id` ou usar lock pessimista, tratado como problema futuro [09:13 Diego].
