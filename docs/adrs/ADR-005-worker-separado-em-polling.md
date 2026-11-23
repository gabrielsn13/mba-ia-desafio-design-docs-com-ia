# ADR-005: Worker em processo separado com polling de 2 segundos

- **Status:** Aceito
- **Origem:** TRANSCRICAO [09:09]–[09:13], [09:28], [09:29]–[09:30]

## Contexto

Os eventos da outbox ([ADR-001](ADR-001-outbox-no-mysql.md)) precisam ser lidos e enviados. O requisito do cliente é latência abaixo de 10 segundos [09:02 Marcos]. O MySQL não tem mecanismo nativo de listener como o `LISTEN/NOTIFY` do Postgres [09:09 Diego].

## Decisão

- **Polling em loop a cada 2 segundos**: busca os eventos pendentes mais antigos em lotes pequenos, processa e marca [09:08–09:09 Diego]. A latência mínima no pior caso é de 2 s, aceita por Larissa e Marcos [09:10].
- O worker roda como **processo separado da API**, para que reinícios da API não percam o worker [09:11 Diego].
- Novo entry-point `src/worker.ts` (análogo a `src/server.ts`) e script `npm run worker` [09:11 Larissa]. A lógica de processamento fica no módulo (`src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts`) [09:28 Bruno].
- Mesmo banco e mesma `DATABASE_URL`, mas **instância própria de `PrismaClient`**, pois o client é por processo [09:30 Bruno].
- **Single-worker** por enquanto.

## Alternativas Consideradas

1. **Trigger no banco**: só executa SQL e não notifica processo externo; exigiria improviso (arquivo, endpoint) [09:09 Diego].
2. **Worker dentro do processo da API**: descartada; se a API reinicia, o worker se perde [09:11 Diego].
3. **Mecanismo reativo via broker (Redis)**: descartada em [ADR-001](ADR-001-outbox-no-mysql.md) por exigir infraestrutura nova.

## Consequências

**Positivas**
- Simples, previsível e com latência bem dentro do alvo de 10 s.
- Deploy e ciclo de vida independentes da API.

**Negativas / trade-offs**
- Consultas periódicas ao MySQL mesmo sem eventos.
- Latência mínima de até 2 s e ordering garantido só com um worker (ver [ADR-004](ADR-004-at-least-once-com-x-event-id.md)).
- Um segundo processo a operar e monitorar.
