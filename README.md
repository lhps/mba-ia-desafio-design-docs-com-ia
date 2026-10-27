# Da Reunião ao Documento: Webhooks de Notificação de Pedidos

> O enunciado original do desafio está no [repositório base](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia#readme). Este README documenta **como** o pacote de design docs foi produzido.

## Sobre o desafio

Uma empresa que opera um Order Management System (Node.js + TypeScript + Prisma/MySQL) decidiu, em uma reunião de cerca de 55 minutos, construir um **sistema de webhooks de saída** para avisar clientes B2B quando o status dos pedidos deles muda. A única coisa registrada da reunião foi a transcrição (`TRANSCRICAO.md`). A tarefa foi transformar essa conversa, junto com o código existente, em um pacote de documentação técnica: PRD, RFC, FDD, ADRs e um tracker de rastreabilidade, detalhado o bastante para o time começar a implementar.

A dificuldade não era escrever muito, e sim escrever **só o que tem origem**. Era preciso separar o que foi decidido do que foi descartado ou adiado, e o que foi dito do que é inferência. Também era preciso confrontar a conversa com o código real, porque em alguns pontos as duas coisas não fecham.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|---|---|
| **Claude Code (Claude Opus 5.5, CLI)** | Ferramenta principal. Leu o repositório e a transcrição, extraiu decisões, requisitos e itens fora de escopo, e redigiu todos os documentos direto nos arquivos do repositório. |
| **Subagente de revisão adversarial (Claude Code, agente `general-purpose`)** | Uma segunda instância, sem o contexto de quem escreveu, auditou os documentos contra a transcrição e o código procurando citações erradas, decisões inventadas e contradições. |
| **Scripts Python de validação (escritos pela IA)** | Checagem mecânica do tracker: formato, percentual de linhas por fonte, se cada `[hh:mm] Nome` corresponde a uma fala real, se cada caminho de código existe, e cobertura dos IDs dos documentos. |

## Workflow adotado

1. **Contextualização.** A IA leu a transcrição inteira e os arquivos centrais do código: `prisma/schema.prisma`, `src/modules/orders/*`, `src/shared/errors/*`, `src/middlewares/*`, `src/app.ts`, `src/routes/index.ts`, `src/server.ts`, `src/config/*`, `src/shared/logger/index.ts`, os testes e o `package.json`. Antes de escrever qualquer documento, listei os achados que mudariam o conteúdo (ver "Iterações").
2. **Extração dirigida.** A transcrição foi classificada em decidido, descartado, adiado, em aberto e detalhe secundário, sempre com `[hh:mm] Nome`. O resumo da Larissa em [09:48] serviu de checklist das decisões fechadas, e a conversa pós-call ([09:50]–[09:53]) foi tratada como fonte de duas decisões extras (UUID e snapshot).
3. **ADRs primeiro.** Sete ADRs: as seis decisões principais do enunciado e mais uma decisão secundária com consequências reais (snapshot do payload e filtro na inserção).
4. **RFC.** Proposta em nível de arquitetura, com seis alternativas descartadas e oito questões em aberto. O detalhe de implementação ficou fora dele.
5. **FDD.** Modelo de dados, fluxos (outbox → worker → retry → DLQ → replay), sete endpoints com exemplos, contrato da notificação, matriz de erros `WEBHOOK_*`, observabilidade e a seção de integração com 16 arquivos reais. As decisões que a reunião **não** tomou ficaram marcadas como "(proposta)" e consolidadas no §15.
6. **PRD por último** entre os documentos grandes, consolidando problema, público, métricas, escopo e requisitos.
7. **Revisão adversarial** por um subagente independente e correção de tudo que procedia.
8. **Tracker e validação mecânica** por script, com os números registrados no próprio `TRACKER.md`.
9. **Checklist final** contra os critérios de aceite do enunciado.

## Prompts customizados

**Prompt 1: extração dirigida da transcrição.** Usado para classificar a reunião antes de gerar qualquer documento. A ideia é forçar a separação entre decidido, descartado e em aberto, em vez de pedir "um PRD a partir da transcrição".

```text
Leia TRANSCRICAO.md inteira. NÃO escreva nenhum documento ainda.
Classifique cada fala relevante em exatamente uma categoria:
  DECIDIDO | DESCARTADO | ADIADO/FUTURO | EM ABERTO (ninguém fechou) | DETALHE SECUNDÁRIO
Para cada item: [hh:mm] Nome, resumo de 1 linha, e quem fechou a decisão.
Regras:
- Uma pergunta não é um requisito. Registre quem RESPONDEU/decidiu, não quem perguntou.
- Se uma fala posterior contradiz uma anterior (ex.: customer_id "do JWT" vs "body ou path"),
  registre as duas e marque a mais recente como vigente.
- Se números não fecham entre si (ex.: quantidade de tentativas vs. soma dos intervalos),
  aponte a ambiguidade em vez de escolher em silêncio.
- Depois, cruze com o código: para cada decisão, diga qual arquivo/função ela toca
  e se algo no código CONTRADIZ ou COMPLICA a decisão (ex.: middleware que reescreve códigos de erro).
```

**Prompt 2: revisão adversarial (usado como foi enviado ao subagente).**

```text
You are an adversarial reviewer for a design-docs package. READ-ONLY: do not edit any file.
Source of truth: TRANSCRICAO.md and the code under src/, prisma/, tests/, package.json.
Documents to audit: docs/PRD.md, docs/RFC.md, docs/FDD.md, docs/adrs/ADR-00*.md.
Rule: every requirement, decision or constraint must be traceable to the transcript or the code;
nothing invented; nothing discarded/postponed may appear as a requirement; docs must not contradict
transcript or code; every code path cited must exist. Items labelled "(proposta)" or listed in
FDD §15 / RFC open questions are allowed — only report them if they contradict the transcript.
Check: 1) every "[hh:mm] Nome" citation — does that speaker say that at that timestamp?
2) claims presented as decided that were NOT decided; 3) contradictions with transcript numbers;
4) contradictions with the code (class/function names, middleware behavior, paths);
5) out-of-scope items leaking in as requirements; 6) duplication across docs (RFC vs FDD altitude).
Output: severity | file:line | problem | suggested fix. Prioritize real errors over style.
```

**Prompt 3: geração de ADR com fronteira de altura.**

```text
Escreva o ADR-00N em MADR (Status, Contexto, Decisão, Alternativas Consideradas, Consequências
positivas/negativas + "Trade-off explícito" em 1 frase). Cada afirmação cita [hh:mm] Nome ou um
caminho real do repositório. Alternativas: só as que foram faladas na reunião; se usar uma alternativa
"plausível" não discutida, rotule-a assim. Não descreva endpoints, payloads de exemplo ou SQL: isso é FDD.
Se a decisão tem um ponto que a reunião não fechou, registre como "ponto não decidido" e aponte para
a questão correspondente no RFC, em vez de decidir por conta própria.
```

## Iterações e ajustes

Foram **4 ciclos principais**: (1) contextualização e extração, (2) geração na ordem ADR → RFC → FDD → PRD com autocorreção, (3) revisão adversarial e correções, (4) validação mecânica do tracker e checklist final. Os ajustes concretos foram estes:

1. **"5 tentativas" é ambíguo.** A primeira leitura era 5 envios no total. Ao cruzar os números, isso não fechava: a agenda 1m/5m/30m/2h/12h tem 5 intervalos e soma "quase 15 horas entre a primeira falha e a última tentativa" ([09:17] Diego), e o contra-exemplo "3 retentaria três vezes em 30 minutos" ([09:16] Diego) só funciona contando **retentativas**. Por outro lado, o resumo diz "total 5 tentativas" ([09:48] Larissa). Em vez de escolher em silêncio, a documentação adota "envio inicial + 5 retentativas" e abre a confirmação como RFC Q-04. O PRD, o FDD e os critérios de aceite sinalizam a interpretação.
2. **O `customer_id` mudou durante a reunião.** O Marcos disse "implícito do JWT" ([09:31]), mas o Bruno e a Larissa corrigiram, porque o JWT é do operador, e a Larissa deixou "body ou path" ([09:32]). Um primeiro rascunho usaria o JWT. A versão final trata o ponto como questão em aberto (RFC Q-02), e o FDD traz a proposta como "(proposta)".
3. **Conflito entre a reunião e o código na validação de https.** A Sofia pediu a regra "no schema Zod" ([09:23]), e o Bruno pediu códigos `WEBHOOK_*` ([09:28]). Lendo `src/middlewares/validate.middleware.ts`, a IA viu que todo `ZodError` vira `VALIDATION_ERROR`, então as duas coisas não convivem. A primeira versão do FDD resolveu movendo a regra para o service sem destacar a divergência. Depois da revisão, ficou explícito que isso **contraria a Sofia** e depende do aceite dela, e o critério de aceite passou a ser condicional.
4. **Código do erro de 404.** `NotFoundError` (`src/shared/errors/http-errors.ts`) sempre emite `NOT_FOUND` e não aceita código. Por isso o FDD especifica que `WebhookNotFoundError` estende `AppError` diretamente. Esse detalhe só apareceu por causa da leitura do código.
5. **Invenções cortadas.** Na autocorreção, removi do PRD um "piloto com a Atlas" que ninguém mencionou. A revisão adversarial encontrou mais:
   - uma **citação errada**: o ADR-002 atribuía a [09:11] Bruno a ideia de um `PrismaClient` separado, quando ele disse o contrário naquele minuto e só esclareceu em [09:30];
   - o código `WEBHOOK_OUTBOX_INSERT_FAILED` apresentado como contrato fechado sem ter sido discutido, o que virou proposta P-14;
   - um argumento contra Redis (perda de atomicidade) que não saiu da reunião, agora rotulado como "argumento adicional";
   - "2xx = sucesso" e "replay mantém o mesmo `event_id`", agora marcados como pressuposto e inferência;
   - a ordem dos passos de `changeStatus` no ADR-001, que estava diferente do código;
   - o histórico de entregas devolvendo `payload` sem relação no modelo de dados, o que levou à inclusão da relação `WebhookDelivery → WebhookOutbox`;
   - detalhe de FDD (Vitest, Supertest, arquivos de teste) no PRD, que foi enxugado.
6. **Arquivos citados que não existem.** O script de validação apontou `src/worker.ts` e `src/modules/webhooks/*` como inexistentes. Eles vêm da própria reunião ([09:11], [09:28]) e são arquivos **a criar**, então o FDD ganhou uma lista explícita de "arquivos a criar" para não confundir com referências quebradas.
7. **Achado do código que virou questão.** Em `src/modules/orders/order.status.ts`, nenhuma transição leva a `PENDING`, e a criação do pedido não passa por `changeStatus`. Assinar `PENDING` nunca dispararia nada. Isso virou RFC Q-07 e a proposta P-10.

## Como navegar a entrega

Ordem sugerida de leitura:

| # | Arquivo | Para quê |
|---|---|---|
| 1 | [`docs/PRD.md`](docs/PRD.md) | Por quê e o quê: problema, público, métricas, escopo, 16 requisitos funcionais, riscos |
| 2 | [`docs/RFC.md`](docs/RFC.md) | Proposta técnica para revisão: abordagem, alternativas descartadas, questões em aberto |
| 3 | [`docs/adrs/`](docs/adrs/README.md) | Uma decisão por arquivo: |
|   | [ADR-001](docs/adrs/ADR-001-outbox-no-mysql.md) | Outbox no MySQL |
|   | [ADR-002](docs/adrs/ADR-002-worker-separado-em-polling.md) | Worker separado, polling de 2 s |
|   | [ADR-003](docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) | Retry com backoff e DLQ |
|   | [ADR-004](docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256, secret por endpoint, rotação |
|   | [ADR-005](docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) | At-least-once com `X-Event-Id` |
|   | [ADR-006](docs/adrs/ADR-006-reuso-dos-padroes-existentes.md) | Reuso dos padrões do projeto |
|   | [ADR-007](docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md) | Snapshot do payload e filtro na inserção |
| 4 | [`docs/FDD.md`](docs/FDD.md) | Como construir: dados, fluxos, contratos, erros, observabilidade, integração com o código, propostas pendentes (§15) |
| 5 | [`docs/TRACKER.md`](docs/TRACKER.md) | De onde veio cada item: 298 linhas com origem na transcrição ou no código |

A fonte usada é a [`TRANSCRICAO.md`](TRANSCRICAO.md), que não foi alterada. O código da aplicação (`src/`, `prisma/`, `tests/`) também ficou intocado: a entrega é só documental.
