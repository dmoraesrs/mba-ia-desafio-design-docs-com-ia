# ADR-005: Garantia de entrega at-least-once com deduplicação por `X-Event-Id`

- **Data da decisão:** reunião técnica de quinta-feira, 09:00 (registrada em `TRANSCRICAO.md`)
- **Decisores:** Diego (Plataforma), Larissa (Tech Lead), Sofia (Segurança), Marcos (Produto)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md)

## Status

Aceito

## Contexto

Com outbox, worker e retry ([ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md)), existem cenários em que o mesmo evento é enviado mais de uma vez: o cliente processa mas a resposta não chega dentro do timeout, o worker cai depois do envio e antes de marcar a entrega, ou um administrador faz replay da DLQ. Precisamos dizer claramente que garantia a plataforma oferece.

## Decisão

1. **At-least-once**: a plataforma garante que o evento é entregue pelo menos uma vez, e o cliente pode receber o mesmo evento duas vezes ([09:24] Diego, [09:26] Larissa).
2. **`X-Event-Id`**: cada evento leva um UUID gerado quando entra na outbox, único por evento; o cliente deduplica por ele ([09:25] Diego). O mesmo valor também aparece no corpo como `event_id` ([09:43] Diego).
3. O id é estável entre retentativas do mesmo evento, já que nasce na inserção da outbox e não no envio ([09:25] Diego). Pela mesma razão, o replay da DLQ recoloca **o mesmo evento** na outbox ([09:18] Diego), preservando o id.
4. **Documentação destacada no portal de desenvolvedor** sobre a necessidade de deduplicação ([09:26] Marcos).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| Exactly-once | Exigiria coordenação dos dois lados e fica muito mais complexo ([09:25] Diego). |
| At-most-once (sem retry) | Incompatível com a política de retry decidida para clientes offline ([09:14] Larissa, [09:15] Diego). |

## Consequências

**Positivas**
- Padrão de mercado, usado por Stripe e GitHub, familiar aos integradores ([09:25] Diego).
- Implementação simples do nosso lado; retentativas seguras.

**Negativas / trade-offs**
- Joga responsabilidade de deduplicação para o cliente ([09:25] Sofia).
- Clientes que não implementarem dedup podem processar o mesmo evento duas vezes.
- Trade-off aceito: simplicidade e robustez da entrega em troca de exigir idempotência do consumidor.
