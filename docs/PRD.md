# PRD — Product Requirements Document

> **Convenção:** `[hh:mm Nome]` = origem na transcrição; **(Proposto)** = meta ou critério sugerido neste PRD, sem origem na reunião, a validar com o PM. Detalhes técnicos estão no [RFC](RFC.md), nos [ADRs](adrs/) e no [FDD](FDD.md).

## 1. Resumo e contexto da feature

O Order Management System passará a notificar sistemas externos, por webhook HTTPS, quando o status de um pedido mudar. A feature atende a um pedido formal de três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) [09:00 Marcos]. Hoje a aplicação não possui nenhum mecanismo de notificação externa; a feature é só de **saída** (a plataforma envia, o cliente recebe) [09:02 Marcos].

## 2. Problema e motivação

Os clientes consultam `GET /orders` periodicamente para descobrir mudanças, o que torna a integração lenta e cara para eles [09:00 Marcos]. A Atlas indicou que, sem a entrega até o fim do trimestre, pode migrar para um concorrente [09:00 Marcos]. Para os clientes, "tempo real" significa menos de 10 segundos, sem precisar atualizar manualmente [09:02 Marcos].

## 3. Público-alvo e cenários de uso

| Público | Cenário |
|---|---|
| Sistemas de integração dos clientes B2B | Receber a notificação quando um pedido vira, por exemplo, `SHIPPED` ou `DELIVERED`, sem polling [09:33 Marcos] |
| Usuários da nossa plataforma que representam o cliente | Cadastrar, editar, remover e listar webhooks pela API, autenticados com o JWT do sistema [09:32 Marcos] |
| Cliente integrador | Consultar o histórico das últimas entregas para diagnosticar falhas [09:34 Marcos] |
| Administrador da plataforma | Reprocessar eventos que esgotaram as tentativas [09:18, 09:36] |
| Time de produto (Marcos) | Documentar a integração no portal de desenvolvedor [09:26, 09:40] |

## 4. Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta | Origem |
|---|---|---|---|
| Notificação percebida como tempo real | Tempo entre a mudança de status e a primeira tentativa de entrega | Menos de 10 s (polling de 2 s) | [09:02 Marcos, 09:09–09:10] |
| Eliminar polling dos clientes em `GET /orders` | Redução de chamadas de polling dos 3 clientes pilotos | Meta numérica a definir após medir a linha de base **(Proposto)** | [09:00 Marcos] |
| Não perder eventos | Eventos com mudança de status confirmada e sem registro de entrega, DLQ ou retry pendente | 0 **(Proposto)** | [09:40 Bruno] |
| Entregar no prazo | Entrega da feature | 3 sprints (Atlas pede fim de novembro) | [09:45–09:47] |
| Não degradar pedidos | Impacto da feature na latência de `changeStatus` | Sem chamada externa na transação; limite numérico a definir **(Proposto)** | [09:04 Bruno] |

## 5. Escopo

### Incluso
- Cadastro e gestão de webhooks por cliente, com filtro por status.
- Notificação de mudança de status com assinatura, retry e DLQ.
- Rotação de secret, histórico de entregas e replay manual de DLQ.

