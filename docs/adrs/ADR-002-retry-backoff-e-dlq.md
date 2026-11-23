# ADR-002: Retry com backoff exponencial e DLQ em tabela separada

- **Status:** Aceito
- **Origem:** TRANSCRICAO [09:14]–[09:19], [09:36], [09:42]

## Contexto

O endpoint do cliente pode estar indisponível. Já houve cliente com indisponibilidade de duas horas em manutenção planejada [09:16 Diego]. É preciso uma política que cubra janelas longas sem deixar eventos pendurados para sempre.

## Decisão

- **5 tentativas** com backoff de **1 min, 5 min, 30 min, 2 h e 12 h** (cerca de 15 h entre a primeira falha e a última tentativa) [09:17 Larissa/Diego].
- **Timeout de 10 segundos** por chamada; estouro de timeout conta como falha e agenda retry [09:42 Diego].
- Esgotadas as tentativas, o evento é movido para a **DLQ**, em tabela separada `webhook_dead_letter`, com payload, motivo da falha e timestamp [09:18 Diego].
- **Replay manual** via `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente [09:18 Diego]. Exige role `ADMIN` (reaproveitando `requireRole` de `src/middlewares/auth.middleware.ts`) e deve registrar em log quem executou o replay [09:36 Sofia/Larissa].

## Alternativas Consideradas

1. **Retry indefinido com backoff**: descartada; evento fica pendurado para sempre se o cliente sumir [09:15 Diego].
2. **3 tentativas**: descartada; cobre só cerca de 30 minutos e mataria eventos em indisponibilidades como a de 2 h [09:16 Diego].
3. **Marcar como `failed` na própria outbox em vez de tabela DLQ**: descartada; a tabela separada deixa a leitura da outbox mais limpa e serve de evidência para debug/reprocessamento [09:18 Diego].

## Consequências

**Positivas**
- Janela de recuperação de aproximadamente 15 h; falhas permanentes ficam isoladas e auditáveis.
- A outbox principal permanece enxuta para o polling.

**Negativas / trade-offs**
- Um evento pode atrasar muito (até cerca de 15 h) antes de ser considerado perdido.
- Reprocessamento é manual; não há alerta automático ao cliente (e-mail foi adiado para a próxima fase [09:37 Larissa]).
- Uma tabela a mais para manter.
