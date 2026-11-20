# Da Reunião ao Documento: Design Docs do Sistema de Webhooks

Pacote de design docs do **Sistema de Webhooks de Notificação de Pedidos** do OMS, produzido com IA a partir da transcrição da reunião técnica (`TRANSCRICAO.md`) e do código existente. O enunciado original está no [repositório base do desafio](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

## Sobre o desafio

Uma empresa com um Order Management System em produção decidiu, numa reunião de ~55 minutos entre tech lead, PM, dois engenheiros e a engenheira de segurança, construir webhooks de saída para avisar clientes B2B quando o status dos seus pedidos muda. Nada foi registrado além da transcrição. A tarefa foi transformar essa conversa, junto com a leitura do código Node.js + TypeScript + Prisma, num pacote de documentos acionável: PRD, RFC, FDD, ADRs e um tracker que liga cada afirmação à sua origem.

A dificuldade não estava em escrever, e sim em filtrar: separar o que foi decidido do que foi descartado (dashboard, e-mail de alerta, exactly-once) ou adiado (rate limiting, múltiplos workers), não inventar o que a reunião deixou em aberto e manter cada documento na sua altura, sem repetir o FDD dentro do RFC.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Claude Code** (modelo Claude Opus 5.5) | Agente principal: leu a transcrição e o código, mapeou os pontos de integração, redigiu ADRs, RFC, FDD, PRD, tracker e este README, e fez commits pequenos a cada etapa. |
| **Subagente de revisão do Claude Code** (instância separada, sem acesso ao raciocínio do agente principal) | Revisão crítica independente dos documentos contra a transcrição e o código, para não confirmar as próprias hipóteses do agente principal. |
| **Scripts Python escritos pela IA** (fora do repositório, em diretório temporário) | Verificação mecânica: timestamps e falantes, caminhos de arquivo, links e âncoras, travessões, contagens do tracker e cobertura. |

## Workflow adotado

1. **Exploração.** Leitura integral de `TRANSCRICAO.md` e dos pontos do código citados na reunião: `src/modules/orders/order.service.ts` (transação de `changeStatus`), `src/modules/orders/order.status.ts`, `src/shared/errors/`, `src/middlewares/` (auth, error, validate, request logger), `src/shared/logger/index.ts`, `src/server.ts`, `src/app.ts`, `src/routes/index.ts`, `src/config/`, `prisma/schema.prisma`, `package.json` e `tests/`. Saída: um mapa mental de "decidido / descartado / adiado / não discutido" com timestamp de cada item.
2. **ADRs primeiro** (7 arquivos). As seis decisões principais da reunião mais uma secundária (payload snapshot), porque ela muda o schema da outbox e o contrato com o cliente.
3. **RFC** em cima dos ADRs: proposta curta, 7 alternativas descartadas e as questões em aberto.
4. **FDD**: modelo de dados, fluxos, contratos, matriz de erros, observabilidade e a seção "Integração com o sistema existente". Ao detalhar, surgiram duas questões novas que voltaram para o RFC (Q-10 e Q-11).
5. **PRD** por último entre os grandes documentos, como consolidação em linguagem de produto.
6. **Revisão crítica independente** por um subagente (prompt 2 abaixo) e correção de cada ponto aceito.
7. **Tracker** montado varrendo os documentos prontos, com IDs alinhados aos IDs dos documentos (`RF-01` ↔ `PRD-FR-01`) para a cobertura ser medida por script.
8. **Verificação mecânica** com scripts e correção do que falhou.
9. **README** no fim.

Convenções adotadas em todos os documentos:
- Toda afirmação relevante cita `[hh:mm] Nome` ou um caminho de arquivo.
- O que a reunião não fechou, mas o FDD precisava definir para ser acionável (caminho do endpoint de rotação, codificação da assinatura, critério de sucesso 2xx, tamanho do lote), aparece marcado **(proposta)** e ligado à regra discutida da qual deriva.
- Ambiguidades da reunião viram questões no RFC, não decisões silenciosas.

## Prompts customizados

**Prompt 1: orquestração da tarefa** (prompt de entrada que dirigiu o agente principal; trecho das regras específicas)

```text
Entregue o desafio inteiro, cumprindo cada critério de aceite.

1. Entrega puramente documental: NÃO altere src/, prisma/, tests/, TRANSCRICAO.md nem
   arquivos de configuração. Nada de instalar dependências: só leia o código.
2. Leia a TRANSCRICAO.md inteira e o código relevante (módulo de pedidos, máquina de
   estados, auditoria, classes de erro, prisma/schema.prisma) antes de escrever.
3. Tudo rastreável: cada requisito, decisão, restrição e alternativa precisa de origem
   na transcrição ([hh:mm] Nome, timestamp real) ou num caminho de arquivo que existe.
   Não invente. O que foi descartado ou adiado na reunião NÃO vira requisito: vai para
   "Fora de escopo" ou "Questões em aberto".
4. Ordem: explorar → ADRs → RFC → FDD → PRD → TRACKER → README. Itere: depois do
   primeiro rascunho, faça uma revisão crítica de cada documento contra a transcrição e
   o código (redundância entre documentos, itens sem origem, timestamps errados) e corrija.
5. Antes de concluir, verifique mecanicamente: (a) todo caminho de arquivo citado nos
   docs existe; (b) todo timestamp citado existe na TRANSCRICAO.md com aquele falante;
   (c) contagens mínimas: >= 8 RF no PRD, >= 4 endpoints com request/response no FDD,
   erros WEBHOOK_*, >= 4 caminhos reais na seção "Integração com o sistema existente",
   tracker com >= 70% de linhas TRANSCRICAO e >= 5 CODIGO, cobertura >= 80%.
```

**Prompt 2: revisão crítica independente** (escrito pelo agente principal para o subagente revisor, que recebeu só o código e os documentos, sem as conclusões do autor)

```text
Você é um revisor técnico cético. NÃO edite nenhum arquivo; apenas leia e reporte.

Fonte da verdade: TRANSCRICAO.md e o código em src/, prisma/schema.prisma, tests/, package.json.
Documentos a revisar: docs/PRD.md, docs/RFC.md, docs/FDD.md, docs/adrs/ADR-*.md.

Leia a TRANSCRICAO.md inteira primeiro. Depois verifique, documento por documento:
1. Atribuição semântica: para cada citação [hh:mm] Nome, a fala daquele falante naquele
   minuto realmente sustenta a afirmação ao lado?
2. Invenção: requisitos, decisões, números ou restrições apresentados como decididos que
   não têm origem na transcrição nem no código. Itens marcados como "(proposta)" são
   aceitáveis, desde que não sejam apresentados como decisão da reunião.
3. Escopo: algo descartado ou adiado na reunião aparecendo como requisito.
4. Contradições: entre documentos (números, endpoints, códigos de erro, tabelas, contagem
   de tentativas) e entre documentos e código (nomes de funções, classes, caminhos,
   comportamento descrito de NotFoundError, validate.middleware, changeStatus).
5. Redundância de altura: trechos do RFC ou do PRD que repetem detalhe do FDD.
6. Qualquer caractere travessão (U+2014 ou U+2013).

Formato: lista numerada com arquivo:linha, categoria, problema em uma frase e correção
sugerida, do mais grave ao menos grave, no máximo 25 itens. Só reporte o que confirmou
lendo os arquivos. Se uma categoria não tiver problemas, diga explicitamente.
```

## Iterações e ajustes

Foram **4 iterações principais**: rascunho guiado por ADRs; ajustes durante o detalhamento do FDD; revisão crítica independente com correções; verificação mecânica final. Os ajustes concretos mais importantes:

1. **Contagem de tentativas contraditória.** Ao escrever o ADR-003, a conta não fechou: a reunião diz "5 tentativas", mas lista 5 intervalos (1m/5m/30m/2h/12h) que somam ~14 h 36 min "entre primeira falha e última tentativa" ([09:17] Diego). Isso só é coerente com 1 envio + 5 retentativas. O primeiro rascunho registrou a leitura numa nota, mas manteve "Teto de 5 tentativas" como decisão aceita. A revisão independente apontou a contradição entre o item e a nota; a correção explicitou a leitura adotada, marcou-a como pendente e concentrou a justificativa numa só questão (RFC Q-04), em vez de repeti-la em três documentos.
2. **`WEBHOOK_INVALID_URL` contra `VALIDATION_ERROR`.** Sofia disse que a regra de https "é só uma validação no schema Zod" ([09:23] Sofia), e Bruno quer o código `WEBHOOK_INVALID_URL` ([09:28] Bruno). Lendo `src/middlewares/validate.middleware.ts`, a IA confirmou que toda falha de schema vira `VALIDATION_ERROR`, de modo que as duas falas não podem ser atendidas literalmente juntas. O FDD passou a declarar a regra num schema Zod aplicado pelo service (marcado como proposta), e a revisão pegou que o ADR-004 ainda dizia o contrário; foi alinhado.
3. **Questões que só apareceram no detalhamento.** Ao escrever os fluxos do worker, ficou claro que um evento em backoff não bloqueia o evento seguinte do mesmo pedido: a "ordem por pedido" prometida na reunião não vale durante retry. Ao ler `OrderService.create`, apareceu que a criação do pedido grava histórico `null → PENDING`, mas a reunião só falou de `changeStatus`. As duas viraram Q-10 e Q-11 no RFC (commit separado), e depois da revisão a ressalva de ordem foi propagada para PRD, ADR-002 e RFC.
4. **Derivações a partir do código.** A máquina de estados (`src/modules/orders/order.status.ts`) não tem nenhuma transição com destino `PENDING`, então um filtro com `PENDING` nunca dispararia, e o FDD exclui esse valor. `tests/setup.ts` apaga `customer` no `beforeEach`; com a FK nova de `webhook_endpoints`, a limpeza precisa vir antes. `NotFoundError` fixa o código `NOT_FOUND`, então `WEBHOOK_NOT_FOUND` precisa estender `AppError` diretamente.
5. **Revisão independente: 22 apontamentos, 20 aceitos.** Entre os aceitos:
   - citação errada: o ADR-002 atribuía a "[09:11] Bruno" a ideia de PrismaClient separado, mas nesse minuto Bruno disse "usar o mesmo Prisma client" (a ideia é dele só às [09:30]);
   - o FDD prometia `requestId` no log de enfileiramento, o que a assinatura de `changeStatus(id, input, userId)` não permite;
   - âncoras quebradas para o RFC e para a seção de riscos do FDD;
   - "p95" e "sem jitter" apresentados como decididos, embora não tenham aparecido na reunião;
   - cenários do PRD que atribuíam falas a clientes que não foram nomeados nelas;
   - códigos de erro além dos três citados por Bruno sem a marcação de proposta;
   - riscos duplicados entre PRD e FDD;
   - detalhe técnico (Vitest, caminhos) no PRD e diagrama de implementação no RFC.

   Rejeitados, com motivo: (a) tirar do RFC os itens adiados (e-mail, permissões do CRUD), porque o enunciado define questões em aberto como pontos "não decididos **ou adiados**"; (b) enxugar a tabela de padrões do ADR-006, porque o critério exige ADR que referencie explicitamente o código, e essa tabela é o contexto da decisão.
6. **Verificação mecânica.** O primeiro cálculo de cobertura contou as variáveis de ambiente `WEBHOOK_POLL_INTERVAL_MS` e similares como se fossem códigos de erro sem linha no tracker. O script foi corrigido para considerar só códigos de erro. Um link checker foi criado depois da revisão ter encontrado âncoras quebradas que o checker de caminhos não via.

## Como navegar a entrega

| Ordem | Arquivo | Para quê |
| --- | --- | --- |
| 1 | [`docs/PRD.md`](docs/PRD.md) | Por que e o quê: problema, público, metas, escopo, requisitos |
| 2 | [`docs/RFC.md`](docs/RFC.md) | Proposta técnica, alternativas descartadas e questões em aberto |
| 3 | [`docs/adrs/`](docs/adrs/README.md) | Uma decisão por arquivo (ADR-001 a ADR-007), com alternativas e consequências |
| 4 | [`docs/FDD.md`](docs/FDD.md) | Como construir: modelo de dados, fluxos, contratos, erros, observabilidade, integração com o código |
| 5 | [`docs/TRACKER.md`](docs/TRACKER.md) | Origem de cada item: timestamp e falante, ou caminho de arquivo |

Para quem vai implementar, o caminho mais curto é RFC → FDD, consultando os ADRs quando precisar do porquê de uma decisão. A transcrição original continua em [`TRANSCRICAO.md`](TRANSCRICAO.md); o código da aplicação (`src/`, `prisma/`, `tests/`) não foi alterado.
