# RFC-001: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Autora** | Larissa (Tech Lead), a partir da reunião técnica registrada em `TRANSCRICAO.md` |
| **Status** | Em revisão |
| **Data** | 2026-11-19 |
| **Revisores** | Diego (Plataforma), Bruno (Pedidos), Sofia (Segurança), Marcos (Produto) |
| **Documentos relacionados** | [PRD](PRD.md), [FDD](FDD.md), [ADRs](adrs/README.md), [Tracker](TRACKER.md) |

## 1. Resumo (TL;DR)

Propomos que o OMS passe a notificar clientes B2B por **webhooks de saída** sempre que o status de um pedido muda. Cada mudança de status grava um evento numa **tabela outbox no MySQL, na mesma transação** de `OrderService.changeStatus`. Um **worker em processo separado** lê a outbox a cada 2 segundos, envia um POST HTTPS **assinado com HMAC-SHA256** (secret por endpoint) e aplica **retry com backoff de 1m/5m/30m/2h/12h**, movendo para uma **DLQ** o que esgotar as tentativas. A entrega é **at-least-once**, com `X-Event-Id` para deduplicação no cliente. Nenhuma infraestrutura nova: o módulo segue os padrões do projeto.

## 2. Contexto e problema

Atlas Comercial, MaxDistribuição e Nova Cargo pediram formalmente notificação em tempo real de mudança de status de pedidos. Hoje eles fazem polling em `GET /orders`, o que deixa a integração lenta e cara; a Atlas sinalizou que pode migrar para um concorrente se a entrega não sair até o fim do trimestre ([09:00] Marcos). Para os clientes, "tempo real" é qualquer coisa abaixo de 10 segundos ([09:02] Marcos). O fluxo é só de saída ([09:02] Marcos).

O OMS não tem nenhum mecanismo de eventos, filas ou notificação externa. A mudança de status é uma transação já pesada (pedido, histórico e estoque) em `src/modules/orders/order.service.ts` ([09:04] Bruno), e qualquer solução precisa preservar sua consistência e seu desempenho.

## 3. Proposta técnica

### 3.1 Visão geral

```
 ┌──────────────────────┐  mesma transação  ┌──────────────┐  polling 2 s  ┌──────────────┐  HTTPS + HMAC  ┌──────────┐
 │ API: mudança de      │ ────────────────▶ │ MySQL:       │ ────────────▶ │ Worker       │ ─────────────▶ │ Cliente  │
 │ status do pedido     │                   │ outbox       │               │ (processo    │                └──────────┘
 └──────────────────────┘                   └──────────────┘ ◀──────────── │  separado)   │
                                                             retry / DLQ   └──────────────┘
```

### 3.2 Componentes

1. **Outbox transacional** ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)). A mudança de status grava o evento, já filtrado pelos status que o cliente assina e com o payload congelado ([ADR-007](adrs/ADR-007-payload-enxuto-com-snapshot-na-insercao.md)), na mesma transação. Se a gravação falhar, o status não muda.
2. **Worker** ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)). Processo Node próprio, mesmo banco e mesma stack, polling de 2 s, instância única para preservar a ordem por pedido (com a ressalva de Q-10).
3. **Resiliência** ([ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)). Timeout por chamada, backoff com progressão fixa, DLQ em tabela própria, replay manual por administrador.
4. **Segurança** ([ADR-004](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md)). HTTPS obrigatório, HMAC-SHA256 do corpo, secret gerada pela plataforma por endpoint, rotação com 24 h de convivência.
5. **Semântica de entrega** ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)). At-least-once; deduplicação pelo cliente via `X-Event-Id`.
6. **API de configuração**. CRUD de webhooks e histórico de entregas para qualquer usuário autenticado; replay da DLQ restrito a `ADMIN` ([09:36] Sofia, [09:37] Sofia).
7. **Padrões do projeto** ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)). Módulo `src/modules/webhooks`, erros `WEBHOOK_*`, Pino, error middleware e `requireRole` existentes.

Contratos, schema das tabelas, matriz de erros e fluxos passo a passo estão no [FDD](FDD.md).

## 4. Alternativas consideradas

| # | Alternativa | Trade-off que levou ao descarte | Origem |
| --- | --- | --- | --- |
| ALT-01 | **Disparo HTTP síncrono** dentro de `changeStatus` | Acoplaria a latência e a disponibilidade do cliente à transação de status: um cliente lento travaria mudanças de outros pedidos, e um cliente fora do ar forçaria um rollback inaceitável. | [09:04] Bruno, [09:06] Diego |
| ALT-02 | **Redis Streams** (ou outro broker) | Mais reativo e escalável, mas exige subir e operar infraestrutura nova; para um time pequeno, overengineering. | [09:07] Larissa, [09:07] Diego |
| ALT-03 | **Trigger no MySQL** para acordar o worker | MySQL não tem `NOTIFY/LISTEN`; trigger só executa SQL, e avisar um processo externo exigiria improviso. Polling de 2 s já cumpre o limite de 10 s. | [09:09] Bruno, [09:09] Diego |
| ALT-04 | **Retry indefinido** / **3 tentativas** | Indefinido deixa eventos pendurados para sempre; 3 tentativas cobrem só ~30 min e não sobrevivem a manutenções de 2 h. | [09:15] Diego, [09:16] Bruno, [09:16] Diego |
| ALT-05 | **Secret global** da plataforma | Um único vazamento comprometeria todos os clientes. | [09:21] Sofia |
| ALT-06 | **Exactly-once** | Exigiria coordenação dos dois lados, muito mais complexo; at-least-once com `event_id` é o padrão de mercado. | [09:25] Diego |
| ALT-07 | **Renderizar o payload no envio** (outbox só com `order_id`) | O evento passaria a refletir o pedido no momento do envio, não da mudança. | [09:51] Bruno, [09:52] Larissa |

