# FDD: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Status** | Pronto para implementação (pendente das questões em aberto do RFC) |
| **Data** | 2026-11-19 |
| **Responsáveis técnicos** | Bruno (Pedidos), Diego (Plataforma); revisão de segurança: Sofia |
| **Base** | [RFC](RFC.md), [ADRs](adrs/README.md), `TRANSCRICAO.md`, código em `src/` e `prisma/` |

Convenções deste documento:
- Citações `[hh:mm] Nome` apontam para `TRANSCRICAO.md`; caminhos apontam para arquivos do repositório.
- Itens marcados com **(proposta)** são escolhas de implementação deste FDD que a reunião não fechou; cada um é derivado de uma regra discutida e precisa de aceite na sessão de revisão que Larissa marcou com Bruno e Diego ([09:50] Larissa).
- Arquivos em `src/modules/webhooks/` e `src/worker.ts` **ainda não existem**: são os arquivos a criar ([09:11] Larissa, [09:27] Bruno, [09:28] Bruno).

---

## 1. Contexto e motivação técnica

O OMS não tem eventos, filas nem chamadas de saída. A única forma de um cliente saber que um pedido mudou é consultar `GET /api/v1/orders` repetidamente ([09:00] Marcos). Toda mudança de status passa por `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), que, dentro de `prisma.$transaction`, valida a transição com `canTransition` (`src/modules/orders/order.status.ts`), debita ou repõe estoque, atualiza `orders` e insere em `order_status_history`. Esse é o único ponto do código onde o evento "status mudou" nasce, e por isso é o ponto de integração ([09:40] Bruno).

Restrições técnicas que moldam o desenho:
- A transação de status não pode depender de rede nem da disponibilidade do cliente ([09:04] Bruno).
- O time é pequeno e não quer infraestrutura nova ([09:07] Diego).
- O MySQL não notifica processos externos ([09:09] Diego).

## 2. Objetivos técnicos

| ID | Objetivo | Origem |
| --- | --- | --- |
| OT-01 | Entregar a notificação em menos de 10 s após o commit da mudança de status, em condições normais (cliente respondendo). Com polling de 2 s, a latência esperada é de até ~2 s mais o tempo de resposta do cliente. | [09:02] Marcos, [09:10] Larissa |
| OT-02 | Zero mudanças de status sem evento correspondente para os webhooks inscritos (atomicidade outbox + status). | [09:40] Bruno, [09:41] Diego |
| OT-03 | Nenhum evento perdido silenciosamente: todo evento termina entregue ou na DLQ. | [09:15] Diego, [09:18] Diego |
| OT-04 | Nenhuma dependência de runtime nova: Prisma, Zod, Pino, Express e o `fetch` nativo do Node 20 (`engines.node >= 20` em `package.json`). | [09:29] Bruno, [09:30] Larissa |
| OT-05 | Módulo estruturalmente idêntico aos existentes, com erros `WEBHOOK_*`. | [09:27] Bruno, [09:29] Larissa |

## 3. Escopo e exclusões

**Incluso**
- Tabelas `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries`, `webhook_dead_letter`.
- `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada em `changeStatus`.
- Worker `src/worker.ts` (novo, a criar) + processador `src/modules/webhooks/webhook.processor.ts` (novo, a criar).
- Endpoints: criar, listar, editar, remover webhook; rotacionar secret; histórico de entregas; replay de DLQ.
- Assinatura HMAC-SHA256, headers de entrega, timeout, retry, DLQ.

**Excluído** (com origem)

| Item | Motivo | Origem |
| --- | --- | --- |
| Webhooks de entrada (cliente enviando para nós) | Só saída | [09:02] Marcos, [09:03] Sofia |
| Arquivamento de linhas entregues da outbox | Fora do escopo desta feature | [09:08] Diego |
| Múltiplos workers / particionamento por `order_id` | Futuro | [09:13] Diego |
| E-mail de alerta ao cliente | Próxima fase | [09:37] Larissa |
| Rate limiting de saída | Observar e decidir depois | [09:39] Larissa |
| Dashboard visual | Projeto separado do time de frontend | [09:40] Larissa |
| Itens do pedido no payload | Mantém payload enxuto | [09:43] Diego |
| Evento na **criação** do pedido (`OrderService.create`, que grava histórico `null → PENDING`) | A reunião só tratou mudança de status via `changeStatus`; incluir exige decisão (ver RFC Q-11) | [09:40] Bruno, `src/modules/orders/order.service.ts` |

## 4. Modelo de dados

Acréscimos em `prisma/schema.prisma`, numa única migração. Todos os ids são UUID `@db.Char(36)`, seguindo o padrão das tabelas atuais ([09:51] Larissa). A única alteração em modelo existente é o campo de relação virtual `webhooks WebhookEndpoint[]` em `Customer`, que não muda a tabela `customers`.

