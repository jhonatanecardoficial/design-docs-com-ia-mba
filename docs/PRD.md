# Product Requirements Document (PRD): Notificações Outbound via Webhooks

| Metadado | Detalhe |
| :--- | :--- |
| **Documento** | Product Requirements Document (PRD) |
| **Produto** | Order Management System (OMS) B2B |
| **Funcionalidade** | Sistema de Webhooks Outbound para Notificação de Pedidos |
| **Status** | Aprovado |
| **Versão** | 1.1.0 |
| **Product Manager** | Marcos |
| **Tech Lead** | Larissa |
| **Data** | 2026-11-25 |
| **Público-Alvo** | Clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo), Engenharia e Operações |

---

## 1. Sumário Executivo e Contexto do Negócio

O Order Management System (OMS) gerencia pedidos corporativos em banco MySQL controlando o ciclo de vida definido na máquina de estados (`PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED` — `prisma/schema.prisma` e `src/modules/orders/order.status.ts`). Três clientes B2B estratégicos — **Atlas Comercial**, **MaxDistribuição** e **Nova Cargo** — precisam reagir em tempo real às mudanças de status dos seus pedidos (`[09:00] Marcos`).

Atualmente, não existe nenhum mecanismo de eventos ou notificações externas no sistema. Para descobrir mudanças de estado, os parceiros executam polling contínuo sobre `GET /api/v1/orders`, tornando suas integrações lentas e onerosas. Este PRD consolida os requisitos de produto e negócio para a implementação do **Sistema de Webhooks Outbound de Notificação de Pedidos**.

---

## 2. Problema e Motivação

### 2.1. O Problema
Conforme relatado na reunião de alinhamento (`[09:00] Marcos`):
- Os clientes **Atlas Comercial**, **MaxDistribuição** e **Nova Cargo** solicitam formalmente notificações automáticas quando o status de seus pedidos é alterado.
- O modelo atual baseado em consultas periódicas em `GET /api/v1/orders` gera latência operacional de sincronização e consumo desnecessário de recursos na integração dos clientes e na API do OMS.
- A **Atlas Comercial** sinalizou que, caso a capacidade de notificação não seja entregue até o fim do trimestre, poderá migrar para um concorrente (`[09:00] Marcos`).

### 2.2. A Solução
Substituir o modelo passivo de consulta (*polling* pelo cliente) por notificações ativas orientadas a eventos (*outbound webhooks*). Toda transição válida de status de pedido executada via `PATCH /api/v1/orders/:id/status` passará a gerar, de forma transacional e assíncrona, requisições HTTPS assinadas via HMAC-SHA256 para os endpoints cadastrados pelo cliente interessado naquele status (`[09:02] Marcos`, `[09:33] Bruno`).

---

## 3. Público-Alvo, Personas e Cenários de Uso

### 3.1. Personas

| Persona | Papel | Necessidade Principal |
| :--- | :--- | :--- |
| **Operador / Engenheiro de Integração do Parceiro B2B** | Representante técnico de clientes como Atlas Comercial, MaxDistribuição e Nova Cargo | Cadastrar URLs HTTPS, selecionar quais status receber (`SHIPPED`, `DELIVERED`, etc.), validar autenticidade via `X-Signature`, deduplicar via `X-Event-Id` e consultar histórico de entregas. |
| **Operador Interno do OMS** | Usuário autenticado que opera pedidos e cadastros na API do OMS | Cadastrar e gerenciar webhooks de clientes B2B usando o JWT do próprio OMS (`[09:32] Marcos, Larissa`). |
| **Administrador da Plataforma (`ADMIN`)** | Engenheiro de Operações / Suporte com perfil `UserRole.ADMIN` | Inspecionar falhas permanentes encaminhadas à Dead Letter Queue e reprocessar mensagens manualmente com registro de auditoria (`[09:36] Sofia, Larissa`). |

