# RFC-001: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Autor** | Lucas Pinho, a partir da reunião técnica conduzida por Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | 2026-10-27 (redação). A reunião de origem foi numa quinta-feira, 09:00; a data exata não consta na transcrição. |
| **Revisores** | Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Eng. de Segurança) |
| **Documentos relacionados** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/README.md) · [Tracker](TRACKER.md) |

---

## 1. Resumo executivo (TL;DR)

Propomos **webhooks outbound** que avisam clientes B2B quando o status de um pedido muda, com latência abaixo de 10 s. A abordagem:

- **Transactional Outbox no MySQL existente.** O evento é gravado na mesma transação de `OrderService.changeStatus`.
- **Worker separado** que lê a outbox por polling a cada 2 s.
- **Retry** com backoff exponencial (1m/5m/30m/2h/12h) e **DLQ** com replay manual por ADMIN.
- **HMAC-SHA256** com secret por endpoint e rotação com grace period de 24 h.
- **Entrega at-least-once** com `X-Event-Id` para o cliente deduplicar.

Nada de infraestrutura nova: o módulo reusa os padrões do projeto. A estimativa é de **3 sprints**, incluindo **2 dias úteis de revisão de segurança** ([09:46] Larissa, [09:46] Sofia).

## 2. Contexto e problema

Atlas Comercial, MaxDistribuição e Nova Cargo pediram formalmente para ser notificados quando os pedidos deles mudam de status. Hoje eles fazem polling em `GET /orders`, o que deixa a integração **lenta e cara**, e a Atlas sinalizou que pode migrar para um concorrente se a solução não sair até o fim do trimestre ([09:00] Marcos).

O OMS atual **não tem nenhum mecanismo de notificação externa, evento ou fila**. A mudança de status acontece em uma transação já pesada, que atualiza `orders`, insere em `order_status_history` e mexe em `products.stockQuantity` (`src/modules/orders/order.service.ts`, [09:04] Bruno). Qualquer solução precisa:

- **não** degradar nem acoplar essa transação à disponibilidade de terceiros;
- **não** perder eventos de mudanças commitadas, nem emitir eventos de mudanças que deram rollback;
- ser operável por um **time pequeno** ([09:07] Diego).

Escopo de direção: apenas **outbound**, da plataforma para o cliente ([09:02] Marcos, [09:03] Sofia).

## 3. Proposta técnica

```
PATCH /orders/:id/status
        │
        ▼
┌──────────────── transação MySQL (changeStatus) ────────────────┐
│ update orders → insert order_status_history → estoque           │
│ → publishWebhookEvent(tx, order, from, to)                      │
│     • busca webhooks ATIVOS do customer que querem `to`          │
│     • insere 1 linha por webhook em webhook_outbox (snapshot)    │
└─────────────────────────────────────────────────────────────────┘
        │ commit (ou rollback de tudo)
        ▼
 webhook_outbox ◄──── polling 2s ──── worker (src/worker.ts, processo próprio)
        │                                   │ POST https + X-Signature (HMAC)
        │                                   ▼
        │                           endpoint do cliente (timeout 10s)
        │                     2xx → entregue │ falha → retry 1m/5m/30m/2h/12h
        ▼                                    ▼
 webhook_deliveries (histórico)       webhook_dead_letter ──► replay ADMIN
```

**Componentes**

1. **Publicação transacional** ([ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md)). Uma função `publishWebhookEvent(tx, …)` recebe o `TransactionClient` da transação de `changeStatus`, filtra os webhooks interessados e grava o payload já renderizado. Se a inserção falhar, a mudança de status inteira dá rollback.
2. **Worker** ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)). Entry-point `src/worker.ts` (`npm run worker`), em processo separado, single-instance, com polling de 2 s em lotes pequenos ordenados por `created_at`.
3. **Entrega e resiliência** ([ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md), [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)). Timeout de 10 s. Resposta fora de 2xx conta como falha e entra no backoff (convenção adotada; não foi dita na reunião). Esgotadas as retentativas, o evento vai para a DLQ. Garantia at-least-once.
4. **Segurança** ([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)). Somente URLs `https`. Assinatura HMAC-SHA256 do corpo com secret própria do endpoint, rotacionável. Payload limitado a 64 KB.
5. **API de gestão** ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)). Módulo `src/modules/webhooks` no padrão do projeto, com CRUD de configuração, rotação de secret e histórico de entregas para qualquer usuário autenticado, e replay de DLQ restrito a `ADMIN`.

Os contratos HTTP, o modelo de dados, a máquina de estados da outbox e a matriz de erros estão no [FDD](FDD.md).

## 4. Alternativas consideradas

| # | Alternativa | Trade-off que levou ao descarte | Fonte |
|---|---|---|---|
| ALT-01 | **Chamada HTTP síncrona dentro de `changeStatus`** | Um cliente lento trava mudanças de status de outros pedidos, e se o cliente estiver fora do ar não dá para fazer rollback do status. A disponibilidade de terceiros vira disponibilidade nossa. | [09:04] Bruno, [09:06] Diego |
| ALT-02 | **Redis Streams / broker dedicado** | Exige subir e operar Redis Cluster, *overengineering* para um time pequeno. | [09:07] Larissa, [09:07] Diego |
| ALT-03 | **Trigger MySQL para acordar o worker** | MySQL não tem `NOTIFY/LISTEN`. A trigger só executa SQL, e notificar um processo externo exigiria gambiarra. O polling de 2 s já atende o SLA de 10 s. | [09:09] Bruno, [09:09] Diego |
| ALT-04 | **Retry indefinido** ou **apenas 3 tentativas** | O indefinido deixa eventos pendurados para sempre. Com 3, a janela cai para ~30 min, curta para manutenções de 2 h que já ocorreram. | [09:15] Diego, [09:16] Diego |
| ALT-05 | **Exactly-once** | Exige coordenação bilateral, com complexidade muito maior. At-least-once + `event_id` cobre "99% dos casos" e é o padrão de Stripe e GitHub. | [09:25] Diego |
| ALT-06 | **Renderizar o payload no envio** | Retentativas horas depois mostrariam um estado diferente da transição anunciada. | [09:52] Larissa |

