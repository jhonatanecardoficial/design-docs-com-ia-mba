# ADR-006: Reuso dos Padrões Arquiteturais, Estruturais e de Observabilidade Existentes

## Status
Aprovado

## Contexto
O Order Management System (OMS) possui convenções arquiteturais bem estabelecidas e maduras em sua base de código TypeScript:
- Organização em módulos de domínio coesos sob `src/modules/` contendo controller, service, repository, routes e schemas (`auth`, `customers`, `orders`, `products`, `users`).
- Tratamento uniforme de exceções de aplicação via classe base `AppError` (`src/shared/errors/app-error.ts`) e classes filhas semânticas (`src/shared/errors/http-errors.ts`), capturadas de forma centralizada por `src/middlewares/error.middleware.ts`.
- Sistema padronizado de validação de schemas em tempo de execução via biblioteca Zod (`src/middlewares/validate.middleware.ts`).
- Middleware de segurança com autenticação JWT e controle de acesso baseado em papéis (`requireRole`) em `src/middlewares/auth.middleware.ts`.
- Logging estruturado em formato JSON com redação automática de dados sensíveis utilizando Pino (`src/shared/logger/index.ts`).
- Centralização de roteamento em `src/routes/index.ts`.

A introdução do novo Sistema de Webhooks não deve criar ilhas tecnológicas, bibliotecas concorrentes de logging ou padrões divergentes de formatação de erro ([09:27]–[09:30] Bruno, Diego, Larissa).

## Decisão
Reaproveitar integralmente os padrões arquiteturais, utilitários compartilhados e convenções estruturais já adotados na aplicação:

1. **Estrutura Modular em `src/modules/webhooks/`:**
   - `webhook.controller.ts`: Manipulação das requisições HTTP REST de CRUD e deliveries.
   - `webhook.service.ts`: Regras de negócio de webhooks, geração e rotação de secret, e despacho.
   - `webhook.repository.ts`: Operações de persistência com o Prisma Client.
   - `webhook.routes.ts`: Definição de endpoints Express conectada ao builder em `src/routes/index.ts`.
   - `webhook.schemas.ts`: Schemas Zod de validação (exigindo `https://` via validação de string, formato de arrays de eventos, paginação).
   - `webhook.processor.ts`: Lógica central de consumo e envio de eventos invocada pelo entrypoint `src/worker.ts`.
2. **Convenção de Nomenclatura de Erros com Prefixo `WEBHOOK_*`:**
   - Todas as falhas de domínio e validação estenderão `AppError` (`src/shared/errors/app-error.ts`) utilizando códigos em uppercase com prefixo obrigatório `WEBHOOK_` ([09:28], [09:29] Bruno, Larissa):
     - `WEBHOOK_NOT_FOUND` (404)
     - `WEBHOOK_INVALID_URL` (400)
     - `WEBHOOK_SECRET_REQUIRED` (400)
     - `WEBHOOK_PAYLOAD_TOO_LARGE` (413)
     - `WEBHOOK_DELIVERY_FAILED` (502)
     - `WEBHOOK_DEAD_LETTER_NOT_FOUND` (404)
     - `WEBHOOK_UNAUTHORIZED_REPLAY` (403)
3. **Tratamento Centralizado de Erros:**
   - O middleware `src/middlewares/error.middleware.ts` interceptará nativamente os erros do módulo de webhooks sem necessidade de customizações invasivas no fluxo de tratamento global da API.
4. **Segurança e RBAC:**
   - O endpoint de replay manual de eventos da DLQ (`POST /admin/webhooks/dead-letter/:id/replay`) utilizará obrigatoriamente a função `requireRole('ADMIN')` já disponível em `src/middlewares/auth.middleware.ts` ([09:36] Sofia, Larissa).
5. **Logging Estruturado com Pino:**
   - Todas as operações da API e do Worker utilizarão o logger instanciado em `src/shared/logger/index.ts`, garantindo que chaves privadas (`secret`, `password`) sejam automaticamente censuradas via `redactPaths` do Pino.

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
- **Observabilidade Unificada:** Métricas, traces e logs trafegam no mesmo formato JSON padronizado com correlação nativa.

### Negativas / Mitigações
- **Acoplamento Tecnológico:** O módulo permanece atrelado ao ecossistema Express, Prisma e MySQL. *Mitigação:* Como o domínio está estritamente encapsulado sob `src/modules/webhooks/`, caso haja necessidade futura de extração para microsserviço independente, a refatoração será cirúrgica.
