# Architectural Decision Records (ADRs)

Este diretório armazena as decisões arquiteturais aprovadas na reunião técnica para o **Sistema de Webhooks Outbound de Notificação de Pedidos**, seguindo o formato MADR (Markdown Architectural Decision Records).

| ADR | Título da Decisão | Status | Origem Principal |
| :--- | :--- | :---: | :--- |
| [ADR-001](ADR-001-outbox-no-mysql.md) | Adoção do Padrão Transactional Outbox no Banco de Dados MySQL | Aprovado | `[09:04]–[09:08] Bruno, Diego, Larissa` |
| [ADR-002](ADR-002-worker-separado-em-polling.md) | Execução de Worker em Processo Separado com Polling a Cada 2 Segundos | Aprovado | `[09:09]–[09:11] Diego, Larissa, Bruno` |
| [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md) | Política de Retry com Backoff Exponencial e Dead Letter Queue (DLQ) | Aprovado | `[09:15]–[09:18] Diego, Bruno, Larissa` |
| [ADR-004](ADR-004-autenticacao-hmac-sha256-secret-por-endpoint.md) | Autenticação por HMAC-SHA256 com Secret Única por Endpoint e Suporte a Rotação | Aprovado | `[09:19]–[09:24] Sofia, Diego, Bruno` |
| [ADR-005](ADR-005-garantia-at-least-once-com-x-event-id.md) | Garantia de Entrega At-Least-Once com Idempotência via Header `X-Event-Id` | Aprovado | `[09:24]–[09:26] Diego, Sofia, Marcos` |
| [ADR-006](ADR-006-reuso-padroes-existentes-do-projeto.md) | Reuso dos Padrões Arquiteturais, Estruturais e de Observabilidade Existentes | Aprovado | `[09:27]–[09:30] Bruno, Diego, Larissa` |
