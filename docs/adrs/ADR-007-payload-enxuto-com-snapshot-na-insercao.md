# ADR-007: Payload enxuto, renderizado como snapshot na inserção da outbox

- **Data da decisão:** reunião técnica de quinta-feira, 09:00, incluindo a conversa após a saída de Marcos e Sofia (registrada em `TRANSCRICAO.md`)
- **Decisores:** Diego (Plataforma), Larissa (Tech Lead), Bruno (Pedidos), Sofia (Segurança)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-004](ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)

## Status

Aceito

## Contexto

Decisão técnica secundária, registrada em ADR porque afeta o schema da outbox e o contrato com os clientes. Havia duas perguntas: o que vai no corpo e nos headers do envio ([09:43] Marcos, [09:44] Marcos), e se a outbox guarda o payload pronto ou só o `order_id` para renderizar no envio ([09:51] Bruno). Como o envio pode acontecer até ~15 h depois da mudança ([ADR-003](ADR-003-retry-com-backoff-e-dlq.md)), o pedido pode ter mudado nesse meio tempo.

## Decisão

1. **Snapshot na inserção**: o payload é renderizado e gravado na outbox no momento da mudança de status, para refletir o estado daquele instante ([09:52] Larissa, [09:52] Diego, [09:52] Bruno).
2. **Corpo JSON enxuto** com `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do pedido como `total_cents`. Itens **não** vão no payload; quem quiser detalhes consulta `GET /orders/:id` ([09:43] Diego, [09:44] Bruno).
3. **Headers**: `X-Event-Id`, `X-Signature`, `X-Timestamp` (momento do envio, para o cliente detectar replay se quiser), `Content-Type: application/json` ([09:44] Diego) e `X-Webhook-Id` com o id do endpoint cadastrado ([09:44] Sofia, [09:45] Diego).
4. **Limite de 64 KB**: payload acima disso não é enviado e gera erro, sem truncar ([09:23] Sofia, [09:24] Diego, [09:24] Larissa).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| Guardar só `order_id` e renderizar no envio | O evento passaria a refletir o estado do pedido no momento do envio, não da mudança, gerando "caso esquisito" ([09:52] Larissa). |
| Incluir os itens do pedido | Infla o payload sem necessidade ([09:43] Diego). |
| Truncar payloads acima do limite | Payload desse tamanho indica erro; melhor falhar ([09:23] Sofia). |

## Consequências

**Positivas**
- Retentativas e replays enviam exatamente o mesmo corpo, o que mantém a assinatura e a deduplicação coerentes.
- Payload pequeno, bem abaixo do limite de 64 KB ([09:24] Diego).
- `X-Webhook-Id` permite ao cliente com vários cadastros saber qual deles recebeu o envio.

**Negativas / trade-offs**
- A outbox armazena JSON por evento, aumentando o volume em disco em relação a guardar só o id.
- O cliente que precisa de itens faz uma chamada extra a `GET /orders/:id`.
- `X-Timestamp` não entra na assinatura (que cobre só o corpo, ver [ADR-004](ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md)); a detecção de replay pelo cliente é opcional e limitada.
- Trade-off aceito: fidelidade histórica do evento em troca de mais armazenamento.
