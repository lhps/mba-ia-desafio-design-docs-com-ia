# Tracker de Rastreabilidade: Webhooks de Notificação de Pedidos

Cada linha liga um item dos documentos à sua origem: a transcrição da reunião (`TRANSCRICAO.md`, localizada por `[hh:mm] Nome`) ou o código base (caminho do arquivo). Quando um item tem mais de uma origem, a coluna Localização traz a **fonte principal**, e as demais aparecem no próprio documento.

**Tipos usados:** Contexto · Requisito Funcional · Requisito Não Funcional · Decisão · Alternativa Descartada · Trade-off · Restrição · Fora de Escopo · Questão em Aberto · Objetivo/Métrica · Risco · Dependência · Contrato · Erro · Integração · Critério de Aceite · Proposta (não decidida na reunião; validar).

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CTX-01 | docs/PRD.md | Contexto | Três clientes B2B (Atlas, MaxDistribuição, Nova Cargo) pediram notificação de mudança de status | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Contexto | Clientes fazem polling em `GET /orders`: integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Risco | Atlas pode migrar para concorrente se a feature não sair até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-04 | docs/PRD.md | Contexto | OMS sem mecanismo de notificação, evento ou fila; nenhum módulo/rota de webhooks registrado | CODIGO | src/routes/index.ts |
| PRD-PUB-01 | docs/PRD.md | Contexto | Clientes acessam a API com JWT de usuários da plataforma que os representam | TRANSCRICAO | [09:32] Marcos |
| PRD-PUB-02 | docs/PRD.md | Contexto | Público secundário: ADMIN que reprocessa a DLQ | TRANSCRICAO | [09:36] Sofia |
| PRD-C-01 | docs/PRD.md | Cenário | Cadastro com filtro SHIPPED/DELIVERED e recebimento sem polling | TRANSCRICAO | [09:33] Marcos |
| PRD-C-02 | docs/PRD.md | Cenário | Manutenção planejada de 2 h coberta por retentativas | TRANSCRICAO | [09:16] Diego |
| PRD-C-03 | docs/PRD.md | Cenário | Rotação de secret após suspeita de vazamento em log | TRANSCRICAO | [09:22] Diego |
| PRD-C-04 | docs/PRD.md | Cenário | Cliente descarta duplicata via `X-Event-Id` | TRANSCRICAO | [09:25] Diego |
| PRD-C-05 | docs/PRD.md | Cenário | Cliente consulta as últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-C-06 | docs/PRD.md | Cenário | ADMIN faz replay de evento da DLQ com auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-OBJ-01 | docs/PRD.md | Objetivo/Métrica | Latência < 10 s entre a mudança e a 1ª tentativa | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo/Métrica | 3 de 3 clientes com webhook ativo até o fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Objetivo/Métrica | 0 mudanças de status commitadas sem evento | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-04 | docs/PRD.md | Objetivo/Métrica | Janela de retentativas ≈ 14,6 h | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-05 | docs/PRD.md | Objetivo/Métrica | Queda no polling de `GET /orders` (linha de base a medir) | TRANSCRICAO | [09:00] Marcos |
| PRD-OOS-01 | docs/PRD.md | Fora de Escopo | E-mail ao cliente quando o webhook falha: próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de Escopo | Dashboard visual: projeto separado do time de frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de Escopo | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de Escopo | Webhooks inbound: só outbound | TRANSCRICAO | [09:02] Marcos |
| PRD-OOS-05 | docs/PRD.md | Fora de Escopo | Ordenação global e múltiplos workers | TRANSCRICAO | [09:13] Larissa |
| PRD-OOS-06 | docs/PRD.md | Fora de Escopo | Exactly-once descartado | TRANSCRICAO | [09:25] Diego |
| PRD-OOS-07 | docs/PRD.md | Fora de Escopo | Arquivamento de linhas entregues (~30 dias) | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-08 | docs/PRD.md | Fora de Escopo | Itens do pedido no payload | TRANSCRICAO | [09:43] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook: url + lista de status; secret gerada pela plataforma e devolvida | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | `customer_id` informado na requisição, não derivado do JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Listar, editar (PATCH) e remover (DELETE) webhooks | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Filtro de status por webhook | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | POST para cada webhook ativo interessado quando o status muda | TRANSCRICAO | [09:06] Diego |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Campos do payload; sem itens | TRANSCRICAO | [09:43] Diego |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Evento é snapshot do momento da mudança | TRANSCRICAO | [09:52] Larissa |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Headers X-Event-Id, X-Signature, X-Timestamp, Content-Type | TRANSCRICAO | [09:44] Diego |
| PRD-FR-08b | docs/PRD.md | Requisito Funcional | Header X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 do corpo com secret exclusiva do endpoint | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Rotação de secret com validade paralela de 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | 5 retentativas em 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | DLQ com payload, motivo e horário | TRANSCRICAO | [09:18] Diego |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Replay manual por ADMIN, com registro de quem fez | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-15 | docs/PRD.md | Requisito Funcional | URL http recusada com erro de validação | TRANSCRICAO | [09:23] Sofia |
| PRD-FR-16 | docs/PRD.md | Requisito Funcional | Status não assinado por nenhum webhook não gera evento | TRANSCRICAO | [09:34] Bruno |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência < 10 s; captura a cada 2 s | TRANSCRICAO | [09:10] Larissa |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Atomicidade entre status e evento | TRANSCRICAO | [09:06] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Cliente lento não afeta mudança de status | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | At-least-once com dedup por X-Event-Id | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-05 | docs/PRD.md | Restrição | Ordenação só por pedido e só com single-worker | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 s | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Limite de 64 KB, erro se ultrapassar | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | CRUD aberto a qualquer usuário autenticado | TRANSCRICAO | [09:37] Sofia |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Processamento em processo separado | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-10 | docs/PRD.md | Restrição | Sem infraestrutura nova; mesmo MySQL e mesma stack | TRANSCRICAO | [09:07] Diego |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Padrões da plataforma e prefixo WEBHOOK_ | TRANSCRICAO | [09:29] Larissa |
| PRD-DEC-01 | docs/PRD.md | Trade-off | Tabela de decisões e trade-offs consolidada a partir dos ADR-001..007 | TRANSCRICAO | [09:48] Larissa |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança de 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Documentação no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-03 | docs/PRD.md | Dependência | Comunicação do prazo aos clientes | TRANSCRICAO | [09:47] Marcos |
| PRD-DEP-04 | docs/PRD.md | Dependência | Sessão de revisão do design com Bruno e Diego | TRANSCRICAO | [09:50] Larissa |
| PRD-DEP-05 | docs/PRD.md | Dependência | Adaptação dos clientes (https, HMAC, dedup, 2xx em 10 s) | TRANSCRICAO | [09:25] Diego |
| PRD-DEP-06 | docs/PRD.md | Dependência | Usuários da plataforma representando clientes | TRANSCRICAO | [09:32] Marcos |
| PRD-CRON-01 | docs/PRD.md | Restrição | Estimativa de 3 sprints e quebra por bloco | TRANSCRICAO | [09:46] Larissa |
| PRD-CRON-02 | docs/PRD.md | Restrição | Revisão de segurança incluída no fim das 3 sprints | TRANSCRICAO | [09:47] Larissa |
| PRD-RISK-01 | docs/PRD.md | Risco | Perder a Atlas por atraso (fim de novembro × 3 sprints) | TRANSCRICAO | [09:45] Marcos |
| PRD-RISK-02 | docs/PRD.md | Risco | Cliente processa duplicata | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-03 | docs/PRD.md | Risco | Vazamento de secret pelo cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-04 | docs/PRD.md | Risco | Rajadas de eventos (50 pedidos/min) | TRANSCRICAO | [09:38] Diego |
| PRD-RISK-05 | docs/PRD.md | Risco | Eventos fora de ordem em retry | TRANSCRICAO | [09:12] Diego |
| PRD-RISK-06 | docs/PRD.md | Risco | Cliente fora do ar por mais de ~15 h vai para a DLQ | TRANSCRICAO | [09:17] Marcos |
| PRD-AC-01 | docs/PRD.md | Critério de Aceite | Cadastro https com filtro devolve a secret | TRANSCRICAO | [09:31] Marcos |
| PRD-AC-02 | docs/PRD.md | Critério de Aceite | Entrega em < 10 s com headers e assinatura | TRANSCRICAO | [09:02] Marcos |
| PRD-AC-03 | docs/PRD.md | Critério de Aceite | Status não assinado não é enviado | TRANSCRICAO | [09:34] Bruno |
| PRD-AC-04 | docs/PRD.md | Critério de Aceite | http recusado | TRANSCRICAO | [09:23] Sofia |
| PRD-AC-05 | docs/PRD.md | Critério de Aceite | Retentativas e DLQ | TRANSCRICAO | [09:17] Larissa |
| PRD-AC-06 | docs/PRD.md | Critério de Aceite | Replay ADMIN com o mesmo X-Event-Id; não-ADMIN negado | TRANSCRICAO | [09:36] Larissa |
| PRD-AC-07 | docs/PRD.md | Critério de Aceite | Secret antiga válida por 24 h após a rotação | TRANSCRICAO | [09:21] Sofia |
| PRD-AC-08 | docs/PRD.md | Critério de Aceite | Histórico com no máximo 100 tentativas | TRANSCRICAO | [09:34] Marcos |
| PRD-AC-09 | docs/PRD.md | Critério de Aceite | Falha no evento impede a mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-AC-10 | docs/PRD.md | Critério de Aceite | Revisão de segurança aprovada antes do deploy | TRANSCRICAO | [09:49] Sofia |
| PRD-TEST-01 | docs/PRD.md | Restrição | Testes em níveis (unitário, API contra banco de teste, atomicidade, ponta a ponta) sobre a suíte existente | CODIGO | tests/orders.test.ts |
| PRD-TEST-02 | docs/PRD.md | Dependência | Revisão de segurança de HMAC e geração de secret | TRANSCRICAO | [09:46] Sofia |
| RFC-META-01 | docs/RFC.md | Contexto | Revisores = participantes; doc aberto pela Tech Lead para revisão | TRANSCRICAO | [09:50] Larissa |
| RFC-TLDR-01 | docs/RFC.md | Decisão | Resumo das decisões confirmado no fechamento da reunião | TRANSCRICAO | [09:48] Larissa |
| RFC-CTX-01 | docs/RFC.md | Restrição | Escopo apenas outbound | TRANSCRICAO | [09:03] Sofia |
| RFC-CTX-02 | docs/RFC.md | Contexto | Transação de mudança de status já é pesada (orders, history, estoque) | CODIGO | src/modules/orders/order.service.ts |
| RFC-CTX-03 | docs/RFC.md | Restrição | Time pequeno, operação simples | TRANSCRICAO | [09:07] Diego |
| RFC-PROP-01 | docs/RFC.md | Decisão | Publicação transacional via `publishWebhookEvent(tx, …)` | TRANSCRICAO | [09:41] Bruno |
| RFC-PROP-02 | docs/RFC.md | Decisão | Worker `src/worker.ts` + `npm run worker` | TRANSCRICAO | [09:11] Larissa |
| RFC-PROP-03 | docs/RFC.md | Decisão | Timeout de 10 s, retry, DLQ, at-least-once | TRANSCRICAO | [09:42] Diego |
| RFC-PROP-04 | docs/RFC.md | Decisão | Segurança: https, HMAC, 64 KB | TRANSCRICAO | [09:22] Sofia |
| RFC-PROP-05 | docs/RFC.md | Decisão | Módulo `src/modules/webhooks` com CRUD aberto e replay ADMIN | TRANSCRICAO | [09:27] Bruno |
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | HTTP síncrono no changeStatus | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Redis Streams / broker | TRANSCRICAO | [09:07] Larissa |
| RFC-ALT-03 | docs/RFC.md | Alternativa Descartada | Trigger MySQL para acordar o worker | TRANSCRICAO | [09:09] Bruno |
| RFC-ALT-04 | docs/RFC.md | Alternativa Descartada | Retry indefinido ou só 3 tentativas | TRANSCRICAO | [09:16] Bruno |
| RFC-ALT-05 | docs/RFC.md | Alternativa Descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa Descartada | Renderizar o payload no envio | TRANSCRICAO | [09:51] Bruno |
| RFC-Q-01 | docs/RFC.md | Questão em Aberto | Rate limiting de saída | TRANSCRICAO | [09:39] Diego |
| RFC-Q-02 | docs/RFC.md | Questão em Aberto | `customer_id` no body ou no path | TRANSCRICAO | [09:32] Larissa |
| RFC-Q-03 | docs/RFC.md | Questão em Aberto | Escala multi-worker e inversão de ordem em retry | TRANSCRICAO | [09:13] Diego |
| RFC-Q-04 | docs/RFC.md | Questão em Aberto | "5 tentativas": retentativas ou envios no total | TRANSCRICAO | [09:48] Larissa |
| RFC-Q-05 | docs/RFC.md | Questão em Aberto | Formato da dupla assinatura no grace period | TRANSCRICAO | [09:21] Sofia |
| RFC-Q-06 | docs/RFC.md | Questão em Aberto | Secret em repouso, X-Timestamp não assinado, SSRF: para a revisão de segurança | TRANSCRICAO | [09:46] Sofia |
| RFC-Q-07 | docs/RFC.md | Questão em Aberto | Notificar criação (PENDING)? Nenhuma transição leva a PENDING | CODIGO | src/modules/orders/order.status.ts |
| RFC-Q-08 | docs/RFC.md | Questão em Aberto | Endurecer roles do CRUD | TRANSCRICAO | [09:37] Sofia |
| RFC-IMP-01 | docs/RFC.md | Trade-off | `PATCH /orders/:id/status` passa a depender da outbox | TRANSCRICAO | [09:40] Bruno |
| RFC-IMP-02 | docs/RFC.md | Trade-off | Crescimento contínuo da outbox (arquivamento fora de escopo) | TRANSCRICAO | [09:08] Diego |
| RFC-IMP-03 | docs/RFC.md | Contexto | Novo processo a operar | TRANSCRICAO | [09:11] Diego |
| RFC-IMP-04 | docs/RFC.md | Dependência | Clientes validam HMAC e deduplicam; documentação no portal | TRANSCRICAO | [09:26] Marcos |
| RFC-RISK-01 | docs/RFC.md | Risco | Prazo: fim de novembro × 3 sprints | TRANSCRICAO | [09:46] Larissa |
| RFC-RISK-02 | docs/RFC.md | Risco | Falha na outbox derruba mudança de status | TRANSCRICAO | [09:41] Diego |
| RFC-RISK-03 | docs/RFC.md | Risco | Cliente não deduplica | TRANSCRICAO | [09:25] Sofia |
| RFC-RISK-04 | docs/RFC.md | Risco | Worker parado acumula atraso | TRANSCRICAO | [09:11] Diego |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox no MySQL, na mesma transação de changeStatus | TRANSCRICAO | [09:08] Larissa |
| ADR-001-CTX | docs/adrs/ADR-001-outbox-no-mysql.md | Contexto | `changeStatus` em `$transaction` atualiza order, history e estoque | CODIGO | src/modules/orders/order.service.ts |
| ADR-001-D2 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Índices em status e created_at; leitura em batch pequeno | TRANSCRICAO | [09:08] Diego |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Disparo síncrono | TRANSCRICAO | [09:06] Diego |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Redis Streams: overengineering | TRANSCRICAO | [09:07] Diego |
| ADR-001-TO | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Consistência e simplicidade em vez de throughput de broker; tabela cresce | TRANSCRICAO | [09:08] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker separado, polling de 2 s | TRANSCRICAO | [09:10] Larissa |
| ADR-002-D2 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Processo separado; mesmo banco, PrismaClient próprio | TRANSCRICAO | [09:11] Diego |
| ADR-002-D3 | docs/adrs/ADR-002-worker-separado-em-polling.md | Restrição | Single-worker; ordenação por `order_id` só nessa condição | TRANSCRICAO | [09:12] Diego |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa Descartada | Trigger MySQL (sem NOTIFY/LISTEN) | TRANSCRICAO | [09:09] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa Descartada | Worker no processo da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT-03 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa Descartada | Múltiplos workers (particionar ou lock) | TRANSCRICAO | [09:13] Diego |
| ADR-002-TO | docs/adrs/ADR-002-worker-separado-em-polling.md | Trade-off | Latência mínima de 2 s aceita | TRANSCRICAO | [09:10] Larissa |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | 5 retentativas, backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| ADR-003-D2 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Interpretação: envio inicial + 5 retentativas (~15 h) | TRANSCRICAO | [09:17] Diego |
| ADR-003-D3 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | DLQ em tabela separada com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| ADR-003-D4 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Replay manual via endpoint admin | TRANSCRICAO | [09:18] Diego |
| ADR-003-ALT-01 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Alternativa Descartada | Retry indefinido | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-02 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Alternativa Descartada | 3 tentativas | TRANSCRICAO | [09:16] Diego |
| ADR-003-ALT-03 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Alternativa Descartada | Marcar `failed` na própria outbox | TRANSCRICAO | [09:17] Larissa |
| ADR-003-TO | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Trade-off | Até 15 h para falha definitiva; 15 h de queda aceitável pelo produto | TRANSCRICAO | [09:17] Marcos |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação de 24 h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-D2 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Assinatura no header X-Signature | TRANSCRICAO | [09:20] Sofia |
| ADR-004-D3 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | Config armazena url, secret, customer_id, ativo | TRANSCRICAO | [09:21] Bruno |
| ADR-004-D4 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Restrição | TLS obrigatório (https) | TRANSCRICAO | [09:23] Sofia |
| ADR-004-ALT-01 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Alternativa Descartada | Secret global | TRANSCRICAO | [09:21] Sofia |
| ADR-004-ALT-02 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Alternativa Descartada | Só TLS, sem assinatura (plausível) | TRANSCRICAO | [09:19] Sofia |
| ADR-004-CONS | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Redação de logs atual não cobre `secret` | CODIGO | src/shared/logger/index.ts |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id para dedup | TRANSCRICAO | [09:26] Larissa |
| ADR-005-D2 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | X-Event-Id é UUID gerado na entrada da outbox | TRANSCRICAO | [09:25] Diego |
| ADR-005-ALT-01 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Alternativa Descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| ADR-005-TO | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Trade-off | Responsabilidade de dedup transferida ao cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Reuso máximo: AppError, Pino, error middleware, módulos, Zod, códigos | TRANSCRICAO | [09:30] Larissa |
| ADR-006-CTX-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Contexto | Padrão de módulo controller/service/repository/routes/schemas | CODIGO | src/modules/orders/order.routes.ts |
| ADR-006-CTX-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Contexto | Hierarquia AppError com errorCode | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-CTX-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Contexto | `requireRole` existente | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-006-D2 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Prefixo WEBHOOK_ em todos os códigos de erro | TRANSCRICAO | [09:28] Bruno |
| ADR-006-D3 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Função `publishWebhookEvent(tx, …)` em vez de injetar repository | TRANSCRICAO | [09:41] Diego |
| ADR-006-ALT-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Alternativa Descartada | Injetar WebhookRepository no OrderService | TRANSCRICAO | [09:41] Diego |
| ADR-006-ALT-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Alternativa Descartada | Reusar NotFoundError genérico (código fixo NOT_FOUND) | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-ALT-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Alternativa Descartada | Compartilhar PrismaClient com a API | TRANSCRICAO | [09:30] Bruno |
| ADR-006-TO | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Trade-off | validate.middleware converte ZodError em VALIDATION_ERROR | CODIGO | src/middlewares/validate.middleware.ts |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Decisão | Payload renderizado (snapshot) na inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-007-D2 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Decisão | Filtro de status aplicado na inserção | TRANSCRICAO | [09:34] Bruno |
| ADR-007-D3 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Decisão | Payload enxuto, sem itens | TRANSCRICAO | [09:44] Bruno |
| ADR-007-ALT-01 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Alternativa Descartada | Guardar só order_id e renderizar no envio | TRANSCRICAO | [09:51] Bruno |
| ADR-007-ALT-02 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Alternativa Descartada | Filtrar no envio | TRANSCRICAO | [09:34] Diego |
| FDD-CTX-01 | docs/FDD.md | Contexto | Passos atuais de `changeStatus` (find, valida, estoque, update, history, reload) | CODIGO | src/modules/orders/order.service.ts |
| FDD-CTX-02 | docs/FDD.md | Contexto | `canTransition`, `shouldDebitStock`, `shouldReplenishStock` | CODIGO | src/modules/orders/order.status.ts |
| FDD-OT-01 | docs/FDD.md | Objetivo/Métrica | 0 status commitados sem linha na outbox | TRANSCRICAO | [09:41] Diego |
| FDD-OT-02 | docs/FDD.md | Objetivo/Métrica | Commit → 1ª tentativa < 10 s | TRANSCRICAO | [09:02] Marcos |
| FDD-OT-03 | docs/FDD.md | Objetivo/Métrica | Cliente lento não afeta PATCH de status | TRANSCRICAO | [09:04] Bruno |
| FDD-OT-04 | docs/FDD.md | Objetivo/Métrica | 5 retentativas em ~14,6 h e depois DLQ | TRANSCRICAO | [09:17] Diego |
| FDD-OT-05 | docs/FDD.md | Objetivo/Métrica | 100% das requisições assinadas | TRANSCRICAO | [09:22] Sofia |
| FDD-OT-06 | docs/FDD.md | Restrição | Zero infraestrutura e dependências novas | TRANSCRICAO | [09:07] Diego |
| FDD-DADOS-01 | docs/FDD.md | Decisão | Model `Webhook` (customerId, url, secret, events, active) | TRANSCRICAO | [09:21] Bruno |
| FDD-DADOS-02 | docs/FDD.md | Decisão | Enum de status da outbox: pendente/processando/entregue/falhou | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-03 | docs/FDD.md | Decisão | IDs UUID Char(36) | TRANSCRICAO | [09:51] Larissa |
| FDD-DADOS-04 | docs/FDD.md | Restrição | Convenções de schema (uuid, @@map, timestamps) | CODIGO | prisma/schema.prisma |
| FDD-DADOS-05 | docs/FDD.md | Decisão | `payload` em MEDIUMTEXT para manter bytes idênticos (HMAC sobre o corpo) | TRANSCRICAO | [09:22] Sofia |
| FDD-DADOS-06 | docs/FDD.md | Decisão | Tabela `webhook_dead_letter` (payload, motivo, timestamp, replay) | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-07 | docs/FDD.md | Proposta | Tabela `webhook_deliveries` para o histórico | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-08 | docs/FDD.md | Decisão | `eventId` separado do `id` para o replay preservar o X-Event-Id | TRANSCRICAO | [09:25] Diego |
| FDD-FLUXO-01 | docs/FDD.md | Decisão | `publishWebhookEvent` chamado após `orderStatusHistory.create` no mesmo tx | TRANSCRICAO | [09:41] Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Decisão | Filtra webhooks ativos do customer que assinam `to_status` | TRANSCRICAO | [09:34] Bruno |
| FDD-FLUXO-03 | docs/FDD.md | Restrição | Erro na outbox faz rollback de tudo | TRANSCRICAO | [09:40] Bruno |
| FDD-FLUXO-04 | docs/FDD.md | Integração | `uuid` v4 já é dependência do projeto | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-FLUXO-05 | docs/FDD.md | Integração | Worker usa `createPrismaClient()` | CODIGO | src/config/database.ts |
| FDD-FLUXO-06 | docs/FDD.md | Proposta | Recuperação PROCESSING → PENDING no startup | TRANSCRICAO | [09:24] Diego |
| FDD-FLUXO-07 | docs/FDD.md | Integração | Shutdown SIGINT/SIGTERM espelhando o servidor | CODIGO | src/server.ts |
| FDD-FLUXO-08 | docs/FDD.md | Decisão | Query de polling a cada 2 s, por created_at, em lote | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-09 | docs/FDD.md | Decisão | Processamento sequencial para manter a ordem | TRANSCRICAO | [09:12] Diego |
| FDD-FLUXO-10 | docs/FDD.md | Proposta | Payload > 64 KB verificado no worker → DLQ sem retry | TRANSCRICAO | [09:23] Sofia |
| FDD-FLUXO-11 | docs/FDD.md | Decisão | Tabela de backoff e `RETRY_DELAYS_MS` | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-12 | docs/FDD.md | Decisão | Movimento para a DLQ em transação | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-13 | docs/FDD.md | Decisão | Replay recoloca na outbox como PENDING | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-14 | docs/FDD.md | Requisito Não Funcional | Replay registra quem fez (auditoria) | TRANSCRICAO | [09:36] Sofia |
| FDD-FLUXO-15 | docs/FDD.md | Decisão | Rotação: previousSecret válida por 24 h | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | `POST /webhooks` (201 com secret) | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | `GET /webhooks?customerId=` (lista paginada) | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | `PATCH /webhooks/:id` | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | `DELETE /webhooks/:id` (204) | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | `POST /webhooks/:id/rotate-secret` | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | `GET /webhooks/:id/deliveries` (últimas 100) | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | `POST /admin/webhooks/dead-letter/:id/replay` (ADMIN, 202) | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Envelope de erro `{ error: { code, message, details } }` | CODIGO | src/middlewares/error.middleware.ts |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Resposta paginada `{ data, pagination }` | CODIGO | src/shared/http/response.ts |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | Corpo do evento outbound | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-11 | docs/FDD.md | Contrato | Headers X-Event-Id/X-Signature/X-Timestamp/Content-Type | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-12 | docs/FDD.md | Contrato | Header X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| FDD-CONTRATO-13 | docs/FDD.md | Contrato | Assinatura `sha256=<hex>` (HMAC do corpo) | TRANSCRICAO | [09:20] Sofia |
| FDD-CONTRATO-14 | docs/FDD.md | Contrato | `order_number` no formato `ORD-NNNNNN` | CODIGO | src/modules/orders/order.service.ts |
| FDD-CONTRATO-15 | docs/FDD.md | Restrição | CRUD com qualquer role autenticada; replay só ADMIN | TRANSCRICAO | [09:37] Sofia |
| FDD-ERR-01 | docs/FDD.md | Erro | `WEBHOOK_NOT_FOUND` (404) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Erro | `WEBHOOK_INVALID_URL` (400) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-03 | docs/FDD.md | Erro | `WEBHOOK_SECRET_REQUIRED` (worker, → DLQ) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-04 | docs/FDD.md | Erro | `WEBHOOK_CUSTOMER_NOT_FOUND` (404) | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-05 | docs/FDD.md | Erro | `WEBHOOK_INVALID_EVENTS` (400) | TRANSCRICAO | [09:33] Marcos |
| FDD-ERR-06 | docs/FDD.md | Erro | `WEBHOOK_INACTIVE` (409 / → DLQ) | TRANSCRICAO | [09:21] Bruno |
| FDD-ERR-07 | docs/FDD.md | Erro | `WEBHOOK_SECRET_ROTATION_IN_PROGRESS` (409, proposta) | TRANSCRICAO | [09:21] Sofia |
| FDD-ERR-08 | docs/FDD.md | Erro | `WEBHOOK_DEAD_LETTER_NOT_FOUND` (404) | TRANSCRICAO | [09:35] Diego |
| FDD-ERR-09 | docs/FDD.md | Erro | `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` (409) | TRANSCRICAO | [09:18] Diego |
| FDD-ERR-10 | docs/FDD.md | Erro | `WEBHOOK_OUTBOX_INSERT_FAILED` (500, rollback), proposta P-14 | TRANSCRICAO | [09:40] Bruno |
| FDD-ERR-11 | docs/FDD.md | Erro | `WEBHOOK_DELIVERY_TIMEOUT` (retentável) | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-12 | docs/FDD.md | Erro | `WEBHOOK_DELIVERY_NETWORK_ERROR` / `WEBHOOK_DELIVERY_HTTP_ERROR` (retentáveis) | TRANSCRICAO | [09:15] Diego |
| FDD-ERR-13 | docs/FDD.md | Erro | `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` (→ DLQ) | TRANSCRICAO | [09:17] Larissa |
| FDD-ERR-14 | docs/FDD.md | Erro | `WEBHOOK_PAYLOAD_TOO_LARGE` (→ DLQ) | TRANSCRICAO | [09:24] Diego |
| FDD-ERR-15 | docs/FDD.md | Restrição | `NotFoundError` fixa `NOT_FOUND`; `ConflictError` aceita code | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERR-16 | docs/FDD.md | Proposta | Regra https no service para emitir WEBHOOK_INVALID_URL (validate gera VALIDATION_ERROR) | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-RES-01 | docs/FDD.md | Requisito Não Funcional | Timeout de 10 s via AbortSignal | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Decisão | Retentativas 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-RES-03 | docs/FDD.md | Decisão | Fallback: DLQ + replay manual | TRANSCRICAO | [09:18] Diego |
| FDD-RES-04 | docs/FDD.md | Fora de Escopo | Sem fallback por e-mail nesta fase | TRANSCRICAO | [09:37] Larissa |
| FDD-RES-05 | docs/FDD.md | Decisão | HTTP fora da transação; worker isolado | TRANSCRICAO | [09:04] Bruno |
| FDD-RES-06 | docs/FDD.md | Proposta | Redirects não seguidos | TRANSCRICAO | [09:46] Sofia |
| FDD-RES-07 | docs/FDD.md | Fora de Escopo | Rate limiting não implementado | TRANSCRICAO | [09:39] Larissa |
| FDD-OBS-01 | docs/FDD.md | Restrição | Logs com Pino; nada novo | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Requisito Não Funcional | Redigir `*.secret` nos logs (vazamento anterior) | TRANSCRICAO | [09:22] Diego |
| FDD-OBS-03 | docs/FDD.md | Integração | `redactPaths` atual não inclui secret | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-04 | docs/FDD.md | Objetivo/Métrica | Alerta se o evento pendente mais antigo tiver > 10 s | TRANSCRICAO | [09:02] Marcos |
| FDD-OBS-05 | docs/FDD.md | Objetivo/Métrica | Eventos/min por webhook como insumo para rate limiting | TRANSCRICAO | [09:38] Diego |
| FDD-OBS-06 | docs/FDD.md | Decisão | Tracing por IDs de correlação (event_id ↔ X-Event-Id) | TRANSCRICAO | [09:25] Diego |
| FDD-OBS-07 | docs/FDD.md | Restrição | requestId não chega ao service, por isso não é propagado à outbox | CODIGO | src/modules/orders/order.controller.ts |
| FDD-OBS-08 | docs/FDD.md | Restrição | Log de auditoria do replay (quem fez) | TRANSCRICAO | [09:36] Sofia |
| FDD-INT-01 | docs/FDD.md | Integração | Hook em `changeStatus` após o history, mesmo `TxClient` | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | `events` validado com `z.nativeEnum(OrderStatus)`; PENDING inalcançável | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Classes Webhook*Error estendem AppError/ConflictError | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-04 | docs/FDD.md | Integração | errorMiddleware sem alteração; P2025 vira NOT_FOUND genérico | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | `authenticate` + `requireRole('ADMIN')` no replay | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Padrão de rota com requireRole | CODIGO | src/modules/users/user.routes.ts |
| FDD-INT-07 | docs/FDD.md | Integração | Montar `/webhooks` e `/admin/webhooks` em `buildApiRouter` | CODIGO | src/routes/index.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Instanciar repository/service/controllers em `buildControllers` | CODIGO | src/app.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Novas variáveis no `envSchema`; worker exige o mesmo `.env` (JWT_SECRET) | CODIGO | src/config/env.ts |
| FDD-INT-10 | docs/FDD.md | Integração | Scripts `worker` / `start:worker` espelhando `dev` / `start` | CODIGO | package.json |
| FDD-INT-11 | docs/FDD.md | Integração | Novos models, relação `Customer.webhooks` e migration | CODIGO | prisma/schema.prisma |
| FDD-INT-12 | docs/FDD.md | Integração | Limpar as novas tabelas antes de `customer.deleteMany()` | CODIGO | tests/setup.ts |
| FDD-INT-13 | docs/FDD.md | Integração | Factory `createTestWebhook` | CODIGO | tests/helpers/factories.ts |
| FDD-INT-14 | docs/FDD.md | Decisão | Entry-point `src/worker.ts` no padrão de `src/server.ts` | TRANSCRICAO | [09:11] Larissa |
| FDD-INT-15 | docs/FDD.md | Decisão | Lógica em `src/modules/webhooks/webhook.worker.ts` | TRANSCRICAO | [09:28] Bruno |
| FDD-DEP-01 | docs/FDD.md | Dependência | Node ≥ 20 (fetch, crypto); nenhum pacote novo | CODIGO | package.json |
| FDD-DEP-02 | docs/FDD.md | Dependência | MySQL 8.0 | CODIGO | docker-compose.yml |
| FDD-DEP-03 | docs/FDD.md | Dependência | Ordem de deploy: migration → API → worker; eventos acumulam sem perda | TRANSCRICAO | [09:06] Diego |
| FDD-DEP-04 | docs/FDD.md | Dependência | Revisão de segurança antes do deploy | TRANSCRICAO | [09:49] Sofia |
| FDD-RISK-01 | docs/FDD.md | Risco | Falha na outbox bloqueia mudança de status | TRANSCRICAO | [09:40] Bruno |
| FDD-RISK-02 | docs/FDD.md | Risco | Worker parado estoura o SLA | TRANSCRICAO | [09:11] Diego |
| FDD-RISK-03 | docs/FDD.md | Risco | Duplicatas no cliente | TRANSCRICAO | [09:24] Diego |
| FDD-RISK-04 | docs/FDD.md | Risco | Inversão de ordem em retry | TRANSCRICAO | [09:12] Diego |
| FDD-RISK-05 | docs/FDD.md | Risco | Vazamento de secret em logs | TRANSCRICAO | [09:22] Diego |
| FDD-RISK-06 | docs/FDD.md | Risco | Crescimento das tabelas | TRANSCRICAO | [09:08] Diego |
| FDD-RISK-07 | docs/FDD.md | Risco | Rajada de eventos | TRANSCRICAO | [09:38] Diego |
| FDD-RISK-08 | docs/FDD.md | Risco | SSRF via URL interna (para a revisão de segurança) | TRANSCRICAO | [09:46] Sofia |
| FDD-RISK-09 | docs/FDD.md | Risco | Prazo de 3 sprints | TRANSCRICAO | [09:47] Larissa |
| FDD-AC-01 | docs/FDD.md | Critério de Aceite | 1 linha na outbox por webhook interessado, no mesmo tx | TRANSCRICAO | [09:34] Bruno |
| FDD-AC-02 | docs/FDD.md | Critério de Aceite | Rollback total se publishWebhookEvent falhar | TRANSCRICAO | [09:40] Bruno |
| FDD-AC-03 | docs/FDD.md | Critério de Aceite | Status não assinado não gera linha | TRANSCRICAO | [09:34] Bruno |
| FDD-AC-04 | docs/FDD.md | Critério de Aceite | Entrega em ≤ 2 s + tempo de resposta | TRANSCRICAO | [09:09] Diego |
| FDD-AC-05 | docs/FDD.md | Critério de Aceite | Corpo idêntico ao payload e assinatura conferível | TRANSCRICAO | [09:22] Sofia |
| FDD-AC-06 | docs/FDD.md | Critério de Aceite | Os 5 headers presentes | TRANSCRICAO | [09:44] Diego |
| FDD-AC-07 | docs/FDD.md | Critério de Aceite | Backoff e DLQ após o 6º envio | TRANSCRICAO | [09:17] Diego |
| FDD-AC-08 | docs/FDD.md | Critério de Aceite | > 64 KB → DLQ | TRANSCRICAO | [09:24] Larissa |
| FDD-AC-09 | docs/FDD.md | Critério de Aceite | Replay: OPERATOR 403, ADMIN 202 com auditoria | TRANSCRICAO | [09:36] Larissa |
| FDD-AC-10 | docs/FDD.md | Critério de Aceite | Dupla assinatura por 24 h após a rotação | TRANSCRICAO | [09:21] Sofia |
| FDD-AC-11 | docs/FDD.md | Critério de Aceite | http → 400; secret nunca em GET/PATCH | TRANSCRICAO | [09:23] Sofia |
| FDD-AC-12 | docs/FDD.md | Critério de Aceite | Histórico com no máximo 100 itens | TRANSCRICAO | [09:34] Marcos |
| FDD-AC-13 | docs/FDD.md | Critério de Aceite | Crash no PROCESSING → reentrega com o mesmo X-Event-Id | TRANSCRICAO | [09:24] Diego |
| FDD-AC-14 | docs/FDD.md | Critério de Aceite | lint e testes verdes; setup limpa as tabelas | CODIGO | tests/setup.ts |
| FDD-AC-15 | docs/FDD.md | Critério de Aceite | Revisão de segurança aprovada | TRANSCRICAO | [09:46] Sofia |
| FDD-PROP-01 | docs/FDD.md | Proposta | customerId no body (POST) e na query (GET) | TRANSCRICAO | [09:32] Larissa |
| FDD-PROP-02 | docs/FDD.md | Proposta | Tabela webhook_deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-PROP-03 | docs/FDD.md | Proposta | DELETE como soft delete | TRANSCRICAO | [09:33] Bruno |
| FDD-PROP-04 | docs/FDD.md | Proposta | Limite de 64 KB checado no worker | TRANSCRICAO | [09:23] Sofia |
| FDD-PROP-05 | docs/FDD.md | Proposta | X-Signature com duas assinaturas no grace period | TRANSCRICAO | [09:21] Sofia |
| FDD-PROP-06 | docs/FDD.md | Proposta | Bloquear nova rotação dentro do grace period | TRANSCRICAO | [09:21] Sofia |
| FDD-PROP-07 | docs/FDD.md | Proposta | Recuperação de órfãos no startup | TRANSCRICAO | [09:24] Diego |
| FDD-PROP-08 | docs/FDD.md | Proposta | WEBHOOK_BATCH_SIZE configurável (default a definir) | TRANSCRICAO | [09:08] Diego |
| FDD-PROP-09 | docs/FDD.md | Proposta | responseBody truncado em 4 KB | TRANSCRICAO | [09:34] Marcos |
| FDD-PROP-10 | docs/FDD.md | Proposta | Rejeitar PENDING em `events` | CODIGO | src/modules/orders/order.status.ts |
| FDD-PROP-11 | docs/FDD.md | Proposta | https validado no service | TRANSCRICAO | [09:23] Sofia |
| FDD-PROP-12 | docs/FDD.md | Proposta | Redirects não seguidos | TRANSCRICAO | [09:46] Sofia |
| FDD-PROP-13 | docs/FDD.md | Proposta | Arquivos webhook.publisher.ts / webhook.errors.ts; webhook.worker.ts (worker ou processor) | TRANSCRICAO | [09:28] Bruno |
| FDD-PROP-14 | docs/FDD.md | Proposta | Encapsular erro da outbox em WebhookOutboxInsertError | TRANSCRICAO | [09:40] Bruno |
| FDD-PROP-15 | docs/FDD.md | Proposta | Detalhes de contrato (rotate path, sha256=, ISO 8601, 202, paginação) | TRANSCRICAO | [09:44] Diego |
| FDD-PROP-16 | docs/FDD.md | Proposta | Sucesso = HTTP 2xx | TRANSCRICAO | [09:42] Diego |
| FDD-PROP-17 | docs/FDD.md | Proposta | Replay mantém o mesmo eventId (inferência) | TRANSCRICAO | [09:18] Diego |
| ADR-003-PEND | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Questão em Aberto | Contagem "5 tentativas" pendente de confirmação | TRANSCRICAO | [09:48] Larissa |
| ADR-004-DIV | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Questão em Aberto | Divergência entre https no Zod e WEBHOOK_INVALID_URL | TRANSCRICAO | [09:23] Sofia |

## Resumo de cobertura

Os números abaixo foram gerados por um script de validação que lê esta tabela, a `TRANSCRICAO.md` e o repositório (ver README, seção "Iterações e ajustes").

| Verificação | Resultado |
|---|---|
| Total de linhas | 298 |
| Linhas com Fonte = TRANSCRICAO | 257 (86%). Em todas elas o par `[hh:mm] Nome` foi conferido: o falante tem fala naquele minuto da transcrição |
| Linhas com Fonte = CODIGO | 41, todas com caminho existente no repositório |
| IDs explícitos nos documentos (PRD-FR/NFR, OBJ, RISK, ALT, Q, OT, P, R) com linha no tracker | 84 de 84 (100%) |
| Linhas inválidas (formato, fonte ou caminho) | 0 |

**Limite da verificação:** o script confere se o falante existe naquele timestamp, mas não confere se o conteúdo da fala é o resumido. Essa conferência semântica foi feita por uma revisão adversarial separada (ver README).
