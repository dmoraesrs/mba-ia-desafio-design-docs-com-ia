# PRD: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Product Manager** | Marcos |
| **Tech Lead** | Larissa |
| **Status** | Aprovado na reunião técnica; em detalhamento |
| **Data** | 2026-11-19 |
| **Documentos relacionados** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) · [Tracker](TRACKER.md) |

## 1. Resumo e contexto

O OMS passa a avisar os sistemas dos clientes B2B, por HTTP, sempre que o status de um pedido deles muda. O cliente cadastra uma URL, escolhe quais status quer receber e passa a receber notificações assinadas, sem precisar consultar a plataforma periodicamente. A feature atende a um pedido formal de três clientes e é a primeira forma de notificação externa do OMS ([09:00] Marcos).

## 2. Problema e motivação

- **Integração lenta e cara.** Atlas Comercial, MaxDistribuição e Nova Cargo descobrem mudanças fazendo polling em `GET /orders` de tempos em tempos ([09:00] Marcos).
- **Risco comercial.** A Atlas sinalizou que pode migrar para um concorrente se a entrega não sair até o fim do trimestre ([09:00] Marcos), e pediu a feature para o fim de novembro ([09:45] Marcos).
- **Lacuna do produto.** O OMS não tem nenhum mecanismo de evento ou notificação de saída; o ciclo de vida do pedido existe (`src/modules/orders/order.status.ts`), mas é invisível para fora.

## 3. Público-alvo e cenários de uso

**Público**
- **Clientes B2B integradores** (inicialmente Atlas Comercial, MaxDistribuição e Nova Cargo), que consomem os eventos nos seus sistemas ([09:00] Marcos).
- **Usuários da plataforma que representam o cliente**, que cadastram e mantêm os webhooks pela API com JWT do OMS ([09:32] Marcos).
- **Administradores do OMS**, que reprocessam entregas que falharam definitivamente ([09:36] Sofia).

**Cenários**
1. *Acompanhar expedição.* A Nova Cargo quer saber só quando um pedido vira `SHIPPED` ou `DELIVERED`; cadastra um webhook com esses dois status e recebe apenas essas notificações ([09:33] Marcos).
2. *Cliente em manutenção.* O endpoint da Atlas fica duas horas fora do ar; as notificações são retentadas automaticamente e chegam quando o endpoint volta ([09:16] Diego).
3. *Investigar uma integração.* O time da MaxDistribuição consulta as últimas entregas para ver o que foi enviado, o que o servidor deles respondeu e quanto tempo levou ([09:34] Marcos).
4. *Secret vazada.* O cliente descobre que a secret apareceu num log; pede uma nova pela API e tem 24 horas para trocar nos seus sistemas sem perder notificações ([09:21] Sofia, [09:22] Diego).
5. *Reprocessar falha definitiva.* Após o cliente corrigir um problema longo, um administrador reenvia o evento que estava na fila de falhas ([09:18] Diego).

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta | Origem |
| --- | --- | --- | --- | --- |
| OBJ-01 | Notificação percebida como "tempo real" | Tempo entre a mudança de status e a entrega bem-sucedida (p95, com o cliente respondendo) | **< 10 segundos** | [09:02] Marcos, [09:10] Larissa |
| OBJ-02 | Reter e ativar os clientes que pediram a feature | Clientes com webhook ativo recebendo eventos | **3 de 3** (Atlas, MaxDistribuição, Nova Cargo) | [09:00] Marcos, [09:47] Marcos |
| OBJ-03 | Entregar no prazo combinado | Sprints até produção, com revisão de segurança incluída | **3 sprints** | [09:46] Larissa, [09:47] Larissa |
| OBJ-04 | Nenhuma notificação perdida | Eventos que não terminam entregues nem na fila de falhas | **0** | [09:15] Diego, [09:18] Diego |
| OBJ-05 | Consistência com o pedido | Mudanças de status (com webhook inscrito) sem evento correspondente | **0** | [09:40] Bruno |

