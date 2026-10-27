# PRD: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Product Manager** | Marcos |
| **Tech Lead** | Larissa |
| **Status** | Aprovado em reunião; documentação em revisão |
| **Prazo alvo** | Fim de novembro, pedido da Atlas Comercial ([09:45] Marcos) |
| **Documentos** | [RFC](RFC.md) (proposta técnica) · [FDD](FDD.md) (implementação) · [ADRs](adrs/README.md) · [Tracker](TRACKER.md) |

---

## 1. Resumo e contexto da feature

Hoje o OMS só informa a mudança de status de um pedido a quem pergunta. Esta feature cria **webhooks de saída**: o cliente B2B cadastra uma URL `https` e passa a receber um `POST` assinado sempre que um pedido dele muda para um dos status que escolheu. A meta é que isso aconteça em **menos de 10 segundos**.

A demanda veio de um pedido formal de três clientes B2B e tem prazo comercial ([09:00] Marcos).

## 2. Problema e motivação

- **Problema:** Atlas Comercial, MaxDistribuição e Nova Cargo descobrem mudanças de status fazendo polling periódico em `GET /orders`. A integração fica **lenta**, porque a mudança só aparece no próximo ciclo de consulta, e **cara**, porque são muitas requisições que não encontram mudança nenhuma ([09:00] Marcos).
- **Risco de negócio:** a Atlas sinalizou que pode **migrar para um concorrente** se a solução não sair até o fim do trimestre ([09:00] Marcos).
- **Lacuna da plataforma:** o OMS não tem nenhum mecanismo de notificação externa, evento ou fila.

## 3. Público-alvo e cenários de uso

**Público primário:** times de integração dos clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo). Eles acessam a API com JWT de usuários da plataforma que os representam ([09:32] Marcos).
**Público secundário:** administradores da plataforma (role `ADMIN`), que reprocessam entregas que falharam ([09:36] Sofia).

| # | Cenário | Origem |
|---|---|---|
| C-01 | O integrador da Atlas cadastra `https://…/orders` assinando só `SHIPPED` e `DELIVERED`. Quando um pedido da Atlas é despachado, o sistema dela recebe o evento em segundos e atualiza o rastreio, sem polling. | [09:31] Marcos, [09:33] Marcos |
| C-02 | O endpoint da MaxDistribuição fica 2 h fora do ar em manutenção planejada. Os eventos do período são reentregues automaticamente quando o endpoint volta. | [09:16] Diego |
| C-03 | O integrador suspeita que vazou a secret em log. Ele pede uma nova pela API e tem 24 h para trocar nos sistemas dele sem perder eventos. | [09:21] Sofia, [09:22] Diego |
| C-04 | O cliente recebe o mesmo evento duas vezes e descarta a duplicata pelo `X-Event-Id`. | [09:25] Diego |
| C-05 | O cliente quer saber por que não recebeu um evento e consulta as últimas 100 entregas: sucesso ou falha, payload, resposta e tempo. | [09:34] Marcos |
| C-06 | Um evento esgotou as tentativas e foi para a DLQ. Um ADMIN dispara o replay depois que o cliente corrige o endpoint, e o replay fica registrado para auditoria. | [09:18] Diego, [09:36] Sofia |

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta | Origem |
|---|---|---|---|---|
| OBJ-01 | Notificação em "tempo real" na percepção do cliente | Latência entre a mudança de status e a 1ª tentativa de entrega | **< 10 s** (esperado ≈ 2 s de captura pelo polling + tempo de envio) | [09:02] Marcos, [09:10] Larissa |
| OBJ-02 | Reter os clientes que pediram a feature | Clientes com webhook ativo em produção | **3 de 3** (Atlas, MaxDistribuição, Nova Cargo) até o **fim de novembro**. *Meta proposta:* a reunião fixou só o prazo pedido pela Atlas | [09:00] Marcos, [09:45] Marcos |
| OBJ-03 | Não perder notificações | Mudanças de status commitadas sem evento registrado para webhooks interessados | **0** | [09:40] Bruno |
| OBJ-04 | Tolerar indisponibilidade do cliente | Janela coberta por retentativas automáticas antes da DLQ | **≈ 14,6 h** (1m+5m+30m+2h+12h) | [09:17] Diego, [09:17] Marcos |
| OBJ-05 | Reduzir o polling dos clientes em `GET /orders` | Requisições de `GET /orders` desses clientes antes e depois | Tendência de queda. A reunião não fixou percentual: **linha de base a medir** antes do go-live | [09:00] Marcos |

