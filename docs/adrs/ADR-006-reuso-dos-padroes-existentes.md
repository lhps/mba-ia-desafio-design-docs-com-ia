# ADR-006 — Reuso dos padrões existentes do projeto no módulo de webhooks

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira ([09:30] Larissa: "Decisão: reuso máximo do que já existe.")
- **Decisores:** Larissa, Bruno, Diego
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-em-polling.md)

## Contexto

O OMS tem padrões consolidados que qualquer feature nova deveria seguir ([09:27] Bruno):

| Padrão | Onde está no código |
|---|---|
| Módulo por domínio com `controller`, `service`, `repository`, `routes`, `schemas` | `src/modules/orders/`, `src/modules/customers/`, `src/modules/products/`, `src/modules/users/` |
| Hierarquia de erros com `statusCode` + `errorCode` em UPPER_SNAKE_CASE | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` (`InsufficientStockError` → `INSUFFICIENT_STOCK`, `InvalidStatusTransitionError` → `INVALID_STATUS_TRANSITION`) |
| Tratamento centralizado de `AppError`, `ZodError` e erros Prisma (`P2002`, `P2025`) | `src/middlewares/error.middleware.ts` |
| Validação de entrada com Zod via middleware | `src/middlewares/validate.middleware.ts` + `*.schemas.ts` |
| Autenticação JWT e autorização por role | `authenticate` e `requireRole` em `src/middlewares/auth.middleware.ts` |
| Logger estruturado Pino com redação de campos sensíveis | `src/shared/logger/index.ts` |
| IDs UUID `@db.Char(36)` em todos os models | `prisma/schema.prisma` |
| Composição de dependências por construtor e montagem de rotas | `src/app.ts` (`buildControllers`), `src/routes/index.ts` (`buildApiRouter`) |

A equipe é pequena ([09:07] Diego). Introduzir bibliotecas ou convenções paralelas aumentaria a carga cognitiva e o custo de manutenção.

## Decisão

O módulo de webhooks será **um módulo igual aos outros** ([09:30] Larissa):

1. **Estrutura:** `src/modules/webhooks/` com `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts` ([09:27] Bruno), mais `webhook.worker.ts` com a lógica de processamento. O Bruno também citou `webhook.processor.ts` como nome possível ([09:28] Bruno).
2. **Erros:** novas classes estendem `AppError` (direta ou indiretamente), e **todos os códigos do módulo usam o prefixo `WEBHOOK_`**: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` etc. ([09:28] Bruno, [09:29] Larissa).
3. **Error middleware sem alteração:** como as novas classes são `AppError`, o `errorMiddleware` já devolve `{ error: { code, message, details } }` com o status correto ([09:29] Bruno).
4. **Logger:** Pino, nada novo ([09:29] Bruno). O worker importa o mesmo `logger`.
5. **Autorização:** o replay da DLQ usa o `requireRole('ADMIN')` existente ([09:36] Larissa), no mesmo padrão de `src/modules/users/user.routes.ts`. O CRUD de configuração usa apenas `authenticate` ([09:37] Sofia).
6. **Banco:** Prisma, mesma `DATABASE_URL`. O worker cria o próprio `PrismaClient` ([09:30] Bruno). Models novos com `id` UUID ([09:51] Larissa).
7. **Integração com pedidos por função, não por injeção de repository:** `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebe o `Prisma.TransactionClient` da transação corrente ([09:41] Bruno, [09:41] Diego).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Injetar `WebhookRepository` no `OrderService`** | Mudaria a assinatura do construtor de `OrderService` (hoje `(orders, prisma)` em `src/app.ts`) e acoplaria pedidos ao módulo de webhooks inteiro. Uma função pura que recebe o `tx` resolve com acoplamento mínimo ([09:41] Diego: "Não precisa injetar repository inteiro"). |
| **Reusar as classes genéricas (`NotFoundError`, `ConflictError`) com os códigos padrão** (alternativa plausível) | `NotFoundError` sempre emite `NOT_FOUND` e não aceita código customizado (`src/shared/errors/http-errors.ts`). O cliente não conseguiria distinguir "webhook não encontrado" de "rota não encontrada". O prefixo `WEBHOOK_` foi decisão explícita ([09:29] Larissa). |
| **Compartilhar a instância `prisma` da API com o worker** | Não é possível: `PrismaClient` é por processo, e o worker roda em processo separado ([09:30] Bruno, ADR-002). |

## Consequências

**Positivas**
- Quem já conhece o código entende o módulo novo de imediato. Revisão e onboarding ficam mais baratos.
- Zero dependências novas: o HTTP de saída usa o `fetch` nativo do Node ≥ 20 (`package.json` → `engines.node`), e o HMAC usa `node:crypto`.
- O formato de erro é consistente para os consumidores da API.

**Negativas**
- O padrão herda limitações atuais. Por exemplo, o `validate.middleware.ts` transforma **qualquer** `ZodError` em `VALIDATION_ERROR`, então regras que precisam emitir `WEBHOOK_*` têm de rodar no service, e não no schema Zod (detalhado no FDD).
- O projeto não tem biblioteca de métricas nem de tracing, então a observabilidade do worker fica limitada a logs estruturados e consultas às tabelas (ver FDD).
- `OrderService.changeStatus` passa a importar uma função do módulo de webhooks. É uma dependência nova entre módulos, mantida mínima.

**Trade-off explícito:** consistência com a base de código vale mais do que a liberdade de escolher ferramentas "ideais" para a feature. Abrimos mão de métricas e tracing de primeira classe nesta fase para não introduzir stack nova.