### Fora de escopo (descartado ou adiado)
| Item | Situação | Origem |
|---|---|---|
| E-mail de alerta quando o webhook falha | Adiado para a próxima fase, após medir o impacto | [09:37 Larissa] |
| Rate limiting de envio por cliente | Não entra; observar e decidir depois | [09:38–09:39 Diego/Larissa] |
| Dashboard visual para o cliente | Descartado nesta fase (projeto separado do frontend) | [09:40 Larissa] |
| Arquivamento de eventos entregues (cerca de 30 dias) | Fora do escopo desta feature | [09:08 Diego] |
| Múltiplos workers e ordering global | Fora; ordering só por pedido, com single-worker | [09:12–09:14] |
| Webhooks de entrada (cliente para nós) | Fora; apenas saída | [09:02 Marcos] |
| Garantia exactly-once | Descartada; o cliente deduplica | [09:25 Diego] |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
|---|---|---|
| RF-01 | Cadastrar webhook com URL e lista de status de interesse; a secret é gerada pela plataforma e devolvida na criação | [09:31 Marcos] |
| RF-02 | O cliente (`customer_id`) é informado na requisição, não vem do JWT | [09:32 Larissa] |
| RF-03 | Editar (PATCH) e remover (DELETE) um webhook | [09:33 Bruno] |
| RF-04 | Listar os webhooks de um cliente | [09:33 Bruno] |
| RF-05 | Cada webhook escolhe quais status deseja receber; o filtro é aplicado ao gerar o evento | [09:33 Marcos, 09:34 Bruno] |
| RF-06 | Notificar a mudança de status de forma consistente com a mudança do pedido (sem evento perdido nem evento sem mudança) | [09:40 Bruno] |
| RF-07 | Reenviar automaticamente em caso de falha (5 tentativas com backoff) e, depois, mover para a DLQ | [09:17–09:18] |
| RF-08 | Assinar cada envio (HMAC-SHA256) com secret única por webhook | [09:20–09:22 Sofia] |
| RF-09 | Rotacionar a secret pela API, mantendo a anterior válida por 24 h | [09:21 Sofia] |
| RF-10 | Identificar cada evento com um id único para deduplicação pelo cliente (`X-Event-Id`) | [09:25 Diego] |
| RF-11 | Consultar o histórico de entregas (últimas 100: sucesso/falha, payload, resposta, tempo) | [09:34 Marcos] |
| RF-12 | Reprocessar manualmente eventos da DLQ, restrito à role ADMIN, com registro de quem executou | [09:18, 09:36] |
| RF-13 | Aceitar somente URLs `https` | [09:23 Sofia] |
| RF-14 | Enviar payload enxuto, sem itens do pedido | [09:43 Diego] |

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
|---|---|---|
| RNF-01 | Latência de notificação abaixo de 10 s (polling de 2 s) | [09:02, 09:09–09:10] |
| RNF-02 | Garantia de entrega at-least-once | [09:24–09:26] |
| RNF-03 | Segurança: HMAC-SHA256, secret por endpoint, TLS obrigatório | [09:19–09:23 Sofia] |
| RNF-04 | Timeout de 10 s por chamada ao cliente | [09:42 Diego] |
| RNF-05 | Limite de 64 KB por payload, com erro (sem truncar) | [09:23–09:24] |
| RNF-06 | Isolamento: o envio não bloqueia nem faz rollback das mudanças de pedido; o worker roda em processo separado | [09:04, 09:11] |
| RNF-07 | Ordenação apenas por pedido e só com single-worker (limitação conhecida) | [09:12–09:13 Larissa] |
| RNF-08 | Sem infraestrutura nova (MySQL existente) | [09:07 Diego] |
| RNF-09 | Revisão de segurança antes do deploy (2 dias úteis) | [09:46 Sofia] |

## 8. Decisões e trade-offs principais

Resumo; o raciocínio completo está nos ADRs.

