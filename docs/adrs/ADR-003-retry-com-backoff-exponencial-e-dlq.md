# ADR-003 — Retry com backoff exponencial (1m/5m/30m/2h/12h) e DLQ em tabela separada

- **Status:** Aceito
- **Data da decisão:** reunião técnica de quinta-feira ([09:17] Larissa: "Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h")
- **Decisores:** Larissa, Diego, Bruno, Marcos
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)

## Contexto

O endpoint do cliente pode estar fora do ar, lento ou devolvendo erro. Já houve cliente com **duas horas** de indisponibilidade por manutenção planejada ([09:16] Diego). Precisamos insistir o bastante para atravessar janelas de manutenção, mas sem deixar eventos pendurados para sempre quando o cliente some ([09:15] Diego).

## Decisão

1. **Backoff exponencial com teto de tentativas** ([09:15] Diego).
2. **Agenda de retentativas:** 1 min, 5 min, 30 min, 2 h, 12 h ([09:17] Diego). O total entre a primeira falha e a última tentativa é de ~14h36min ("quase 15 horas", [09:17] Diego), considerado aceitável pelo produto ([09:17] Marcos).
3. **Interpretação de "5 tentativas" (pendente de confirmação, [RFC Q-04](../RFC.md#5-questões-em-aberto)).** A reunião fechou o número 5 e a agenda, mas não o que conta como "tentativa". Esta documentação interpreta como são **5 retentativas após o envio inicial**, ou seja, até **6 chamadas HTTP** por evento. Justificativa, tirada da própria transcrição:
   - a agenda tem 5 intervalos e soma "quase 15 horas" *entre a primeira falha e a última tentativa* ([09:17] Diego), o que só fecha se os 5 intervalos forem usados;
   - o contra-exemplo do Diego para "3" foi "retentaria três vezes em 30 minutos" ([09:16] Diego), e 1m+5m+30m ≈ 36 min. Ele estava contando **retentativas**.
   - O resumo da reunião diz "total 5 tentativas" ([09:48] Larissa). A divergência de vocabulário fica registrada no RFC para confirmação. Se o time decidir por 5 chamadas no total, basta remover o intervalo de 12 h.
4. **Uma tentativa falha quando:** há timeout de 10 s ([09:42] Diego), erro de rede ou resposta HTTP fora da faixa 2xx. *Pressuposto:* o critério "2xx = sucesso" não foi dito na reunião; é a convenção adotada aqui.
5. **DLQ em tabela separada `webhook_dead_letter`**, guardando payload, motivo da falha e timestamp ([09:18] Diego). Esgotadas as retentativas, o evento sai do fluxo ativo da outbox e vai para a DLQ.
6. **Reprocessamento manual** via `POST /admin/webhooks/dead-letter/:id/replay`. O endpoint recoloca o evento na outbox como pendente ([09:18] Diego), exige role `ADMIN` e registra quem fez o replay ([09:36] Sofia, [09:36] Larissa).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Retry indefinido com backoff** | O evento fica pendurado para sempre se o cliente sumir ([09:15] Diego). |
| **3 tentativas (mais agressivo)** | Cobre só ~30 minutos e mataria eventos durante manutenções planejadas de 2 h, que já aconteceram ([09:16] Bruno, [09:16] Diego). |
| **Marcar como `failed` na própria outbox (sem tabela de DLQ)** | Mistura eventos mortos com o fluxo ativo e polui a leitura da outbox. A tabela separada deixa a outbox limpa e serve de evidência para debug e reprocessamento ([09:17] Larissa, [09:18] Diego). |

## Consequências

**Positivas**
- Atravessa indisponibilidades de até ~14,6 h sem intervenção.
- A outbox principal só contém trabalho vivo, e a DLQ concentra o que precisa de atenção humana.
- O replay é auditável (quem e quando).

**Negativas**
- Um evento pode chegar ao cliente até ~15 h depois do fato. O payload é um snapshot ([ADR-007](ADR-007-snapshot-do-payload-na-insercao.md)), então o cliente recebe o estado da época e precisa considerar o `timestamp`.
- Não há aviso proativo ao cliente quando o webhook dele está falhando. Notificação por e-mail ficou para uma fase futura ([09:37] Larissa).
- O replay depende de ação humana de um ADMIN. Não existe reprocessamento automático da DLQ.
- Retentativas aumentam a chance de entrega duplicada, o que reforça a semântica at-least-once ([ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)).

**Trade-off explícito:** priorizamos cobrir janelas longas de indisponibilidade em vez de ter feedback rápido de falha definitiva. Ter um teto, em vez de retry indefinido, garante que nenhum evento fica eternamente em processamento.