## 5. Questões em aberto

| # | Questão | Situação | Origem |
| --- | --- | --- | --- |
| Q-01 | **Rate limiting de saída** por cliente (ex.: 50 mudanças em um minuto geram 50 chamadas). | Fora do escopo; observar em produção e decidir depois. | [09:38] Diego, [09:39] Diego, [09:39] Larissa |
| Q-02 | **Escala para múltiplos workers**: particionar por `order_id` ou usar lock pessimista. | Adiado; limitação conhecida enquanto for single-worker. | [09:13] Diego, [09:13] Larissa |
| Q-03 | **Assinatura durante o grace period de 24 h**: o envio é assinado com a secret nova, a antiga ou ambas? A reunião definiu a convivência, não o mecanismo. | Não decidido; levar para a revisão de segurança. | [09:21] Sofia, [09:46] Sofia |
| Q-04 | **Contagem das tentativas**: "5 tentativas" com 5 intervalos e ~15 h "entre primeira falha e última tentativa" só fecha como 1 envio inicial + 5 retentativas. Este pacote adota essa leitura. | Confirmar com Larissa e Diego. | [09:17] Diego, [09:48] Larissa |
| Q-05 | **Onde aplicar o limite de 64 KB**: na inserção (dentro da transação, o que faria a mudança de status falhar) ou no worker (evento vai para a DLQ). O FDD propõe o worker. | Não decidido. | [09:24] Larissa, [09:40] Bruno |
| Q-06 | **Notificação por e-mail** quando o webhook falha repetidamente. | Adiado para próxima fase, após medir impacto. | [09:37] Larissa, [09:38] Marcos |
| Q-07 | **Endurecer permissões do CRUD** de webhooks (hoje qualquer role autenticada). | "Mais pra frente". | [09:37] Sofia |
| Q-08 | **Proteção da secret em repouso**: ela precisa ser recuperável para assinar; a forma (criptografia, KMS, etc.) não foi discutida. | Levar para a revisão de segurança. | [09:21] Bruno, [09:46] Sofia |
| Q-09 | **Classificação de respostas**: a reunião só tratou timeout e cliente offline como falha; não definiu se respostas 4xx devem ser retentadas. O FDD trata qualquer não 2xx como falha retentável. | Confirmar. | [09:14] Larissa, [09:42] Diego |
| Q-10 | **Ordenação durante retry**: a garantia "por `order_id`, single-worker" pressupõe envio em ordem de `created_at`, mas um evento em backoff não bloqueia o evento seguinte do mesmo pedido, que pode chegar antes. Bloquear os seguintes (head-of-line) ou aceitar e documentar? | Não discutido; o FDD documenta a limitação. | [09:12] Diego, [09:13] Larissa |
| Q-11 | **Evento na criação do pedido**: `OrderService.create` grava histórico `null → PENDING`, mas a reunião só tratou `changeStatus`. Criação deve gerar evento? | Não discutido; fora do escopo até decisão. | [09:40] Bruno |

## 6. Impacto e riscos

**Impacto no sistema existente**
- `changeStatus` ganha uma leitura (webhooks do cliente) e uma escrita (outbox) por transação.
- Quatro tabelas novas (endpoints, outbox, entregas e dead letter) em uma migração Prisma; nenhuma tabela existente é alterada.
- Novo processo (`npm run worker`) para implantar e monitorar.
- Nenhuma mudança de contrato nos endpoints atuais.

**Riscos principais** (tabela completa com probabilidade e mitigação no [PRD](PRD.md#10-riscos-e-mitigação) e no [FDD](FDD.md#13-riscos-e-mitigação))
- Aumento de latência ou contenção na transação de status.
- Worker parado sem alarme, acumulando pendentes.
- Clientes sem deduplicação processando eventos duplicados.
- Prazo de três sprints apertado, dependente da revisão de segurança ([09:46] Larissa, [09:46] Sofia).

## 7. Decisões relacionadas

- [ADR-001: Padrão Outbox no MySQL existente](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002: Worker em processo separado, polling de 2 s](adrs/ADR-002-worker-separado-em-polling.md)
- [ADR-003: Retry com backoff exponencial e DLQ](adrs/ADR-003-retry-com-backoff-e-dlq.md)
- [ADR-004: HMAC-SHA256 com secret por endpoint](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005: At-least-once com `X-Event-Id`](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)
- [ADR-006: Reuso dos padrões existentes](adrs/ADR-006-reuso-dos-padroes-existentes.md)
- [ADR-007: Payload enxuto com snapshot na inserção](adrs/ADR-007-payload-enxuto-com-snapshot-na-insercao.md)