```prisma
enum WebhookOutboxStatus {
  PENDING      // pendente
  PROCESSING   // processando
  FAILED       // falhou (movido para a DLQ ou cancelado)
  DELIVERED    // entregue
}

model WebhookEndpoint {
  id                      String    @id @default(uuid()) @db.Char(36)
  customerId              String    @db.Char(36)
  url                     String    @db.VarChar(2048)
  secret                  String    @db.VarChar(128)
  previousSecret          String?   @db.VarChar(128)
  previousSecretExpiresAt DateTime?
  events                  Json      // lista de OrderStatus assinados
  active                  Boolean   @default(true)
  deletedAt               DateTime?
  createdById             String    @db.Char(36)
  createdAt               DateTime  @default(now())
  updatedAt               DateTime  @updatedAt

  customer Customer @relation(fields: [customerId], references: [id])

  @@index([customerId, active])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id            String              @id @default(uuid()) @db.Char(36) // = event_id / X-Event-Id
  webhookId     String              @db.Char(36)
  orderId       String              @db.Char(36)
  eventType     String              @db.VarChar(64)  // "order.status_changed"
  payload       Json                                  // snapshot renderizado na inserção
  status        WebhookOutboxStatus @default(PENDING)
  attempts      Int                 @default(0)
  nextAttemptAt DateTime            @default(now())
  lastError     String?             @db.VarChar(500)
  deliveredAt   DateTime?
  createdAt     DateTime            @default(now())
  updatedAt     DateTime            @updatedAt

  @@index([status])
  @@index([createdAt])
  @@index([status, nextAttemptAt])
  @@map("webhook_outbox")
}

model WebhookDelivery {
  id           String   @id @default(uuid()) @db.Char(36)
  webhookId    String   @db.Char(36)
  eventId      String   @db.Char(36)
  attempt      Int
  success      Boolean
  statusCode   Int?
  responseBody String?  @db.Text
  durationMs   Int
  errorCode    String?  @db.VarChar(64)
  createdAt    DateTime @default(now())

  @@index([webhookId, createdAt])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {
  id           String    @id @default(uuid()) @db.Char(36)
  eventId      String    @db.Char(36)
  webhookId    String    @db.Char(36)
  payload      Json
  reason       String    @db.VarChar(500)
  failedAt     DateTime  @default(now())
  replayedAt   DateTime?
  replayedById String?   @db.Char(36)

  @@index([webhookId])
  @@map("webhook_dead_letter")
}
```

Notas de rastreabilidade do modelo:
- Endpoint guarda url, secret, customer_id e estado ativo ([09:21] Bruno, [09:21] Sofia); `previousSecret` + `previousSecretExpiresAt` implementam a convivência de 24 h ([09:21] Sofia); `events` é a lista de status assinados ([09:33] Marcos).
- Outbox: status pendente/processando/falhou/entregue com índices em `status` e `created_at` ([09:08] Diego). `nextAttemptAt` e o índice composto são **(proposta)** para agendar o backoff ([09:17] Larissa).
- Uma linha de outbox **por endpoint de destino** **(proposta)**: o filtro é por webhook ([09:33] Bruno) e cada envio leva o seu `X-Webhook-Id` ([09:44] Sofia), então retry e DLQ precisam ser independentes por destino. O `id` da linha é o `event_id`.
- Entregas: sucesso/falha, payload, resposta e tempo de resposta ([09:34] Marcos). O payload vem do join com a outbox pelo `eventId`.
- Dead letter: payload, motivo e timestamp ([09:18] Diego); `replayedById` atende à auditoria do replay ([09:36] Sofia).
- `deletedAt` (remoção lógica) é **(proposta)**: preserva histórico de entregas e DLQ do endpoint removido ([09:33] Bruno pede DELETE; a forma não foi discutida).
- O tamanho de armazenamento de `responseBody` não foi discutido; recomenda-se truncar na gravação para não inflar a tabela **(proposta)**.

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox (API, dentro da transação)

```
PATCH /api/v1/orders/:id/status
 └ OrderController.changeStatus
    └ OrderService.changeStatus  ── prisma.$transaction(tx) ─────────────────────────┐
       1. findUnique(order)  → NotFoundError                                          │
       2. from === to        → ConflictError(INVALID_STATUS_TRANSITION)               │
       3. canTransition      → InvalidStatusTransitionError                           │
       4. debitStock / replenishStock (conforme order.status.ts)                      │
       5. tx.order.update(status = to)                                                │
       6. tx.orderStatusHistory.create(...)                                           │
       7. publishWebhookEvent(tx, order, from, to)            ◀── NOVO                │
          a. tx.webhookEndpoint.findMany({ customerId, active: true, deletedAt: null })│
          b. filtra endpoints cujo events contém `to`                                 │
          c. nenhum? → retorna sem inserir                                            │
          d. para cada endpoint: id = uuid; payload = render(order, from, to, id)     │
             tx.webhookOutbox.create({ id, webhookId, orderId, payload, PENDING })    │
       8. refresh e retorno                                                           │
    commit ◀──────────────────────────────────────────────────────────────────────────┘
    (qualquer exceção em 1..8 → rollback de tudo, inclusive da outbox)
```

Regras:
- O passo 7 roda **depois** das escritas de status e histórico e **dentro** da mesma transação; falha na outbox faz rollback do status ([09:40] Bruno).
- O filtro é aplicado aqui, e não no envio ([09:34] Bruno).
- O payload é renderizado aqui (snapshot) ([09:52] Larissa). `order` já tem `orderNumber`, `customerId` e `totalCents` carregados pelo `findUnique` do passo 1, então nenhuma consulta extra ao pedido é necessária.
- O `timestamp` do corpo é o momento da mudança de status (renderização) **(proposta)**; o momento do envio vai no header `X-Timestamp` ([09:44] Diego).

### 5.2 Processamento pelo worker

```
src/worker.ts (bootstrap)
 ├ prisma = createPrismaClient()      // instância própria, mesma DATABASE_URL
 ├ recupera órfãos: PROCESSING há mais de 2 × timeout → PENDING   (proposta)
 └ loop a cada WEBHOOK_POLL_INTERVAL_MS (2000):
    1. SELECT pendentes: status = PENDING AND nextAttemptAt <= now()
       ORDER BY createdAt ASC LIMIT WEBHOOK_BATCH_SIZE
    2. marca o lote como PROCESSING
    3. para cada evento, em ordem de createdAt:
       a. carrega endpoint; removido ou inativo → status FAILED, lastError
          "webhook inactive", sem envio e sem DLQ (proposta); log
       b. body = JSON.stringify(payload); Buffer.byteLength(body) > 65536
          → DLQ com WEBHOOK_PAYLOAD_TOO_LARGE, sem retry (seção 5.4)
       c. assina body (seção 6.8) e monta headers
       d. fetch(url, { method: POST, body, headers, signal: AbortSignal.timeout(10000) })
       e. grava webhook_deliveries (attempt, success, statusCode, responseBody, durationMs)
       f. 2xx → status DELIVERED, deliveredAt = now()
          caso contrário → fluxo de retry (5.3)
    4. dorme até o próximo ciclo
 └ SIGINT/SIGTERM: termina o evento em andamento, não pega novo lote, $disconnect()
```

