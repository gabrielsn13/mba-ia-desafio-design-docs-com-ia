# ADR-001: Padrão Outbox no MySQL existente

- **Status:** Aceito
- **Data:** reunião técnica (quinta-feira, 09:00)
- **Decisores:** Larissa (Tech Lead), Diego, Bruno
- **Origem:** TRANSCRICAO [09:03]–[09:08] e [09:40]–[09:41]

## Contexto

Clientes B2B (Atlas, MaxDistribuição, Nova Cargo) querem ser notificados em menos de 10 segundos quando o status de um pedido muda [09:00 Marcos, 09:02 Marcos]. Hoje a mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de um `prisma.$transaction` que já atualiza `orders`, insere em `order_status_history` e debita/repõe `stock_quantity`.

Disparar o HTTP de forma síncrona dentro dessa transação foi considerado inviável: um cliente lento travaria mudanças de status de outros pedidos, e um cliente fora do ar forçaria rollback [09:04 Bruno].

## Decisão

Adotar o **padrão Outbox no MySQL existente**. Na mesma transação que atualiza `orders` e `order_status_history`, `changeStatus` insere uma linha na tabela `webhook_outbox`. Se a inserção falhar, a transação inteira sofre rollback [09:40 Bruno]. Um worker separado (ver [ADR-005](ADR-005-worker-separado-em-polling.md)) lê a tabela e faz as chamadas HTTP.

- Estados do evento: pendente, processando, falhou, entregue [09:08 Diego].
- Índices em `status` e `created_at` [09:08 Diego].
- Chave primária UUID, seguindo o padrão do projeto (`@default(uuid()) @db.Char(36)` em `prisma/schema.prisma`) [09:51 Larissa].
- O filtro de eventos por webhook acontece na inserção: se nenhum webhook do customer quer aquele status, nenhuma linha é criada [09:34 Bruno].

## Alternativas Consideradas

1. **Chamada HTTP síncrona dentro de `changeStatus`**: descartada. Acopla a latência do cliente à transação de pedidos e não tem resposta aceitável para cliente offline [09:04 Bruno, 09:06 Diego].
2. **Redis Streams (ou fila equivalente)**: descartada. Exigiria subir nova infraestrutura; para um time pequeno seria overengineering [09:07 Larissa e Diego].
3. **Trigger no banco para notificar o worker**: descartada para o disparo (ver [ADR-005](ADR-005-worker-separado-em-polling.md)).

## Consequências

**Positivas**
- Atomicidade: se a transação commitou, o evento existe; se deu rollback, o evento some junto [09:06 Diego].
- Nenhuma infraestrutura nova; reutiliza o MySQL e o Prisma já existentes.

**Negativas / trade-offs**
- Aumenta o trabalho dentro da transação de `changeStatus` (um INSERT a mais) e adiciona carga de leitura (polling) ao MySQL.
- A tabela cresce; o arquivamento de linhas entregues (cerca de 30 dias) fica **fora do escopo** desta feature [09:08 Diego].
- Troca-se throughput potencial de uma fila dedicada por simplicidade operacional.