## 5. Questões em aberto

| # | Questão | Situação na reunião | Proposta / próximo passo | Dono sugerido |
|---|---|---|---|---|
| Q-01 | **Rate limiting de saída por cliente.** 50 pedidos mudando em 1 min viram 50 chamadas. | Adiado: "observar e decidir depois" ([09:38] Diego, [09:39] Larissa) | Medir volume por webhook (ver métricas no FDD) e reabrir se aparecer reclamação ou 429. | Diego |
| Q-02 | **`customer_id` no body ou no path** dos endpoints de configuração | Não decidido. Larissa disse "no body ou no path" ([09:32] Larissa) depois de descartar o JWT. | O FDD propõe `customerId` no **body** do `POST` e na **query** do `GET` de listagem. Confirmar na sessão de revisão ([09:50] Larissa). | Bruno |
| Q-03 | **Escala para múltiplos workers e ordenação** | Adiado: "problema do futuro" (particionar por `order_id` ou lock pessimista) ([09:13] Diego) | Manter single-worker. Detalhe derivado: mesmo com um único worker, um evento em retry pode ser **ultrapassado** por um evento mais novo do mesmo pedido ([09:12] Diego × [09:17] Diego). Decidir se isso é aceitável ou se o worker deve segurar eventos de um `order_id` que tenha retry pendente. | Diego / Larissa |
| Q-04 | **"5 tentativas" = 5 retentativas ou 5 envios no total?** | Ambíguo. A agenda tem 5 intervalos e "quase 15 h" ([09:17] Diego), mas o resumo diz "total 5 tentativas" ([09:48] Larissa). | ADR-003 assume envio inicial + 5 retentativas (6 chamadas), que é o único modelo coerente com ~15 h. Confirmar. | Larissa |
| Q-05 | **Formato da assinatura durante o grace period de 24 h** | Não discutido. Só a regra "antiga válida por 24 h" ([09:21] Sofia). | O FDD propõe `X-Signature: sha256=<nova>,sha256=<antiga>` durante o grace period. | Sofia |
| Q-06 | **Proteção da secret em repouso, `X-Timestamp` não assinado e destino da URL (rede interna)** | Fora da pauta. Sofia reservou 2 dias para revisar HMAC e geração de secret ([09:46] Sofia). | Levar como itens da revisão de segurança antes do deploy. | Sofia |
| Q-07 | **Notificar criação de pedido (`PENDING`)?** | Não discutido. O gancho acordado é só `changeStatus` ([09:40] Bruno). | Pelo código, a criação grava `fromStatus: null → PENDING` em `create()`, e nenhuma transição leva a `PENDING` (`src/modules/orders/order.status.ts`). Proposta: rejeitar `PENDING` no filtro de eventos nesta fase. | Marcos |
| Q-08 | **Endurecer permissões do CRUD de configuração** | Adiado: "Por enquanto [qualquer role autenticada]. Mais pra frente a gente pode endurecer." ([09:37] Sofia) | Reavaliar após o go-live. | Sofia |

## 6. Impacto e riscos

**Impacto no sistema**
- **Pedidos:** a transação de `changeStatus` ganha uma leitura e N inserções. A disponibilidade de `PATCH /orders/:id/status` passa a depender da saúde de `webhook_outbox`, por design ([09:40] Bruno).
- **Banco:** quatro tabelas novas (config, outbox, histórico de entregas, DLQ). Crescimento contínuo da outbox até existir arquivamento, que está fora do escopo ([09:08] Diego).
- **Operação:** um processo novo para deploy e monitoramento (`npm run worker`).
- **Clientes:** precisam validar o HMAC e deduplicar por `event_id`. O Marcos vai documentar isso no portal ([09:26] Marcos, [09:40] Marcos).

**Riscos principais** (a matriz completa com probabilidade e impacto está no [PRD](PRD.md#11-riscos-e-mitigação) e no [FDD](FDD.md#13-riscos-e-mitigação))

| Risco | Mitigação |
|---|---|
| Prazo: a Atlas quer a feature até o fim de novembro e a estimativa é de 3 sprints ([09:45] Marcos, [09:46] Larissa) | Revisão de segurança já reservada no cronograma ([09:47] Larissa). Escopo enxuto, sem e-mail nem dashboard. |
| Falha na inserção da outbox derruba mudanças de status | Inserção simples, com índices e sem chamadas externas na transação. Testes de integração cobrindo o rollback. |
| Cliente não deduplica e processa duas vezes | Documentação destacada no portal ([09:26] Marcos). |
| Worker parado acumula atraso | Métrica de idade do evento pendente mais antigo e alerta (FDD §10). |

## 7. Decisões relacionadas

| ADR | Decisão |
|---|---|
| [ADR-001](adrs/ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL existente |
| [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) | Worker em processo separado, polling 2 s, single-worker |
| [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) | Retry 1m/5m/30m/2h/12h e DLQ em tabela separada |
| [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256, secret por endpoint, rotação com grace period de 24 h |
| [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) | At-least-once com `X-Event-Id` |
| [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) | Reuso dos padrões do projeto |
| [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md) | Snapshot do payload e filtro na inserção |
