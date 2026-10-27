# ADR-004 — Autenticação por HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h

- **Status:** Aceito (sujeito à revisão de segurança antes do deploy, [09:46] Sofia)
- **Data da decisão:** reunião técnica de quinta-feira ([09:22] Sofia: "Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h.")
- **Decisores:** Sofia (Segurança), Diego, Bruno
- **Relacionados:** [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)

## Contexto

Os webhooks levam dados de pedidos para endpoints **fora da nossa infraestrutura** ([09:19] Sofia). O cliente precisa conseguir verificar duas coisas: (a) que a requisição saiu de nós e (b) que o payload não foi adulterado no caminho ([09:19] Sofia). Há também um histórico concreto de vazamento: um cliente já vazou secret em log de aplicação ([09:22] Diego).

## Decisão

1. **Assinatura HMAC-SHA256 calculada sobre o corpo do request**, enviada no header `X-Signature` ([09:20] Sofia). SHA-256 por ser o padrão de mercado, com biblioteca disponível em qualquer stack ([09:20] Sofia).
2. **Uma secret por endpoint de webhook**, nunca uma secret global da plataforma. Se uma vaza, as outras continuam seguras ([09:21] Sofia).
3. **A secret é gerada pela plataforma** e devolvida ao cliente na criação do webhook ([09:31] Marcos). Ela fica na configuração do webhook junto com `url`, `customer_id` e o estado ativo ([09:21] Bruno, [09:21] Sofia).
4. **Rotação sob demanda pela API.** Ao rotacionar, a secret antiga continua válida por **24 horas** em paralelo com a nova, para o cliente migrar os sistemas dele. Depois disso, a antiga deixa de valer ([09:21] Sofia).
5. **TLS obrigatório.** A URL do webhook tem de ser `https`, e `http` é recusado com erro de validação ([09:23] Sofia). Isso foi tratado como validação no schema Zod, não como decisão arquitetural. **Divergência registrada:** o FDD (§15, P-11) propõe mover a regra para o service, para emitir `WEBHOOK_INVALID_URL` em vez de `VALIDATION_ERROR`. Isso contraria a fala da Sofia em [09:23] e depende do aceite dela.

> **Ponto de implementação não decidido na reunião:** como sinalizar as **duas** assinaturas durante o grace period. A proposta do FDD é enviar no `X-Signature` uma lista separada por vírgula (`sha256=<nova>,sha256=<antiga>`). Isso ainda precisa de validação da Sofia (ver [RFC, questões em aberto](../RFC.md#5-questões-em-aberto)).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
|---|---|
| **Secret global da plataforma** | Um único vazamento comprometeria todos os clientes ([09:21] Sofia). |
| **Sem assinatura, confiando apenas em TLS** (alternativa plausível, não defendida na reunião) | O TLS protege o transporte, mas não deixa o cliente provar que a origem é a plataforma. Qualquer um que conheça a URL poderia forjar eventos. É exatamente o requisito levantado em [09:19] Sofia. |
| **Rotação com troca imediata (sem grace period)** | Obrigaria o cliente a trocar a secret em todos os sistemas dele no mesmo instante, com risco de rejeitar eventos legítimos. O período de 24 h existe para isso ([09:21] Sofia). |

## Consequências

**Positivas**
- O cliente verifica origem e integridade com uma biblioteca padrão.
- O raio de impacto de um vazamento fica restrito a um endpoint, e a rotação permite reagir sem downtime.
- Segue o mesmo modelo usado no mercado (Stripe e GitHub, citados em [09:25] Diego para a semântica de entrega).

**Negativas**
- A plataforma precisa guardar a secret de forma **recuperável** (para assinar), então não dá para usar hash. A forma de proteção em repouso não foi discutida e entra na revisão de segurança ([09:46] Sofia).
- A secret aparece em respostas da API (criação e rotação) e não pode vazar em logs. O logger Pino (`src/shared/logger/index.ts`) só redige `authorization`, `cookie`, `password`, `passwordHash`, `token` e `accessToken`, então será preciso incluir `*.secret` na lista de redação (ver FDD).
- A assinatura cobre apenas o **corpo**. O `X-Timestamp` não é assinado ([09:44] Diego), então a proteção contra replay do lado do cliente é limitada. Fica registrado como risco para a revisão de segurança.
- Durante o grace period, o worker precisa assinar com duas secrets.

**Trade-off explícito:** aceitamos a complexidade de gerenciar secrets por endpoint (armazenamento recuperável, rotação, dupla assinatura temporária) em troca de isolar o impacto de vazamentos, que no nosso histórico já aconteceram.
