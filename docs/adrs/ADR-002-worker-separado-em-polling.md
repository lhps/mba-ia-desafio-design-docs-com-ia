# ADR-002 — Worker em processo separado, lendo a outbox por polling de 2 segundos

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira ([09:10] Larissa: "Worker em polling, 2s. [...] Aceitamos.")
- **Decisores:** Larissa, Diego, Bruno, Marcos
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)

## Contexto

Com a outbox definida ([ADR-001](ADR-001-outbox-no-mysql.md)), alguém precisa ler os eventos pendentes e fazer as chamadas HTTP. Requisitos que moldam a decisão:

- **Latência:** para os clientes, "tempo real" é qualquer coisa **abaixo de 10 segundos** ([09:02] Marcos).
- **Banco:** MySQL 8 (`docker-compose.yml`, `prisma/schema.prisma`). Ele não tem mecanismo nativo de notificação para processos externos ([09:09] Diego).
- **Disponibilidade:** o worker não pode morrer junto com a API: "se a API reinicia, perde o worker" ([09:11] Diego).
- **Ordenação:** a Larissa perguntou se eventos do mesmo pedido em sequência rápida (PAID → PROCESSING → SHIPPED) chegariam em ordem ([09:12] Larissa). A resposta, que virou a decisão, é do Diego ([09:12] Diego).

## Decisão

1. **Polling em loop a cada 2 segundos.** A cada ciclo, o worker busca os eventos pendentes mais antigos (por `created_at`), processa e marca o resultado ([09:09] Diego). A latência de captura no pior caso é de ~2 s, abaixo do teto de 10 s ([09:10] Larissa).
2. **Processo Node separado da API.** Nova entry-point `src/worker.ts`, ao lado de `src/server.ts`, com script `npm run worker` ([09:11] Larissa, [09:28] Bruno). A lógica de processamento fica dentro do módulo, em `src/modules/webhooks/webhook.worker.ts`. O Bruno sugeriu "webhook.worker.ts ou webhook.processor.ts" ([09:28] Bruno), e esta documentação adota o primeiro nome.
3. **Mesmo banco e mesma stack, outro `PrismaClient`.** O worker usa a mesma `DATABASE_URL`, mas instancia o próprio `PrismaClient`, porque o client é por processo ([09:11] Diego, [09:30] Bruno). Em [09:11] Bruno chegou a falar em "usar o mesmo Prisma client", mas em [09:30] ele mesmo esclareceu: a stack é a mesma e a instância é nova. Ele pode chamar `createPrismaClient()` de `src/config/database.ts`.
4. **Uma única instância (single-worker).** A ordenação é implícita por `created_at` e, portanto, por `order_id`, apenas enquanto houver um só worker ([09:12] Diego). **Não** existe garantia de ordenação global. Isso fica registrado como limitação conhecida ([09:13] Larissa), e os clientes não pediram ordenação global ([09:14] Marcos).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Trigger no banco para reagir à inserção** | Trigger do MySQL só executa SQL e não notifica processo externo. Não existe equivalente ao `NOTIFY/LISTEN` do Postgres. Avisar o worker exigiria improvisos como escrever em arquivo ou chamar um endpoint ([09:09] Bruno, [09:09] Diego). |
| **Worker dentro do processo da API** | O ciclo de vida do worker ficaria acoplado ao da API: em restart ou deploy, o processamento para ([09:11] Diego). |
| **Vários workers em paralelo** | Perde a ordenação por pedido. Exigiria particionar por `order_id` ou usar lock pessimista. Adiado como "problema do futuro" ([09:13] Diego). |

## Consequências

**Positivas**
- Simples de implementar, testar e operar, sem dependência nova.
- Latência previsível: ~2 s de captura, bem dentro do SLA de 10 s.
- Deploy e restart da API não interrompem entregas, e vice-versa.

**Negativas**
- Consultas ao banco a cada 2 s mesmo sem eventos (custo baixo graças aos índices em `status` e `created_at`, ver ADR-001).
- O single-worker é um ponto único de processamento: se ele cair, as entregas atrasam até voltar. Os eventos não se perdem, ficam na outbox.
- A escala horizontal está bloqueada até alguém decidir uma estratégia de particionamento (questão em aberto no [RFC](../RFC.md#5-questões-em-aberto)).
- Um evento em retry (ADR-003) pode ser ultrapassado por um evento mais novo do mesmo pedido. A ordenação "por `order_id`" vale para a primeira tentativa, não para retentativas. Essa consequência foi inferida a partir de [09:12] e [09:17] e está aberta no RFC.

**Trade-off explícito:** aceitamos 2 s de latência mínima e ausência de escala horizontal em troca de não depender de infraestrutura de mensageria nem de mecanismos que o MySQL não oferece.
