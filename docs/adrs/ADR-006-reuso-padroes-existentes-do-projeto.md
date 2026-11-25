# ADR-006: Reuso dos Padrões Arquiteturais, Estruturais e de Observabilidade Existentes

## Status
Aprovado

## Contexto
O Order Management System (OMS) possui convenções arquiteturais bem estabelecidas e maduras em sua base de código TypeScript:
- Organização em módulos de domínio coesos sob `src/modules/` contendo controller, service, repository, routes e schemas (`auth`, `customers`, `orders`, `products`, `users`).
- Tratamento uniforme de exceções de aplicação via classe base `AppError` (`src/shared/errors/app-error.ts`) e classes filhas semânticas (`src/shared/errors/http-errors.ts`), capturadas e serializadas no formato `{ error: { code, message, details? } }` por `src/middlewares/error.middleware.ts`.
- Sistema padronizado de validação de schemas em tempo de execução via biblioteca Zod (`src/middlewares/validate.middleware.ts`).
- Envelope de paginação padronizado (`{ data, pagination: { page, pageSize, total, totalPages } }`) via `paginated()` em `src/shared/http/response.ts`.
- Middleware de segurança com autenticação JWT e controle de acesso baseado em papéis (`requireRole`) em `src/middlewares/auth.middleware.ts`.
- Logging estruturado em formato JSON com `redactPaths` utilizando Pino (`src/shared/logger/index.ts`) e correlação de requisições com `X-Request-Id` (`req.id`) em `src/middlewares/request-logger.middleware.ts`.
- Centralização de roteamento em `src/routes/index.ts` sob o prefixo `/api/v1` em `src/app.ts`.

A introdução do novo Sistema de Webhooks não deve criar ilhas tecnológicas, bibliotecas concorrentes de logging ou padrões divergentes de formatação de erro ([09:27]–[09:30] Bruno, Diego, Larissa).

## Decisão
Reaproveitar integralmente os padrões arquiteturais, utilitários compartilhados e convenções estruturais já adotados na aplicação:

1. **Estrutura Modular em `src/modules/webhooks/`:**
   - `webhook.controller.ts`: Manipulação das requisições HTTP REST de CRUD (`POST`, `GET`, `PATCH`, `DELETE`), rotação e deliveries.
   - `webhook.service.ts`: Regras de negócio de webhooks, geração e rotação de secret, e consulta de entregas/replay.
   - `webhook.repository.ts`: Operações de persistência com o Prisma Client.
   - `webhook.routes.ts`: Definição de endpoints Express conectada a `buildApiRouter` em `src/routes/index.ts` (sob `/api/v1`).
   - `webhook.schemas.ts`: Schemas Zod de validação (exigindo `https://`, `customerId`, lista `events` de `OrderStatus`, paginação `page`/`pageSize`).
   - `webhook.publisher.ts`: Função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` invocada na transação de `OrderService.changeStatus`.
   - `webhook.processor.ts`: Lógica de consumo, assinatura e envio HTTP invocada pelo entrypoint separado `src/worker.ts`.
2. **Convenção de Nomenclatura de Erros com Prefixo `WEBHOOK_*`:**
   - Todas as falhas de domínio e validação estenderão `AppError` (`src/shared/errors/app-error.ts`) utilizando códigos em uppercase com prefixo obrigatório `WEBHOOK_` ([09:28], [09:29] Bruno, Larissa):
     - `WEBHOOK_NOT_FOUND` (404)
     - `WEBHOOK_INVALID_URL` (400)
     - `WEBHOOK_INVALID_EVENTS` (400)
     - `WEBHOOK_SECRET_REQUIRED` (400)
     - `WEBHOOK_PAYLOAD_TOO_LARGE` (413)
     - `WEBHOOK_DELIVERY_FAILED` (502)
     - `WEBHOOK_DEAD_LETTER_NOT_FOUND` (404)
     - `WEBHOOK_ALREADY_REPLAYED` (409)
     - `WEBHOOK_UNAUTHORIZED_REPLAY` (403)
3. **Tratamento Centralizado de Erros e Paginação:**
   - O middleware `src/middlewares/error.middleware.ts` interceptará nativamente os erros do módulo de webhooks no formato `{ error: { code, message, details? } }`.
   - Listagens (`GET /api/v1/webhooks` e `GET /api/v1/webhooks/:id/deliveries`) utilizarão o helper `paginated()` de `src/shared/http/response.ts`.
4. **Segurança e RBAC:**
   - O endpoint de replay manual de eventos da DLQ (`POST /api/v1/admin/webhooks/dead-letter/:id/replay`) utilizará obrigatoriamente `authenticate` e `requireRole('ADMIN')` disponíveis em `src/middlewares/auth.middleware.ts` ([09:36] Sofia, Larissa).
5. **Logging Estruturado e Tracing com Pino:**
   - API e Worker utilizarão a fábrica `createLogger()` de `src/shared/logger/index.ts`. Na implementação da feature, o array `redactPaths` já existente (que atualmente mascara `authorization`, `cookie`, `password`, `passwordHash`, `token`, `accessToken`) **será estendido** para censurar também `*.secret`, `*.previousSecret`, `*.newSecret`, `*.headers["x-signature"]` e `req.headers["x-signature"]`.
   - O `req.id` gerado em `src/middlewares/request-logger.middleware.ts` servirá de base para `correlationId` propagado até o worker.

## Alternativas Consideradas

### 1. Construção de um Microsserviço Independente em Outro Repositório/Stack
- **Descrição:** Desenvolver o sistema de webhooks como um serviço completamente isolado (ex.: em Go ou Python com banco próprio).
- **Trade-off de Descarte:** Rejeitado pelo time ([09:07] Larissa, Diego). A equipe de engenharia é enxuta, e criar um novo ciclo de vida, repositório, CI/CD e monitoramento representaria sobrecarga excessiva desnecessária quando a arquitetura modular com outbox atende com excelência.

### 2. Adoção de Nova Biblioteca de Logger para o Worker (ex.: Winston)
- **Descrição:** Usar Winston ou `console.log` nativo no processo do worker.
- **Trade-off de Descarte:** Rejeitado ([09:29] Bruno). Fragmentaria a ingestão de telemetria no Elastic/Grafana/CloudWatch. O Pino já configurado possui alta performance e formatação idêntica em desenvolvimento e produção.

## Consequências

### Positivas
- **Velocidade de Entrega:** Desenvolvedores do time utilizam as abstrações e middlewares com os quais já têm familiaridade diária.
- **Homogeneidade de Código:** A base permanece consistente, limpa e padronizada em todos os domínios.
- **Observabilidade Unificada:** Métricas, traces (`correlationId` / `traceparent`) e logs trafegam no mesmo formato JSON padronizado.

### Negativas / Mitigações
- **Acoplamento Tecnológico:** O módulo permanece atrelado ao ecossistema Express, Prisma e MySQL. *Mitigação:* Como o domínio está estritamente encapsulado sob `src/modules/webhooks/`, caso haja necessidade futura de extração para microsserviço independente, a refatoração será cirúrgica.