Acompanhamento sem meta nesta fase: volume de polling dos três clientes em `GET /orders` e volume de eventos por cliente, que embasam as decisões futuras de e-mail de alerta e rate limiting ([09:37] Larissa, [09:39] Diego).

## 5. Escopo

### 5.1 Incluso
- Cadastro, listagem, edição e remoção de webhooks por cliente, com escolha de status.
- Notificação automática a cada mudança de status de pedido, para os webhooks inscritos naquele status.
- Assinatura verificável das notificações, secret por webhook e rotação de secret.
- Retentativa automática, fila de falhas definitivas e reprocessamento por administrador.
- Histórico das últimas entregas por webhook.

### 5.2 Fora de escopo

| Item | Situação | Origem |
| --- | --- | --- |
| **Notificação por e-mail** quando o webhook do cliente falha repetidamente | Adiado para a próxima fase, depois de medir o impacto | [09:37] Larissa, [09:38] Marcos |
| **Dashboard visual** para o cliente ver seus webhooks | Descartado nesta fase; projeto separado do time de frontend | [09:39] Marcos, [09:40] Larissa |
| **Rate limiting** de envio por cliente | Não entra; observar em produção e decidir depois | [09:38] Diego, [09:39] Larissa |
| **Webhooks de entrada** (cliente enviando para a plataforma) | Descartado: o fluxo é só de saída | [09:02] Marcos, [09:03] Sofia |
| **Garantia de ordem global** entre pedidos e **escala para múltiplos workers** | Não entra; ordem garantida só por pedido, com um único worker | [09:13] Larissa, [09:14] Marcos |
| **Garantia de entrega exatamente uma vez** | Descartado; o cliente deduplica | [09:25] Diego |
| **Arquivamento** de notificações antigas | Fora desta feature | [09:08] Diego |
| **Permissões mais restritas** para o cadastro de webhooks | Adiado ("mais pra frente") | [09:37] Sofia |
| **Itens do pedido** na notificação | Descartado; detalhes via `GET /orders/:id` | [09:43] Diego |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| RF-01 | O usuário autenticado cadastra um webhook informando o cliente, a URL de destino e a lista de status que deseja receber; o cliente é informado na requisição, não deduzido do login. | [09:31] Marcos, [09:32] Larissa |
| RF-02 | A plataforma gera a secret do webhook e a devolve na resposta do cadastro. | [09:31] Marcos |
| RF-03 | O usuário lista os webhooks de um cliente. | [09:33] Bruno |
| RF-04 | O usuário edita um webhook (URL, status assinados, ativo/inativo). | [09:33] Bruno |
| RF-05 | O usuário remove um webhook. | [09:33] Bruno |
| RF-06 | A cada mudança de status de pedido, a plataforma notifica os webhooks ativos do cliente do pedido que assinam o novo status; webhooks que não assinam aquele status não recebem nada. | [09:33] Marcos, [09:34] Bruno, [09:40] Bruno |
| RF-07 | A notificação informa o identificador do evento, o tipo, o momento, o pedido (id, número, total), o cliente e os status de origem e destino, sem os itens. | [09:43] Diego |
| RF-08 | A notificação reflete o pedido no momento da mudança de status, mesmo que ele mude depois. | [09:52] Larissa |
| RF-09 | Cada notificação é assinada com a secret do webhook, para o cliente verificar origem e integridade. | [09:19] Sofia, [09:22] Sofia |
| RF-10 | Cada notificação traz um identificador único de evento, para o cliente descartar duplicatas, e o identificador do webhook cadastrado. | [09:25] Diego, [09:44] Sofia |
| RF-11 | O usuário solicita uma nova secret pela API; a anterior continua válida por 24 horas. | [09:21] Sofia |
| RF-12 | Notificações não entregues são retentadas automaticamente em intervalos crescentes (1 min, 5 min, 30 min, 2 h, 12 h). | [09:15] Diego, [09:17] Larissa |
| RF-13 | Notificações que esgotam as tentativas ficam registradas numa fila de falhas, com conteúdo e motivo. | [09:18] Diego |
| RF-14 | Um administrador reenvia uma notificação da fila de falhas; usuários sem perfil de administrador não podem; quem reenviou fica registrado. | [09:18] Diego, [09:36] Sofia, [09:36] Larissa |
| RF-15 | O usuário consulta as últimas 100 entregas de um webhook, com sucesso ou falha, conteúdo enviado, resposta recebida e tempo de resposta. | [09:34] Marcos |
| RF-16 | URLs que não usam HTTPS são recusadas no cadastro e na edição. | [09:23] Sofia |

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| RNF-01 | **Latência**: entrega em menos de 10 s; atraso mínimo aceito de até 2 s pelo ciclo de leitura. | [09:02] Marcos, [09:10] Larissa |
| RNF-02 | **Isolamento**: mudanças de status nunca esperam nem falham por causa do endpoint do cliente. | [09:04] Bruno |
| RNF-03 | **Consistência**: se o status mudou, a notificação existe; se a mudança foi desfeita, a notificação não existe. | [09:06] Diego, [09:40] Bruno |
| RNF-04 | **Entrega pelo menos uma vez**: o cliente pode receber a mesma notificação mais de uma vez e deve deduplicar. | [09:24] Diego |
| RNF-05 | **Timeout** de 10 s por tentativa de entrega. | [09:42] Diego |
| RNF-06 | **Tamanho**: notificação limitada a 64 KB; acima disso, não é enviada e gera erro. | [09:24] Larissa |
| RNF-07 | **Segurança**: HTTPS obrigatório; HMAC-SHA256; secret única por webhook, nunca global. | [09:20] Sofia, [09:21] Sofia, [09:23] Sofia |
| RNF-08 | **Ordem**: garantida apenas por pedido e enquanto houver um único processador. | [09:13] Larissa |
| RNF-09 | **Operação**: o envio roda em processo separado da API, para que reinícios de um não afetem o outro. | [09:11] Diego |
| RNF-10 | **Sem infraestrutura nova** e com os padrões atuais do projeto (módulos, erros, logs). | [09:07] Diego, [09:30] Larissa |
| RNF-11 | **Auditoria**: reenvios manuais registram o usuário responsável. | [09:36] Sofia |

