# Matriz de Rastreabilidade e Proveniência (TRACKER)

| Metadado | Detalhe |
| :--- | :--- |
| **Documento** | Matriz de Rastreabilidade Técnica e de Requisitos |
| **Projeto** | Sistema de Webhooks Outbound para Notificação de Pedidos |
| **Cobertura** | `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` e `docs/adrs/ADR-001` a `ADR-006` |
| **Fontes Rastreadas** | `TRANSCRICAO.md` e Código-Fonte Real do Repositório |

---

## 1. Visão Geral da Matriz

Esta matriz mapeia cada requisito funcional, requisito não funcional, exclusão de escopo, decisão arquitetural, contrato público, regra de resiliência, requisito de observabilidade, dependência e risco de implementação à sua fonte original na transcrição da reunião (`TRANSCRICAO`) ou nos arquivos concretos da base de código (`CODIGO`).

---

## 2. Tabela de Rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **TRK-01** | `docs/PRD.md` | Requisito de Negócio | Demanda dos clientes B2B Atlas Comercial, MaxDistribuição e Nova Cargo por notificações e risco de churn | `TRANSCRICAO` | `[09:00] Marcos` |
| **TRK-02** | `docs/PRD.md` | Meta Quantitativa | Latência máxima aceitável inferior a 10 segundos considerada "tempo real" pelos clientes | `TRANSCRICAO` | `[09:02] Marcos` |
| **TRK-03** | `docs/PRD.md` | Escopo de Produto | Escopo estritamente outbound (saída para os clientes); inbound fora desta entrega | `TRANSCRICAO` | `[09:02] Marcos` / `[09:03] Sofia` |
| **TRK-04** | `docs/adrs/ADR-001-outbox-no-mysql.md` | Alternativa Descartada | Rejeição de disparo HTTP síncrono na transação de pedidos para evitar locks de estoque e inconsistência | `TRANSCRICAO` | `[09:04] Bruno` / `[09:06] Diego` |
| **TRK-05** | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão Arquitetural | Adoção do padrão Transactional Outbox persistido em tabela `webhook_outbox` no MySQL existente | `TRANSCRICAO` | `[09:06] Diego` / `[09:08] Larissa` |
| **TRK-06** | `docs/RFC.md` | Alternativa Descartada | Descarte de Redis Streams / RabbitMQ / Kafka no MVP para evitar overengineering e nova infra | `TRANSCRICAO` | `[09:07] Larissa` / `[09:07] Diego` |
| **TRK-07** | `docs/FDD.md` | Escopo / Risco | Expurgo/arquivamento de eventos entregues após 30 dias postecipado (fora de escopo do MVP) | `TRANSCRICAO` | `[09:08] Diego` |
| **TRK-08** | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão Arquitetural | Worker executando polling contínuo no MySQL a cada 2 segundos com leitura em lote indexada | `TRANSCRICAO` | `[09:08] Diego` / `[09:09] Diego` |
| **TRK-09** | `docs/RFC.md` | Alternativa Descartada | Descarte de triggers no MySQL por ausência de `NOTIFY/LISTEN` nativo como no PostgreSQL | `TRANSCRICAO` | `[09:09] Diego` |
| **TRK-10** | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão / Dependência | Worker rodando em processo Node.js separado via novo entrypoint `src/worker.ts` e script `npm run worker` | `TRANSCRICAO` | `[09:11] Diego` / `[09:11] Larissa` |
| **TRK-11** | `docs/FDD.md` | Risco / Limitação | Ordenação garantida apenas por `order_id` enquanto houver single-worker; escala futura por partição/lock | `TRANSCRICAO` | `[09:12] Diego` / `[09:13] Larissa` |
| **TRK-12** | `docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md` | Decisão Arquitetural | Teto de 5 retentativas com backoff exponencial para tolerar manutenções sem pendurar eventos | `TRANSCRICAO` | `[09:15] Diego` / `[09:16] Diego` |
| **TRK-13** | `docs/FDD.md` | Estratégia de Resiliência | Escala progressiva de backoff em 1 min, 5 min, 30 min, 2 horas e 12 horas (~15h totais) | `TRANSCRICAO` | `[09:17] Diego` / `[09:17] Larissa` |
| **TRK-14** | `docs/adrs/ADR-003-retry-backoff-exponencial-e-dlq.md` | Decisão Arquitetural | Isolamento de falhas permanentes na tabela separada `webhook_dead_letter` (DLQ) | `TRANSCRICAO` | `[09:18] Diego` |
| **TRK-15** | `docs/FDD.md` | Requisito Funcional | Endpoint administrativo `POST /api/v1/admin/webhooks/dead-letter/:id/replay` para reprocessamento manual | `TRANSCRICAO` | `[09:18] Diego` / `[09:35] Diego` |
| **TRK-16** | `docs/adrs/ADR-004-autenticacao-hmac-sha256-secret-por-endpoint.md` | Decisão de Segurança | Assinatura criptográfica HMAC-SHA256 sobre o payload enviada no cabeçalho `X-Signature` | `TRANSCRICAO` | `[09:20] Sofia` / `[09:20] Bruno` |
| **TRK-17** | `docs/adrs/ADR-004-autenticacao-hmac-sha256-secret-por-endpoint.md` | Requisito de Segurança | Secret única por endpoint cadastrado (rejeição de secret global para conter vazamentos) | `TRANSCRICAO` | `[09:21] Sofia` / `[09:22] Diego` |
| **TRK-18** | `docs/FDD.md` | Requisito Funcional | Endpoint `rotate-secret` mantendo a secret anterior válida em paralelo por 24 horas (grace period) | `TRANSCRICAO` | `[09:21] Sofia` / `[09:22] Sofia` |
| **TRK-19** | `docs/FDD.md` | Requisito Não Funcional | Obrigatoriedade de protocolo HTTPS nas URLs cadastradas com rejeição de HTTP no schema Zod | `TRANSCRICAO` | `[09:23] Sofia` |
| **TRK-20** | `docs/FDD.md` | Requisito Não Funcional | Limite máximo de 64 KB para payload outbound; rejeição com erro explícito sem truncamento | `TRANSCRICAO` | `[09:23] Sofia` / `[09:24] Diego` |
| **TRK-21** | `docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md` | Decisão Arquitetural | Garantia at-least-once com `X-Event-Id` UUID imutável para deduplicação no cliente receptor | `TRANSCRICAO` | `[09:24] Diego` / `[09:25] Diego` |
| **TRK-22** | `docs/PRD.md` | Dependência / Docs | Documentar claramente o comportamento at-least-once e `X-Event-Id` no portal do desenvolvedor | `TRANSCRICAO` | `[09:26] Marcos` |
| **TRK-23** | `docs/adrs/ADR-006-reuso-padroes-existentes-do-projeto.md` | Padrão Estrutural | Organização do novo domínio em `src/modules/webhooks/` e processador em `webhook.processor.ts` | `TRANSCRICAO` | `[09:27] Bruno` / `[09:28] Bruno` |
| **TRK-24** | `docs/FDD.md` | Matriz de Erros | Extensão de `AppError` com códigos padronizados usando prefixo `WEBHOOK_*` | `TRANSCRICAO` | `[09:28] Bruno` / `[09:29] Larissa` |
| **TRK-25** | `docs/FDD.md` | Observabilidade | Reuso do logger Pino existente e do middleware global de erros sem adicionar novo framework de log | `TRANSCRICAO` | `[09:29] Bruno` |
| **TRK-26** | `docs/FDD.md` | Dependências técnicas | Processo worker instanciando seu próprio `PrismaClient` conectado à mesma `DATABASE_URL` | `TRANSCRICAO` | `[09:30] Bruno` |
| **TRK-27** | `docs/FDD.md` | Contrato HTTP | `POST /webhooks` com `customerId` no body (não extraído do JWT) e retorno da `secret` gerada | `TRANSCRICAO` | `[09:31] Marcos` / `[09:32] Larissa` |
| **TRK-28** | `docs/FDD.md` | Contrato HTTP | CRUD completo (`POST`, `GET`, `PATCH`, `DELETE`) para endpoints de webhook por cliente | `TRANSCRICAO` | `[09:33] Bruno` |
| **TRK-29** | `docs/FDD.md` | Regra de Negócio | Filtro por lista de status selecionados aplicado no momento da inserção na outbox (economiza tabela) | `TRANSCRICAO` | `[09:33] Marcos` / `[09:34] Bruno` |
| **TRK-30** | `docs/PRD.md` | Requisito Funcional | Consulta de histórico de entregas (`GET /webhooks/:id/deliveries`) com payload, response e latência | `TRANSCRICAO` | `[09:34] Marcos` |
| **TRK-31** | `docs/FDD.md` | Requisito de Segurança | Replay na DLQ exige role `ADMIN` via `requireRole` e registra quem executou o replay | `TRANSCRICAO` | `[09:36] Sofia` / `[09:36] Larissa` |
| **TRK-32** | `docs/PRD.md` | Escopo de Produto | Notificação de falha por e-mail adiada para próxima fase (fora de escopo do MVP) | `TRANSCRICAO` | `[09:37] Marcos` / `[09:37] Larissa` |
| **TRK-33** | `docs/RFC.md` | Ponto em Aberto / Risco | Rate limiting outbound em rajadas deixado em aberto para observação pós-lançamento | `TRANSCRICAO` | `[09:38] Diego` / `[09:39] Larissa` |
| **TRK-34** | `docs/PRD.md` | Escopo de Produto | Dashboard visual no frontend excluído do escopo (apenas API REST e docs de integração) | `TRANSCRICAO` | `[09:39] Marcos` / `[09:40] Larissa` |
| **TRK-35** | `docs/FDD.md` | Integração / Contrato | Função pura transacional `publishWebhookEvent(tx, order, fromStatus, toStatus)` no `OrderService` | `TRANSCRICAO` | `[09:40] Bruno` / `[09:41] Diego` |
| **TRK-36** | `docs/FDD.md` | Estratégia de Resiliência | Timeout estrito de 10 segundos para chamadas HTTP outbound executadas pelo worker | `TRANSCRICAO` | `[09:42] Diego` / `[09:42] Sofia` |
| **TRK-37** | `docs/FDD.md` | Contrato Outbound | Payload enxuto (`event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, status, `customer_id`, `total_cents`, sem `items`) | `TRANSCRICAO` | `[09:43] Diego` / `[09:44] Bruno` |
| **TRK-38** | `docs/FDD.md` | Contrato de Headers | Cabeçalhos outbound `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id` | `TRANSCRICAO` | `[09:44] Diego` / `[09:44] Sofia` |
| **TRK-39** | `docs/PRD.md` | Planejamento / Risco | Cronograma de 3 sprints até fim de novembro incluindo 2 dias úteis de revisão de segurança pela Sofia | `TRANSCRICAO` | `[09:45] Larissa` / `[09:46] Sofia` |
| **TRK-40** | `docs/adrs/ADR-001-outbox-no-mysql.md` | Modelagem de Dados | Chave primária em UUID na tabela `webhook_outbox` alinhada ao padrão de todo o sistema | `TRANSCRICAO` | `[09:51] Larissa` |
| **TRK-41** | `docs/adrs/ADR-001-outbox-no-mysql.md` | Modelagem de Dados | Armazenamento de snapshot completo renderizado do payload no instante da inserção na outbox | `TRANSCRICAO` | `[09:52] Larissa` / `[09:52] Diego` |
| **TRK-42** | `docs/PRD.md` | Métrica / Meta | Uso dos status do ciclo real de pedidos para notificação contínua até a entrega ao destinatário | `TRANSCRICAO` | `[09:12] Larissa` / `[09:33] Marcos` |
| **TRK-43** | `docs/FDD.md` | Compatibilidade | Garantia de não degradação das rotas atuais de pedidos e integração sem quebrar fluxos existentes | `TRANSCRICAO` | `[09:04] Bruno` / `[09:40] Bruno` |
| **TRK-44** | `docs/FDD.md` | Critério de Segurança | Validação periódica contra vazamento de secrets e inspeção formal de geração de HMAC antes de subir | `TRANSCRICAO` | `[09:22] Diego` / `[09:46] Sofia` |
| **TRK-45** | `docs/FDD.md` | Integração Existente | Transação `prisma.$transaction` em `OrderService.changeStatus(id, input, userId)` | `CODIGO` | `src/modules/orders/order.service.ts` (L126-179) |
| **TRK-46** | `docs/FDD.md` | Domínio Existente | Enum `OrderStatus` real (`PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`) e transições | `CODIGO` | `src/modules/orders/order.status.ts` (L1-37) |
| **TRK-47** | `docs/RFC.md` | Contrato HTTP Real | Verbo HTTP real `PATCH /:id/status` com middleware `authenticate` e validação Zod | `CODIGO` | `src/modules/orders/order.routes.ts` (L19-23) |
| **TRK-48** | `docs/FDD.md` | Esquema de Dados Real | Modelos `OrderStatus`, `Customer`, `Order`, `OrderStatusHistory` e uso estrito de `@db.Char(36)` | `CODIGO` | `prisma/schema.prisma` (L16-23, L40-54, L74-97) |
| **TRK-49** | `docs/adrs/ADR-006-reuso-padroes-existentes-do-projeto.md` | Padrão de Erros | Classe base `AppError(message, statusCode, errorCode, details)` para subclasses de domínio | `CODIGO` | `src/shared/errors/app-error.ts` (L1-16) |
| **TRK-50** | `docs/FDD.md` | Segurança Existente | Middlewares `authenticate` (JWT) e `requireRole('ADMIN')` (`FORBIDDEN` HTTP 403) | `CODIGO` | `src/middlewares/auth.middleware.ts` (L27-61) |
| **TRK-51** | `docs/FDD.md` | Envelope de Erro Real | Serialização padronizada `{ error: { code, message, details? } }` para `AppError`, `ZodError` e Prisma | `CODIGO` | `src/middlewares/error.middleware.ts` (L14-65) |
| **TRK-52** | `docs/FDD.md` | Tracing / Correlação | Middleware `requestLogger` gerando `X-Request-Id` (`req.id`) e propagando contexto entre API e worker | `CODIGO` | `src/middlewares/request-logger.middleware.ts` (L5-28) |
| **TRK-53** | `docs/adrs/ADR-006-reuso-padroes-existentes-do-projeto.md` | Logging Existente | Instância Pino `createLogger()` com `redactPaths` base a ser estendido com paths de secret/assinatura | `CODIGO` | `src/shared/logger/index.ts` (L4-32) |
| **TRK-54** | `docs/FDD.md` | Paginação Real | Helper `paginated(data, page, pageSize, total)` produzindo `{ data, pagination }` | `CODIGO` | `src/shared/http/response.ts` (L1-24) |
| **TRK-55** | `docs/FDD.md` | Roteamento Real | Montagem modular de rotas em `buildApiRouter(controllers)` sob o prefixo `/api/v1` | `CODIGO` | `src/routes/index.ts` (L13-31) e `src/app.ts` (L55-76) |
| **TRK-56** | `docs/FDD.md` | Dependência / Ambiente | Validação de variáveis de ambiente com `envSchema` (Zod) e fábrica `createPrismaClient()` | `CODIGO` | `src/config/env.ts` (L3-27) e `src/config/database.ts` (L4-10) |
| **TRK-57** | `docs/FDD.md` | Dependências / Runtime | Node.js `>= 20` (ESM), Express `4.21.1`, Prisma `5.22.0`, Zod `3.23.8`, Pino `9.5.0`, uuid `11.0.3` | `CODIGO` | `package.json` (L7-34) |
| **TRK-58** | `docs/FDD.md` | Infraestrutura MySQL | Serviço `mysql:8.0` em `utf8mb4_unicode_ci` com suporte ACID e índices para outbox | `CODIGO` | `docker-compose.yml` (L1-25) |

---

## 3. Análise de Cobertura e Métricas

- **Total de Itens Mapeados:** 58
- **Itens Provenientes de Discussão Técnica (`TRANSCRICAO`):** 44 itens (**75.9%** do total, superando o mínimo de $\ge 70\%$)
- **Itens Vinculados a Arquivos Reais da Base de Código (`CODIGO`):** 14 itens (**24.1%** do total, superando o mínimo de $\ge 5$ itens)
- **Cobertura Documental:** Mapeia 100% das decisões dos 6 ADRs, requisitos do PRD, proposta e questões do RFC e todas as 13 seções técnicas do FDD (incluindo Tracing, Dependências/Compatibilidade e Riscos/Mitigação).