### 3.2. Cenários de Uso Principais
1. **UC-01 (Cadastro e Filtro de Eventos):** Operador cadastra uma URL HTTPS associada ao `customerId`, escolhe a lista de status desejados (ex.: `["SHIPPED", "DELIVERED"]`) e recebe uma chave secreta exclusiva gerada pela plataforma (`[09:31]–[09:33] Marcos`).
2. **UC-02 (Notificação em Tempo Real de Mudança de Status):** Quando um pedido passa de `PROCESSING` para `SHIPPED`, os endpoints ativos desse cliente interessados em `SHIPPED` recebem um POST assinado em menos de 10 segundos (`[09:02] Marcos`, `[09:33] Bruno`).
3. **UC-03 (Tolerância a Indisponibilidade Temporária):** Se o servidor do cliente estiver em janela de manutenção (ex.: 2 horas), o worker retenta o envio progressivamente em até 5 retentativas (~15 horas) sem intervenção manual (`[09:16]–[09:17] Diego`).
4. **UC-04 (Auditoria e Replay na DLQ):** Após esgotamento de retries, o evento vai para a Dead Letter Queue; quando o cliente regulariza seu endpoint, um usuário `ADMIN` aciona o replay manual (`[09:18] Diego`, `[09:36] Sofia`).
5. **UC-05 (Rotação Segura de Secret):** Em caso de suspeita de vazamento ou política de troca de chaves, um novo segredo é gerado mantendo o segredo anterior válido por 24 horas para migração sem downtime (`[09:21]–[09:22] Sofia, Diego`).

---

## 4. Objetivos e Métricas de Sucesso

| Objetivo / Meta | Métrica e Alvo Quantitativo | Origem / Critério de Medição |
| :--- | :--- | :--- |
| **Latência de Notificação ("Tempo Real")** | **< 10 segundos** entre o commit da mudança de status e o disparo do evento (intervalo de polling configurado em **2 segundos** no worker). | Definido em `[09:02] Marcos` e `[09:09] Larissa, Diego`. |
| **Janela de Resiliência a Falhas Externas** | **5 retentativas** progressivas (`1m / 5m / 30m / 2h / 12h`), cobrindo **~15 horas** (~14h36m) antes de encaminhar para DLQ. | Definido em `[09:16]–[09:17] Diego, Marcos`. |
| **Prazo de Entrega para Retenção B2B** | Conclusão integral em **3 sprints** (incluindo **2 dias úteis** reservados para revisão de segurança) até o **fim de novembro**. | Definido em `[09:00] Marcos`, `[09:45]–[09:47] Larissa, Sofia`. |
| **Redução de Polling em `GET /api/v1/orders`** | **KPI pós-lançamento:** queda significativa do tráfego de polling dos clientes **Atlas Comercial**, **MaxDistribuição** e **Nova Cargo** (linha de base a ser medida antes/depois do rollout piloto). | Motivação de negócio descrita em `[09:00] Marcos`. |
| **Confiabilidade de Persistência Transacional** | **100%** dos eventos de transição comitados persistidos atomicamente na outbox (zero perda silenciosa por falha de rede). | Garantia transacional definida em `[09:06] Diego`, `[09:40]–[09:41] Bruno`. |

---

## 5. Escopo do Produto

### 5.1. Em Escopo (MVP)
- Emissão exclusiva de **webhooks outbound** para o evento `order.status_changed` (`[09:02]–[09:03] Marcos, Sofia`).
- Cadastro, listagem, edição (`PATCH`), exclusão (`DELETE`) e histórico de entregas (`GET /:id/deliveries`) via endpoints REST autenticados sob `/api/v1/webhooks` (`[09:31]–[09:34] Marcos, Bruno`).
- Filtro por lista de status de pedido (`OrderStatus`) aplicado no momento da inserção na tabela outbox (`[09:33]–[09:34] Marcos, Bruno`).
- Garantia transacional atômica com `OrderService.changeStatus`, worker em processo separado com polling de 2s, assinatura HMAC-SHA256 com secret única por endpoint e rotação de 24h, semântica *at-least-once* com `X-Event-Id`, limite de 64 KB (erro se exceder) e timeout de 10s.
- Tabela `webhook_dead_letter` (DLQ) com endpoint de replay manual exclusivo para role `ADMIN` e registro de auditoria (`[09:18] Diego`, `[09:36] Sofia`).

### 5.2. Fora de Escopo (Explicitamente Descartado ou Postecipado)