## 5. Escopo

### 5.1 Incluso
- Cadastro, listagem, edição e remoção de webhooks por cliente, com filtro de status.
- Notificação de **mudança de status de pedido** (`order.status_changed`), originada em `PATCH /orders/:id/status`.
- Assinatura HMAC-SHA256 com secret por endpoint e rotação com grace period de 24 h.
- Retentativas automáticas, fila de falhas definitivas (DLQ) e replay manual por ADMIN.
- Histórico das últimas 100 entregas por webhook.

### 5.2 Fora de escopo

| Item | Situação | Origem |
|---|---|---|
| **Notificação por e-mail** quando o webhook do cliente falha repetidamente | **Adiado** para a próxima fase, depois de medir o impacto | [09:37] Larissa, [09:38] Marcos |
| **Dashboard visual** para o cliente gerenciar webhooks | **Descartado** nesta feature; é projeto separado do time de frontend. Só endpoints de API. | [09:40] Larissa |
| **Rate limiting** de envio por cliente | **Adiado**: observar e decidir depois | [09:39] Diego, [09:39] Larissa |
| **Webhooks de entrada** (cliente → plataforma) | **Descartado**: os clientes só querem receber | [09:02] Marcos |
| **Garantia de ordenação global** e múltiplos workers | **Adiado** como limitação conhecida | [09:13] Larissa, [09:14] Marcos |
| **Garantia exactly-once** | **Descartado** em favor de at-least-once | [09:25] Diego |
| **Arquivamento** de eventos entregues (~30 dias) | **Adiado**, fora desta feature | [09:08] Diego |
| **Itens do pedido** no payload | **Descartado**: quem precisar consulta `GET /orders/:id` | [09:43] Diego |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
|---|---|---|
| PRD-FR-01 | O cliente cadastra um webhook informando a URL e a lista de status que quer receber. A **secret é gerada pela plataforma** e devolvida na criação. | [09:31] Marcos |
| PRD-FR-02 | O cliente identificado no cadastro é informado na requisição (body ou path), **não** derivado do JWT, que pertence ao usuário operador. | [09:32] Bruno, [09:32] Larissa |
| PRD-FR-03 | É possível **listar** os webhooks de um customer, **editar** (PATCH) e **remover** (DELETE) um webhook. | [09:33] Bruno |
| PRD-FR-04 | Cada webhook tem um **filtro de status**, e o cliente só recebe as mudanças para os status escolhidos (ex.: só `SHIPPED` e `DELIVERED`). | [09:33] Marcos |
| PRD-FR-05 | Quando o status de um pedido muda, cada webhook **ativo** do customer do pedido que assina o novo status recebe um `POST` com o evento. | [09:00] Marcos, [09:06] Diego |
| PRD-FR-06 | O evento contém `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do pedido, como `total_cents`. **Sem itens.** | [09:43] Diego |
| PRD-FR-07 | O evento reflete o pedido **no momento da mudança** (snapshot), mesmo que seja entregue depois. | [09:52] Larissa |
| PRD-FR-08 | Cada requisição leva os headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`. | [09:44] Diego, [09:44] Sofia |
| PRD-FR-09 | Cada requisição é **assinada com HMAC-SHA256** sobre o corpo, usando uma secret **exclusiva daquele endpoint**. | [09:20] Sofia, [09:21] Sofia |
| PRD-FR-10 | O cliente pode **rotacionar a secret** pela API. A secret anterior continua válida por **24 h** e depois expira. | [09:21] Sofia |
| PRD-FR-11 | Entregas que falham são **retentadas automaticamente** 5 vezes, com intervalos de 1 min, 5 min, 30 min, 2 h e 12 h. Contar como 5 *retentativas* após o envio inicial é interpretação desta documentação, ainda a confirmar ([RFC Q-04](RFC.md#5-questões-em-aberto)). | [09:17] Diego, [09:17] Larissa |
| PRD-FR-12 | Esgotadas as retentativas, o evento vai para uma **fila de falhas (DLQ)** com payload, motivo e horário da falha. | [09:18] Diego |
| PRD-FR-13 | Um usuário **ADMIN** pode reprocessar manualmente um evento da DLQ. A ação registra quem a fez. | [09:18] Diego, [09:36] Sofia |
| PRD-FR-14 | O cliente consulta o **histórico das últimas 100 entregas** de um webhook: sucesso ou falha, payload, resposta e tempo de resposta. | [09:34] Marcos |
| PRD-FR-15 | Cadastro ou edição com URL **http** (sem TLS) é **recusado** com erro de validação. | [09:23] Sofia |
| PRD-FR-16 | Se nenhum webhook do cliente assina o novo status, nenhum evento é gerado. | [09:34] Bruno |

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
|---|---|---|
| PRD-NFR-01 | **Latência:** notificação em menos de 10 s; captura do evento a cada 2 s. | [09:02] Marcos, [09:10] Larissa |
| PRD-NFR-02 | **Atomicidade:** se o status mudou, o evento existe; se a mudança falhou, não existe evento. | [09:06] Diego, [09:40] Bruno |
| PRD-NFR-03 | **Isolamento:** a lentidão ou indisponibilidade de um cliente não afeta a mudança de status de nenhum pedido. | [09:04] Bruno |
| PRD-NFR-04 | **Entrega at-least-once:** duplicatas são possíveis, e o cliente deduplica por `X-Event-Id`. | [09:24] Diego, [09:25] Diego |
| PRD-NFR-05 | **Ordenação:** garantida apenas por pedido, e só enquanto houver um único worker. Não há ordenação global. | [09:12] Diego, [09:13] Larissa |
| PRD-NFR-06 | **Timeout:** o cliente tem 10 s para responder, senão a tentativa conta como falha. | [09:42] Diego |
| PRD-NFR-07 | **Tamanho:** evento acima de 64 KB não é enviado e é tratado como erro. | [09:23] Sofia, [09:24] Diego |
| PRD-NFR-08 | **Segurança:** TLS obrigatório, secret por endpoint, CRUD acessível a qualquer usuário autenticado, replay restrito a ADMIN. | [09:23] Sofia, [09:21] Sofia, [09:37] Sofia, [09:36] Sofia |
| PRD-NFR-09 | **Disponibilidade:** o processamento roda em processo separado e sobrevive a restarts da API. | [09:11] Diego |
| PRD-NFR-10 | **Operação:** nenhuma infraestrutura nova; mesmo MySQL e mesma stack. | [09:07] Diego, [09:11] Diego |
| PRD-NFR-11 | **Consistência com a plataforma:** mesmos padrões de API, erros (prefixo `WEBHOOK_`) e logs do restante do OMS. | [09:30] Larissa, [09:29] Larissa |

## 8. Decisões e trade-offs principais

| Decisão | Ganho | Custo aceito | ADR |
|---|---|---|---|
| Outbox no MySQL | Nenhum evento perdido ou fantasma; sem infraestrutura nova | Tabela cresce até existir arquivamento | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Worker separado, polling de 2 s | Simples; atende < 10 s | Latência mínima de ~2 s; sem escala horizontal | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| 5 retentativas em ~15 h + DLQ | Atravessa manutenções longas | Evento pode chegar horas depois; replay manual | [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) |
| HMAC por endpoint + rotação | Vazamento isolado; troca sem downtime | Gestão de secrets mais complexa | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| At-least-once + `X-Event-Id` | Implementação simples e padrão de mercado | O cliente precisa deduplicar | [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| Reuso dos padrões do projeto | Manutenção barata para um time pequeno | Sem métricas ou tracing dedicados nesta fase | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) |
| Snapshot do evento + filtro na geração | Evento fiel à transição; menos registros | Mudança no filtro não afeta eventos já gerados | [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md) |

## 9. Dependências

| Dependência | Tipo | Origem |
|---|---|---|
| Revisão de segurança da Sofia (HMAC e geração de secret), com **2 dias úteis** antes do deploy | Interna, bloqueante | [09:46] Sofia, [09:49] Sofia |
| Documentação no **portal do desenvolvedor** (integração, validação HMAC, deduplicação) | Interna (Produto) | [09:26] Marcos, [09:40] Marcos |
| Comunicação do prazo aos clientes | Interna (Produto) | [09:47] Marcos, [09:49] Marcos |
| Sessão de revisão do design com Bruno e Diego antes de codar | Interna (Engenharia) | [09:50] Larissa |
| Clientes adaptarem os sistemas: endpoint https, validação de assinatura, dedup, resposta 2xx em até 10 s | Externa | [09:23] Sofia, [09:25] Diego, [09:42] Diego |
| Usuários da plataforma que representam cada cliente, para autenticar na API | Pré-existente | [09:32] Marcos |

## 10. Cronograma e marcos

Estimativa da Tech Lead: **3 sprints**, com a revisão de segurança incluída no fim ([09:46] Larissa, [09:47] Larissa):

| Bloco | Esforço estimado |
|---|---|
| Modelagem da outbox e da DLQ | 1 sprint |
| Worker e retry | 1 sprint |
| CRUD de configuração e histórico de entregas | ½ sprint |
| Integração com pedidos e testes ponta a ponta | ½ sprint |
| HMAC, schemas, validações e revisão da Sofia | "mais um pouco", dentro das 3 sprints |

## 11. Riscos e mitigação

| # | Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|---|
| RISK-01 | **Perder a Atlas** por atraso: o prazo é o fim de novembro e a estimativa é de 3 sprints | Média | Alto | Escopo enxuto (sem e-mail, dashboard ou rate limiting); revisão de segurança já no cronograma; Marcos confirma o prazo com o cliente ([09:47] Marcos) |
| RISK-02 | Cliente **processa evento duplicado**, porque at-least-once transfere a deduplicação para ele | Média | Médio | `X-Event-Id` estável e documentação destacada no portal ([09:25] Sofia, [09:26] Marcos) |
| RISK-03 | **Vazamento de secret** por parte do cliente, como já aconteceu | Média | Alto | Secret por endpoint limita o raio de impacto; rotação com 24 h de transição ([09:21] Sofia, [09:22] Diego) |
| RISK-04 | **Rajadas de eventos** sobrecarregam o endpoint do cliente (ex.: 50 pedidos em 1 min) | Média | Médio | Monitorar volume por webhook e reabrir a decisão de rate limiting se necessário ([09:38] Diego, [09:39] Larissa) |
| RISK-05 | **Eventos fora de ordem** para o mesmo pedido em caso de retry | Baixa | Médio | O payload traz `from_status`, `to_status` e `timestamp`; limitação documentada ([09:13] Larissa) |
| RISK-06 | Cliente fora do ar por **mais de ~15 h** perde eventos para a DLQ | Baixa | Médio | Replay manual por ADMIN; produto considera aceitável ([09:17] Marcos) |

## 12. Critérios de aceitação

1. Um cliente cadastra um webhook `https` com filtro `["SHIPPED","DELIVERED"]` e recebe a secret na resposta.
2. Ao mudar um pedido desse cliente para `SHIPPED`, o endpoint recebe o `POST` em menos de 10 s, com o payload e os 5 headers definidos e assinatura válida.
3. Ao mudar para `PAID` (não assinado), nada é enviado.
4. Cadastro com `http://` é recusado.
5. Com o endpoint fora do ar, o evento é retentado em 1m/5m/30m/2h/12h e, se continuar falhando, aparece na DLQ.
6. Um ADMIN reprocessa o evento da DLQ e o cliente o recebe com o **mesmo** `X-Event-Id` (inferência, ver ADR-005). Um usuário não-ADMIN recebe acesso negado.
7. Depois de rotacionar a secret, o cliente consegue validar com a antiga por 24 h e, depois disso, só com a nova.
8. O histórico de entregas mostra no máximo as últimas 100 tentativas, com status, payload, resposta e tempo.
9. Uma falha na geração do evento impede a mudança de status. Nenhum pedido muda de status sem gerar o evento devido.
10. A revisão de segurança está aprovada antes do deploy.

## 13. Estratégia de testes e validação

| Nível | O que valida | Como |
|---|---|---|
| Unitário | Regras de retry, assinatura, filtro de status e montagem do evento | Suíte automatizada do projeto |
| Integração da API | Cadastro, rotação, histórico, replay e permissões | Testes de API contra banco de teste |
| Atomicidade | Nenhuma mudança de status sem evento, e vice-versa | Teste forçando falha na geração do evento |
| Ponta a ponta | Entrega, assinatura, timeout, retry, DLQ | Cliente simulado com sucesso, erro, lentidão e queda |

Detalhamento técnico dos testes: [FDD §14](FDD.md#14-critérios-de-aceite-técnicos).
| Segurança | HMAC, geração de secret, exposição de secret em respostas e logs | Revisão dedicada da Sofia, com 2 dias úteis ([09:46] Sofia) |
| Validação com clientes | Integração real dos 3 clientes (OBJ-02) e latência (OBJ-01) | Acompanhamento das métricas do [FDD §10](FDD.md#10-observabilidade) depois do go-live |
