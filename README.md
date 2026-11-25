# Da Reunião ao Documento: Design Docs Gerados por IA

## 1. Sobre o Desafio

Este desafio técnico do MBA em Engenharia de Software com IA (Full Cycle) teve como objetivo transformar a transcrição bruta de uma reunião técnica de alinhamento (`TRANSCRICAO.md`) e uma aplicação existente em Node.js/TypeScript em um pacote completo, rigoroso e acionável de **Design Docs** para um **Sistema de Webhooks Outbound de Notificação de Pedidos**.

O cenário simula uma demanda real de engenharia corporativa: um Order Management System (OMS) com persistência em MySQL via Prisma ORM (`src/modules/orders/order.service.ts`) precisa notificar três clientes B2B estratégicos (**Atlas Comercial**, **MaxDistribuição** e **Nova Cargo**) em menos de 10 segundos sempre que o status de um pedido mudar, substituindo o modelo atual de polling sobre `GET /api/v1/orders`. Todo o pacote de documentação respeita a separação de níveis entre produto (`PRD`), arquitetura (`RFC`), decisões pontuais (`ADRs`), especificação de implementação (`FDD`) e rastreabilidade (`TRACKER`), sem modificar o código da aplicação base.

---

## 2. Ferramentas de IA Utilizadas

Durante todo o ciclo de engenharia reversa do código-fonte, análise de transcrição, produção documental e auditoria cruzada, foram utilizadas as seguintes ferramentas:

- **Antigravity AI Agent (Google DeepMind Engine):** Orquestrador principal para leitura estática do repositório (`src/` e `prisma/schema.prisma`), extração de contratos reais (rotas `/api/v1`, máquina de estados `OrderStatus`, envelopes de erro e paginação) e verificação automatizada de consistência cruzada entre os documentos.
- **Modelos Claude / Gemini em Fluxo Multi-Agente e Revisão Crítica:** Empregados para sintetizar decisões arquiteturais em formato MADR, estruturar diagramas de sequência Mermaid e refinar especificações técnicas de observabilidade (logs Pino, métricas e tracing distribuído W3C/OpenTelemetry), dependências e matriz de riscos.

---

## 3. Workflow Adotado

Para preservar as fronteiras entre cada documento e evitar duplicação ou alucinações, seguiu-se um fluxo incremental em 7 etapas com ciclos de auditoria:

```text
[1. Exploração Código + TRANSCRICAO.md] ──> [2. 6 ADRs (MADR)] ──> [3. RFC Conciso]
                                                                            │
[6. Matriz TRACKER (58 itens)] <── [5. PRD Consolidado] <── [4. FDD (13 Seções)]
          │
[7. README do Processo + Dupla Auditoria de Consistência]
```

1. **Contexto Estático & Mapeamento:** Leitura integral das 324 linhas de `TRANSCRICAO.md` e inspeção direta dos arquivos reais do OMS (`prisma/schema.prisma`, `src/app.ts`, `src/routes/index.ts`, `src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`, `src/modules/orders/order.routes.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/request-logger.middleware.ts`, `src/shared/errors/app-error.ts`, `src/shared/http/response.ts`, `src/shared/logger/index.ts`, `src/config/env.ts`, `package.json`, `docker-compose.yml`).
2. **ADRs em Formato MADR (`docs/adrs/ADR-001` a `ADR-006`):** Registro isolado das 6 decisões fechadas (Transactional Outbox no MySQL, Worker separado em polling de 2s, Backoff exponencial com 5 retentativas + DLQ, Autenticação HMAC-SHA256 com secret por endpoint e carência de 24h, Entrega at-least-once com `X-Event-Id` e Reuso dos padrões existentes).
3. **Proposta Técnica (`docs/RFC.md`):** Documento conciso voltado para revisão pelos 5 participantes da reunião, apresentando proposta geral, 4 alternativas descartadas com trade-offs, 4 questões em aberto e links diretos para os ADRs.
4. **Especificação de Implementação (`docs/FDD.md`):** Documento profundo em 13 seções acionáveis: contexto, objetivos, escopo, integração detalhada com 11+ caminhos reais do código (incluindo assinatura exata de `changeStatus` e `publishWebhookEvent`), modelo relacional Prisma, 4 diagramas Mermaid, 7 contratos REST `/api/v1` + contrato outbound `order.status_changed`, matriz de erros `WEBHOOK_*`, resiliência, observabilidade completa (logs Pino, métricas e tracing distribuído), dependências e compatibilidade, riscos e mitigação, e critérios técnicos.
5. **Documento de Produto (`docs/PRD.md`):** Consolidação em 12 seções cobrindo personas, metas quantitativas ancoradas na reunião, 11 requisitos funcionais, 6 requisitos não funcionais e exclusões explícitas de escopo.
6. **Matriz de Rastreabilidade (`docs/TRACKER.md`):** Mapeamento de 58 itens (`75.9%` `TRANSCRICAO` e `24.1%` `CODIGO` com faixas de linha verificadas).
7. **Auditoria Cruzada & README:** Revisão rigorosa contra a transcrição, contra o código e contra cada critério de aceite do desafio.