1. **Webhooks Inbound (Recepção):** O sistema não receberá callbacks de terceiros nesta entrega (`[09:02]–[09:03] Marcos, Sofia`).
2. **Dashboard Visual / Painel no Frontend:** Não haverá interface gráfica para gerenciamento ou visualização de entregas — responsabilidade fora de escopo do projeto de backend, ficando apenas endpoints e documentação para integração (`[09:39]–[09:40] Marcos, Larissa`).
3. **Fallback para E-mail em Caso de Falha:** Avisos por e-mail caso um webhook falhe consecutivamente (ex.: 3 erros seguidos) foram adiados para uma fase posterior (`[09:37] Marcos, Larissa`).
4. **Expurgo Automático de 30 Dias:** Arquivamento ou exclusão automatizada de registros `DELIVERED` após 30 dias ficou explicitamente fora do escopo desta feature (`[09:08] Diego`).
5. **Rate Limiting Outbound por Cliente:** Controle automático de rajadas de disparos por cliente ficou em aberto para observação pós-lançamento (`[09:38]–[09:39] Diego, Larissa`).

---

## 6. Requisitos Funcionais (RFs)

- **RF-01 (Cadastro de Webhook):** Disponibilizar `POST /api/v1/webhooks` para registrar URL HTTPS, `customerId` (informado no payload, não extraído do JWT), descrição opcional e lista de status de pedido assinados (`events`), gerando e retornando a `secret` na criação (`[09:31]–[09:33] Marcos, Larissa`).
- **RF-02 (Listagem de Webhooks por Cliente):** Disponibilizar `GET /api/v1/webhooks` para listar os endpoints cadastrados para um `customerId`, omitindo as chaves secretas (`[09:33] Bruno`).
- **RF-03 (Edição e Remoção de Webhook):** Disponibilizar `PATCH /api/v1/webhooks/:id` para editar URL, status ativo/inativo ou filtro de eventos e `DELETE /api/v1/webhooks/:id` para remover o cadastro (`[09:33] Bruno`).
- **RF-04 (Emissão Orientada a Transição de Status com Filtro Prévio):** Toda transição válida em `OrderService.changeStatus` deve persistir o evento `order.status_changed` na outbox dentro da mesma transação SQL **somente** para endpoints ativos do cliente que assinaram o `toStatus` (`[09:33]–[09:34] Bruno`, `[09:40] Bruno`).
- **RF-05 (Assinatura HMAC-SHA256 e Cabeçalhos Padrão):** Toda entrega outbound deve incluir `X-Signature` (HMAC-SHA256 com secret única por endpoint), `X-Event-Id` (UUID), `X-Timestamp` (ISO 8601) e `X-Webhook-Id` (`[09:20]–[09:21] Sofia`, `[09:44] Diego, Sofia`).
- **RF-06 (Rotação de Secret com Janela de 24h):** Disponibilizar `POST /api/v1/webhooks/:id/rotate-secret` mantendo a secret anterior válida em paralelo por **24 horas** para transição sem interrupção (`[09:21]–[09:22] Sofia`).
- **RF-07 (Deduplicação / At-Least-Once via `X-Event-Id`):** Garantir envio *at-least-once* preservando o mesmo UUID `X-Event-Id` entre retentativas para permitir idempotência no cliente (`[09:24]–[09:25] Diego`).
- **RF-08 (Retry com Backoff Exponencial):** Reenfileirar automaticamente entregas com falha ou timeout usando escala de 5 retentativas (`1m`, `5m`, `30m`, `2h`, `12h`) (`[09:15]–[09:17] Diego, Larissa`).
- **RF-09 (Armazenamento em Dead Letter Queue):** Mover eventos que falharem após as 5 retentativas para a tabela `webhook_dead_letter`, registrando payload, motivo e timestamp (`[09:18] Diego`).
- **RF-10 (Consulta de Histórico de Entregas):** Disponibilizar `GET /api/v1/webhooks/:id/deliveries` retornando histórico paginado de entregas (até os últimos 100 disparos, com sucesso/falha, payload, response e tempo de resposta — `[09:34] Marcos`).
- **RF-11 (Replay Manual Restrito a `ADMIN`):** Disponibilizar `POST /api/v1/admin/webhooks/dead-letter/:id/replay` restrito a usuários com papel `ADMIN` (`requireRole('ADMIN')`), registrando obrigatoriamente quem executou o replay (`[09:35]–[09:36] Sofia, Larissa`).

