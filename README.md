# Design Docs com IA: Webhooks de Notificação de Pedidos

## Sobre o desafio

A partir da transcrição de uma reunião técnica (`TRANSCRICAO.md`) e do código existente do Order Management System (Node/TypeScript/Prisma/MySQL), foi produzido o pacote de design docs da feature **Webhooks de Notificação de Pedidos**. O entregável é somente documentação: `src/`, `prisma/` e `tests/` não foram alterados.

## Ferramentas de IA utilizadas

- **GitHub Copilot app** (agente em sessão com worktree, leitura de arquivos, edição e execução de comandos).
- **Modelo:** Claude Sonnet 5.5.
- Uso: ler a transcrição e o código, propor a estrutura, redigir os documentos e validar mecanicamente (caminhos de código, timestamps e proporção de fontes do Tracker).

## Workflow adotado

1. Análise do enunciado e plano em etapas, aprovado antes de começar.
2. Leitura integral da transcrição e dos arquivos-chave do código, com levantamento de decisões, itens fora de escopo e referências reais.
3. **ADRs** (7): uma decisão por arquivo, com alternativas e consequências.
4. **RFC**: proposta com alternativas, questões em aberto e links para os ADRs.
5. **FDD**: design técnico, endpoints, matriz de erros e integração com o código existente.
6. **PRD**: problema, requisitos funcionais/não funcionais e fora de escopo.
7. **Tracker**: rastreabilidade de cada item para a transcrição (`[hh:mm] Nome`) ou para o código (caminho real).
8. README e revisão final contra o checklist de aceite.

Cada etapa foi entregue em um PR próprio, revisado e mergeado antes da seguinte.

## Prompts customizados

Prompt de contexto e regras (início do trabalho):

```text
Leia TRANSCRICAO.md por completo e os arquivos principais do código (service de pedidos,
schema Prisma, erros, middlewares, rotas). Liste: decisões tomadas, itens fora de escopo,
e referências reais no código. Não invente nada sem fonte rastreável; o que for detalhe
de design sem origem deve ser marcado como "(Proposto)". Não altere src/, prisma/ ou tests/.
```

Prompt de validação do Tracker:

```text
Valide docs/TRACKER.md: (1) pelo menos 70% das linhas com fonte TRANSCRICAO no formato
[hh:mm] Nome; (2) pelo menos 5 linhas CODIGO com caminhos que existem no repositório;
(3) cada [hh:mm] Nome deve existir na transcrição. Reporte as divergências.
```

## Iterações e ajustes

- O placeholder `docs/adrs/README.md` foi removido: o nome conflitava com o padrão `ADR-NNN-*`.
- Ao sobrescrever placeholders com acentos, a edição por texto falhou (problema de codificação); passou-se a apagar e recriar o arquivo.
- FDD corrigido em dois pontos: o padrão de resposta 204 nos DELETE e a descrição do request logger (que não é `pino-http`; define `req.id`).
- Detalhes sem origem (tabela `webhook_deliveries`, `next_attempt_at`, lote de 10, endpoint `rotate-secret`, métricas, metas do PRD) foram marcados como "Proposto" e separados na seção 2 do Tracker.
- Alternativas não discutidas na reunião foram rotuladas como "plausíveis, não discutidas".
- Ambiguidade de "5 tentativas" resolvida explicitamente (5 esperas após a primeira falha, cerca de 15 h) e registrada como questão em aberto.
- Na revisão final contra o enunciado, o Tracker não listava 3 grupos de códigos de erro do FDD e usava IDs diferentes dos do PRD; foram adicionados e a correspondência foi documentada.
- A validação automática do Tracker detectou um timestamp errado (`[09:45]` em vez de `[09:44]`), corrigido.

No total, foram 4 ciclos principais de geração, revisão crítica e ajuste: ADRs, RFC/FDD (com as correções do FDD), PRD/Tracker (com a separação dos itens "Proposto") e a revisão final contra o enunciado.

## Como navegar a entrega

| Documento | Caminho |
|---|---|
| Transcrição (fonte) | `TRANSCRICAO.md` |
| PRD | `docs/PRD.md` |
| RFC | `docs/RFC.md` |
| FDD | `docs/FDD.md` |
| ADRs (7) | `docs/adrs/ADR-001-...` a `ADR-007-...` |
| Tracker | `docs/TRACKER.md` |

Ordem de leitura sugerida: PRD, RFC, ADRs, FDD e, por fim, o Tracker para auditar a origem de cada item.
