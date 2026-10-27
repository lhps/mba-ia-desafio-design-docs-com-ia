# ADR-007 — Payload renderizado (snapshot) na inserção da outbox e filtro de eventos na inserção

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira, pós-call ([09:52] Bruno: "Beleza, snapshot. Decidido.") e bloco de requisitos ([09:34] Bruno/Diego)
- **Decisores:** Larissa, Diego, Bruno
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)

## Contexto

Duas perguntas sobre o momento em que o evento é materializado ficaram para o fim da reunião:

1. A linha da outbox guarda o **payload pronto** ou só o `order_id`, para o worker montar o payload na hora do envio ([09:51] Bruno)?
2. O filtro de status que cada webhook quer receber ([09:33] Marcos) é aplicado **na inserção** na outbox ou **no envio** ([09:34] Diego)?

Com retry de até ~15 h ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)) e replay manual, o tempo entre o fato e a entrega pode ser longo. Nesse intervalo o pedido pode mudar de novo.

## Decisão

1. **Snapshot na inserção.** O payload JSON completo é renderizado e gravado na outbox dentro da transação de `changeStatus`. O evento reflete o estado do pedido **no momento da mudança de status**, mesmo que o pedido mude depois ([09:52] Larissa, [09:52] Diego).
2. **Filtro na inserção.** Ao publicar o evento, só são criadas linhas na outbox para os webhooks **ativos** do `customer_id` do pedido cuja lista de status inclui o `to_status`. Se nenhum webhook do cliente quer aquele status, nada é inserido ([09:34] Bruno, [09:34] Diego).
3. **Payload enxuto.** `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos da ordem como `total_cents`. **Sem itens.** Quem precisar de detalhes consulta `GET /orders/:id` ([09:43] Diego, [09:44] Bruno).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Guardar só `order_id` e renderizar no envio** | Um evento "PAID → PROCESSING" entregue horas depois mostraria o pedido já `SHIPPED`, criando o "caso esquisito" de um evento que não bate com a transição que anuncia ([09:52] Larissa). Também exigiria uma leitura extra de `orders` por envio. |
| **Filtrar no envio (inserir sempre, descartar no worker)** | Gera linhas inúteis na outbox para eventos que nenhum webhook quer ([09:34] Bruno: "Economiza linha na tabela"). |
| **Payload completo com itens** | Infla o payload sem necessidade, já que existe `GET /orders/:id` para detalhes ([09:43] Diego). |

## Consequências

**Positivas**
- O evento é imutável e fiel à transição. Retries e replays enviam exatamente o mesmo corpo, o que é coerente com `X-Event-Id` ([ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)) e com a assinatura HMAC sobre o corpo ([ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md)).
- O worker não toca em `orders`: lê e envia.
- A outbox só contém o que algum webhook realmente quer.

**Negativas**
- A transação de `changeStatus` ganha uma leitura (webhooks ativos do customer) e N inserções, uma por webhook interessado.
- Mudanças feitas na configuração do webhook **depois** da inserção (por exemplo, remover um status do filtro) não afetam eventos já enfileirados.
- Os dados ficam duplicados entre `orders` e `webhook_outbox`. É aceitável porque o snapshot é intencional.

**Trade-off explícito:** pagamos um custo extra pequeno na transação de escrita e alguma duplicação de dados em troca de eventos imutáveis e de um worker simples e desacoplado do modelo de pedidos.
