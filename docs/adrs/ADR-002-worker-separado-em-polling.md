# ADR-002: Worker em processo separado, lendo a outbox por polling

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira, 09:00 (registrada em `TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)

## Contexto

Com a outbox definida ([ADR-001](ADR-001-outbox-no-mysql.md)), falta decidir como os eventos saem da tabela e onde esse processamento roda. O requisito de produto é entregar em menos de 10 segundos, que é o que os clientes consideram "tempo real" ([09:02] Marcos). O MySQL não oferece notificação nativa para processos externos ([09:09] Diego). A API sobe hoje por `src/server.ts`, que cria o app, abre o `PrismaClient` de `src/config/database.ts` e trata `SIGINT`/`SIGTERM`.

## Decisão

1. **Polling em loop a cada 2 segundos**: o worker busca os eventos pendentes mais antigos em lote pequeno, processa e marca o resultado ([09:09] Diego, [09:08] Diego, [09:10] Larissa). Latência mínima de 2 segundos no pior caso, aceita explicitamente ([09:10] Larissa, [09:10] Marcos).
2. **Processo separado da API**: se a API reinicia, o worker não pode morrer junto ([09:11] Diego). Nova entry-point `src/worker.ts`, nos moldes de `src/server.ts`, com script `npm run worker` ([09:11] Larissa). A lógica de processamento fica dentro do módulo, em `src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts` ([09:28] Bruno).
3. **Mesmo banco, mesma stack, instância própria de PrismaClient**: mesma `DATABASE_URL`, mas um `PrismaClient` novo, porque cliente Prisma é por processo ([09:11] Bruno, [09:11] Diego, [09:30] Bruno).
4. **Instância única (single-worker)**: o worker processa em ordem de `created_at`, o que dá ordenação implícita por `order_id`. Não há garantia de ordenação global, e a garantia por pedido só vale enquanto houver um único worker; isso é limitação conhecida ([09:12] Diego, [09:13] Larissa, [09:14] Marcos).

## Alternativas consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| Trigger no banco para acionar o worker | Trigger do MySQL só executa SQL, não notifica processo externo; não existe equivalente ao `NOTIFY/LISTEN` do Postgres, e qualquer gambiarra (escrever arquivo, chamar endpoint) "fica esquisito" ([09:09] Bruno, [09:09] Diego). |
| Worker dentro do processo da API | Reinício da API derruba o worker ([09:11] Diego). |
| Compartilhar o PrismaClient da API | Impossível entre processos Node distintos ([09:30] Bruno). |
| Múltiplos workers em paralelo | Perde a ordenação por pedido; particionar por `order_id` ou lock pessimista ficam como "problema do futuro" ([09:12] Diego, [09:13] Diego). |

## Consequências

**Positivas**
- Implementação simples, sem dependência nova; 2 s atende com folga o limite de 10 s ([09:09] Diego).
- Deploy da API e do worker são independentes; uma queda de um não afeta o outro.
- Ordenação por pedido sem mecanismo extra, enquanto houver um worker.

**Negativas / trade-offs**
- Consultas periódicas ao banco mesmo sem eventos.
- Latência mínima de 2 s, contra quase zero de uma solução reativa.
- Escala horizontal bloqueada pela ordenação: aumentar vazão exige uma nova decisão (particionamento ou lock).
- Um novo processo para operar, monitorar e manter no ar.
- Trade-off aceito: reatividade e escala horizontal trocadas por simplicidade e ordenação por pedido.