- Polling de 2 s, lote pequeno, mais antigos primeiro ([09:09] Diego, [09:08] Diego). O tamanho do lote não foi definido na reunião; `WEBHOOK_BATCH_SIZE` tem valor inicial 20 **(proposta)**, a calibrar.
- Instância única; processamento sequencial dentro do lote preserva a ordem por `created_at` ([09:12] Diego).
- Recuperação de órfãos **(proposta)**: um evento pode ficar em `PROCESSING` se o worker cair no meio do lote. Devolvê-lo a `PENDING` pode gerar reenvio, o que é coerente com at-least-once ([09:24] Diego).
- Encerramento gracioso nos moldes de `src/server.ts`.

### 5.3 Retry com backoff

| Falha nº | Próxima tentativa após | Acumulado desde a 1ª falha |
| --- | --- | --- |
| 1 (envio inicial) | 1 min | 1 min |
| 2 | 5 min | 6 min |
| 3 | 30 min | 36 min |
| 4 | 2 h | 2 h 36 min |
| 5 | 12 h | 14 h 36 min |
| 6 | (sem nova tentativa) → DLQ | |

- Progressão conforme [09:17] Larissa. A tabela implementa 1 envio inicial + 5 retentativas, leitura pendente de confirmação (RFC Q-04).
- Implementação: `BACKOFF_MS = [60_000, 300_000, 1_800_000, 7_200_000, 43_200_000]`; em cada falha, `attempts += 1`; se `attempts <= BACKOFF_MS.length`, `nextAttemptAt = now + BACKOFF_MS[attempts - 1]` e `status = PENDING`; senão, DLQ.
- Conta como falha: timeout de 10 s ([09:42] Diego), erro de rede e qualquer status fora de 2xx **(proposta, RFC Q-09)**.
- Sem jitter **(proposta)**: a reunião não tratou do tema, e com um único worker não há tempestade de retentativas a evitar.

### 5.4 DLQ e replay

```
Falha definitiva (tentativas esgotadas ou payload > 64 KB):
  $transaction:
    webhookDeadLetter.create({ eventId, webhookId, payload, reason })
    webhookOutbox.update({ status: FAILED, lastError: reason })

POST /api/v1/admin/webhooks/dead-letter/:id/replay   (authenticate + requireRole('ADMIN'))
  $transaction:
    dl = findUnique(id)               → WEBHOOK_DEAD_LETTER_NOT_FOUND
    dl.replayedAt != null             → WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED   (proposta)
    webhookOutbox.update(dl.eventId, { status: PENDING, attempts: 0, nextAttemptAt: now(), lastError: null })
    webhookDeadLetter.update(id, { replayedAt: now(), replayedById: req.user.id })
  logger.info({ deadLetterId, eventId, webhookId, userId }, 'webhook_dead_letter_replayed')
```

- Tabela separada e replay manual que recoloca o evento como pendente ([09:18] Diego). Por reaproveitar a linha da outbox, o `event_id` é o mesmo, o que permite ao cliente deduplicar ([09:25] Diego).
- `ADMIN` obrigatório e log de quem fez ([09:36] Sofia, [09:36] Larissa).
- Zerar `attempts` no replay dá ao evento um novo ciclo completo de tentativas **(proposta)**.

## 6. Contratos públicos

Todas as rotas ficam sob `/api/v1` (`src/app.ts`), exigem `Authorization: Bearer <jwt>` (`authenticate` em `src/middlewares/auth.middleware.ts`) e respondem erros no envelope do error middleware: `{ "error": { "code", "message", "details?" } }`. Erros de schema seguem o padrão atual: 400 `VALIDATION_ERROR` com `details[]` (`src/middlewares/validate.middleware.ts`).

O `customerId` vem no corpo (criação) ou na query (listagem), nunca do JWT, porque o JWT é do usuário operador ([09:32] Bruno, [09:32] Larissa). A reunião aceitou "body ou path"; este FDD usa body/query para manter a base `/webhooks` citada em `GET /webhooks/:id/deliveries` ([09:34] Marcos) **(proposta)**.

Valores aceitos em `events` **(proposta derivada do código)**: `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`. `PENDING` é excluído porque nenhuma transição de `order.status.ts` tem `PENDING` como destino, e um filtro com ele nunca dispararia.

### 6.1 `POST /api/v1/webhooks`: cadastrar webhook

Origem: [09:31] Marcos, [09:33] Marcos, [09:23] Sofia.

Request:
```http
POST /api/v1/webhooks
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{
  "customerId": "4f6c2a7e-1b9d-4c3e-9a51-0d8e7f2b6c11",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created` (a secret só é exibida aqui e na rotação **(proposta)**):
```json
{
  "id": "b7a1d3e2-5c4f-4e8a-9b06-2f1e3d4c5a69",
  "customerId": "4f6c2a7e-1b9d-4c3e-9a51-0d8e7f2b6c11",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_EXEMPLO_FICTICIO_NAO_E_UMA_SECRET_REAL",
  "createdAt": "2026-11-19T12:00:00.000Z"
}
```