| Decisão | Trade-off aceito | ADR |
|---|---|---|
| Outbox no MySQL | Mais carga no banco em troca de zero infra nova e atomicidade | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Retry de 5 tentativas e DLQ | Evento pode demorar até cerca de 15 h para ser considerado perdido; replay manual | [ADR-002](adrs/ADR-002-retry-backoff-e-dlq.md) |
| HMAC-SHA256 por endpoint | Gestão de rotação e de duas secrets no grace period | [ADR-003](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md) |
| At-least-once | Cliente precisa deduplicar | [ADR-004](adrs/ADR-004-at-least-once-com-x-event-id.md) |
| Worker separado em polling de 2 s | Latência mínima de até 2 s; ordering só com um worker | [ADR-005](adrs/ADR-005-worker-separado-em-polling.md) |
| Reuso dos padrões do projeto | Herda limitações atuais (roles globais) | [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| Payload em snapshot | Mais armazenamento; evento reflete o instante da mudança | [ADR-007](adrs/ADR-007-snapshot-do-payload-e-publish-webhook-event.md) |

## 9. Dependências

- Time de Segurança (Sofia): revisão de HMAC e geração de secret antes do deploy [09:46].
- Produto (Marcos): documentar no portal de desenvolvedor a deduplicação e a integração via API [09:26, 09:40].
- Clientes: implementar endpoint `https`, deduplicação por `X-Event-Id` e, opcionalmente, verificação de assinatura [09:25–09:26].
- Plataforma: banco MySQL e infraestrutura para executar o worker como processo separado [09:11].
- Time de frontend: dashboard, em projeto futuro e separado [09:40].

## 10. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Atraso frente ao pedido da Atlas (fim de novembro contra 3 sprints) | Média | Alto | Confirmar o prazo com a Atlas; Marcos já ficou de comunicar [09:47–09:49]; reservar os 2 dias da revisão de segurança |
| Vazamento de secret de um cliente | Média | Alto | Secret por endpoint e rotação com grace de 24 h [09:21–09:22] |
| Cliente processa o mesmo evento duas vezes | Média | Médio | `X-Event-Id` e documentação destacada no portal [09:25–09:26] |
| Cliente indisponível por longos períodos | Média | Médio | Retry de até cerca de 15 h, DLQ e replay [09:16–09:18] |
| Rajadas de eventos sobrecarregam clientes | Média | Médio | Fora do escopo; observar e decidir depois [09:39] |
| Falha na outbox bloqueia mudanças de status | Baixa | Alto | Comportamento deliberado [09:40]; testes ponta a ponta |

*Probabilidade e impacto são avaliação deste PRD **(Proposto)**; os riscos em si vêm da reunião.*

## 11. Critérios de aceitação

1. Um usuário autenticado cadastra um webhook `https` com lista de status e recebe a secret uma única vez (RF-01, RF-13).
2. Ao mudar o status de um pedido de um cliente com webhook interessado, o cliente recebe o evento em menos de 10 s (RF-06, RNF-01).
3. Webhooks sem interesse no novo status não recebem nada (RF-05).
4. O cliente valida a assinatura e deduplica por `X-Event-Id` (RF-08, RF-10).
5. Com o endpoint fora do ar, o sistema retenta com backoff e, após 5 tentativas, o evento aparece na DLQ (RF-07).
6. Um ADMIN reprocessa um evento da DLQ e a ação é registrada em log; um OPERATOR recebe `403` (RF-12).
7. Após rotacionar a secret, assinaturas com a antiga continuam válidas por 24 h (RF-09).
8. O histórico de entregas mostra as últimas 100 tentativas com resultado e tempo de resposta (RF-11).
9. Uma falha do endpoint do cliente nunca impede a mudança de status de um pedido (RNF-06).

## 12. Estratégia de testes e validação

- **Testes automatizados (Vitest, padrão atual em `tests/`):** unidade para filtro de eventos, assinatura HMAC e cálculo de backoff; integração para a transação de `changeStatus` com outbox (rollback em falha) e para os endpoints (autorização, validação de URL `https`). O repositório já tem `tests/orders.test.ts` e `tests/auth.test.ts` como referência **(Proposto)**.
- **Testes ponta a ponta:** worker contra um servidor de teste que simule sucesso, erro 5xx, timeout e indisponibilidade; verificação do caminho até a DLQ e do replay [09:46 Larissa].
- **Segurança:** revisão manual do código de HMAC e de geração de secret pela Sofia, com 2 dias úteis reservados [09:46].
- **Validação com o cliente:** piloto com os três clientes solicitantes, medindo o tempo de entrega e a redução de polling em `GET /orders` **(Proposto)**.