---

## 4. Prompts Customizados

Abaixo estão dois prompts customizados elaborados para guiar a IA sem invenção de requisitos e garantindo aderência estrita ao código TypeScript existente.

### Prompt 1: Extração Fiel da Transcrição e Separação de Escopo

```markdown
Você é uma Tech Lead sênior especialista em sistemas distribuídos Node.js/TypeScript e MySQL.
Analise `TRANSCRICAO.md` e `prisma/schema.prisma` + `src/modules/orders/` para mapear os fatos técnicos exatos da reunião.

Regras Inegociáveis:
1. Utilize EXCLUSIVAMENTE os nomes dos clientes citados em TRANSCRICAO.md (Atlas Comercial, MaxDistribuição, Nova Cargo) e os valores reais do enum `OrderStatus` em `prisma/schema.prisma` (PENDING, PAID, PROCESSING, SHIPPED, DELIVERED, CANCELLED). Nunca invente status como CONFIRMED ou DISPATCHED.
2. Liste separadamente tudo o que foi DESCARTADO ou POSTECIPADO (inbound, dashboard, e-mail fallback, expurgo de 30 dias, rate limiting de saída, brokers externos, triggers MySQL), citando `[hh:mm] Falante`.
3. Registre literalmente o contrato do payload outbound definido em `[09:43] Diego`, a assinatura pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` e o filtro prévio de status na inserção da outbox (`[09:34] Bruno`).
```

### Prompt 2: Refinamento do FDD com Tracing, Dependências, Riscos e Contratos Reais

```markdown
Atue como Engenheiro de Software Staff e revise `docs/FDD.md` para torná-lo 100% acionável e fiel à base de código real.

