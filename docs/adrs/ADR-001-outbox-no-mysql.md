# ADR-001 — Padrão Outbox no MySQL existente

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira, 09:00 ([09:08] Larissa: "Tá decidido então: outbox em MySQL")
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md)

## Contexto

Três clientes B2B precisam ser notificados quando o status de um pedido muda ([09:00] Marcos). A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de um `prisma.$transaction` que já:

1. debita ou repõe `products.stockQuantity` conforme `shouldDebitStock` / `shouldReplenishStock` (`src/modules/orders/order.status.ts`);
2. atualiza `orders.status`;
3. insere uma linha em `order_status_history`.

Disparar o HTTP para o cliente **dentro** dessa transação foi rejeitado: um cliente lento seguraria a transação (e os locks de estoque) e bloquearia mudanças de status de outros pedidos ([09:04] Bruno); e se o cliente estiver fora do ar não faz sentido dar rollback na mudança de status ([09:04] Bruno). Precisamos de um mecanismo assíncrono que **não perca** eventos nem **invente** eventos de transações que deram rollback.

## Decisão

Adotar o **padrão Transactional Outbox** usando o **MySQL que a aplicação já usa** ([09:06] Diego):

- Criar a tabela `webhook_outbox`. Na mesma transação de `changeStatus`, inserir uma linha por evento a entregar.
- Se a transação principal commitar, o evento existe; se der rollback, o evento desaparece junto. Não há estado intermediário inconsistente ([09:06] Diego).
- Se a inserção na outbox falhar, a transação inteira dá rollback: "não pode ter caso de status mudar e evento não sair" ([09:40] Bruno, [09:41] Diego).
- A tabela tem índice em `status` (pendente, processando, falhou, entregue) e em `created_at` ([09:08] Diego). O consumidor lê os pendentes em lotes pequenos ([09:08] Diego). O consumidor está descrito no [ADR-002](ADR-002-worker-separado-em-polling.md).
- Chave primária UUID, seguindo o padrão de todos os models em `prisma/schema.prisma` ([09:51] Larissa).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Disparo síncrono no `changeStatus`** | Acopla a latência do cliente à transação de pedidos; um cliente lento trava mudanças de status de outros pedidos; não há resposta aceitável para "cliente fora do ar" (rollback do status não é opção) ([09:04] Bruno, [09:06] Diego: "Síncrono está fora de questão"). |
| **Redis Streams (ou broker equivalente)** | Exige subir e operar infraestrutura nova (Redis Cluster) para um time pequeno, o que foi classificado como *overengineering* ([09:07] Larissa, [09:07] Diego). *Argumento adicional, não levantado na reunião:* publicar direto no broker depois do commit perderia a atomicidade com a transação MySQL. |

## Consequências

**Positivas**
- Atomicidade entre a mudança de status e o registro do evento, sem coordenação distribuída.
- Zero infraestrutura nova: mesmo banco, mesmo Prisma, mesmo backup e mesma operação.
- A outbox serve de trilha auditável do que deveria ter sido enviado.

**Negativas**
- O banco transacional passa a acumular linhas de evento. O arquivamento de linhas entregues (≈30 dias) foi explicitamente deixado **fora do escopo** desta feature ([09:08] Diego). A tabela cresce sem limite até que isso seja feito.
- Cada mudança de status ganha um `INSERT` extra na transação, que já é pesada ([09:04] Bruno).
- A latência passa a depender do intervalo de leitura do consumidor (ver ADR-002).

**Trade-off explícito:** trocamos throughput e isolamento de um broker dedicado por consistência transacional e simplicidade operacional. Para o volume atual e um time pequeno, consistência e simplicidade valem mais.
