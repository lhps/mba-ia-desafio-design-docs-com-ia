# Architectural Decision Records

Este diretório guarda os ADRs (Architectural Decision Records) do projeto. Cada decisão arquitetural relevante fica em um arquivo próprio, nomeado no formato `ADR-NNN-titulo-em-kebab-case.md`, no modelo MADR com as seções Status, Contexto, Decisão, Alternativas Consideradas e Consequências.

## Índice: Sistema de Webhooks de Notificação de Pedidos

| ADR | Decisão | Status |
|---|---|---|
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL existente | Aceito |
| [ADR-002](ADR-002-worker-separado-em-polling.md) | Worker em processo separado, polling de 2 s, single-worker | Aceito |
| [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) | Retry com backoff 1m/5m/30m/2h/12h e DLQ em tabela separada | Aceito |
| [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação com grace period de 24 h | Aceito (pendente de revisão de segurança) |
| [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md) | Entrega at-least-once com deduplicação por `X-Event-Id` | Aceito |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes.md) | Reuso dos padrões existentes do projeto | Aceito |
| [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md) | Payload em snapshot e filtro de eventos na inserção da outbox | Aceito |