---

## 7. Requisitos Não-Funcionais (RNFs)

- **RNF-01 (Latência e Polling):** Worker separado executando polling a cada **2 segundos** com ordenação garantida apenas por `order_id` enquanto houver instância única (`single-worker`), atendendo ao limite de **< 10 segundos** (`[09:02] Marcos`, `[09:09]–[09:13] Diego, Larissa`).
- **RNF-02 (Timeout de Disparo HTTP):** Tempo limite rígido de **10 segundos** por requisição outbound (`[09:42] Diego, Sofia`).
- **RNF-03 (TLS Obrigatório):** Apenas URLs com protocolo `https://` podem ser cadastradas; URLs `http://` devem ser recusadas na validação (`[09:23] Sofia`).
- **RNF-04 (Limite de Tamanho de Payload):** Teto rígido de **64 KB** por payload; eventos maiores devem retornar erro explícito e **nunca** serem truncados (`[09:23]–[09:24] Sofia, Diego, Larissa`).
- **RNF-05 (Isolamento de Processos e Falhas):** O worker roda como processo separado (`src/worker.ts`) conectado ao mesmo banco MySQL para evitar que quedas da API parem envios ou clientes lentos travem pedidos (`[09:04] Bruno`, `[09:11] Diego`).
- **RNF-06 (Conformidade de Logs, Erros e Documentação ao Cliente):** Reutilizar `AppError` (`WEBHOOK_*`), Pino logger (com extensão de `redactPaths` para mascarar segredos e assinaturas) e documentar a idempotência por `X-Event-Id` no portal do desenvolvedor (`[09:26] Marcos`, `[09:28]–[09:29] Bruno, Larissa`).

---

## 8. Decisões e Trade-offs Principais

1. **Transactional Outbox no MySQL vs. Infraestrutura Externa (`[09:06]–[09:07] Diego, Larissa`):**
   - *Decisão:* Persistir eventos na tabela `webhook_outbox` dentro da transação de `OrderService.changeStatus`.
   - *Trade-off:* Evita custo operacional de Redis/RabbitMQ/Kafka para uma equipe pequena e impede o *Dual-Write Problem*, aceitando latência mínima do ciclo de polling (2s).
2. **At-Least-Once com `X-Event-Id` vs. Exactly-Once (`[09:24]–[09:25] Diego, Sofia`):**
   - *Decisão:* Garantir entrega pelo menos uma vez e enviar `X-Event-Id` único por evento.
   - *Trade-off:* Transfere ao cliente a lógica de deduplicação, porém segue o padrão de mercado (Stripe/GitHub) e evita protocolos complexos entre sistemas independentes.
3. **Secret Única por Endpoint + Carência de 24h vs. Secret Global (`[09:21]–[09:22] Sofia, Diego`):**
   - *Decisão:* Chave exclusiva por endpoint com rotação preservando a chave anterior por 24 horas.
   - *Trade-off:* Aumenta ligeiramente a complexidade de verificação e armazenamento de credenciais, mas isola incidentes de vazamento por cliente e elimina downtime nas trocas de chave.

---

## 9. Dependências

- **Banco de Dados e ORM:** MySQL 8.0 compatível com transações ACID + Prisma Client `5.22.0` (`prisma/schema.prisma`, `src/config/database.ts`).
- **Módulo de Pedidos Existente:** Extensão transacional no método `changeStatus` em `src/modules/orders/order.service.ts` respeitando a máquina de estados em `src/modules/orders/order.status.ts`.
- **Segurança e Middleware Compartilhado:** Reaproveitamento de `authenticate` e `requireRole('ADMIN')` (`src/middlewares/auth.middleware.ts`), `errorMiddleware` (`src/middlewares/error.middleware.ts`) e Pino (`src/shared/logger/index.ts`).
- **Revisão Obrigatória de Segurança:** Reserva de pelo menos **2 dias úteis** para revisão de código pela Engenharia de Segurança (HMAC e geração de secrets) antes do deploy em produção (`[09:46]`, `[09:49] Sofia`).
- **Documentação para Parceiros:** Atualização do portal de desenvolvedor explicando validação de HMAC-SHA256 e deduplicação por `X-Event-Id` (`[09:26]`, `[09:40] Marcos`).