| Status | Quando |
| --- | --- |
| 201 | Criado |
| 400 `VALIDATION_ERROR` | `events` vazio ou com valor inválido, campos ausentes, `url` malformada |
| 400 `WEBHOOK_INVALID_URL` | URL não é `https` |
| 401 `UNAUTHORIZED` | Sem token ou token inválido |
| 404 `WEBHOOK_CUSTOMER_NOT_FOUND` | `customerId` não existe |

O prefixo `whsec_` no exemplo é ilustrativo; formato e tamanho da secret ficam para a revisão de segurança (seção 6.8).

### 6.2 `GET /api/v1/webhooks?customerId=...&page=1&pageSize=20`: listar webhooks de um cliente

Origem: [09:33] Bruno.

Request:
```http
GET /api/v1/webhooks?customerId=4f6c2a7e-1b9d-4c3e-9a51-0d8e7f2b6c11&page=1&pageSize=20
Authorization: Bearer eyJhbGciOi...
```

Response `200 OK` (formato de `paginated` em `src/shared/http/response.ts`; sem secret):
```json
{
  "data": [
    {
      "id": "b7a1d3e2-5c4f-4e8a-9b06-2f1e3d4c5a69",
      "customerId": "4f6c2a7e-1b9d-4c3e-9a51-0d8e7f2b6c11",
      "url": "https://hooks.atlascomercial.com.br/oms",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-11-19T12:00:00.000Z",
      "updatedAt": "2026-11-19T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

| Status | Quando |
| --- | --- |
| 200 | Lista (pode ser vazia) |
| 400 `VALIDATION_ERROR` | `customerId` ausente ou não UUID |
| 401 `UNAUTHORIZED` | Sem token |

### 6.3 `PATCH /api/v1/webhooks/:id`: editar webhook

Origem: [09:33] Bruno.

Request (todos os campos opcionais, pelo menos um obrigatório; `secret` e `customerId` não são editáveis):
```http
PATCH /api/v1/webhooks/b7a1d3e2-5c4f-4e8a-9b06-2f1e3d4c5a69
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{ "events": ["PAID", "SHIPPED", "DELIVERED"], "active": false }
```

Response `200 OK` (recurso atualizado, sem secret):
```json
{
  "id": "b7a1d3e2-5c4f-4e8a-9b06-2f1e3d4c5a69",
  "customerId": "4f6c2a7e-1b9d-4c3e-9a51-0d8e7f2b6c11",
  "url": "https://hooks.atlascomercial.com.br/oms",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": false,
  "createdAt": "2026-11-19T12:00:00.000Z",
  "updatedAt": "2026-11-19T15:30:00.000Z"
}
```

| Status | Quando |
| --- | --- |
| 200 | Atualizado |
| 400 `VALIDATION_ERROR` | Corpo vazio, campo desconhecido, `events` inválido |
| 400 `WEBHOOK_INVALID_URL` | Nova `url` não é `https` |
| 404 `WEBHOOK_NOT_FOUND` | Id inexistente ou removido |

Semântica: mudanças valem para eventos **inseridos depois** da edição, pois o filtro roda na inserção ([09:34] Bruno). Desativar (`active: false`) faz o worker cancelar os pendentes daquele endpoint (seção 5.2, passo 3a).

### 6.4 `DELETE /api/v1/webhooks/:id`: remover webhook

Origem: [09:33] Bruno.

Request (sem corpo):
```http
DELETE /api/v1/webhooks/b7a1d3e2-5c4f-4e8a-9b06-2f1e3d4c5a69
Authorization: Bearer eyJhbGciOi...
```

Response `204 No Content` (sem corpo).

| Status | Quando |
| --- | --- |
| 204 | Removido (remoção lógica, `deletedAt`) |
| 404 `WEBHOOK_NOT_FOUND` | Id inexistente ou já removido |

### 6.5 `POST /api/v1/webhooks/:id/rotate-secret`: rotacionar secret

Origem: [09:21] Sofia. O caminho é **(proposta)**; a reunião definiu apenas "endpoint pro cliente conseguir pedir nova secret pela API".

Request (sem corpo):
```http
POST /api/v1/webhooks/b7a1d3e2-5c4f-4e8a-9b06-2f1e3d4c5a69/rotate-secret
Authorization: Bearer eyJhbGciOi...
```

Response `200 OK`:
```json
{
  "id": "b7a1d3e2-5c4f-4e8a-9b06-2f1e3d4c5a69",
  "secret": "whsec_EXEMPLO_FICTICIO_NAO_E_UMA_SECRET_REAL",
  "previousSecretExpiresAt": "2026-11-20T12:00:00.000Z"
}
```

Semântica: a secret atual vira `previousSecret`, com validade de 24 h; uma nova é gerada ([09:21] Sofia, [09:22] Sofia). Uma segunda rotação dentro do grace period descarta a anterior imediatamente **(proposta)**.

| Status | Quando |
| --- | --- |
| 200 | Rotacionada |
| 404 `WEBHOOK_NOT_FOUND` | Id inexistente ou removido |

### 6.6 `GET /api/v1/webhooks/:id/deliveries`: histórico de entregas

Origem: [09:34] Marcos. Retorna as 100 tentativas de envio mais recentes (cada retentativa é um envio e conta como um item), da mais nova para a mais antiga.

Request:
```http
GET /api/v1/webhooks/b7a1d3e2-5c4f-4e8a-9b06-2f1e3d4c5a69/deliveries
Authorization: Bearer eyJhbGciOi...
```

Response `200 OK`:
```json
{
  "data": [
    {
      "id": "e2c4a6b8-0d1f-4e3a-8c5b-7d9f1a3c5e70",
      "eventId": "9c1e3a5b-7d9f-4b2d-8e0a-1c3e5a7b9d21",
      "attempt": 2,
      "success": true,
      "statusCode": 200,
      "durationMs": 184,
      "payload": { "event_id": "9c1e3a5b-7d9f-4b2d-8e0a-1c3e5a7b9d21", "event_type": "order.status_changed", "...": "..." },
      "responseBody": "{\"received\":true}",
      "errorCode": null,
      "createdAt": "2026-11-19T12:01:03.412Z"
    },
    {
      "id": "a1b3c5d7-e9f0-4a2b-9c4d-6e8f0a2b4c61",
      "eventId": "9c1e3a5b-7d9f-4b2d-8e0a-1c3e5a7b9d21",
      "attempt": 1,
      "success": false,
      "statusCode": null,
      "durationMs": 10000,
      "payload": { "event_id": "9c1e3a5b-7d9f-4b2d-8e0a-1c3e5a7b9d21", "...": "..." },
      "responseBody": null,
      "errorCode": "WEBHOOK_DELIVERY_TIMEOUT",
      "createdAt": "2026-11-19T12:00:02.950Z"
    }
  ]
}
```

| Status | Quando |
| --- | --- |
| 200 | Histórico (pode ser vazio) |
| 404 `WEBHOOK_NOT_FOUND` | Id inexistente |

### 6.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay`: reprocessar evento da DLQ

