# FDD: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Status** | Pronto para implementação (pendências listadas em §15) |
| **Autor** | Lucas Pinho |
| **Revisão técnica** | Larissa, Bruno, Diego ([09:50] Larissa: sessão de revisão antes de codar) · Segurança: Sofia ([09:46]) |
| **Base** | [RFC](RFC.md) · [ADRs](adrs/README.md) · [PRD](PRD.md) · rastreabilidade em [TRACKER](TRACKER.md) |

> Convenção: cada item cita a origem. `[hh:mm] Nome` remete à `TRANSCRICAO.md`, e caminhos `src/...` remetem ao código. Os itens marcados como **(proposta)** são decisões de implementação que a reunião **não** tomou. Eles estão consolidados em §15 para validação.
>
> **Arquivos a criar (ainda não existem no repositório):** `src/worker.ts`, `src/modules/webhooks/` (`webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts`, `webhook.worker.ts`, `webhook.publisher.ts`, `webhook.errors.ts`). Todos os outros caminhos citados neste documento já existem no código base.

---

## 1. Contexto e motivação técnica

O OMS não tem nenhum mecanismo de notificação externa. Os clientes B2B fazem polling em `GET /api/v1/orders` para descobrir mudanças de status ([09:00] Marcos). A mudança de status fica em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), que, dentro de `this.prisma.$transaction(async (tx) => …)`:

1. carrega a ordem com `items`;
2. valida a transição com `canTransition` (`src/modules/orders/order.status.ts`);
3. debita ou repõe estoque (`shouldDebitStock`: `PENDING→PAID`; `shouldReplenishStock`: `PAID|PROCESSING→CANCELLED`);
4. faz `tx.order.update` e `tx.orderStatusHistory.create`;
5. relê a ordem com relações e retorna.

Essa transação é o **único** ponto onde o status de um pedido existente muda. É nela que o evento precisa nascer, de forma atômica ([09:40] Bruno, [ADR-001](adrs/ADR-001-outbox-no-mysql.md)). Fazer HTTP dentro dela está descartado ([09:04] Bruno).

## 2. Objetivos técnicos

| ID | Objetivo | Medida | Origem |
|---|---|---|---|
| OT-01 | Atomicidade entre status e evento | 0 mudanças de status commitadas sem linha correspondente na outbox (para webhooks interessados) | [09:40] Bruno, [09:41] Diego |
| OT-02 | Latência de entrega | Do commit à 1ª tentativa: < 10 s (esperado ≤ 2 s de captura + duração do POST) | [09:02] Marcos, [09:10] Larissa |
| OT-03 | Isolamento de falhas do cliente | Cliente lento ou fora do ar não afeta `PATCH /orders/:id/status` | [09:04] Bruno |
| OT-04 | Resiliência | Até 5 retentativas em ~14,6 h, depois DLQ, sem perda (contagem por interpretação, RFC Q-04) | [09:17] Diego |
| OT-05 | Autenticidade e integridade | 100% das requisições assinadas com HMAC-SHA256 da secret do endpoint | [09:22] Sofia |
| OT-06 | Zero infraestrutura nova | Sem dependências npm novas, sem serviços novos além do processo worker | [09:07] Diego, [09:30] Larissa |

## 3. Escopo e exclusões

**Incluso:** tabelas `webhooks`, `webhook_outbox`, `webhook_deliveries`, `webhook_dead_letter`; módulo `src/modules/webhooks`; `publishWebhookEvent` chamado em `changeStatus`; entry-point `src/worker.ts` + `npm run worker`; os 7 endpoints de §6; assinatura, rotação, retry, DLQ e replay.

**Excluído** (com origem):

| Item | Situação | Origem |
|---|---|---|
| Webhooks inbound (cliente → plataforma) | Fora de escopo | [09:02] Marcos, [09:03] Sofia |
| Notificação por e-mail quando o webhook falha | Próxima fase | [09:37] Larissa |
| Dashboard visual para o cliente | Projeto separado do time de frontend | [09:40] Larissa |
| Rate limiting de saída | Observar e decidir depois | [09:39] Larissa |
| Arquivamento de linhas entregues (~30 dias) | Fora desta feature | [09:08] Diego |
| Múltiplos workers e ordenação global | Futuro | [09:13] Diego, [09:13] Larissa |
| Exactly-once | Descartado | [09:25] Diego |
| Itens do pedido no payload | Descartado (cliente usa `GET /orders/:id`) | [09:43] Diego |
| Endurecimento de roles no CRUD de configuração | Futuro | [09:37] Sofia |

## 4. Modelo de dados

Acréscimos a `prisma/schema.prisma`, seguindo as convenções existentes: `id` UUID `@db.Char(36)` ([09:51] Larissa), `createdAt`/`updatedAt` e `@@map` em snake_case, como nos models atuais.

