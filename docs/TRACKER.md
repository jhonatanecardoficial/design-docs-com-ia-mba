# Matriz de Rastreabilidade e Proveniência (TRACKER)

| Metadado | Detalhe |
| :--- | :--- |
| **Documento** | Matriz de Rastreabilidade Técnica e de Requisitos |
| **Projeto** | Webhooks Outbound para Transição de Status de Pedidos |
| **Cobertura** | Decisões Técnicas, PRD, RFC, FDD e ADRs (ADR-001 a ADR-006) |
| **Fontes Rastreadas** | `TRANSCRICAO.md` e Código-Fonte do Repositório |

---

## 1. Visão Geral da Matriz

Esta matriz mapeia cada decisão arquitetural, requisito de produto, contrato de implementação e estratégia de resiliência à sua fonte original de discussão na reunião técnica (`TRANSCRICAO.md`) ou à estrutura consolidada da base de código do projeto (`CODIGO`).

---

## 2. Tabela de Rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **TRK-01** | PRD / RFC | Requisito de Negócio | Demanda formal dos clientes B2B (Atlas, MaxDistribuição e Nova Cargo) para notificações de pedidos | TRANSCRICAO | `[09:00] Marcos` |
| **TRK-02** | PRD / FDD | Requisito Não-Funcional | Latência máxima de 10 segundos aceitável para ser considerado "tempo real" | TRANSCRICAO | `[09:02] Marcos` |
| **TRK-03** | PRD / RFC | Escopo de Produto | Escopo restrito a webhooks de saída (outbound); clientes apenas recebem notificações | TRANSCRICAO | `[09:02] Marcos` / `[09:03] Sofia` |
| **TRK-04** | ADR-001 / FDD | Decisão Arquitetural | Rejeição de disparos síncronos HTTP dentro da transação de pedidos para evitar travamento do banco | TRANSCRICAO | `[09:04] Bruno` |
| **TRK-05** | ADR-001 / RFC | Decisão Arquitetural | Adoção do padrão Transactional Outbox gravado atomicamente no MySQL na transação do pedido | TRANSCRICAO | `[09:06] Diego` |
| **TRK-06** | ADR-001 / RFC | Decisão Arquitetural | Descarte de Redis Streams ou Kafka no MVP para evitar infraestrutura adicional e overengineering | TRANSCRICAO | `[09:07] Larissa` / `[09:07] Diego` |
| **TRK-07** | FDD / PRD | Escopo de Produto | Expurgo automático de eventos após 30 dias deliberadamente postecipado e fora do escopo | TRANSCRICAO | `[09:08] Diego` |
| **TRK-08** | ADR-002 / FDD | Decisão Arquitetural | Worker executando polling periódico a cada 2 segundos no MySQL para ler eventos pendentes | TRANSCRICAO | `[09:09] Diego` / `[09:10] Larissa` |
| **TRK-09** | ADR-002 / RFC | Decisão Arquitetural | Descarte de triggers no MySQL para disparos de processos externos pela ausência de NOTIFY nativo | TRANSCRICAO | `[09:09] Diego` |
| **TRK-10** | ADR-002 / FDD | Decisão Arquitetural | Worker operando como processo Node.js separado (`src/worker.ts`) e comando `npm run worker` | TRANSCRICAO | `[09:11] Larissa` / `[09:11] Diego` |
| **TRK-11** | FDD / RFC | Premissa Técnica | Garantia de ordenação mantida apenas por `order_id` no modelo single-worker | TRANSCRICAO | `[09:12] Diego` / `[09:13] Larissa` |
| **TRK-12** | ADR-003 / FDD | Decisão Arquitetural | Política de retry com 5 tentativas para cobrir janelas operacionais de manutenção dos clientes | TRANSCRICAO | `[09:15] Diego` / `[09:16] Diego` |
| **TRK-13** | ADR-003 / FDD | Requisito Funcional | Progressão de backoff exponencial nos intervalos de 1m, 5m, 30m, 2h e 12h (~15h total) | TRANSCRICAO | `[09:17] Diego` / `[09:17] Larissa` |
| **TRK-14** | ADR-003 / FDD | Decisão Arquitetural | Persistência de falhas permanentes em tabela isolada `webhook_dead_letter` (DLQ) | TRANSCRICAO | `[09:18] Diego` |
| **TRK-15** | FDD / PRD | Requisito Funcional | Endpoint administrativo para replay manual de registros retidos na DLQ | TRANSCRICAO | `[09:18] Diego` |
| **TRK-16** | ADR-004 / FDD | Decisão Arquitetural | Assinatura criptográfica HMAC-SHA256 enviada no cabeçalho `X-Signature` | TRANSCRICAO | `[09:20] Sofia` / `[09:20] Bruno` |
| **TRK-17** | ADR-004 / FDD | Requisito de Segurança | Segredo criptográfico compartilhado único por endpoint cadastrado (sem segredo global) | TRANSCRICAO | `[09:21] Sofia` |
| **TRK-18** | ADR-004 / FDD | Requisito Funcional | Rotação de secret com período de carência (grace period) de 24 horas para migração sem downtime | TRANSCRICAO | `[09:21] Sofia` / `[09:22] Sofia` |
| **TRK-19** | PRD / FDD | Requisito Não-Funcional | Protocolo HTTPS obrigatório para cadastro de endpoints de webhook | TRANSCRICAO | `[09:23] Sofia` |
| **TRK-20** | PRD / FDD | Requisito Não-Funcional | Limite rígido de tamanho de payload configurado em 64 KB com rejeição em caso de estouro | TRANSCRICAO | `[09:24] Diego` / `[09:24] Larissa` |
| **TRK-21** | ADR-005 / FDD | Decisão Arquitetural | Garantia de entrega at-least-once com cabeçalho `X-Event-Id` UUID para deduplicação no cliente | TRANSCRICAO | `[09:24] Diego` / `[09:25] Diego` |
| **TRK-22** | ADR-006 / FDD | Padrão Arquitetural | Criação do módulo estruturado em `src/modules/webhooks/` seguindo arquitetura existente | TRANSCRICAO | `[09:27] Bruno` |
| **TRK-23** | ADR-006 / FDD | Padrão Arquitetural | Separação da lógica do processador em `webhook.worker.ts` ou `webhook.processor.ts` | TRANSCRICAO | `[09:28] Bruno` |
| **TRK-24** | ADR-006 / FDD | Tratamento de Erros | Uso da classe `AppError` e códigos padronizados com prefixo `WEBHOOK_*` | TRANSCRICAO | `[09:28] Bruno` / `[09:29] Larissa` |
| **TRK-25** | ADR-006 / FDD | Observabilidade | Reuso da biblioteca Pino para logs estruturados e middleware de erros global | TRANSCRICAO | `[09:29] Bruno` |
| **TRK-26** | ADR-002 / FDD | Conectividade de Banco | Worker instanciando cliente próprio de PrismaClient mantendo a mesma DATABASE_URL | TRANSCRICAO | `[09:30] Bruno` |
| **TRK-27** | PRD / FDD | Requisito Funcional | API de cadastro `POST /webhooks` retornando secret gerado automaticamente | TRANSCRICAO | `[09:31] Marcos` |
| **TRK-28** | PRD / FDD | Requisito Funcional | Filtro de eventos por status de pedido desejado pelo cliente no endpoint de webhook | TRANSCRICAO | `[09:33] Marcos` |
| **TRK-29** | FDD / RFC | Otimização Técnica | Filtragem de eventos realizada antes da inserção na outbox para poupar espaço em disco | TRANSCRICAO | `[09:34] Bruno` / `[09:34] Diego` |
| **TRK-30** | PRD / FDD | Requisito Funcional | Consulta do histórico de entregas paginado via `GET /webhooks/:id/deliveries` | TRANSCRICAO | `[09:34] Marcos` |
| **TRK-31** | FDD / PRD | Requisito de Segurança | Replay de mensagens na DLQ restrito a usuários com papel `ADMIN` e auditoria de operador | TRANSCRICAO | `[09:36] Sofia` / `[09:36] Larissa` |
| **TRK-32** | PRD / RFC | Escopo de Produto | Fallback de notificações para e-mail deliberadamente descartado e fora de escopo | TRANSCRICAO | `[09:37] Larissa` |
| **TRK-33** | RFC / PRD | Ponto em Aberto | Rate limiting de disparo outbound para clientes categorizado como ponto em aberto | TRANSCRICAO | `[09:39] Diego` / `[09:39] Larissa` |
| **TRK-34** | PRD / RFC | Escopo de Produto | Dashboard visual no frontend expressamente excluído do escopo inicial | TRANSCRICAO | `[09:40] Larissa` |
| **TRK-35** | FDD / RFC | Integração Transacional | Inserção atômica do evento na outbox acoplada ao método `changeStatus` de pedidos | TRANSCRICAO | `[09:40] Bruno` / `[09:41] Diego` |
| **TRK-36** | FDD / RFC | Design de Código | Criação da função pura transacional `publishWebhookEvent(tx, order, fromStatus, toStatus)` | TRANSCRICAO | `[09:41] Bruno` |
| **TRK-37** | FDD / PRD | Requisito Não-Funcional | Timeout de conexão e leitura HTTP configurado rigidamente em 10 segundos no worker | TRANSCRICAO | `[09:42] Diego` / `[09:42] Sofia` |
| **TRK-38** | FDD / PRD | Contrato de API | Payload outbound contendo `event`, `timestamp`, `order_id`, `from_status`, `to_status` e totais | TRANSCRICAO | `[09:43] Diego` / `[09:44] Bruno` |
| **TRK-39** | FDD / PRD | Contrato de Cabeçalhos | Envio obrigatório dos cabeçalhos `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id` | TRANSCRICAO | `[09:44] Diego` / `[09:44] Sofia` |
| **TRK-40** | PRD / RFC | Planejamento de Projeto | Prazo fixado em 3 sprints para lançamento com 2 dias de homologação pela Segurança | TRANSCRICAO | `[09:46] Larissa` / `[09:46] Sofia` |
| **TRK-41** | FDD / ADR-001 | Modelagem de Dados | Identificadores de outbox modelados com UUID em conformidade com as entidades do projeto | TRANSCRICAO | `[09:51] Larissa` |
| **TRK-42** | FDD / RFC | Modelagem de Dados | Evento da outbox armazena snapshot completo renderizado do payload no momento da transação | TRANSCRICAO | `[09:52] Larissa` / `[09:52] Diego` |
| **TRK-43** | FDD / RFC | Código Existente | Transação do método `changeStatus` com atualização de status, histórico e estoque | CODIGO | `src/modules/orders/order.service.ts` (L126-179) |
| **TRK-44** | FDD / ADR-001 | Código Existente | Definição das entidades relacionais `Order`, `OrderStatusHistory`, `OrderStatus` e `Customer` | CODIGO | `prisma/schema.prisma` (L51-85) |
| **TRK-45** | FDD / ADR-006 | Código Existente | Definição da classe base `AppError` e formatação padrão de erros JSON da aplicação | CODIGO | `src/shared/errors/app-error.ts` (L1-43) |
| **TRK-46** | FDD / ADR-006 | Código Existente | Middleware de autenticação JWT e validação de role de usuário com `requireRole('ADMIN')` | CODIGO | `src/middlewares/auth.middleware.ts` (L45-72) |
| **TRK-47** | FDD / ADR-006 | Código Existente | Middleware global de captura e serialização de erros express (`errorMiddleware`) | CODIGO | `src/middlewares/error.middleware.ts` (L1-55) |
| **TRK-48** | FDD / ADR-006 | Código Existente | Instância centralizada do logger Pino com configuração de níveis e redaction de campos | CODIGO | `src/shared/logger/index.ts` (L1-35) |
| **TRK-49** | FDD / ADR-006 | Código Existente | Roteador principal da aplicação agregando os módulos e prefixos da API REST | CODIGO | `src/routes/index.ts` (L1-25) |
| **TRK-50** | FDD / ADR-002 | Código Existente | Ponto de entrada da aplicação HTTP e configuração de porta (`server.ts`) | CODIGO | `src/server.ts` (L1-30) |

---

## 3. Análise de Cobertura e Métricas

- **Total de Itens Mapeados:** 50
- **Itens Provenientes de Discussão Técnica (`TRANSCRICAO`):** 42 itens (**84.0%** do total, superando o requisito de $\ge 70\%$)
- **Itens Vinculados a Arquivos Reais da Base de Código (`CODIGO`):** 8 itens (**16.0%** do total, superando o requisito de $\ge 5$ itens)
- **Cobertura de Documentos:** Cobre 100% das 6 ADRs, seções do PRD, FDD e RFC.
