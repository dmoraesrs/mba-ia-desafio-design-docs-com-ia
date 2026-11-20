# ADR-004: Assinatura HMAC-SHA256 com secret única por endpoint e rotação com grace period

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira, 09:00 (registrada em `TRANSCRICAO.md`)
- **Decisores:** Sofia (Segurança), Larissa (Tech Lead), Diego (Plataforma)
- **Relacionados:** [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-007](ADR-007-payload-enxuto-com-snapshot-na-insercao.md)

## Contexto

Os webhooks são somente de saída: a plataforma notifica o cliente, o cliente não envia nada ([09:02] Marcos, [09:03] Sofia). Os eventos levam dados de pedido para fora da nossa infraestrutura, e o cliente precisa conseguir verificar que a requisição veio da plataforma e que o payload não foi adulterado ([09:19] Sofia). Já houve cliente que vazou secret em log da própria aplicação ([09:22] Diego).

## Decisão

1. **HMAC-SHA256 sobre o corpo do request**, enviado no header `X-Signature`; o cliente verifica do lado dele ([09:20] Sofia, [09:22] Sofia).
2. **Secret única por endpoint de webhook**, nunca uma secret global da plataforma ([09:21] Sofia). A configuração do webhook guarda url, secret, customer_id e estado ativo ([09:21] Bruno, [09:21] Sofia).
3. **A secret é gerada pela plataforma e devolvida na criação do webhook** ([09:31] Marcos).
4. **Rotação pela API**, com a secret anterior válida por 24 horas em paralelo; depois disso ela deixa de valer ([09:21] Sofia, [09:22] Sofia).
5. **TLS obrigatório**: URL precisa ser `https`; `http` é recusado com erro de validação no schema Zod ([09:23] Sofia).
6. **Revisão de segurança dedicada** do código de HMAC e de geração de secret, com pelo menos dois dias úteis antes do deploy ([09:46] Sofia, [09:47] Larissa).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| Secret global da plataforma | "Se vaza uma, vaza tudo" ([09:21] Sofia). |
| Secret sem rotação | Um vazamento do lado do cliente, como o que já aconteceu, não teria remediação sem quebrar a integração ([09:21] Sofia, [09:22] Diego). |
| Aceitar URLs `http` | Payload com dados de pedido trafegaria em claro; recusado na validação ([09:23] Sofia). |

## Consequências

**Positivas**
- Autenticidade e integridade verificáveis com bibliotecas padrão de mercado ([09:20] Sofia).
- Vazamento de uma secret afeta um único endpoint.
- O cliente consegue rotacionar sem janela de indisponibilidade.

**Negativas / trade-offs**
- A plataforma precisa guardar a secret de forma recuperável para assinar (não dá para guardar só hash), o que aumenta o cuidado com o armazenamento. A forma de proteção em repouso não foi discutida e fica para a revisão da Sofia.
- Durante as 24 h de grace period, não foi definido como o envio é assinado (com a secret nova, com a antiga ou com ambas). Registrado como questão em aberto no [RFC](../RFC.md#questões-em-aberto).
- A assinatura cobre só o corpo; o `X-Timestamp` não é assinado, o que limita a proteção contra replay do lado do cliente (ver [ADR-007](ADR-007-payload-enxuto-com-snapshot-na-insercao.md)).
- Trade-off aceito: complexidade de gestão de secrets por endpoint em troca de isolamento de vazamentos.
