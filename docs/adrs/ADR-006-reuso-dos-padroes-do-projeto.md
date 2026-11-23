# ADR-006: Reuso máximo dos padrões existentes do projeto

- **Status:** Aceito
- **Origem:** TRANSCRICAO [09:27]–[09:30], [09:36]; CODIGO (arquivos citados abaixo)

## Contexto

A aplicação já possui convenções claras. Introduzir novas bibliotecas ou estruturas para webhooks aumentaria o custo de manutenção para um time pequeno [09:07 Diego].

## Decisão

O módulo de webhooks segue o que já existe [09:30 Larissa]:

| Padrão | Arquivo/módulo existente | Uso em webhooks |
|---|---|---|
| Módulo por domínio (controller, service, repository, routes, schemas) | `src/modules/orders/`, `src/modules/customers/` | Novo `src/modules/webhooks/` [09:27 Bruno] |
| `AppError` e subclasses | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` | Erros específicos com prefixo `WEBHOOK_` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) [09:28–09:29] |
| Middleware de erro centralizado (trata `AppError`, Zod, Prisma) | `src/middlewares/error.middleware.ts` | Sem alterações [09:29 Bruno] |
| Autenticação e role | `authenticate` e `requireRole` em `src/middlewares/auth.middleware.ts` | CRUD autenticado; replay da DLQ com `requireRole('ADMIN')` [09:36–09:37] |
| Validação Zod | `src/middlewares/validate.middleware.ts` e `*.schemas.ts` | Schemas do módulo |
| Logger Pino | `src/shared/logger/index.ts` | Logs do worker e da API [09:29 Bruno] |
| Registro de rotas | `src/routes/index.ts` (`buildApiRouter`) | Novo router de webhooks |
| IDs UUID | `prisma/schema.prisma` | Tabelas novas com UUID [09:51 Larissa] |
| Entry-point | `src/server.ts` | Modelo para `src/worker.ts` |

## Alternativas Consideradas

1. **Biblioteca de filas/webhooks de terceiros** (ex.: BullMQ): exigiria Redis, descartado em [ADR-001](ADR-001-outbox-no-mysql.md). Alternativa plausível, não detalhada na reunião.
2. **Novo formato de erro específico do módulo**: quebraria a consistência do contrato `{ error: { code, message, details } }` produzido por `error.middleware.ts`.

## Consequências

**Positivas**
- Curva de aprendizado baixa e consistência com o restante da API.
- O middleware de erro captura os novos erros sem mudanças.

**Negativas / trade-offs**
- O módulo herda limitações dos padrões atuais (por exemplo, `requireRole` é por role global, sem escopo por customer; endurecimento ficou para depois [09:37 Sofia]).
- O worker não passa pelo `error.middleware.ts` e precisa de tratamento de erros e logs próprios.