Origem: [09:18] Diego, [09:35] Diego, [09:36] Sofia.

Request (sem corpo; JWT de usuário com role `ADMIN`):
```http
POST /api/v1/admin/webhooks/dead-letter/5d7f9a1c-3e5b-4d7f-9a1c-3e5b7d9f1a32/replay
Authorization: Bearer eyJhbGciOi...
```


Response `202 Accepted` (o envio é assíncrono, feito pelo worker no próximo ciclo):
```json
{
  "deadLetterId": "5d7f9a1c-3e5b-4d7f-9a1c-3e5b7d9f1a32",
  "eventId": "9c1e3a5b-7d9f-4b2d-8e0a-1c3e5a7b9d21",
  "status": "PENDING",
  "replayedAt": "2026-11-20T09:15:00.000Z",
  "replayedById": "0a2c4e6f-8b1d-4f3a-9c5e-7b9d1f3a5c74"
}
```

| Status | Quando |
| --- | --- |
| 202 | Evento recolocado na outbox |
| 401 `UNAUTHORIZED` | Sem token |
| 403 `FORBIDDEN` | Usuário não é `ADMIN` (código existente, emitido por `requireRole`) |
| 404 `WEBHOOK_DEAD_LETTER_NOT_FOUND` | Id inexistente |
| 409 `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | Já reprocessado **(proposta)** |

### 6.8 Contrato de saída: requisição enviada ao cliente

Origem: [09:43] Diego, [09:44] Diego, [09:44] Sofia, [09:20] Sofia.

```http
POST https://hooks.atlascomercial.com.br/oms
Content-Type: application/json
X-Event-Id: 9c1e3a5b-7d9f-4b2d-8e0a-1c3e5a7b9d21
X-Webhook-Id: b7a1d3e2-5c4f-4e8a-9b06-2f1e3d4c5a69
X-Timestamp: 2026-11-19T12:00:02.950Z
X-Signature: 3b9f0c1e5a7d2f4b6c8e0a1d3f5b7c9e1a2d4f6b8c0e3a5d7f9b1c2e4a6d8f0b

{
  "event_id": "9c1e3a5b-7d9f-4b2d-8e0a-1c3e5a7b9d21",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-19T12:00:00.731Z",
  "order_id": "1e3c5a7b-9d0f-4e2a-8c4b-6d8f0a2c4e95",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "4f6c2a7e-1b9d-4c3e-9a51-0d8e7f2b6c11",
  "total_cents": 125990
}
```

| Elemento | Regra | Origem |
| --- | --- | --- |
| `X-Signature` | `HMAC-SHA256(secret, corpo bruto)` em hexadecimal minúsculo. Codificação hex **(proposta)** | [09:20] Sofia, [09:22] Sofia |
| Durante o grace period | **(proposta, RFC Q-03)**: duas assinaturas separadas por vírgula, nova primeiro, para que clientes migrados e não migrados validem | [09:21] Sofia |
| `X-Event-Id` | UUID da linha da outbox; igual a `event_id` do corpo; estável entre retentativas e replay | [09:25] Diego |
| `X-Timestamp` | Momento do envio, ISO 8601; não entra na assinatura | [09:44] Diego |
| `X-Webhook-Id` | Id do endpoint cadastrado | [09:44] Sofia |
| Corpo | Snapshot da inserção, sem itens; limite 64 KB | [09:43] Diego, [09:52] Larissa, [09:24] Larissa |
| Resposta esperada | Qualquer 2xx em até 10 s = entregue **(proposta para o critério 2xx)** | [09:42] Diego |

Geração de secret **(proposta, sujeita à revisão da Sofia)**: `crypto.randomBytes(32)` do módulo nativo `node:crypto`, sem dependência nova. Comparação de assinaturas no lado do cliente deve ser em tempo constante; o portal de desenvolvedor deve trazer exemplo ([09:26] Marcos).

## 7. Matriz de erros

A reunião nomeou `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED` "etc." e fixou o prefixo ([09:28] Bruno, [09:29] Larissa). Os demais códigos da tabela são **(proposta)**, cada um derivado da regra indicada na coluna Origem.

Classes novas em `src/modules/webhooks/webhook.errors.ts` (novo, a criar), estendendo as de `src/shared/errors/http-errors.ts` (que aceitam `code` customizado em `BadRequestError`, `ConflictError` e `UnprocessableEntityError`) ou `AppError` diretamente quando a base fixa o código (caso de `NotFoundError`, que sempre usa `NOT_FOUND`).

| Código | HTTP | Onde ocorre | Classe proposta | Retentável? | Origem |
| --- | --- | --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | PATCH, DELETE, rotate, deliveries | `WebhookNotFoundError extends AppError` | n/a | [09:28] Bruno |
| `WEBHOOK_INVALID_URL` | 400 | POST, PATCH: URL sem `https` | `WebhookInvalidUrlError extends BadRequestError` | n/a | [09:28] Bruno, [09:23] Sofia |
| `WEBHOOK_SECRET_REQUIRED` | (interno) | Worker: endpoint sem secret ao assinar; vai direto para DLQ, nunca envia sem assinatura | `WebhookSecretRequiredError extends AppError` | Não | [09:28] Bruno, [09:21] Sofia |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | POST: `customerId` inexistente | `WebhookCustomerNotFoundError extends AppError` | n/a | [09:32] Larissa |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Replay: id inexistente | `WebhookDeadLetterNotFoundError extends AppError` | n/a | [09:18] Diego |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | Replay repetido **(proposta)** | `extends ConflictError` | n/a | [09:18] Diego |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | (interno) | Worker: corpo > 64 KB; DLQ sem retry | `extends UnprocessableEntityError` | Não | [09:24] Larissa |
| `WEBHOOK_DELIVERY_TIMEOUT` | (interno) | Worker: sem resposta em 10 s | registrado em `webhook_deliveries.errorCode` | Sim | [09:42] Diego |
| `WEBHOOK_DELIVERY_FAILED` | (interno) | Worker: erro de rede ou status fora de 2xx | registrado em `webhook_deliveries.errorCode` | Sim | [09:14] Larissa, [09:15] Diego |
| `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | (interno) | Worker: tentativas esgotadas; motivo gravado na DLQ | `reason` em `webhook_dead_letter` | Não | [09:17] Larissa |