```prisma
enum WebhookOutboxStatus {
  PENDING      // "pendente"    [09:08] Diego
  PROCESSING   // "processando"
  DELIVERED    // "entregue"
  FAILED       // "falhou": movido para a DLQ ou descartado
}

model Webhook {
  id                      String    @id @default(uuid()) @db.Char(36)
  customerId              String    @db.Char(36)                 // [09:21] Bruno
  url                     String    @db.VarChar(2048)            // https obrigatório [09:23] Sofia
  secret                  String    @db.VarChar(128)             // por endpoint [09:21] Sofia
  previousSecret          String?   @db.VarChar(128)             // rotação [09:21] Sofia
  previousSecretExpiresAt DateTime?                              // now() + 24h na rotação
  events                  Json                                   // OrderStatus[] [09:33] Marcos
  active                  Boolean   @default(true)               // "estado ativo" [09:21] Bruno
  deletedAt               DateTime?                              // (proposta) soft delete, ver §15
  createdAt               DateTime  @default(now())
  updatedAt               DateTime  @updatedAt

  customer   Customer            @relation(fields: [customerId], references: [id])
  outbox     WebhookOutbox[]
  deliveries WebhookDelivery[]

  @@index([customerId, active])
  @@map("webhooks")
}

model WebhookOutbox {
  id            String              @id @default(uuid()) @db.Char(36)
  eventId       String              @unique @db.Char(36)       // X-Event-Id [09:25] Diego
  webhookId     String              @db.Char(36)
  orderId       String              @db.Char(36)               // sem FK: snapshot sobrevive a DELETE /orders
  eventType     String              @db.VarChar(64)            // "order.status_changed"
  payload       String              @db.MediumText             // JSON serializado (snapshot) [09:52] Larissa
  status        WebhookOutboxStatus @default(PENDING)
  attemptCount  Int                 @default(0)
  nextAttemptAt DateTime            @default(now())
  lastError     String?             @db.VarChar(500)
  deliveredAt   DateTime?
  createdAt     DateTime            @default(now())
  updatedAt     DateTime            @updatedAt

  webhook    Webhook           @relation(fields: [webhookId], references: [id])
  deliveries WebhookDelivery[]

  @@index([status])                                            // [09:08] Diego
  @@index([createdAt])                                         // [09:08] Diego
  @@index([status, nextAttemptAt, createdAt])                  // query do worker (§5.2)
  @@index([webhookId])
  @@map("webhook_outbox")
}

model WebhookDelivery {                                          // histórico [09:34] Marcos
  id             String   @id @default(uuid()) @db.Char(36)
  webhookId      String   @db.Char(36)
  outboxId       String   @db.Char(36)
  eventId        String   @db.Char(36)
  attemptNumber  Int
  success        Boolean
  responseStatus Int?
  responseBody   String?  @db.Text                             // truncado (proposta: 4 KB)
  durationMs     Int                                           // "tempo de resposta"
  errorCode      String?  @db.VarChar(64)                      // WEBHOOK_DELIVERY_*
  createdAt      DateTime @default(now())

  webhook Webhook       @relation(fields: [webhookId], references: [id])
  outbox  WebhookOutbox @relation(fields: [outboxId], references: [id])  // payload vem via join

  @@index([webhookId, createdAt])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {                                        // [09:18] Diego
  id            String    @id @default(uuid()) @db.Char(36)
  outboxId      String    @unique @db.Char(36)
  eventId       String    @db.Char(36)
  webhookId     String    @db.Char(36)
  payload       String    @db.MediumText                       // "com a payload"
  failureReason String    @db.VarChar(500)                     // "motivo da falha"
  errorCode     String    @db.VarChar(64)
  failedAt      DateTime  @default(now())                      // "timestamp"
  replayedAt    DateTime?
  replayedById  String?   @db.Char(36)                         // auditoria [09:36] Sofia

  @@index([webhookId])
  @@index([failedAt])
  @@map("webhook_dead_letter")
}
```

Notas de modelagem:
- **`payload` é `MediumText`, não `Json`.** A coluna `JSON` do MySQL normaliza o documento (reordena chaves, remove espaços). Como a assinatura HMAC é calculada **sobre o corpo** ([09:22] Sofia), guardamos a string exata que será enviada, para que todas as retentativas mandem bytes idênticos. `TEXT` (65.535 bytes) é 1 byte menor que o limite de 64 KB ([09:24] Diego), por isso o tipo é `MEDIUMTEXT`.
- **`Customer` ganha** `webhooks Webhook[]`.
- **`event_id` ≠ `id` da outbox:** o replay recoloca o **mesmo** evento na fila ([09:18] Diego), e o `event_id` precisa ser estável para o cliente deduplicar ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)).

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox (API, dentro de `changeStatus`)

```
OrderService.changeStatus(id, input, userId)
└─ prisma.$transaction(async (tx) => {
     order = tx.order.findUnique({ id, include: items })     // existente
     valida transição / estoque / update / history            // existente
     ▶ await publishWebhookEvent(tx, order, from, to)         // NOVO [09:41] Bruno
     refreshed = tx.order.findUnique(...)                     // existente
   })
```

`publishWebhookEvent(tx, order, fromStatus, toStatus)` em `src/modules/webhooks/webhook.publisher.ts` **(proposta de nome de arquivo)**:

1. `webhooks = tx.webhook.findMany({ where: { customerId: order.customerId, active: true, deletedAt: null } })`.
2. Mantém apenas os webhooks cujo `events` contém `toStatus`. Se não sobrar nenhum, **retorna sem inserir nada** ([09:34] Bruno).
3. Para cada webhook interessado:
   - `eventId = uuidv4()` (pacote `uuid`, já usado em `src/middlewares/request-logger.middleware.ts`);
   - monta o payload (§7.1) e faz `JSON.stringify` **uma única vez**;
   - `tx.webhookOutbox.create({ eventId, webhookId, orderId, eventType: 'order.status_changed', payload, status: PENDING, nextAttemptAt: now })`.
4. Qualquer exceção de banco é encapsulada em `WebhookOutboxInsertError` (500 `WEBHOOK_OUTBOX_INSERT_FAILED`) **(proposta)** e relançada. A `$transaction` faz rollback de **tudo**: status, histórico, estoque e outbox ([09:40] Bruno). O encapsulamento existe porque, sem ele, um erro Prisma cru chegaria ao `errorMiddleware` como 500 `INTERNAL_SERVER_ERROR`, ou como 409 `CONFLICT` no caso de P2002.

O limite de 64 KB **não** é verificado aqui. Uma exceção nesse ponto derrubaria a mudança de status, e Sofia pediu "não envia / erra", não "bloqueia o pedido" ([09:23] Sofia). A verificação fica no worker (§5.2, passo 4) **(proposta)**.

### 5.2 Processamento pelo worker

Entry-point `src/worker.ts` → `startWebhookWorker()` em `src/modules/webhooks/webhook.worker.ts` ([09:28] Bruno).

**Startup**
1. Cria o próprio `PrismaClient` via `createPrismaClient()` (`src/config/database.ts`) ([09:30] Bruno).
2. **Recuperação de órfãos:** `UPDATE webhook_outbox SET status='PENDING' WHERE status='PROCESSING'`. Esses eventos ficaram travados por crash no ciclo anterior. Reenviá-los pode gerar duplicata, o que é aceito pela semântica at-least-once ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)) **(proposta)**.
3. Registra `SIGINT`/`SIGTERM` no mesmo padrão de `src/server.ts`: termina o item em andamento, faz `prisma.$disconnect()` e sai.

**Ciclo (a cada `WEBHOOK_POLL_INTERVAL_MS` = 2000)** ([09:09] Diego)
1. Busca o lote:
   ```sql
   SELECT * FROM webhook_outbox
    WHERE status = 'PENDING' AND next_attempt_at <= NOW()
    ORDER BY created_at ASC
    LIMIT :WEBHOOK_BATCH_SIZE;      -- "batch pequeno" [09:08] Diego
   ```
