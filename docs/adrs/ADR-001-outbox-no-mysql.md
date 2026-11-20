# ADR-001: Padrão Outbox no MySQL existente

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira, 09:00 (registrada em `TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram para ser notificados quando o status dos seus pedidos muda; hoje eles fazem polling em `GET /orders` ([09:00] Marcos). A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de um `prisma.$transaction` que atualiza `orders`, insere em `order_status_history` e, conforme a transição, debita ou repõe `stock_quantity` ([09:04] Bruno). A aplicação não tem hoje fila, broker nem mecanismo de eventos.

O evento de notificação precisa refletir exatamente o que foi commitado: se o status mudou, o evento tem que existir; se a transação deu rollback, o evento não pode existir ([09:06] Diego, [09:40] Bruno).

## Decisão

Registrar cada evento de notificação numa tabela `webhook_outbox` do **mesmo MySQL** da aplicação, **dentro da mesma transação** que muda o status do pedido ([09:06] Diego, [09:08] Larissa). Um worker separado lê essa tabela e faz as chamadas HTTP (ver [ADR-002](ADR-002-worker-separado-em-polling.md)).

Regras que fazem parte da decisão:

1. Falha ao inserir na outbox faz rollback da mudança de status: não pode existir status alterado sem evento ([09:40] Bruno, [09:41] Diego).
2. A tabela tem índice em `status` (pendente, processando, falhou, entregue) e em `created_at` ([09:08] Diego).
3. O filtro de eventos por webhook é aplicado **na inserção**: se nenhum webhook ativo do cliente assina aquele status, nenhuma linha é criada ([09:34] Bruno, [09:34] Diego).
4. Identificadores em UUID, seguindo o padrão do schema (`@default(uuid()) @db.Char(36)` em `prisma/schema.prisma`) ([09:51] Larissa).
5. Arquivamento de linhas entregues (algo como 30 dias) fica fora do escopo desta feature ([09:08] Diego).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| Disparo HTTP síncrono dentro de `changeStatus` | A transação já é pesada; um cliente lento travaria mudanças de status de outros pedidos, e um cliente fora do ar não justifica rollback do status ([09:04] Bruno, [09:06] Diego). |
| Redis Streams (ou broker equivalente) | Exige subir e operar infraestrutura nova; para um time pequeno, Redis Cluster seria overengineering ([09:07] Larissa, [09:07] Diego). |
| Filtrar eventos na hora do envio | Geraria linhas na outbox que nunca seriam enviadas; filtrar na inserção economiza linhas ([09:34] Bruno). |
| Id autoincremental na outbox | Fugiria do padrão do projeto, onde todas as entidades usam UUID ([09:51] Larissa). |

## Consequências

**Positivas**
- Atomicidade entre mudança de status e registro do evento sem coordenação distribuída: "não tem inconsistência possível" ([09:06] Diego).
- Nenhuma infraestrutura nova: mesmo banco, mesmo Prisma, mesmo backup.
- A mudança de status não depende da disponibilidade do cliente.

**Negativas / trade-offs**
- A transação de `changeStatus` ganha uma escrita extra e uma leitura dos webhooks do cliente; mais trabalho dentro de uma transação que já é pesada.
- O MySQL passa a servir como fila: crescimento da tabela exige arquivamento, que ficou para depois ([09:08] Diego).
- Latência de entrega fica limitada pelo intervalo de polling do worker (ver [ADR-002](ADR-002-worker-separado-em-polling.md)).
- O trade-off aceito: troca-se a reatividade e a escala horizontal de um broker pela simplicidade operacional e pela garantia transacional.