Erros existentes reutilizados sem mudança: `VALIDATION_ERROR` (schemas Zod), `UNAUTHORIZED` e `FORBIDDEN` (`src/middlewares/auth.middleware.ts`). Erros "internos" não chegam a um cliente HTTP: aparecem em log, em `webhook_deliveries.errorCode` e em `webhook_dead_letter.reason`, e são expostos ao cliente pelo endpoint de histórico (6.6).

## 8. Estratégias de resiliência

| Aspecto | Estratégia | Origem |
| --- | --- | --- |
| Isolamento | API e worker em processos distintos; falha do cliente nunca afeta a transação de status | [09:04] Bruno, [09:11] Diego |
| Timeout | 10 s por chamada, via `AbortSignal.timeout` | [09:42] Diego |
| Retries | Backoff 1m/5m/30m/2h/12h; contagem conforme RFC Q-04 | [09:17] Larissa |
| Fallback | DLQ em tabela própria + replay manual por `ADMIN`. Não há fallback por e-mail nesta fase | [09:18] Diego, [09:37] Larissa |
| Idempotência | `X-Event-Id` estável; reenvios são esperados (at-least-once) | [09:25] Diego |
| Payload | Limite de 64 KB, falha sem truncar; aplicado no worker **(proposta, RFC Q-05)** para não bloquear mudanças de status | [09:24] Larissa |
| Crash do worker | Órfãos em `PROCESSING` voltam a `PENDING` no boot **(proposta)** | [09:24] Diego |
| Ordenação | Sequencial, single-worker, por `created_at`. **Limitação:** um evento em backoff não bloqueia o seguinte do mesmo pedido, que pode chegar antes (RFC Q-10). O corpo traz `from_status`/`to_status` e `timestamp` para o cliente reconciliar | [09:12] Diego, [09:13] Larissa |

## 9. Observabilidade

Sem ferramentas novas ([09:29] Bruno, [09:30] Larissa): tudo via `logger` Pino (`src/shared/logger/index.ts`) e consultas às tabelas do módulo.

**Logs** (JSON estruturado, nomes em snake_case como `server_started` e `http_request` já usados no projeto):

| Evento de log | Processo | Campos |
| --- | --- | --- |
| `webhook_event_enqueued` | API | `eventId`, `webhookId`, `orderId`, `fromStatus`, `toStatus` |
| `webhook_delivery_succeeded` | Worker | `eventId`, `webhookId`, `orderId`, `attempt`, `statusCode`, `durationMs` |
| `webhook_delivery_failed` | Worker | idem + `errorCode`, `nextAttemptAt` |
| `webhook_moved_to_dead_letter` | Worker | `eventId`, `webhookId`, `reason`, `attempts` |
| `webhook_dead_letter_replayed` | API | `deadLetterId`, `eventId`, `webhookId`, `userId` (auditoria, [09:36] Sofia) |
| `webhook_worker_started` / `webhook_worker_stopped` | Worker | `pollIntervalMs`, `batchSize` / `signal` |
| `webhook_poll_cycle` (nível `debug`) | Worker | `fetched`, `delivered`, `failed`, `cycleMs` |

O worker usa `logger.child({ component: 'webhook-worker' })`. Os caminhos `*.secret` e `*.previousSecret` entram em `redactPaths` para que secrets nunca apareçam em log ([09:22] Diego relata vazamento de secret em log de cliente).

**Métricas** (derivadas, sem stack de métricas no projeto hoje):

| Métrica | Cálculo | Uso |
| --- | --- | --- |
| `webhook_delivery_latency_ms` | `deliveredAt - createdAt` em `webhook_outbox` | Acompanhar a meta de < 10 s ([09:02] Marcos) |
| `webhook_outbox_pending` e idade do pendente mais antigo | `COUNT`/`MIN(createdAt)` com `status = PENDING` | Detectar worker parado |
| `webhook_delivery_success_rate` | `success` em `webhook_deliveries` por janela | Saúde por cliente |
| `webhook_delivery_duration_ms` (p95) | `durationMs` em `webhook_deliveries` | Clientes lentos |
| `webhook_dead_letter_total` | `COUNT` em `webhook_dead_letter` sem `replayedAt` | Fila de intervenção manual |
| Volume por cliente/minuto | `COUNT` em `webhook_outbox` agrupado | Insumo para a decisão de rate limiting ([09:39] Diego) |