2. Processa os itens **sequencialmente**, na ordem de `created_at`. Isso dá a ordenação por `order_id` em single-worker ([09:12] Diego).
3. Para cada item, `UPDATE … SET status='PROCESSING' WHERE id=? AND status='PENDING'`. Se nenhuma linha for afetada, pula o item.
4. Carrega o webhook:
   - se foi removido (`deletedAt != null`) → `status=FAILED`, `lastError='WEBHOOK_NOT_FOUND'`, **sem DLQ** **(proposta)**;
   - se está inativo (`active=false`) → move para a DLQ com `WEBHOOK_INACTIVE`, para permitir replay após reativação **(proposta)**;
   - se `Buffer.byteLength(payload) > WEBHOOK_MAX_PAYLOAD_BYTES` (65.536) → DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`, **sem retentativas**, porque é erro determinístico ([09:23] Sofia, [09:24] Diego).
5. Assina e envia (§7.2 e §7.3): `fetch(url, { method: 'POST', body: payload, headers, signal: AbortSignal.timeout(10_000), redirect: 'manual' })` ([09:42] Diego).
6. Grava `webhook_deliveries` (tentativa, status HTTP, corpo truncado, `durationMs`, `errorCode`).
7. **Sucesso** = resposta 2xx → `status=DELIVERED`, `deliveredAt=now()`, `attemptCount+=1`.
8. **Falha** (timeout, rede, não-2xx) → vai para o fluxo §5.3.
9. Quando o lote termina, espera 2 s e repete. Se o lote veio cheio, a proposta é buscar o próximo imediatamente para drenar atrasos.

### 5.3 Retry com backoff

`attemptCount` conta **envios realizados**. Agenda das retentativas ([09:17] Diego):

| Envio que falhou (`attemptCount` após a falha) | Próxima ação | `nextAttemptAt` |
|---|---|---|
| 1 (envio inicial) | retentativa 1 | +1 min |
| 2 | retentativa 2 | +5 min |
| 3 | retentativa 3 | +30 min |
| 4 | retentativa 4 | +2 h |
| 5 | retentativa 5 | +12 h |
| 6 | **DLQ** (§5.4) | n/a |

```ts
const RETRY_DELAYS_MS = [60_000, 300_000, 1_800_000, 7_200_000, 43_200_000]; // 1m/5m/30m/2h/12h
// falha: attemptCount += 1
// attemptCount <= RETRY_DELAYS_MS.length ? reagenda : DLQ
```

Ao reagendar: `status=PENDING`, `nextAttemptAt = now + RETRY_DELAYS_MS[attemptCount-1]`, `lastError = <código>`. A interpretação "5 retentativas após o envio inicial" está justificada no [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) e aberta para confirmação no RFC Q-04.

### 5.4 DLQ

Em uma única transação:
1. `webhook_outbox.status = FAILED`, `lastError = <código final>`;
2. `INSERT webhook_dead_letter { outboxId, eventId, webhookId, payload, failureReason, errorCode, failedAt }` ([09:18] Diego);
3. log `warn` `webhook_moved_to_dead_letter`.

### 5.5 Replay manual da DLQ

`POST /api/v1/admin/webhooks/dead-letter/:id/replay` ([09:18] Diego), protegido por `authenticate` + `requireRole('ADMIN')` ([09:36] Larissa):

1. Carrega o registro da DLQ. Se não existir → `WEBHOOK_DEAD_LETTER_NOT_FOUND`. Se `replayedAt != null` → `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`.
2. Se o webhook estiver inativo ou removido → `WEBHOOK_INACTIVE`.
3. Na mesma transação: a outbox vai para `status=PENDING`, `attemptCount=0`, `nextAttemptAt=now()` (o **mesmo `eventId`**, por inferência de "recoloca na outbox", ver ADR-005), e a DLQ recebe `replayedAt=now()`, `replayedById=req.user.id`.
4. Log `info` `webhook_dead_letter_replayed` com `{ deadLetterId, eventId, webhookId, userId }`, como auditoria de "quem fez o replay" ([09:36] Sofia).

### 5.6 Rotação de secret

`POST /api/v1/webhooks/:id/rotate-secret` ([09:21] Sofia):
1. Se já existe rotação em andamento (`previousSecretExpiresAt > now()`) → `WEBHOOK_SECRET_ROTATION_IN_PROGRESS` **(proposta)**. Uma segunda rotação dentro da janela invalidaria imediatamente a secret que o cliente ainda está migrando.
2. `previousSecret = secret`, `previousSecretExpiresAt = now() + 24h`, `secret = generateSecret()`.
3. Retorna a nova secret **uma vez**.
4. O worker assina com as duas enquanto `previousSecretExpiresAt > now()` (§7.3). Depois disso, ignora `previousSecret`.

## 6. Contratos públicos (API de gestão)

Base: `/api/v1` (`src/app.ts`). Todos os endpoints exigem `Authorization: Bearer <JWT>` (`authenticate`). Erros seguem o envelope do `errorMiddleware`: `{ "error": { "code", "message", "details?" } }`.

Autorização: CRUD, rotação e histórico aceitam **qualquer role autenticada** ([09:37] Sofia). O replay exige **`ADMIN`** ([09:36] Sofia).

Identificação do cliente: `customerId` **não** vem do JWT, porque o JWT é do usuário operador ([09:32] Bruno, [09:32] Larissa). A proposta é usar o body no `POST` e a query no `GET` de listagem (RFC Q-02).

### 6.1 `POST /api/v1/webhooks`: cadastrar webhook ([09:31] Marcos)

Request:
```http
POST /api/v1/webhooks
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{
  "customerId": "8b4f2a8e-3c1d-4e59-9a7b-2f6c1d0e9a11",
  "url": "https://hooks.atlascomercial.com.br/oms/orders",
  "events": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created` (a **única** resposta, junto com a rotação, que expõe a secret):
```json
{
  "id": "c1e0b6a4-7f2d-4b8e-8f21-5d9a0c3e4b77",
  "customerId": "8b4f2a8e-3c1d-4e59-9a7b-2f6c1d0e9a11",
  "url": "https://hooks.atlascomercial.com.br/oms/orders",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "3f9a1c0e5b7d4e2a8c6f0b1d3e5a7c9e2b4d6f8a0c1e3b5d7f9a2c4e6b8d0f1a",
  "createdAt": "2026-11-03T13:02:11.482Z",
  "updatedAt": "2026-11-03T13:02:11.482Z"
}
```

| Status | Código | Quando |
|---|---|---|
| 201 | n/a | criado |
| 400 | `VALIDATION_ERROR` | body fora do schema Zod (campos ausentes, `customerId` não-UUID, `url` não é URL) |
| 400 | `WEBHOOK_INVALID_URL` | esquema diferente de `https` ([09:23] Sofia) |
| 400 | `WEBHOOK_INVALID_EVENTS` | `events` vazio, com duplicatas ou contendo `PENDING` (RFC Q-07) |
| 401 | `UNAUTHORIZED` | sem token ou token inválido |
| 404 | `WEBHOOK_CUSTOMER_NOT_FOUND` | `customerId` inexistente |

### 6.2 `GET /api/v1/webhooks?customerId=…&page=1&pageSize=20`: listar webhooks de um customer ([09:33] Bruno)

Response `200 OK`, usando `paginated()` de `src/shared/http/response.ts`. Nunca devolve `secret`:
```json
{
  "data": [
    {
      "id": "c1e0b6a4-7f2d-4b8e-8f21-5d9a0c3e4b77",
      "customerId": "8b4f2a8e-3c1d-4e59-9a7b-2f6c1d0e9a11",
      "url": "https://hooks.atlascomercial.com.br/oms/orders",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "secretRotation": null,
      "createdAt": "2026-11-03T13:02:11.482Z",
      "updatedAt": "2026-11-03T13:02:11.482Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```
`secretRotation` vale `{ "previousSecretExpiresAt": "<ISO>" }` durante o grace period.

| Status | Código | Quando |
|---|---|---|
| 200 | n/a | sucesso (lista vazia se o cliente não tiver webhooks) |
| 400 | `VALIDATION_ERROR` | `customerId` ausente ou não-UUID |
| 401 | `UNAUTHORIZED` | n/a |
| 404 | `WEBHOOK_CUSTOMER_NOT_FOUND` | customer inexistente |

### 6.3 `PATCH /api/v1/webhooks/:id`: editar ([09:33] Bruno)

Request (todos os campos opcionais, pelo menos um obrigatório):
```json
{ "url": "https://hooks.atlascomercial.com.br/v2/orders", "events": ["PAID", "SHIPPED", "DELIVERED"], "active": true }
```
Response `200 OK`: mesmo formato de um item de §6.2, sem `secret`.

| Status | Código | Quando |
|---|---|---|
| 200 | n/a | atualizado |
| 400 | `VALIDATION_ERROR` | body vazio ou tipos inválidos |
| 400 | `WEBHOOK_INVALID_URL` | URL não-https |
| 400 | `WEBHOOK_INVALID_EVENTS` | lista inválida |
| 401 | `UNAUTHORIZED` | n/a |
| 404 | `WEBHOOK_NOT_FOUND` | id inexistente ou removido |

A mudança de `events` só afeta eventos **futuros**. O que já está na outbox não muda, porque é snapshot ([ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md)).

### 6.4 `DELETE /api/v1/webhooks/:id`: remover ([09:33] Bruno)

Response `204 No Content`, sem corpo. Faz soft delete (`deletedAt=now()`, `active=false`) para preservar o histórico. Eventos pendentes desse webhook são descartados pelo worker (§5.2, passo 4) **(proposta)**.

| Status | Código | Quando |
|---|---|---|
| 204 | n/a | removido |
| 401 | `UNAUTHORIZED` | n/a |
| 404 | `WEBHOOK_NOT_FOUND` | inexistente ou já removido |

### 6.5 `POST /api/v1/webhooks/:id/rotate-secret`: rotacionar secret ([09:21] Sofia)

Request: sem corpo. Response `200 OK`:
```json
{
  "id": "c1e0b6a4-7f2d-4b8e-8f21-5d9a0c3e4b77",
  "secret": "9d2b7e4a1c8f3b6e0a5d2c9f7b4e1a8d3c6f0b9e2a7d4c1f8b5e3a0d6c9f2b7e",
  "previousSecretExpiresAt": "2026-11-04T13:10:00.000Z"
}
```

| Status | Código | Quando |
|---|---|---|
| 200 | n/a | rotacionada; a antiga vale por mais 24 h |
| 401 | `UNAUTHORIZED` | n/a |
| 404 | `WEBHOOK_NOT_FOUND` | n/a |
| 409 | `WEBHOOK_INACTIVE` | webhook inativo |
| 409 | `WEBHOOK_SECRET_ROTATION_IN_PROGRESS` | já existe rotação dentro do grace period **(proposta)** |

### 6.6 `GET /api/v1/webhooks/:id/deliveries?limit=100`: histórico de entregas ([09:34] Marcos)

Retorna as **últimas 100** tentativas (`limit` entre 1 e 100, default 100), da mais recente para a mais antiga, com sucesso ou falha, payload, response e tempo de resposta:
```json
{
  "data": [
    {
      "id": "0a7c3e5f-1b9d-4c2e-8a6f-3d5b7e9c1a2f",
      "eventId": "5c2e9a7b-3d1f-4e8a-9b6c-0d2f4a6c8e1b",
      "attemptNumber": 2,
      "success": true,
      "responseStatus": 200,
      "responseBody": "{\"received\":true}",
      "durationMs": 184,
      "errorCode": null,
      "payload": "{\"event_id\":\"5c2e9a7b-…\",\"event_type\":\"order.status_changed\",…}",
      "createdAt": "2026-11-03T14:21:07.019Z"
    },
    {
      "id": "e4b1d8a2-6c3f-4a9e-b7d0-1f5c3a8e6b2d",
      "eventId": "5c2e9a7b-3d1f-4e8a-9b6c-0d2f4a6c8e1b",
      "attemptNumber": 1,
      "success": false,
      "responseStatus": null,
      "responseBody": null,
      "durationMs": 10000,
      "errorCode": "WEBHOOK_DELIVERY_TIMEOUT",
      "payload": "{\"event_id\":\"5c2e9a7b-…\",…}",
      "createdAt": "2026-11-03T14:20:05.331Z"
    }
  ]
}
```

| Status | Código | Quando |
|---|---|---|
| 200 | n/a | sucesso |
| 400 | `VALIDATION_ERROR` | `limit` fora de 1..100 |
| 401 | `UNAUTHORIZED` | n/a |
| 404 | `WEBHOOK_NOT_FOUND` | n/a |

### 6.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay`: replay da DLQ ([09:18] Diego, [09:36] Sofia)

Request: sem corpo. Response `202 Accepted`, porque o evento volta para a fila e será entregue de forma assíncrona:
```json
{
  "deadLetterId": "7e3a9c1f-5b2d-4e8a-a6c4-9f1d3b5e7a20",
  "outboxId": "b2d4f6a8-0c1e-4a3b-9d5f-7e9a1c3b5d70",
  "eventId": "5c2e9a7b-3d1f-4e8a-9b6c-0d2f4a6c8e1b",
  "status": "PENDING",
  "replayedAt": "2026-11-04T09:12:44.120Z",
  "replayedById": "a9f3c7e1-2b4d-4f6a-8c0e-1d3b5f7a9c2e"
}
```

| Status | Código | Quando |
|---|---|---|
| 202 | n/a | evento recolocado na outbox |
| 401 | `UNAUTHORIZED` | n/a |
| 403 | `FORBIDDEN` | role ≠ `ADMIN` (`requireRole`) |
| 404 | `WEBHOOK_DEAD_LETTER_NOT_FOUND` | id inexistente |
| 409 | `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | já reprocessado |
| 409 | `WEBHOOK_INACTIVE` | webhook inativo ou removido |

## 7. Contrato da notificação enviada ao cliente (outbound)

### 7.1 Corpo ([09:43] Diego)

```json
{
  "event_id": "5c2e9a7b-3d1f-4e8a-9b6c-0d2f4a6c8e1b",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-03T14:20:03.912Z",
  "order_id": "f0a1b2c3-d4e5-4f60-8a71-92b3c4d5e6f7",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "8b4f2a8e-3c1d-4e59-9a7b-2f6c1d0e9a11",
  "total_cents": 15990
}
```
- `timestamp`: momento da mudança de status (inserção na outbox), em ISO 8601.
- `order_number` segue o formato `ORD-NNNNNN` gerado por `reserveOrderNumber` (`src/modules/orders/order.service.ts`).
- Sem `items` ([09:43] Diego).

### 7.2 Headers ([09:44] Diego, [09:44] Sofia)

| Header | Valor | Observação |
|---|---|---|
| `Content-Type` | `application/json` | n/a |
| `X-Event-Id` | `event_id` (UUID) | estável entre retentativas e replay; usado para dedup ([09:25] Diego) |
| `X-Webhook-Id` | `id` do webhook | ajuda clientes com vários cadastros ([09:44] Sofia) |
| `X-Timestamp` | ISO 8601 do **envio** | permite ao cliente detectar replay ([09:44] Diego) |
| `X-Signature` | `sha256=<hex>` | HMAC-SHA256 do corpo; ver §7.3 |

### 7.3 Assinatura ([09:20] Sofia, [09:22] Sofia)

```ts
import { createHmac, randomBytes, timingSafeEqual } from 'node:crypto';

export const generateSecret = (): string => randomBytes(32).toString('hex'); // (proposta) validar na revisão [09:46] Sofia

export function sign(rawBody: string, secret: string): string {
  return `sha256=${createHmac('sha256', secret).update(rawBody, 'utf8').digest('hex')}`;
}

// Durante o grace period de 24h (proposta, RFC Q-05):
// X-Signature: sha256=<hmac com secret nova>,sha256=<hmac com secret antiga>
```
Verificação sugerida para o portal do cliente: recalcular o HMAC sobre o **corpo bruto** e comparar com `timingSafeEqual` contra cada valor do header.

## 8. Matriz de erros

Todas as classes ficam em `src/modules/webhooks/webhook.errors.ts`, estendem `AppError` (`src/shared/errors/app-error.ts`) e são serializadas pelo `errorMiddleware` sem nenhuma alteração ([09:29] Bruno).

**Erros da API (HTTP)**

| Código | HTTP | Classe | Gatilho | Endpoint(s) |
|---|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | `WebhookNotFoundError` | webhook inexistente ou removido | 6.3–6.6 |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `WebhookCustomerNotFoundError` | `customerId` inexistente | 6.1, 6.2 |
| `WEBHOOK_INVALID_URL` | 400 | `WebhookInvalidUrlError` | URL sem `https` ([09:23] Sofia) | 6.1, 6.3 |
| `WEBHOOK_INVALID_EVENTS` | 400 | `WebhookInvalidEventsError` | `events` vazio, duplicado ou com `PENDING` | 6.1, 6.3 |
| `WEBHOOK_INACTIVE` | 409 | `WebhookInactiveError` | operação sobre webhook inativo | 6.5, 6.7 |
| `WEBHOOK_SECRET_ROTATION_IN_PROGRESS` | 409 | `WebhookSecretRotationInProgressError` | rotação dentro do grace period | 6.5 |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `WebhookDeadLetterNotFoundError` | id da DLQ inexistente | 6.7 |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | `WebhookDeadLetterAlreadyReplayedError` | replay repetido | 6.7 |
| `WEBHOOK_OUTBOX_INSERT_FAILED` | 500 | `WebhookOutboxInsertError` | falha ao gravar a outbox em `changeStatus`; **rollback** da mudança de status **(proposta, P-14)** | `PATCH /orders/:id/status` |

**Erros do worker** (não chegam ao cliente HTTP da API; ficam em `webhook_deliveries.errorCode`, `webhook_outbox.lastError` e `webhook_dead_letter.errorCode`)

| Código | Retentável? | Gatilho |
|---|---|---|
| `WEBHOOK_DELIVERY_TIMEOUT` | Sim | sem resposta em 10 s ([09:42] Diego) |
| `WEBHOOK_DELIVERY_NETWORK_ERROR` | Sim | DNS, conexão recusada, TLS inválido |
| `WEBHOOK_DELIVERY_HTTP_ERROR` | Sim | resposta fora de 2xx (inclui 3xx, já que redirect não é seguido) |
| `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | Não (→ DLQ) | 6º envio falhou |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | Não (→ DLQ) | corpo > 64 KB ([09:24] Diego) |
| `WEBHOOK_INACTIVE` | Não (→ DLQ) | webhook desativado com evento pendente |
| `WEBHOOK_SECRET_REQUIRED` | Não (→ DLQ) | webhook sem secret no momento do envio (invariante violada) ([09:28] Bruno) |

Detalhes de implementação ligados ao código existente:
- `NotFoundError` emite sempre `NOT_FOUND` e não aceita código. Por isso `WebhookNotFoundError` estende `AppError` diretamente: `super('Webhook not found', 404, 'WEBHOOK_NOT_FOUND')`. Já as classes 409 podem estender `ConflictError(message, code)`, que aceita código.
- **Validação de https: conflito entre duas falas da reunião.** A Sofia disse que a regra "é só uma validação no schema Zod" ([09:23] Sofia). O Bruno citou `WEBHOOK_INVALID_URL` como código do módulo ([09:28] Bruno). Como o `validate.middleware.ts` converte qualquer `ZodError` em `VALIDATION_ERROR`, as duas coisas não convivem. Proposta (P-11, **diverge de [09:23] Sofia e depende do aceite dela**): o Zod valida só `z.string().url()`, e o `WebhookService` verifica `new URL(url).protocol === 'https:'` e lança `WebhookInvalidUrlError`. Se a Sofia não aceitar, a regra volta para o Zod, a resposta passa a ser 400 `VALIDATION_ERROR` com `details[].path = "url"`, e `WEBHOOK_INVALID_URL` sai da matriz. O mesmo raciocínio vale para `WEBHOOK_INVALID_EVENTS`.

## 9. Estratégias de resiliência

| Mecanismo | Valor | Origem |
|---|---|---|
| Timeout por chamada HTTP | 10 s (`AbortSignal.timeout`) | [09:42] Diego |
| Retentativas | 5, com backoff 1m/5m/30m/2h/12h (~14h36min) | [09:17] Diego |
| Fallback após esgotar | DLQ + replay manual ADMIN | [09:18] Diego |
| Fallback de notificação (e-mail) | **Não** nesta fase | [09:37] Larissa |
| Isolamento | HTTP fora da transação de pedidos; worker em processo próprio | [09:04] Bruno, [09:11] Diego |
| Durabilidade | Outbox na mesma transação; eventos sobrevivem a restart da API e do worker | [09:06] Diego |
| Recuperação de crash | `PROCESSING → PENDING` no startup **(proposta)** | at-least-once [09:24] Diego |
| Erros determinísticos | Payload > 64 KB e webhook inativo vão direto para a DLQ, sem gastar retentativas **(proposta)** | [09:23] Sofia |
| Redirects | Não seguidos (`redirect: 'manual'`) **(proposta, revisão de segurança)** | [09:46] Sofia |
| Rate limiting de saída | Não implementado; observar | [09:39] Larissa |

## 10. Observabilidade

O projeto tem **apenas** logger Pino (`src/shared/logger/index.ts`). Não há biblioteca de métricas nem de tracing, e a decisão é não adicionar stack nova ([09:29] Bruno, [09:30] Larissa). Por isso, a observabilidade desta fase se apoia em logs estruturados, consultas SQL e IDs de correlação.

### 10.1 Logs (Pino)

O worker usa `logger.child({ component: 'webhook-worker' })`. O campo `service` da base continua sendo `order-management-api`.

| Evento (msg) | Nível | Campos |
|---|---|---|
| `webhook_event_enqueued` | info | `eventId`, `webhookId`, `orderId`, `toStatus` (logado após o commit) |
| `webhook_worker_started` / `webhook_worker_stopped` | info | `pollIntervalMs`, `batchSize`, `orphansRecovered` |
| `webhook_delivery_succeeded` | info | `eventId`, `webhookId`, `orderId`, `attempt`, `responseStatus`, `durationMs` |
| `webhook_delivery_failed` | warn | os mesmos + `errorCode`, `nextAttemptAt` |
| `webhook_moved_to_dead_letter` | warn | `eventId`, `webhookId`, `errorCode`, `attempts` |
| `webhook_dead_letter_replayed` | info | `deadLetterId`, `eventId`, `userId` (auditoria [09:36] Sofia) |
| `webhook_secret_rotated` | info | `webhookId`, `previousSecretExpiresAt`, `userId` |

**Redação:** incluir `'*.secret'` e `'*.previousSecret'` em `redactPaths` de `src/shared/logger/index.ts`. Já houve vazamento de secret em log ([09:22] Diego).

### 10.2 Métricas

Calculadas a partir dos logs (agregador de logs) e de consultas às tabelas. Não há exporter no projeto.

| Métrica | Fonte | Uso / alerta |
|---|---|---|
| `outbox_pending_count` | `COUNT(*) WHERE status='PENDING'` | crescimento contínuo indica worker lento ou parado |
| `outbox_oldest_pending_age_s` | `NOW() - MIN(created_at) WHERE status='PENDING' AND attempt_count=0` | **alerta se > 10 s** (SLA [09:02] Marcos) |
| `delivery_latency_ms` (commit → 1ª tentativa) | `webhook_deliveries.createdAt - webhook_outbox.createdAt` (attempt 1) | objetivo do PRD (< 10 s) |
| `delivery_attempts_total{result}` | logs `webhook_delivery_*` | taxa de sucesso por webhook ou cliente |
| `delivery_duration_ms` | `webhook_deliveries.durationMs` | clientes lentos, perto do timeout de 10 s |
| `dead_letter_total` | `COUNT(*)` em `webhook_dead_letter` com `replayedAt IS NULL` | fila de ação manual |
| `events_per_webhook_per_min` | logs `webhook_event_enqueued` | insumo para decidir rate limiting (RFC Q-01) |

### 10.3 Tracing

Não existe tracing distribuído (OpenTelemetry) no projeto, e ele não entra nesta fase. A correlação ponta a ponta usa IDs:
- `orderId` liga `order_status_history` → `webhook_outbox` → `webhook_deliveries`;
- `eventId` aparece em todos os logs do evento **e** chega ao cliente como `X-Event-Id`, o que permite investigação conjunta com ele;
- `X-Webhook-Id` identifica qual cadastro gerou o envio ([09:44] Sofia);
- o `requestId` gerado em `src/middlewares/request-logger.middleware.ts` **não** é propagado para a outbox, porque `changeStatus` não o recebe. Propagá-lo exigiria mudar a assinatura do controller e do service, o que fica como melhoria futura.

## 11. Integração com o sistema existente

| # | Arquivo existente | Como o módulo de webhooks se integra |
|---|---|---|
| 1 | `src/modules/orders/order.service.ts` | Em `changeStatus`, logo após `tx.orderStatusHistory.create(...)` e antes da releitura `refreshed`, chamar `await publishWebhookEvent(tx, order, from, to)`. O `tx` é o mesmo `TxClient` (`Prisma.TransactionClient`) já tipado no arquivo, e qualquer erro propaga e faz rollback ([09:40] Bruno, [09:41] Bruno). O construtor `OrderService(orders, prisma)` **não muda** ([09:41] Diego). `create()` **não** publica evento nesta fase (RFC Q-07). |
| 2 | `src/modules/orders/order.status.ts` | Fonte da verdade dos status válidos. `webhook.schemas.ts` usa `z.nativeEnum(OrderStatus)` (como `order.schemas.ts`) e rejeita `PENDING`, já que nenhuma entrada de `transitions` tem `PENDING` como destino. |
| 3 | `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | Classes `Webhook*Error` estendem `AppError` (404, 400, 500) ou `ConflictError` (409, aceita `code`). `NotFoundError` não serve porque fixa `NOT_FOUND` ([09:28] Bruno). |
| 4 | `src/middlewares/error.middleware.ts` | **Sem alteração.** O ramo `err instanceof AppError` já serializa `{ code, message, details }` ([09:29] Bruno). Erros Prisma `P2025` em `update`/`delete` de webhook viram 404 `NOT_FOUND` genérico, então o service deve checar a existência antes e lançar `WEBHOOK_NOT_FOUND`. |
| 5 | `src/middlewares/auth.middleware.ts` | `authenticate` em todas as rotas. `requireRole('ADMIN')` na rota de replay, no mesmo padrão de `src/modules/users/user.routes.ts` ([09:36] Larissa). `req.user.id` alimenta `replayedById`. |
| 6 | `src/middlewares/validate.middleware.ts` | Validação de `params`, `body` e `query` com os schemas de `webhook.schemas.ts`. Atenção: produz `VALIDATION_ERROR`, por isso as regras com código `WEBHOOK_*` ficam no service (§8). |
| 7 | `src/routes/index.ts` | Adicionar `webhooks: WebhookController` e `adminWebhooks: WebhookAdminController` ao tipo `Controllers`; `router.use('/webhooks', buildWebhookRouter(...))` e `router.use('/admin/webhooks', buildWebhookAdminRouter(...))`. |
| 8 | `src/app.ts` | Em `buildControllers`, instanciar `WebhookRepository(prisma)`, `WebhookService` e os controllers, no mesmo padrão dos demais módulos. |
| 9 | `src/server.ts` | Modelo para o novo `src/worker.ts`: bootstrap, `logger.info(..., 'worker_started')`, shutdown em `SIGINT`/`SIGTERM` com `prisma.$disconnect()` e `bootstrap().catch(...)` com `logger.fatal` ([09:11] Larissa). |
| 10 | `src/config/database.ts` | O worker chama `createPrismaClient()` para ter a **própria** instância. Não importa o singleton `prisma` da API ([09:30] Bruno). |
| 11 | `src/config/env.ts` | Adicionar ao `envSchema`: `WEBHOOK_POLL_INTERVAL_MS` (default 2000), `WEBHOOK_HTTP_TIMEOUT_MS` (10000), `WEBHOOK_MAX_PAYLOAD_BYTES` (65536), `WEBHOOK_BATCH_SIZE` (default a definir, ver §15). Como `env.ts` valida no import e exige `JWT_SECRET`, o worker precisa do mesmo `.env` da API. |
| 12 | `src/shared/logger/index.ts` | Reusar `logger` (child `component: 'webhook-worker'`) e acrescentar `*.secret` e `*.previousSecret` a `redactPaths`. |
| 13 | `src/shared/http/response.ts` | `paginated()` e `PaginatedResponse` na listagem de webhooks (§6.2). |
| 14 | `prisma/schema.prisma` | Novos models e enum (§4), relação `Customer.webhooks`, nova migration em `prisma/migrations/`. |
| 15 | `package.json` | Scripts `"worker": "tsx --env-file=.env src/worker.ts"` (dev) e `"start:worker": "node --env-file=.env dist/worker.js"`, espelhando `dev` e `start` ([09:11] Larissa). Nenhuma dependência nova: `fetch` e `node:crypto` nativos (`engines.node >= 20`) e `uuid` já instalado. |
| 16 | `tests/setup.ts` e `tests/helpers/factories.ts` | `beforeEach` precisa limpar `webhookDelivery`, `webhookDeadLetter`, `webhookOutbox` e `webhook` **antes** de `customer.deleteMany()` (FK). Adicionar uma factory `createTestWebhook`. |

## 12. Dependências e compatibilidade

- **Runtime:** Node ≥ 20 (`fetch`, `AbortSignal.timeout`, `node:crypto`), TypeScript 5.6, Express 4.21, Prisma 5.22, MySQL 8.0 (`docker-compose.yml`), Zod 3.23, Pino 9.5, `uuid` 11. **Nenhum pacote novo.**
- **Banco:** migration aditiva (4 tabelas e 1 enum), sem alterar colunas existentes. É compatível com deploy sem downtime, desde que a migration rode **antes** do deploy da API, já que `changeStatus` passa a escrever em `webhook_outbox`.
- **API:** só endpoints novos. `PATCH /orders/:id/status` mantém contrato e resposta. O único efeito novo é um possível `500 WEBHOOK_OUTBOX_INSERT_FAILED`.
- **Ordem de deploy:** (1) migration → (2) API → (3) worker. O worker pode subir depois: os eventos acumulam na outbox sem perda.
- **Clientes:** precisam aceitar `POST` https, validar `X-Signature`, deduplicar por `X-Event-Id` e responder 2xx em até 10 s. A documentação no portal fica com o Marcos ([09:26] Marcos, [09:40] Marcos).
- **Dependência organizacional:** revisão de segurança da Sofia, com 2 dias úteis, antes do deploy ([09:46] Sofia, [09:49] Sofia).

## 13. Riscos e mitigação

| # | Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|---|
| R-01 | Falha ao inserir na outbox bloqueia mudança de status | Baixa | Alto | Insert simples sem I/O externo; teste de integração do rollback; alerta em `WEBHOOK_OUTBOX_INSERT_FAILED` |
| R-02 | Worker parado ou lento estoura o SLA de 10 s | Média | Alto | Alerta `outbox_oldest_pending_age_s > 10`; eventos persistidos sem perda; sugestão (não discutida): supervisor de processo com restart automático |
| R-03 | Duplicatas no cliente (at-least-once, recuperação de órfãos) | Média | Médio | `X-Event-Id` estável; documentação no portal ([09:26] Marcos) |
| R-04 | Inversão de ordem quando um evento entra em retry e o seguinte do mesmo pedido é entregue antes | Média | Médio | Payload traz `from_status`, `to_status` e `timestamp` para o cliente reconciliar; decisão em aberto (RFC Q-03) |
| R-05 | Vazamento de secret em logs | Média | Alto | `redactPaths` com `*.secret`; secret só nas respostas 201/200 de criação e rotação ([09:22] Diego) |
| R-06 | Crescimento ilimitado de `webhook_outbox` e `webhook_deliveries` | Alta | Médio | Índices em `status` e `created_at`; arquivamento planejado para depois ([09:08] Diego) |
| R-07 | Rajada de eventos sobrecarrega o cliente | Média | Médio | Monitorar `events_per_webhook_per_min`; rate limiting em aberto ([09:39] Larissa) |
| R-08 | URL apontando para rede interna (SSRF) | Baixa | Alto | Não discutido. Levar à revisão de segurança ([09:46] Sofia); `redirect: 'manual'` |
| R-09 | Prazo: 3 sprints contra o fim de novembro | Média | Alto | Escopo enxuto; ordem de entrega da estimativa da Larissa ([09:46] Larissa) |

## 14. Critérios de aceite técnicos

1. `PATCH /orders/:id/status` para um status assinado cria exatamente 1 linha `PENDING` em `webhook_outbox` por webhook ativo interessado, na mesma transação.
2. Se `publishWebhookEvent` lança erro, o status do pedido, o histórico e o estoque **não** são alterados (teste com falha simulada).
3. Um status que nenhum webhook do customer assina **não** cria linha na outbox.
4. O worker entrega um evento pendente em ≤ 2 s + tempo de resposta (medido contra um servidor HTTP local de teste).
5. O corpo enviado é byte a byte igual a `webhook_outbox.payload`, e `X-Signature` confere com `sign(payload, secret)`.
6. Headers `X-Event-Id`, `X-Webhook-Id`, `X-Timestamp`, `X-Signature` e `Content-Type: application/json` estão presentes.
7. Respostas não-2xx ou timeout acima de 10 s reagendam com 1m/5m/30m/2h/12h, e o 6º envio falho cria um registro em `webhook_dead_letter` com payload, motivo e timestamp. Se a resposta à RFC Q-04 for "5 envios no total", passa a ser o 5º envio.
8. Um payload acima de 64 KB não é enviado e vai para a DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`.
9. O replay com role `OPERATOR` responde 403. Com `ADMIN`, responde 202, preserva o `eventId` e registra `replayedById` e o log de auditoria.
10. Durante as 24 h seguintes à rotação, `X-Signature` traz as duas assinaturas. Depois disso, só a nova.
11. Cadastro com `http://` responde 400 `WEBHOOK_INVALID_URL`, ou `VALIDATION_ERROR` se a P-11 não for aceita. `GET /webhooks` e `PATCH` nunca retornam `secret`.
12. `GET /webhooks/:id/deliveries` retorna no máximo 100 itens, do mais recente para o mais antigo.
13. Matar o worker com um evento em `PROCESSING` e reiniciá-lo resulta em reentrega com o mesmo `X-Event-Id`.
14. `npm run lint` e `npm test` continuam verdes, e `tests/setup.ts` limpa as novas tabelas.
15. A revisão de segurança da Sofia está aprovada antes do deploy.

## 15. Decisões de implementação propostas (não tomadas na reunião)

Itens que o FDD precisou definir para ficar acionável. **Validar na sessão de revisão** ([09:50] Larissa) e na revisão de segurança ([09:46] Sofia).

| # | Proposta | Motivo | Relacionado |
|---|---|---|---|
| P-01 | `customerId` no body (POST) e na query (GET lista) | A reunião deixou "body ou path" em aberto | RFC Q-02 |
| P-02 | Tabela `webhook_deliveries` para o histórico | Necessária para "últimos 100, sucesso/falha, payload, response, tempo" | [09:34] Marcos |
| P-03 | DELETE como soft delete; pendentes do webhook removido descartados | Preserva histórico e DLQ | [09:33] Bruno |
| P-04 | Payload > 64 KB verificado no worker e enviado direto para a DLQ | Verificar na inserção derrubaria a mudança de status | [09:23] Sofia |
| P-05 | `X-Signature` com duas assinaturas durante o grace period | Formato não discutido | RFC Q-05 |
| P-06 | Rotação bloqueada (409) enquanto há grace period ativo | Evita invalidar a secret que o cliente está migrando | [09:21] Sofia |
| P-07 | Recuperação `PROCESSING → PENDING` no startup do worker | Crash entre envio e marcação | ADR-005 |
| P-08 | `WEBHOOK_BATCH_SIZE` configurável; valor default a definir com o Diego | A transcrição diz só "batch pequeno" | [09:08] Diego |
| P-09 | `responseBody` truncado em 4 KB no histórico | Evita que um cliente infle a tabela com respostas grandes | [09:34] Marcos |
| P-10 | `events` rejeita `PENDING` | Nenhuma transição leva a `PENDING` (`order.status.ts`) | RFC Q-07 |
| P-11 | Regra `https` no service, e não no Zod. **Diverge de [09:23] Sofia; requer o aceite dela.** | Emitir `WEBHOOK_INVALID_URL` em vez de `VALIDATION_ERROR` | [09:23] Sofia × [09:28] Bruno |
| P-12 | Redirects não seguidos | Segurança de saída | [09:46] Sofia |
| P-13 | Arquivo `webhook.publisher.ts` para `publishWebhookEvent` e `webhook.errors.ts` para as classes de erro; `webhook.worker.ts` escolhido entre "worker ou processor" | Organização dentro do módulo | [09:28] Bruno |
| P-14 | Encapsular erros da outbox em `WebhookOutboxInsertError` (500 `WEBHOOK_OUTBOX_INSERT_FAILED`) | Evitar que um erro Prisma cru vire 500 genérico ou 409 `CONFLICT` | [09:40] Bruno |
| P-15 | Detalhes de contrato não ditos na reunião: path `/webhooks/:id/rotate-secret`; prefixo `sha256=` no `X-Signature`; `X-Timestamp` em ISO 8601; replay respondendo 202; 409 `WEBHOOK_INACTIVE` na rotação; paginação na listagem | A transcrição define a capacidade, não a forma | [09:21] Sofia, [09:44] Diego |
| P-16 | Sucesso = HTTP 2xx; resposta 3xx conta como falha | Critério de sucesso não explicitado | [09:42] Diego |
| P-17 | Replay mantém o mesmo `eventId` | Inferência de "recoloca na outbox" | [09:18] Diego |