Checklist obrigatório para o FDD:
1. Garanta todas as seções técnicas esperadas: Contexto, Objetivos, Escopo/Exclusões, Integração com o sistema existente, Modelo Prisma (`@db.Char(36)`), Diagramas Mermaid, Contratos HTTP públicos (`/api/v1` usando o formato real de erro de `error.middleware.ts` e paginação de `shared/http/response.ts`), Matriz `WEBHOOK_*` (payload > 64 KB falha com erro, sem truncar), Estratégias de Resiliência, Observabilidade (Logs Pino + extensão do `redactPaths`, Métricas e Tracing distribuído propagando `X-Request-Id` e W3C `traceparent` da API até `src/worker.ts`), Dependências e Compatibilidade (ancoradas em `package.json` e `docker-compose.yml`), Riscos e Mitigação, e Critérios de Aceitação.
2. Valide que nenhum endpoint, campo do modelo `Order` ou caminho de arquivo mencionado contradiz `src/` ou `prisma/`.
```

---

## 5. Iterações e Ajustes

A produção documental passou por **4 ciclos iterativos** de geração, revisão crítica e correção:

1. **Iteração 1 — Separação entre RFC e FDD (Nível de Abstração):**
   - *Problema inicial:* A primeira versão do RFC incluía definições extensas de schema Prisma e payloads JSON completos, duplicando o papel do FDD e ultrapassando o limite conciso esperado.
   - *Ajuste:* Todo o detalhamento de implementação foi concentrado em `docs/FDD.md`, mantendo `docs/RFC.md` enxuto (~3 páginas) focado em proposta arquitetural, trade-offs, questões em aberto e referências cruzadas para `docs/adrs/`.
2. **Iteração 2 — Filtragem Rigorosa de Escopo Descartado:**
   - *Problema inicial:* Versões preliminares do PRD listavam *fallback por e-mail*, *dashboard visual* e *limpeza automática de 30 dias* como entregáveis do MVP.
   - *Ajuste:* Cruzamento com `[09:08] Diego`, `[09:37] Larissa` e `[09:40] Larissa`, movendo esses pontos formalmente para "Fora de Escopo / Postecipado" e mantendo rate limiting outbound em "Questões em Aberto".
3. **Iteração 3 — Correção do Mecanismo de Autenticação Outbound:**
   - *Problema inicial:* Sugestão equivocada de usar `Authorization: Bearer <JWT>` no disparo do webhook para o cliente.
   - *Ajuste:* Alinhamento às falas de `[09:20]–[09:22] Sofia`, especificando `X-Signature` (HMAC-SHA256 sobre *raw body*) com secret única por endpoint e período de transição de 24h na rotação.
4. **Iteração 4 — Auditoria Técnica Profunda Pós-Revisão (Tracing, Dependências, Riscos e Fidelidade ao Código Real):**
   - *Problema detectado:* Na revisão, verificou-se que `docs/FDD.md` omitia explicitamente as seções **Dependências e compatibilidade** e **Riscos e mitigação**, além de não especificar **Tracing distribuído** na Seção 10. A auditoria pente-fino adicional também revelou divergências sutis geradas pela IA: status fictícios (`CONFIRMED`/`DISPATCHED`), omissão de `customerId`/`events`/`PATCH /webhooks/:id` e `publishWebhookEvent`, payload outbound diferente de `[09:43] Diego`, verbo `PUT` em vez de `PATCH /api/v1/orders/:id/status`, envelope de erro e paginação divergentes de `error.middleware.ts` e `shared/http/response.ts`, falsa premissa de que `redactPaths` do Pino já mascarava `secret`, e faixas de linha imprecisas no `TRACKER.md`.
   - *Ajuste:* Reescrita estrutural completa de `docs/FDD.md` (13 seções cobrindo 100% dos requisitos, com Tracing W3C/OpenTelemetry, matriz técnica de Dependências/Compatibilidade e 7 Riscos de Implementação), atualização coordenada de `docs/PRD.md`, `docs/RFC.md`, `docs/adrs/*` e recalibração integral das 58 linhas do `docs/TRACKER.md`.

---

## 6. Como Navegar a Entrega

```text
.
├── README.md                                        # Documentação do processo e guia de leitura
├── TRANSCRICAO.md                                   # Transcrição original da reunião técnica (inalterada)
└── docs/
    ├── PRD.md                                       # Product Requirements Document (Visão de Produto)
    ├── RFC.md                                       # Request for Comments (Proposta Técnica e Trade-offs)
    ├── FDD.md                                       # Feature Design Document (13 seções acionáveis + Tracing/Deps/Riscos)
    ├── TRACKER.md                                   # Matriz de Rastreabilidade e Proveniência (58 itens)
    └── adrs/                                        # Architecture Decision Records (MADR)
        ├── README.md                                # Índice consolidado dos 6 ADRs
        ├── ADR-001-outbox-no-mysql.md
        ├── ADR-002-worker-separado-em-polling.md
        ├── ADR-003-retry-backoff-exponencial-e-dlq.md
        ├── ADR-004-autenticacao-hmac-sha256-secret-por-endpoint.md
        ├── ADR-005-garantia-at-least-once-com-x-event-id.md
        └── ADR-006-reuso-padroes-existentes-do-projeto.md
```

### Ordem Sugerida de Leitura
1. **Product Managers e Stakeholders:** Inicie por [`docs/PRD.md`](docs/PRD.md) para entender o problema B2B, metas quantitativas, requisitos e limites claros do MVP.
2. **Arquitetos e Tech Leads:** Leia [`docs/RFC.md`](docs/RFC.md) para revisar a proposta sistêmica, alternativas descartadas e questões em aberto, aprofundando-se em [`docs/adrs/README.md`](docs/adrs/README.md) e nos ADRs individuais.
3. **Engenheiros de Implementação:** Utilize [`docs/FDD.md`](docs/FDD.md) como guia direto de codificação (integração com `src/`, schema Prisma, diagramas de fluxo, contratos `/api/v1`, erros `WEBHOOK_*`, observabilidade com logs/métricas/tracing, dependências e riscos).
4. **Revisores e Auditores:** Consulte [`docs/TRACKER.md`](docs/TRACKER.md) para comprovar a proveniência de cada requisito ou decisão na transcrição e no código-fonte real.