**Tracing**: o projeto não tem tracing distribuído; a correlação é feita por identificadores propagados em todos os logs. `eventId` liga o log da API (`webhook_event_enqueued`) a cada tentativa do worker, à linha de `webhook_deliveries` e ao header `X-Event-Id` que o cliente recebe, permitindo seguir um evento de ponta a ponta inclusive no lado do cliente. Na API, a ligação entre a requisição `PATCH /orders/:id/status` (logada com `requestId` e `path` por `src/middlewares/request-logger.middleware.ts`) e os eventos que ela gerou é feita pelo `orderId` presente nos dois logs. Propagar o `requestId` até a outbox exigiria mudar a assinatura de `changeStatus(id, input, userId)` e de `publishWebhookEvent`, o que não foi discutido; fica como melhoria possível. Adotar OpenTelemetry fica fora desta fase pela decisão de reuso ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)).

## 10. Integração com o sistema existente

| Arquivo existente | Como o módulo de webhooks se integra |
| --- | --- |
| `src/modules/orders/order.service.ts` | `changeStatus` passa a chamar `await publishWebhookEvent(tx, order, from, to)` logo após `tx.orderStatusHistory.create(...)` e antes do `findUnique` de refresh, usando o mesmo `tx` do `prisma.$transaction` ([09:41] Bruno). Nenhuma injeção nova no construtor de `OrderService` ([09:41] Diego). `create()` não muda (seção 3). O tipo `TxClient = Prisma.TransactionClient` já declarado no arquivo é o tipo do primeiro parâmetro da função. |
| `src/modules/orders/order.status.ts` | Fonte dos valores válidos para `events`: o schema do módulo aceita só os status que aparecem como destino em `transitions` (todos menos `PENDING`). Nenhuma alteração no arquivo. |
| `prisma/schema.prisma` | Recebe o enum `WebhookOutboxStatus` e os quatro modelos da seção 4, mais a relação virtual `webhooks` em `Customer`. Migração nova em `prisma/migrations/`. |
| `src/shared/errors/http-errors.ts` e `src/shared/errors/app-error.ts` | Base das classes `Webhook*Error` (seção 7). `BadRequestError`, `ConflictError` e `UnprocessableEntityError` já aceitam `code`; para 404 com código próprio, estende-se `AppError`, porque `NotFoundError` fixa `NOT_FOUND`. |
| `src/middlewares/error.middleware.ts` | Sem alteração: o ramo `err instanceof AppError` já serializa `statusCode`, `errorCode` e `details` das novas classes ([09:29] Bruno). |
| `src/middlewares/auth.middleware.ts` | `authenticate` em todas as rotas do módulo (como em `src/modules/orders/order.routes.ts`); `requireRole('ADMIN')` no replay, como já é feito em `src/modules/users/user.routes.ts` ([09:36] Larissa). |
| `src/middlewares/validate.middleware.ts` | Valida body/query/params com os schemas de `src/modules/webhooks/webhook.schemas.ts` (novo, a criar). A regra `https` vive num schema Zod exportado (`httpsUrlSchema`, [09:23] Sofia), aplicado pelo service para lançar `WEBHOOK_INVALID_URL` **(proposta)**, porque este middleware converte toda falha de schema em `VALIDATION_ERROR`. |
| `src/routes/index.ts` | Registra `router.use('/webhooks', buildWebhookRouter(...))` e `router.use('/admin/webhooks', buildWebhookAdminRouter(...))`; o tipo `Controllers` ganha `webhooks` e `webhooksAdmin`. |
| `src/app.ts` | `buildControllers` instancia `WebhookRepository`, `WebhookService` e os controllers, no mesmo estilo dos demais módulos. |
| `src/server.ts` | Modelo para `src/worker.ts` (novo, a criar): bootstrap assíncrono, `logger.fatal` em falha, `SIGINT`/`SIGTERM` com `prisma.$disconnect()` ([09:11] Larissa). |
| `src/config/database.ts` | O worker chama `createPrismaClient()` (já exportada) para ter sua própria instância com a mesma `DATABASE_URL` ([09:30] Bruno). |
| `src/config/env.ts` | Novas variáveis no `envSchema`: `WEBHOOK_POLL_INTERVAL_MS` (padrão 2000), `WEBHOOK_HTTP_TIMEOUT_MS` (padrão 10000), `WEBHOOK_BATCH_SIZE` (padrão 20, proposta). `.env.example` recebe os nomes. |
| `src/shared/logger/index.ts` | Inclusão de `*.secret` e `*.previousSecret` em `redactPaths`. |
| `src/shared/http/response.ts` | `paginated()` na listagem de webhooks. |
| `package.json` | Scripts `worker` (`node --env-file=.env dist/worker.js`) e `worker:dev` (`tsx watch --env-file=.env src/worker.ts`), espelhando `start` e `dev` ([09:11] Larissa). Nenhuma dependência nova. |
| `tests/setup.ts` | O `beforeEach` passa a limpar `webhookDelivery`, `webhookDeadLetter`, `webhookOutbox` e `webhookEndpoint` **antes** de `customer`, por causa da FK de `webhook_endpoints.customerId`. |
| `tests/orders.test.ts` e `tests/helpers/factories.ts` | Novos casos de integração da outbox ao lado dos testes de transição existentes; factory `createTestWebhook`. |

Estrutura do novo módulo ([09:27] Bruno, [09:28] Bruno):

