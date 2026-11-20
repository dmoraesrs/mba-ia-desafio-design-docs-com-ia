# ADR-006: Reuso máximo dos padrões existentes do projeto

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira, 09:00 (registrada em `TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-em-polling.md)

## Contexto

O OMS já tem convenções consolidadas no código, mapeadas antes de escrever este ADR:

| Padrão | Onde está no código |
| --- | --- |
| Módulo por domínio com controller, service, repository, routes e schemas | `src/modules/orders/` (`order.controller.ts`, `order.service.ts`, `order.repository.ts`, `order.routes.ts`, `order.schemas.ts`) e demais módulos em `src/modules/` |
| Hierarquia de erros com `statusCode` e `errorCode` em UPPER_SNAKE_CASE | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` (`InvalidStatusTransitionError` com `INVALID_STATUS_TRANSITION`, `InsufficientStockError` com `INSUFFICIENT_STOCK`) |
| Error middleware centralizado que trata `AppError`, `ZodError` e erros conhecidos do Prisma | `src/middlewares/error.middleware.ts` |
| Logger Pino com redaction de campos sensíveis | `src/shared/logger/index.ts` |
| Autorização por role | `requireRole` em `src/middlewares/auth.middleware.ts`, já usado em `src/modules/users/user.routes.ts` |
| Validação de entrada com Zod | `src/middlewares/validate.middleware.ts` e os `*.schemas.ts` |
| Registro de rotas e composição de dependências | `src/routes/index.ts`, `buildControllers` em `src/app.ts` |

A feature introduz um domínio novo e um processo novo (worker). Sem uma decisão explícita, o risco é cada parte trazer biblioteca e convenção próprias.

## Decisão

Reusar ao máximo o que existe: `AppError`, Pino, error middleware, padrão de módulos, padrão de schemas Zod e padrão de códigos de erro. Webhook é um módulo igual aos outros ([09:30] Larissa). Concretamente:

1. Novo módulo `src/modules/webhooks/` com controller, service, repository, routes e schemas ([09:27] Bruno).
2. Erros do módulo estendem as classes de `src/shared/errors/` e usam códigos com prefixo `WEBHOOK_`, como `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` ([09:28] Bruno, [09:29] Larissa).
3. Logs via o `logger` Pino existente, sem biblioteca nova ([09:29] Bruno). O error middleware trata os novos erros sem alteração ([09:29] Bruno).
4. Replay da DLQ protegido com `requireRole('ADMIN')` ([09:36] Larissa).
5. Integração com pedidos por uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o client da transação corrente, em vez de injetar um repository inteiro no `OrderService` ([09:41] Bruno, [09:41] Diego).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| Injetar `WebhookRepository` no construtor de `OrderService` | Acopla pedidos ao módulo de webhooks mais do que o necessário; uma função pura recebendo o `tx` basta ([09:41] Bruno, [09:41] Diego). |
| Introduzir bibliotecas novas de log ou de erro para o módulo | Contraria a decisão de "não vamos botar nada novo" ([09:29] Bruno, [09:30] Larissa). |

## Consequências

**Positivas**
- Curva de aprendizado mínima: quem conhece `orders` conhece `webhooks`.
- Respostas de erro com o mesmo envelope `{ error: { code, message, details } }` que o error middleware já produz.
- Nenhuma dependência nova para revisar ou atualizar.

**Negativas / trade-offs**
- O worker não passa pelo Express, então o error middleware não o cobre: o tratamento de erro do worker precisa ser feito no próprio loop, apenas reusando as classes de erro e o logger.
- O padrão herda as limitações atuais (por exemplo, `requestLogger` gera correlação por request HTTP, e não existe tracing distribuído no projeto).
- Trade-off aceito: consistência e rapidez de entrega em troca de não otimizar o módulo com ferramentas especializadas.
