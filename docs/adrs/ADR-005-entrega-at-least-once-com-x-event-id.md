# ADR-005 — Garantia de entrega at-least-once com deduplicação por `X-Event-Id`

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira ([09:26] Larissa: "At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão.")
- **Decisores:** Larissa, Diego, Sofia, Marcos
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)

## Contexto

Em entrega HTTP assíncrona com retry ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)), existem cenários em que o cliente processa o evento e a plataforma **não fica sabendo**: a resposta se perde, estoura o timeout de 10 s depois de o cliente já ter processado, ou o worker cai entre o envio e o registro do sucesso. Precisamos definir qual garantia oferecemos e como o cliente lida com ela.

## Decisão

1. **Semântica at-least-once:** todo evento registrado na outbox será entregue **uma ou mais vezes**. Duplicatas são possíveis e o cliente precisa estar preparado ([09:24] Diego).
2. **`X-Event-Id`:** UUID gerado no momento em que o evento entra na outbox. É único por evento ([09:25] Diego) e se mantém **idêntico** em todas as retentativas. Nos replays da DLQ, manter o mesmo id é uma *inferência*: "recoloca na outbox" ([09:18] Diego) indica o mesmo evento, mas a identidade no replay não foi discutida. O mesmo valor vai no campo `event_id` do payload ([09:43] Diego).
3. **A deduplicação é responsabilidade do cliente**, que usa o `event_id` ([09:25] Diego). A responsabilidade transferida foi reconhecida ([09:25] Sofia) e será documentada com destaque no portal do desenvolvedor ([09:26] Marcos).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Exactly-once** | Exige coordenação dos dois lados (protocolo de confirmação e estado compartilhado), com complexidade muito maior. At-least-once com `event_id` resolve "99% dos casos" e é o padrão de Stripe e GitHub ([09:25] Diego). |
| **At-most-once (sem retry)** (alternativa plausível, oposta ao que foi decidido) | Incompatível com a decisão de retry com backoff ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)) e com a expectativa dos clientes de não perder mudanças de status ([09:00] Marcos). |

## Consequências

**Positivas**
- Nenhum evento commitado se perde: outbox + retry + DLQ + replay.
- O modelo de integração é familiar para quem já consumiu webhooks de Stripe ou GitHub.
- A implementação do worker fica simples. Em caso de dúvida, reenviar é sempre seguro do nosso lado.

**Negativas**
- Clientes que não deduplicam podem processar o mesmo evento duas vezes. O risco é mitigado pela documentação no portal ([09:26] Marcos), mas a plataforma não controla isso.
- O worker passa a poder reprocessar livremente. Por exemplo, ao reiniciar ele devolve à fila eventos que ficaram travados em "processando", o que gera duplicatas por design.

**Trade-off explícito:** trocamos a conveniência de exactly-once para o cliente por uma implementação simples e robusta do nosso lado. O custo vai para o cliente na forma de deduplicação por `event_id`.