```
src/worker.ts                                   entry-point do worker (novo)
src/modules/webhooks/
  webhook.controller.ts   webhook.service.ts    webhook.repository.ts
  webhook.routes.ts       webhook.schemas.ts    webhook.errors.ts
  webhook.publisher.ts    → publishWebhookEvent(tx, order, from, to)
  webhook.processor.ts    → ciclo do worker (seleção, envio, retry, DLQ)
  webhook.signature.ts    → HMAC-SHA256 e geração de secret
```

## 11. Dependências e compatibilidade

- **Runtime**: Node 20+ (`fetch` e `AbortSignal.timeout` nativos), Prisma 5.22.0, Zod 3.23.8, Pino 9.5.0, `uuid` 11.0.3, todos já em `package.json`. Nenhuma dependência nova.
- **Banco**: MySQL existente, mesma `DATABASE_URL` para API e worker. Migração apenas aditiva.
- **Compatibilidade de API**: nenhuma rota existente muda de contrato. `PATCH /orders/:id/status` continua com as mesmas respostas; só fica um pouco mais lento por causa da escrita na outbox.
- **Deploy**: novo processo `npm run worker` com uma única réplica ([09:12] Diego). A migração precisa rodar antes da API nova subir; o worker pode subir depois sem perda, porque os eventos ficam na outbox.
- **Contrato com clientes**: versão inicial; mudanças de payload devem ser aditivas. Documentação no portal de desenvolvedor por Marcos ([09:26] Marcos, [09:40] Marcos).
- **Pessoas**: revisão de segurança da Sofia, com ao menos dois dias úteis antes do deploy ([09:46] Sofia).

## 12. Critérios de aceite técnicos

| ID | Critério verificável | Origem |
| --- | --- | --- |
| CAT-01 | Mudança de status de pedido cujo cliente tem webhook ativo inscrito em `to` cria exatamente 1 linha `PENDING` por endpoint na mesma transação. | [09:40] Bruno |
| CAT-02 | Falha forçada no insert da outbox faz `PATCH /orders/:id/status` falhar e o pedido mantém o status anterior, sem linha nova em `order_status_history` nem alteração de estoque. | [09:40] Bruno, [09:41] Diego |
| CAT-03 | Cliente sem webhook inscrito no status: nenhuma linha na outbox. | [09:34] Bruno |
| CAT-04 | Worker entrega evento pendente em até 2 s + tempo de resposta, com headers `X-Event-Id`, `X-Webhook-Id`, `X-Timestamp`, `X-Signature` e `Content-Type` corretos. | [09:09] Diego, [09:44] Diego, [09:44] Sofia |
| CAT-05 | `X-Signature` confere com HMAC-SHA256 do corpo bruto usando a secret do endpoint. | [09:22] Sofia |
| CAT-06 | Endpoint que não responde em 10 s gera tentativa com `WEBHOOK_DELIVERY_TIMEOUT` e reagendamento em 1 min. | [09:42] Diego, [09:17] Larissa |
| CAT-07 | Após a última retentativa falhar, existe linha em `webhook_dead_letter` com payload e motivo, e a outbox fica `FAILED`. | [09:18] Diego |
| CAT-08 | Replay por `ADMIN` devolve o evento a `PENDING` com o mesmo `event_id`; por `OPERATOR` retorna 403; o log registra o `userId`. | [09:36] Sofia, [09:36] Larissa |
| CAT-09 | Cadastro com URL `http://` retorna 400 `WEBHOOK_INVALID_URL`. | [09:23] Sofia |
| CAT-10 | Secrets distintas por endpoint; rotação mantém a anterior válida por 24 h. | [09:21] Sofia |
| CAT-11 | Payload acima de 64 KB não é enviado e vai para a DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`. | [09:24] Larissa |
| CAT-12 | `GET /webhooks/:id/deliveries` retorna no máximo 100 itens com sucesso/falha, payload, resposta e duração. | [09:34] Marcos |
| CAT-13 | Corpo do evento contém exatamente os campos da seção 6.8 e não contém itens. | [09:43] Diego |
| CAT-14 | Reiniciar a API não interrompe o worker, e vice-versa. | [09:11] Diego |
| CAT-15 | Nenhum log contém o valor de uma secret. | [09:22] Diego |
| CAT-16 | `npm test` existente continua passando com a nova limpeza em `tests/setup.ts`. | `tests/setup.ts` |

## 13. Riscos e mitigação

| Risco | Prob. | Impacto | Mitigação |
| --- | --- | --- | --- |
| Transação de status mais lenta ou com mais contenção por causa da leitura de endpoints e insert na outbox | Média | Médio | Índice `(customerId, active)`; payload pequeno; medir latência de `PATCH /orders/:id/status` antes e depois |
| Worker parado sem ninguém perceber | Média | Alto | Métrica de idade do pendente mais antigo; logs de start/stop; supervisão do processo pela infraestrutura |
| Evento preso em `PROCESSING` após crash | Baixa | Médio | Recuperação de órfãos no boot (proposta) |
| Eventos do mesmo pedido fora de ordem durante retry | Baixa | Médio | Corpo com `from_status`, `to_status` e `timestamp`; decisão em RFC Q-10 ([09:13] Larissa) |
| Secret exposta por log ou resposta da própria plataforma | Baixa | Alto | Redaction de `*.secret` no logger; secret só nas respostas de criação e rotação; revisão da Sofia ([09:46] Sofia) |
| Crescimento da outbox e de `webhook_deliveries` | Alta (longo prazo) | Baixo | Arquivamento planejado para depois ([09:08] Diego); índices por `createdAt` |

Riscos de produto (cliente sem deduplicação, vazamento de secret pelo cliente, permissões abertas do CRUD, rajadas de eventos, prazo) estão no [PRD, seção 10](PRD.md#10-riscos-e-mitigação).
