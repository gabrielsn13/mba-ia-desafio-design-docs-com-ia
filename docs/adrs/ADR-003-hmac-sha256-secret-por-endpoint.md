# ADR-003: Autenticação HMAC-SHA256 com secret por endpoint e rotação

- **Status:** Aceito
- **Origem:** TRANSCRICAO [09:19]–[09:22], [09:44]–[09:45]

## Contexto

Os eventos carregam dados de pedidos e saem para endpoints fora da nossa infraestrutura. O cliente precisa validar que a requisição veio de nós e que o payload não foi adulterado [09:19 Sofia]. Já houve cliente que vazou uma secret em log da própria aplicação [09:22 Diego].

## Decisão

- Assinar o **corpo do request** com **HMAC-SHA256** e enviar a assinatura no header `X-Signature` [09:20 Sofia].
- Uma **secret única por endpoint de webhook**, nunca global [09:21 Sofia]. A secret é gerada pela plataforma e devolvida na criação [09:31 Marcos].
- **Rotação** pela API: ao rotacionar, a secret antiga continua válida por **24 horas** em paralelo (grace period) e depois deixa de valer [09:21 Sofia].
- Headers complementares: `X-Timestamp` (permite ao cliente detectar replay) e `X-Webhook-Id` (identifica o cadastro) [09:44–09:45].
- URL do webhook obrigatoriamente `https`, validada no schema Zod (não é decisão arquitetural) [09:23 Sofia].

## Alternativas Consideradas

1. **Secret global da plataforma**: descartada; o vazamento de uma comprometeria todos os clientes [09:21 Sofia].
2. **Secret sem rotação**: descartada; cenário de vazamento já ocorreu e exige troca sem janela de indisponibilidade.
3. **Sem assinatura (confiar só em TLS)**: TLS protege o transporte, mas não prova a origem nem a integridade fim a fim do payload para o cliente [09:19 Sofia].

## Consequências

**Positivas**
- Vazamento contido a um único endpoint; troca de secret sem downtime do cliente.
- HMAC-SHA256 é padrão de mercado, com bibliotecas disponíveis na maioria dos clientes [09:20 Sofia].

**Negativas / trade-offs**
- Durante o grace period há duas secrets válidas por endpoint, o que exige armazenar a secret anterior e seu prazo.
- A secret precisa ser recuperável para assinar (não pode ser guardada só como hash); o armazenamento seguro fica para o FDD e para a revisão de segurança da Sofia [09:46].
- O cliente carrega a responsabilidade de verificar a assinatura.