---

## 10. Riscos e Mitigação

| Risco Identificado | Probabilidade | Impacto | Plano de Mitigação |
| :--- | :---: | :---: | :--- |
| **Clientes lentos ou fora do ar travarem o processo de pedidos** | Alta | Alto | Desacoplamento total via Transactional Outbox + processo worker separado + timeout rígido de **10 segundos** + backoff exponencial em 5 retentativas (`[09:04] Bruno`, `[09:42] Diego`). |
| **Vazamento de chave secreta de cliente** | Baixa | Alto | Secret exclusiva por endpoint (nunca global), rotação com janela de 24h, proibição de exibir `secret` no `GET /webhooks` e extensão de `redactPaths` no Pino (`[09:21]–[09:22] Sofia, Diego`). |
| **Acúmulo excessivo de linhas na tabela `webhook_outbox`** | Média | Médio | Filtro prévio de eventos na inserção (`[09:34] Bruno`), índices compostos por status e data, e previsão de rotina de expurgo de 30 dias pós-MVP (`[09:08] Diego`). |
| **Entrega duplicada ou fora de ordem em clientes B2B** | Média | Médio | Envio de `X-Event-Id` para idempotência e `X-Timestamp`/snapshot imutável no payload, operando com single-worker ordenado por `created_at` no MVP (`[09:12]`, `[09:25]`, `[09:52] Diego, Larissa`). |
| **Rajada de disparos para um cliente sem throttling** | Média | Médio | Controle de tamanho de lote no worker, backoff em caso de HTTP 429/5xx e monitoramento em produção para avaliar rate limiting outbound (`[09:38]–[09:39] Diego`). |

---

## 11. Critérios de Aceitação

- [ ] Operador autenticado cadastra, lista, edita (`PATCH`) e remove endpoints `https://` vinculados a `customerId` e lista de `OrderStatus`, recebendo `secret` apenas na criação e rotação.
- [ ] Alterações em `PATCH /api/v1/orders/:id/status` gravam snapshot em `webhook_outbox` na mesma transação SQL apenas quando há endpoint ativo inscrito no novo status.
- [ ] Disparo outbound pelo worker separado (`src/worker.ts`) respeita timeout de 10s e envia payload compacto com headers `X-Event-Id`, `X-Signature` (HMAC-SHA256), `X-Timestamp`, `X-Webhook-Id` e contexto de trace.
- [ ] Payloads maiores que 64 KB e URLs `http://` são rejeitados com erro explícito (sem truncar).
- [ ] Falhas externas disparam até 5 retentativas (`1m/5m/30m/2h/12h`) e, ao esgotar, movem o evento para `webhook_dead_letter`.
- [ ] Apenas usuários `ADMIN` conseguem acionar `POST /api/v1/admin/webhooks/dead-letter/:id/replay`, registrando auditoria do operador; outros perfis recebem `403`.
- [ ] Rotação de secret mantém a chave anterior funcional por 24 horas antes da revogação definitiva.

---

## 12. Estratégia de Testes e Validação

1. **Testes Unitários (Vitest):** Geração e verificação de HMAC-SHA256, cálculo da janela de rotação de 24h, validação de schemas Zod (`https://`, `OrderStatus`, 64 KB sem truncar) e escalonamento de backoff exponencial.
2. **Testes de Integração com API e Banco (`tests/` com Vitest + Supertest):**
   - Atomicidade e rollback em `OrderService.changeStatus` quando o estoque falha ou o payload excede 64 KB.
   - Ciclo completo CRUD (`/api/v1/webhooks`), filtro prévio de `events` na outbox, histórico `/deliveries`, bloqueio `403` para `OPERATOR` no replay e autorização `200` para `ADMIN`.
3. **Homologação com Clientes B2B e Revisão de Segurança:** Testes ponta a ponta contra endpoints HTTPS de staging medindo latência (< 10s), seguidos dos **2 dias úteis de revisão pela Engenharia de Segurança (Sofia)** antes da liberação gradual para Atlas Comercial, MaxDistribuição e Nova Cargo (`[09:46] Sofia`).