## 8. Decisões e trade-offs principais

| Decisão | Ganho | Custo aceito | ADR |
| --- | --- | --- | --- |
| Registrar o evento no próprio banco, junto da mudança de status | Consistência total e nada de infraestrutura nova | Atraso mínimo de 2 s; banco usado como fila | [ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| Retentar por ~15 h e depois parar | Cobre manutenções longas sem eventos pendurados para sempre | Após 15 h, depende de reenvio manual | [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| Secret por webhook com rotação | Vazamento isolado e trocável sem parada | Mais gestão de secrets | [ADR-004](adrs/ADR-004-assinatura-hmac-sha256-com-secret-por-endpoint.md) |
| Entrega pelo menos uma vez | Robustez e simplicidade; padrão de mercado | Cliente precisa deduplicar ([09:25] Sofia) | [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| Reaproveitar padrões do projeto | Entrega mais rápida e previsível | Sem ferramentas especializadas | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) |
| Notificação enxuta e congelada no momento da mudança | Fiel ao que aconteceu; leve | Cliente busca itens em outra chamada | [ADR-007](adrs/ADR-007-payload-enxuto-com-snapshot-na-insercao.md) |

## 9. Dependências

| Dependência | Tipo | Origem |
| --- | --- | --- |
| Revisão de segurança da Sofia (HMAC e geração de secret), com pelo menos 2 dias úteis antes do deploy | Pessoas / prazo | [09:46] Sofia, [09:49] Sofia |
| Sessão de revisão do design com Bruno e Diego antes de começar a codar | Processo | [09:50] Larissa |
| Documentação de integração no portal de desenvolvedor, com destaque para deduplicação | Produto (Marcos) | [09:26] Marcos, [09:40] Marcos |
| Confirmação do prazo com a Atlas e atualização dos clientes | Comercial (Marcos) | [09:47] Marcos, [09:49] Marcos |
| Fluxo de mudança de status existente (`src/modules/orders/order.service.ts`) e usuários com perfil `ADMIN` (`prisma/schema.prisma`) | Sistema | [09:40] Bruno, [09:36] Larissa |
| Novo processo de envio implantado e mantido no ar pela operação | Infraestrutura | [09:11] Larissa |

## 10. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Atraso na entrega leva a Atlas a migrar para o concorrente | Média | Alto | Escopo enxuto (e-mail, dashboard e rate limiting fora); estimativa de 3 sprints com revisão de segurança já incluída; Marcos confirma prazo com a Atlas ([09:47] Larissa, [09:47] Marcos) |
| Cliente não deduplica e processa a mesma notificação duas vezes | Média | Médio | Identificador de evento em toda notificação; documentação destacada no portal ([09:26] Marcos) |
| Vazamento de secret por parte do cliente | Média (já ocorreu, [09:22] Diego) | Alto | Secret por webhook, rotação com 24 h de convivência, revisão de segurança |
| Cliente fora do ar por mais de ~15 h perde notificações para a fila de falhas | Baixa | Médio | Fila de falhas com reenvio por administrador; histórico de entregas visível ao cliente; e-mail de alerta avaliado na próxima fase |
| Rajadas de notificações sobrecarregam endpoints de clientes | Média | Médio | Medir volume por cliente e decidir rate limiting com dados ([09:39] Diego) |
| Qualquer usuário autenticado pode configurar webhooks de qualquer cliente | Média | Alto | Aceito nesta fase ([09:37] Sofia); restringir em fase posterior |

## 11. Critérios de aceitação

1. Os três clientes conseguem cadastrar, listar, editar e remover webhooks pela API (RF-01 a RF-05).
2. Uma mudança de status de pedido gera notificação apenas para os webhooks do cliente que assinam aquele status, entregue em menos de 10 s com o endpoint respondendo (RF-06, RNF-01).
3. O cliente valida a assinatura de uma notificação recebida usando a secret devolvida no cadastro (RF-02, RF-09).
4. Com o endpoint do cliente fora do ar, a notificação é retentada na progressão definida e, esgotadas as tentativas, aparece na fila de falhas; um administrador consegue reenviá-la e um operador não (RF-12 a RF-14).
5. O histórico mostra as últimas entregas com resultado, conteúdo, resposta e tempo (RF-15).
6. Cadastro com URL `http://` é recusado (RF-16).
7. Uma mudança de status que falha não gera notificação, e nenhuma mudança concluída fica sem notificação (RNF-03).
8. A revisão de segurança da Sofia foi concluída antes do deploy.

Os critérios técnicos verificáveis estão em [FDD, seção 12](FDD.md#12-critérios-de-aceite-técnicos).

## 12. Estratégia de testes e validação

| Camada | O que validar | Como |
| --- | --- | --- |
| Unitária | Assinatura, cálculo do próximo intervalo de retry, montagem da notificação, filtro por status | Vitest, como já configurado em `vitest.config.ts` |
| Integração (API + banco) | Cadastro e validações; geração da notificação junto da mudança de status e ausência dela quando a mudança falha; permissão de administrador no reenvio | Vitest + Supertest com MySQL, no padrão de `tests/orders.test.ts` |
| Ponta a ponta | Processo de envio contra um receptor HTTP de teste: entrega, timeout, retentativas, fila de falhas, reenvio | Receptor local controlado pelo teste |
| Segurança | HMAC, geração e rotação de secret, ausência de secrets em logs | Revisão dedicada da Sofia, 2 dias úteis antes do deploy ([09:46] Sofia) |
| Regressão | Fluxo de pedidos existente continua igual | Suíte atual (`npm test`) |
| Validação de produto | Métricas da seção 4 após a entrada em produção | Acompanhamento dos indicadores de latência e entrega definidos no [FDD](FDD.md#9-observabilidade) |
