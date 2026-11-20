# ADR-003: Retry com backoff exponencial fixo e DLQ em tabela separada

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira, 09:00 (registrada em `TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Sofia (Segurança), Marcos (Produto)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)

## Contexto

O endpoint do cliente pode estar fora do ar ou lento. Já houve cliente com indisponibilidade de duas horas por manutenção planejada ([09:16] Diego). É preciso decidir quantas vezes tentar, com que intervalo, e o que fazer quando as tentativas acabam ([09:14] Larissa).

## Decisão

1. **Backoff exponencial com progressão fixa 1 min, 5 min, 30 min, 2 h, 12 h** ([09:17] Diego, [09:17] Larissa). A janela é de quase 15 horas entre a primeira falha e a última tentativa ([09:17] Diego), aceita pelo produto ([09:17] Marcos).
2. **Teto de 5 tentativas**, depois falha permanente ([09:15] Diego, [09:16] Larissa, [09:48] Larissa). Ver a nota de interpretação abaixo.
3. **Timeout de 10 segundos** por chamada HTTP; sem resposta nesse prazo, conta como falha e vai para retry ([09:42] Diego).
4. **DLQ em tabela separada `webhook_dead_letter`**, com payload, motivo da falha e timestamp, para manter a leitura da outbox limpa e servir de evidência para debug e reprocessamento ([09:18] Diego).
5. **Reprocessamento manual** por `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego, [09:35] Diego). Exige role `ADMIN` via `requireRole` existente e registra quem fez o replay ([09:36] Sofia, [09:36] Larissa).

**Nota de interpretação (pendente de confirmação).** A reunião diz "5 tentativas" e lista 5 intervalos, somando cerca de 14 h 36 min "entre primeira falha e última tentativa" ([09:17] Diego). As duas falas só fecham se os 5 intervalos forem as esperas entre o envio inicial e as 5 retentativas, ou seja, 1 envio inicial mais 5 retentativas. Este pacote adota essa leitura e a registra como questão em aberto no [RFC](../RFC.md#questões-em-aberto) para confirmação com Larissa e Diego.

## Alternativas consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| Retry indefinido com backoff | Evento pode ficar pendurado para sempre se o cliente sumiu ([09:15] Diego). |
| 3 tentativas | Com indisponibilidade de manhã, três tentativas em 30 minutos matariam o evento cedo demais ([09:16] Bruno, [09:16] Diego). |
| Marcar como `failed` na própria outbox em vez de tabela de DLQ | Polui a leitura da outbox principal; tabela separada fica mais limpa e serve de evidência ([09:17] Larissa, [09:18] Diego). |
| Notificar o cliente por e-mail quando as entregas falham | Fora de escopo desta fase; talvez numa próxima, depois de medir impacto ([09:37] Larissa). |

## Consequências

**Positivas**
- Cobre indisponibilidades comuns (inclusive manutenções de horas) sem intervenção manual.
- Nenhum evento é perdido silenciosamente: o que esgota tentativas fica na DLQ com o motivo.
- Replay auditável e restrito a administradores.

**Negativas / trade-offs**
- Um evento pode chegar com até ~15 h de atraso; a partir daí depende de ação humana.
- Progressão fixa não se adapta ao tipo de erro (um 4xx permanente também consome as 5 tentativas; a classificação de erros não foi discutida).
- Sem e-mail de alerta, o cliente só descobre falhas consultando o histórico de entregas.
- Trade-off aceito: limitar a janela de retry em troca de não acumular eventos eternamente pendentes.
