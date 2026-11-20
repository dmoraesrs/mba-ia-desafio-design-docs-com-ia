# Tracker de Rastreabilidade

Referência cruzada entre cada item registrado no pacote de documentos e a sua origem na reunião (`TRANSCRICAO.md`) ou no código do repositório.

- **Fonte `TRANSCRICAO`**: a localização é o timestamp e o falante da fala que dá origem ao item. Quando várias falas contribuem, a linha aponta a fala decisiva; as demais aparecem citadas no próprio documento.
- **Fonte `CODIGO`**: a localização é o caminho do arquivo que existe hoje no repositório.
- Itens marcados como **(proposta)** no FDD apontam para a regra da reunião da qual foram derivados.

## Tabela

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | `docs/PRD.md` | Contexto | Três clientes B2B pediram notificação de mudança de status; hoje fazem polling em `GET /orders` | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | `docs/PRD.md` | Restrição de negócio | Atlas pediu a entrega para o fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-CTX-03 | `docs/PRD.md` | Contexto | Ciclo de vida do pedido definido na máquina de estados | CODIGO | `src/modules/orders/order.status.ts` |
| PRD-CTX-04 | `docs/PRD.md` | Contexto | Código atual sem eventos, filas ou chamadas de saída | CODIGO | `src/modules/orders/order.service.ts` |
| PRD-PUB-01 | `docs/PRD.md` | Público-alvo | Usuários da plataforma que representam o cliente cadastram via API com JWT | TRANSCRICAO | [09:32] Marcos |
| PRD-PUB-02 | `docs/PRD.md` | Público-alvo | Administradores reprocessam a fila de falhas | TRANSCRICAO | [09:36] Sofia |
| PRD-CEN-01 | `docs/PRD.md` | Cenário | Assinar só `SHIPPED` e `DELIVERED` | TRANSCRICAO | [09:33] Marcos |
| PRD-CEN-02 | `docs/PRD.md` | Cenário | Endpoint do cliente fora do ar por 2 h em manutenção | TRANSCRICAO | [09:16] Diego |
| PRD-CEN-03 | `docs/PRD.md` | Cenário | Consulta das últimas entregas para investigar integração | TRANSCRICAO | [09:34] Marcos |
| PRD-CEN-04 | `docs/PRD.md` | Cenário | Secret vazada em log do cliente | TRANSCRICAO | [09:22] Diego |
| PRD-OBJ-01 | `docs/PRD.md` | Objetivo / métrica | Entrega em menos de 10 s | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | `docs/PRD.md` | Objetivo / métrica | 3 de 3 clientes solicitantes ativos | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-03 | `docs/PRD.md` | Objetivo / métrica | Entrega em 3 sprints com revisão de segurança | TRANSCRICAO | [09:47] Larissa |
| PRD-OBJ-04 | `docs/PRD.md` | Objetivo / métrica | Zero notificações perdidas (entregue ou fila de falhas) | TRANSCRICAO | [09:18] Diego |
| PRD-OBJ-05 | `docs/PRD.md` | Objetivo / métrica | Zero mudanças de status sem evento | TRANSCRICAO | [09:40] Bruno |
| PRD-OOS-01 | `docs/PRD.md` | Fora de escopo | E-mail de alerta adiado para a próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | `docs/PRD.md` | Fora de escopo | Dashboard visual é projeto do time de frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-03 | `docs/PRD.md` | Fora de escopo | Rate limiting: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-04 | `docs/PRD.md` | Fora de escopo | Webhooks de entrada | TRANSCRICAO | [09:02] Marcos |
| PRD-OOS-05 | `docs/PRD.md` | Fora de escopo | Ordem global e múltiplos workers | TRANSCRICAO | [09:13] Larissa |
| PRD-OOS-06 | `docs/PRD.md` | Fora de escopo | Exactly-once | TRANSCRICAO | [09:25] Diego |
| PRD-OOS-07 | `docs/PRD.md` | Fora de escopo | Arquivamento de notificações antigas | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-08 | `docs/PRD.md` | Fora de escopo | Permissões mais restritas no CRUD, "mais pra frente" | TRANSCRICAO | [09:37] Sofia |
| PRD-OOS-09 | `docs/PRD.md` | Fora de escopo | Itens do pedido fora da notificação | TRANSCRICAO | [09:43] Diego |
| PRD-FR-01 | `docs/PRD.md` | Requisito Funcional | Cadastrar webhook com cliente, URL e status assinados | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-01b | `docs/PRD.md` | Restrição | Cliente informado na requisição, não deduzido do JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-02 | `docs/PRD.md` | Requisito Funcional | Secret gerada pela plataforma e devolvida no cadastro | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | `docs/PRD.md` | Requisito Funcional | Listar webhooks de um cliente | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | `docs/PRD.md` | Requisito Funcional | Editar webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | `docs/PRD.md` | Requisito Funcional | Remover webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | `docs/PRD.md` | Requisito Funcional | Notificar webhooks ativos que assinam o novo status | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-07 | `docs/PRD.md` | Requisito Funcional | Conteúdo da notificação, sem itens | TRANSCRICAO | [09:43] Diego |
| PRD-FR-08 | `docs/PRD.md` | Requisito Funcional | Notificação congelada no momento da mudança | TRANSCRICAO | [09:52] Larissa |
| PRD-FR-09 | `docs/PRD.md` | Requisito Funcional | Notificação assinada com a secret do webhook | TRANSCRICAO | [09:19] Sofia |
| PRD-FR-10 | `docs/PRD.md` | Requisito Funcional | Identificador único de evento para deduplicação | TRANSCRICAO | [09:25] Diego |
| PRD-FR-10b | `docs/PRD.md` | Requisito Funcional | Identificador do webhook cadastrado em cada envio | TRANSCRICAO | [09:44] Sofia |
| PRD-FR-11 | `docs/PRD.md` | Requisito Funcional | Rotação de secret com 24 h de convivência | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-12 | `docs/PRD.md` | Requisito Funcional | Retentativa automática 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-13 | `docs/PRD.md` | Requisito Funcional | Fila de falhas com conteúdo e motivo | TRANSCRICAO | [09:18] Diego |
| PRD-FR-14 | `docs/PRD.md` | Requisito Funcional | Reenvio só por administrador, com registro de autoria | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-15 | `docs/PRD.md` | Requisito Funcional | Últimas 100 entregas com resultado, conteúdo, resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-16 | `docs/PRD.md` | Requisito Funcional | Recusar URL sem HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-01 | `docs/PRD.md` | Requisito Não Funcional | Latência < 10 s, atraso mínimo de 2 s aceito | TRANSCRICAO | [09:10] Larissa |
| PRD-NFR-02 | `docs/PRD.md` | Requisito Não Funcional | Mudança de status isolada do endpoint do cliente | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-03 | `docs/PRD.md` | Requisito Não Funcional | Consistência entre status e notificação | TRANSCRICAO | [09:06] Diego |
| PRD-NFR-04 | `docs/PRD.md` | Requisito Não Funcional | Entrega pelo menos uma vez | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-05 | `docs/PRD.md` | Requisito Não Funcional | Timeout de 10 s | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-06 | `docs/PRD.md` | Requisito Não Funcional | Limite de 64 KB com erro | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-07 | `docs/PRD.md` | Requisito Não Funcional | HTTPS, HMAC-SHA256, secret por webhook | TRANSCRICAO | [09:20] Sofia |
| PRD-NFR-08 | `docs/PRD.md` | Requisito Não Funcional | Ordem só por pedido e com um processador | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-09 | `docs/PRD.md` | Requisito Não Funcional | Envio em processo separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-10 | `docs/PRD.md` | Requisito Não Funcional | Sem infraestrutura nova, padrões atuais | TRANSCRICAO | [09:30] Larissa |
| PRD-NFR-11 | `docs/PRD.md` | Requisito Não Funcional | Auditoria do reenvio manual | TRANSCRICAO | [09:36] Sofia |
| PRD-DEP-01 | `docs/PRD.md` | Dependência | Revisão de segurança com 2 dias úteis | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | `docs/PRD.md` | Dependência | Sessão de revisão do design com Bruno e Diego | TRANSCRICAO | [09:50] Larissa |
| PRD-DEP-03 | `docs/PRD.md` | Dependência | Documentação no portal de desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-04 | `docs/PRD.md` | Dependência | Confirmação de prazo com a Atlas | TRANSCRICAO | [09:47] Marcos |
| PRD-DEP-05 | `docs/PRD.md` | Dependência | Perfil `ADMIN` existente | CODIGO | `prisma/schema.prisma` |
| PRD-DEP-06 | `docs/PRD.md` | Dependência | Novo processo de envio implantado e mantido pela operação | TRANSCRICAO | [09:11] Larissa |
| PRD-RISK-01 | `docs/PRD.md` | Risco | Atraso pode levar a Atlas ao concorrente | TRANSCRICAO | [09:00] Marcos |
| PRD-RISK-02 | `docs/PRD.md` | Risco | Cliente sem deduplicação | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-03 | `docs/PRD.md` | Risco | Vazamento de secret pelo cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-04 | `docs/PRD.md` | Risco | Rajadas de notificações | TRANSCRICAO | [09:38] Diego |
| PRD-RISK-05 | `docs/PRD.md` | Risco | CRUD aberto a qualquer role autenticada | TRANSCRICAO | [09:37] Sofia |
| PRD-RISK-06 | `docs/PRD.md` | Risco | Cliente fora do ar por mais de ~15 h tem notificações na fila de falhas | TRANSCRICAO | [09:17] Marcos |
| PRD-TEST-01 | `docs/PRD.md` | Estratégia de teste | Revisão de segurança antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-TEST-02 | `docs/PRD.md` | Estratégia de teste | Portal explica assinatura e deduplicação | TRANSCRICAO | [09:26] Marcos |
| RFC-CTX-01 | `docs/RFC.md` | Contexto | Transação de status já pesada (orders, history, estoque) | TRANSCRICAO | [09:04] Bruno |
| RFC-CTX-02 | `docs/RFC.md` | Contexto | Ponto de mudança de status no código | CODIGO | `src/modules/orders/order.service.ts` |
| RFC-PROP-01 | `docs/RFC.md` | Decisão | Outbox no MySQL na mesma transação | TRANSCRICAO | [09:08] Larissa |
| RFC-PROP-02 | `docs/RFC.md` | Decisão | Worker separado com polling de 2 s | TRANSCRICAO | [09:10] Larissa |
| RFC-PROP-03 | `docs/RFC.md` | Decisão | CRUD para qualquer autenticado; replay só `ADMIN` | TRANSCRICAO | [09:37] Sofia |
| RFC-ALT-01 | `docs/RFC.md` | Alternativa descartada | Disparo HTTP síncrono no `changeStatus` | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | `docs/RFC.md` | Alternativa descartada | Redis Streams / broker | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | `docs/RFC.md` | Alternativa descartada | Trigger no MySQL para acordar o worker | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | `docs/RFC.md` | Alternativa descartada | Retry indefinido ou só 3 tentativas | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-05 | `docs/RFC.md` | Alternativa descartada | Secret global | TRANSCRICAO | [09:21] Sofia |
| RFC-ALT-06 | `docs/RFC.md` | Alternativa descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-07 | `docs/RFC.md` | Alternativa descartada | Renderizar payload no envio | TRANSCRICAO | [09:52] Larissa |
| RFC-Q-01 | `docs/RFC.md` | Questão em aberto | Rate limiting de saída | TRANSCRICAO | [09:39] Diego |
| RFC-Q-02 | `docs/RFC.md` | Questão em aberto | Escala para múltiplos workers | TRANSCRICAO | [09:13] Diego |
| RFC-Q-03 | `docs/RFC.md` | Questão em aberto | Assinatura durante o grace period | TRANSCRICAO | [09:21] Sofia |
| RFC-Q-04 | `docs/RFC.md` | Questão em aberto | Contagem das tentativas (5 vs 1 + 5) | TRANSCRICAO | [09:17] Diego |
| RFC-Q-05 | `docs/RFC.md` | Questão em aberto | Onde aplicar o limite de 64 KB | TRANSCRICAO | [09:24] Larissa |
| RFC-Q-06 | `docs/RFC.md` | Questão em aberto | E-mail de alerta | TRANSCRICAO | [09:37] Larissa |
| RFC-Q-07 | `docs/RFC.md` | Questão em aberto | Endurecer permissões do CRUD | TRANSCRICAO | [09:37] Sofia |
| RFC-Q-08 | `docs/RFC.md` | Questão em aberto | Proteção da secret em repouso | TRANSCRICAO | [09:46] Sofia |
| RFC-Q-09 | `docs/RFC.md` | Questão em aberto | Retentar ou não respostas 4xx | TRANSCRICAO | [09:42] Diego |
| RFC-Q-10 | `docs/RFC.md` | Questão em aberto | Ordenação por pedido durante retry | TRANSCRICAO | [09:12] Diego |
| RFC-Q-11 | `docs/RFC.md` | Questão em aberto | Evento na criação do pedido | CODIGO | `src/modules/orders/order.service.ts` |
| RFC-RISK-01 | `docs/RFC.md` | Risco | Prazo de três sprints dependente da revisão de segurança | TRANSCRICAO | [09:46] Larissa |
| FDD-OT-01 | `docs/FDD.md` | Objetivo técnico | Entrega < 10 s | TRANSCRICAO | [09:02] Marcos |
| FDD-OT-02 | `docs/FDD.md` | Objetivo técnico | Atomicidade outbox + status | TRANSCRICAO | [09:41] Diego |
| FDD-OT-03 | `docs/FDD.md` | Objetivo técnico | Todo evento termina entregue ou na DLQ | TRANSCRICAO | [09:15] Diego |
| FDD-OT-04 | `docs/FDD.md` | Restrição | Sem dependência nova; Node 20 com `fetch` nativo | CODIGO | `package.json` |
| FDD-OT-05 | `docs/FDD.md` | Objetivo técnico | Módulo idêntico aos existentes, erros `WEBHOOK_*` | TRANSCRICAO | [09:27] Bruno |
| FDD-MOD-01 | `docs/FDD.md` | Decisão | Ids UUID `Char(36)` como no schema atual | TRANSCRICAO | [09:51] Larissa |
| FDD-MOD-02 | `docs/FDD.md` | Decisão | Endpoint guarda url, secret, customer_id, ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-MOD-03 | `docs/FDD.md` | Decisão | Status da outbox e índices em status e `created_at` | TRANSCRICAO | [09:08] Diego |
| FDD-MOD-04 | `docs/FDD.md` | Decisão | Tabela `webhook_dead_letter` com payload, motivo, timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-MOD-05 | `docs/FDD.md` | Integração | Novos modelos e relação virtual em `Customer` | CODIGO | `prisma/schema.prisma` |
| FDD-FLUXO-01 | `docs/FDD.md` | Fluxo | `publishWebhookEvent(tx, ...)` dentro da transação | TRANSCRICAO | [09:41] Bruno |
| FDD-FLUXO-02 | `docs/FDD.md` | Fluxo | Filtro de status na inserção | TRANSCRICAO | [09:34] Bruno |
| FDD-FLUXO-03 | `docs/FDD.md` | Fluxo | Worker lê pendentes mais antigos em lote pequeno | TRANSCRICAO | [09:08] Diego |
| FDD-FLUXO-04 | `docs/FDD.md` | Fluxo | Polling a cada 2 s | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-05 | `docs/FDD.md` | Fluxo | Ordem por `created_at`, single-worker | TRANSCRICAO | [09:12] Diego |
| FDD-FLUXO-06 | `docs/FDD.md` | Fluxo | Retry com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| FDD-FLUXO-07 | `docs/FDD.md` | Fluxo | Replay recoloca evento como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-08 | `docs/FDD.md` | Fluxo | Recuperação de órfãos em `PROCESSING` (proposta, coerente com at-least-once) | TRANSCRICAO | [09:24] Diego |
| FDD-CONTRATO-01 | `docs/FDD.md` | Contrato | `POST /webhooks` | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | `docs/FDD.md` | Contrato | `GET /webhooks?customerId=` | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | `docs/FDD.md` | Contrato | `PATCH /webhooks/:id` | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | `docs/FDD.md` | Contrato | `DELETE /webhooks/:id` | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | `docs/FDD.md` | Contrato | `POST /webhooks/:id/rotate-secret` (caminho proposto) | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | `docs/FDD.md` | Contrato | `GET /webhooks/:id/deliveries` | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | `docs/FDD.md` | Contrato | `POST /admin/webhooks/dead-letter/:id/replay` | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | `docs/FDD.md` | Contrato | Corpo JSON do evento enviado ao cliente | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-09 | `docs/FDD.md` | Contrato | Headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `Content-Type` | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-10 | `docs/FDD.md` | Contrato | Header `X-Webhook-Id` | TRANSCRICAO | [09:44] Sofia |
| FDD-CONTRATO-11 | `docs/FDD.md` | Restrição | `events` exclui `PENDING` (nenhuma transição tem PENDING como destino) | CODIGO | `src/modules/orders/order.status.ts` |
| FDD-CONTRATO-12 | `docs/FDD.md` | Contrato | Envelope de erro `{ error: { code, message, details } }` | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-CONTRATO-13 | `docs/FDD.md` | Contrato | Falha de schema vira 400 `VALIDATION_ERROR` | CODIGO | `src/middlewares/validate.middleware.ts` |
| FDD-CONTRATO-14 | `docs/FDD.md` | Contrato | Listagem no formato `paginated` | CODIGO | `src/shared/http/response.ts` |
| FDD-ERRO-01 | `docs/FDD.md` | Erro | `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | `docs/FDD.md` | Restrição | Prefixo `WEBHOOK_` em todos os códigos | TRANSCRICAO | [09:29] Larissa |
| FDD-ERRO-03 | `docs/FDD.md` | Erro | `WEBHOOK_CUSTOMER_NOT_FOUND` (cliente informado na requisição) | TRANSCRICAO | [09:32] Larissa |
| FDD-ERRO-04 | `docs/FDD.md` | Erro | `WEBHOOK_PAYLOAD_TOO_LARGE` | TRANSCRICAO | [09:24] Larissa |
| FDD-ERRO-05 | `docs/FDD.md` | Erro | `WEBHOOK_DELIVERY_TIMEOUT` | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-06 | `docs/FDD.md` | Erro | `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | TRANSCRICAO | [09:17] Larissa |
| FDD-ERRO-07 | `docs/FDD.md` | Erro | `WEBHOOK_DEAD_LETTER_NOT_FOUND` / `ALREADY_REPLAYED` | TRANSCRICAO | [09:18] Diego |
| FDD-ERRO-08 | `docs/FDD.md` | Integração | Classes de erro base com `code` customizável; `NotFoundError` fixa `NOT_FOUND` | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-ERRO-10 | `docs/FDD.md` | Erro | `WEBHOOK_DELIVERY_FAILED` para erro de rede ou status fora de 2xx (critério 2xx em proposta) | TRANSCRICAO | [09:15] Diego |
| FDD-ERRO-09 | `docs/FDD.md` | Integração | `AppError` como base para 404 com código próprio | CODIGO | `src/shared/errors/app-error.ts` |
| FDD-RES-01 | `docs/FDD.md` | Resiliência | Timeout de 10 s via `AbortSignal.timeout` | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | `docs/FDD.md` | Resiliência | Sem fallback por e-mail nesta fase | TRANSCRICAO | [09:37] Larissa |
| FDD-OBS-01 | `docs/FDD.md` | Observabilidade | Logs via Pino existente, sem ferramenta nova | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | `docs/FDD.md` | Observabilidade | Log de auditoria do replay com `userId` | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-03 | `docs/FDD.md` | Observabilidade | Redaction de secrets no logger | CODIGO | `src/shared/logger/index.ts` |
| FDD-OBS-04 | `docs/FDD.md` | Observabilidade | Correlação por `requestId` na API | CODIGO | `src/middlewares/request-logger.middleware.ts` |
| FDD-OBS-05 | `docs/FDD.md` | Observabilidade | Métrica de volume por cliente como insumo para rate limiting | TRANSCRICAO | [09:39] Diego |
| FDD-INT-01 | `docs/FDD.md` | Integração | `changeStatus` chama `publishWebhookEvent` com o `tx` | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | `docs/FDD.md` | Integração | Error middleware trata as novas classes sem mudança | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INT-03 | `docs/FDD.md` | Integração | `authenticate` e `requireRole('ADMIN')` | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INT-04 | `docs/FDD.md` | Integração | Precedente de `requireRole('ADMIN')` em rotas | CODIGO | `src/modules/users/user.routes.ts` |
| FDD-INT-05 | `docs/FDD.md` | Integração | Registro dos routers `/webhooks` e `/admin/webhooks` | CODIGO | `src/routes/index.ts` |
| FDD-INT-06 | `docs/FDD.md` | Integração | Composição em `buildControllers` | CODIGO | `src/app.ts` |
| FDD-INT-07 | `docs/FDD.md` | Integração | `src/server.ts` como modelo do worker | CODIGO | `src/server.ts` |
| FDD-INT-08 | `docs/FDD.md` | Integração | Worker usa `createPrismaClient()` própria | CODIGO | `src/config/database.ts` |
| FDD-INT-09 | `docs/FDD.md` | Integração | Novas variáveis no `envSchema` | CODIGO | `src/config/env.ts` |
| FDD-INT-10 | `docs/FDD.md` | Integração | Limpeza das novas tabelas antes de `customer` no setup de testes | CODIGO | `tests/setup.ts` |
| FDD-INT-11 | `docs/FDD.md` | Integração | Scripts `worker` / `worker:dev` em `package.json` | TRANSCRICAO | [09:11] Larissa |
| FDD-INT-12 | `docs/FDD.md` | Integração | Instância própria de PrismaClient por processo | TRANSCRICAO | [09:30] Bruno |
| FDD-INT-13 | `docs/FDD.md` | Integração | Estrutura de pastas do módulo e do worker | TRANSCRICAO | [09:28] Bruno |
| FDD-CAT-01 | `docs/FDD.md` | Critério de aceite | Uma linha `PENDING` por endpoint inscrito, na mesma transação | TRANSCRICAO | [09:40] Bruno |
| FDD-CAT-02 | `docs/FDD.md` | Critério de aceite | Falha na outbox desfaz a mudança de status | TRANSCRICAO | [09:41] Diego |
| FDD-CAT-03 | `docs/FDD.md` | Critério de aceite | Sem webhook inscrito, nenhuma linha | TRANSCRICAO | [09:34] Bruno |
| FDD-CAT-04 | `docs/FDD.md` | Critério de aceite | Headers corretos na entrega | TRANSCRICAO | [09:44] Diego |
| FDD-CAT-05 | `docs/FDD.md` | Critério de aceite | Assinatura confere com HMAC-SHA256 do corpo | TRANSCRICAO | [09:22] Sofia |
| FDD-CAT-06 | `docs/FDD.md` | Critério de aceite | Timeout de 10 s gera reagendamento em 1 min | TRANSCRICAO | [09:42] Diego |
| FDD-CAT-07 | `docs/FDD.md` | Critério de aceite | Última falha cria linha na DLQ e outbox `FAILED` | TRANSCRICAO | [09:18] Diego |
| FDD-CAT-08 | `docs/FDD.md` | Critério de aceite | Replay `ADMIN` ok, `OPERATOR` 403 | TRANSCRICAO | [09:36] Larissa |
| FDD-CAT-09 | `docs/FDD.md` | Critério de aceite | URL `http://` retorna `WEBHOOK_INVALID_URL` | TRANSCRICAO | [09:23] Sofia |
| FDD-CAT-10 | `docs/FDD.md` | Critério de aceite | Secret distinta por endpoint e rotação com 24 h | TRANSCRICAO | [09:21] Sofia |
| FDD-CAT-11 | `docs/FDD.md` | Critério de aceite | Payload > 64 KB vai para a DLQ sem envio | TRANSCRICAO | [09:24] Larissa |
| FDD-CAT-12 | `docs/FDD.md` | Critério de aceite | Histórico com no máximo 100 itens | TRANSCRICAO | [09:34] Marcos |
| FDD-CAT-13 | `docs/FDD.md` | Critério de aceite | Corpo com os campos definidos e sem itens | TRANSCRICAO | [09:43] Diego |
| FDD-CAT-14 | `docs/FDD.md` | Critério de aceite | Reinício da API não derruba o worker | TRANSCRICAO | [09:11] Diego |
| FDD-CAT-15 | `docs/FDD.md` | Critério de aceite | Nenhum log contém secret | TRANSCRICAO | [09:22] Diego |
| FDD-CAT-16 | `docs/FDD.md` | Critério de aceite | Suíte existente continua passando | CODIGO | `tests/setup.ts` |
| FDD-RISK-01 | `docs/FDD.md` | Risco | Crescimento da outbox, arquivamento depois | TRANSCRICAO | [09:08] Diego |
| FDD-RISK-03 | `docs/FDD.md` | Risco | Eventos fora de ordem durante retry | TRANSCRICAO | [09:13] Larissa |
| FDD-RISK-04 | `docs/FDD.md` | Risco | Secret exposta por log da plataforma; mitigado por redaction | CODIGO | `src/shared/logger/index.ts` |
| FDD-RISK-07 | `docs/FDD.md` | Risco | Transação de status mais lenta pela escrita na outbox | TRANSCRICAO | [09:04] Bruno |
| FDD-DEP-01 | `docs/FDD.md` | Dependência | Documentação de integração no portal | TRANSCRICAO | [09:40] Marcos |
| ADR-001 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Outbox no MySQL, mesma transação | TRANSCRICAO | [09:06] Diego |
| ADR-001-ALT-01 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Alternativa descartada | Redis Cluster é overengineering para time pequeno | TRANSCRICAO | [09:07] Diego |
| ADR-001-REF | `docs/adrs/ADR-001-outbox-no-mysql.md` | Referência de código | Transação de `changeStatus` | CODIGO | `src/modules/orders/order.service.ts` |
| ADR-002 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão | Worker em processo separado | TRANSCRICAO | [09:11] Diego |
| ADR-002-D2 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão | Entry-point `src/worker.ts` e `npm run worker` | TRANSCRICAO | [09:11] Larissa |
| ADR-002-D3 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão | Mesmo banco, mesma stack, outro processo | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT-01 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Alternativa descartada | Trigger do banco (Bruno perguntou) | TRANSCRICAO | [09:09] Bruno |
| ADR-002-TO | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Trade-off | Latência mínima de 2 s aceita | TRANSCRICAO | [09:10] Marcos |
| ADR-002-LIM | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Restrição | Clientes não pediram ordem global | TRANSCRICAO | [09:14] Marcos |
| ADR-003 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Decisão | Teto de 5 tentativas | TRANSCRICAO | [09:16] Larissa |
| ADR-003-D1 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Decisão | Backoff exponencial até falha permanente e DLQ | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-01 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Alternativa descartada | 3 tentativas (proposta do Bruno) | TRANSCRICAO | [09:16] Bruno |
| ADR-003-ALT-02 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Alternativa descartada | `failed` na própria outbox | TRANSCRICAO | [09:17] Larissa |
| ADR-003-TO | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Trade-off | 15 h de indisponibilidade é problema do cliente | TRANSCRICAO | [09:17] Marcos |
| ADR-003-RESUMO | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Decisão | Resumo final: 5 tentativas e DLQ separada | TRANSCRICAO | [09:48] Larissa |
| ADR-004 | `docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md` | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação 24 h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-D1 | `docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md` | Decisão | Algoritmo SHA-256 (padrão de mercado) | TRANSCRICAO | [09:20] Sofia |
| ADR-004-CTX | `docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md` | Contexto | Webhook só de saída | TRANSCRICAO | [09:03] Sofia |
| ADR-004-REV | `docs/adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md` | Restrição | Revisão de segurança antes de subir | TRANSCRICAO | [09:49] Sofia |
| ADR-005 | `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` | Decisão | At-least-once com `X-Event-Id` | TRANSCRICAO | [09:26] Larissa |
| ADR-005-CTX | `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` | Contexto | Retry para cliente offline | TRANSCRICAO | [09:14] Larissa |
| ADR-005-TO | `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` | Trade-off | Responsabilidade de dedup no cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-005-MIT | `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` | Mitigação | Destaque no portal de desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| ADR-006 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Decisão | Reuso máximo de AppError, Pino, error middleware, módulos, Zod | TRANSCRICAO | [09:30] Larissa |
| ADR-006-D1 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Decisão | Módulo `src/modules/webhooks` | TRANSCRICAO | [09:27] Bruno |
| ADR-006-D5 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Decisão | Função pura recebendo `tx`, sem injetar repository | TRANSCRICAO | [09:41] Diego |
| ADR-006-REF-01 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Referência de código | Padrão de módulo | CODIGO | `src/modules/orders/order.routes.ts` |
| ADR-006-REF-02 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Referência de código | Hierarquia de erros | CODIGO | `src/shared/errors/http-errors.ts` |
| ADR-006-REF-03 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Referência de código | Error middleware | CODIGO | `src/middlewares/error.middleware.ts` |
| ADR-006-REF-04 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Referência de código | Logger Pino | CODIGO | `src/shared/logger/index.ts` |
| ADR-007 | `docs/adrs/ADR-007-payload-enxuto-com-snapshot-na-insercao.md` | Decisão | Snapshot do payload na inserção | TRANSCRICAO | [09:52] Bruno |
| ADR-007-D2 | `docs/adrs/ADR-007-payload-enxuto-com-snapshot-na-insercao.md` | Decisão | Payload enxuto | TRANSCRICAO | [09:44] Bruno |
| ADR-007-D3 | `docs/adrs/ADR-007-payload-enxuto-com-snapshot-na-insercao.md` | Decisão | Header `X-Webhook-Id` aceito | TRANSCRICAO | [09:45] Diego |
| ADR-007-D4 | `docs/adrs/ADR-007-payload-enxuto-com-snapshot-na-insercao.md` | Decisão | 64 KB é teto generoso | TRANSCRICAO | [09:24] Diego |
| ADR-007-ALT-01 | `docs/adrs/ADR-007-payload-enxuto-com-snapshot-na-insercao.md` | Alternativa descartada | Truncar payload grande | TRANSCRICAO | [09:23] Sofia |
| ADR-007-ALT-02 | `docs/adrs/ADR-007-payload-enxuto-com-snapshot-na-insercao.md` | Alternativa descartada | Guardar só `order_id` | TRANSCRICAO | [09:51] Bruno |
