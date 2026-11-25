# ADR-001: Adoção do Padrão Outbox no Banco de Dados MySQL

## Status
Aprovado

## Contexto
O Order Management System (OMS) necessita notificar clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) sempre que o status de um pedido for alterado. Atualmente, o método `changeStatus` em `src/modules/orders/order.service.ts` executa uma transação atômica complexa que atualiza a tabela `orders`, registra o histórico em `order_status_history` e executa operações de baixa ou reposição de estoque em `products` (linhas 126–179).

Disparar requisições HTTP diretamente de forma síncrona dentro dessa transação bloquearia conexões com o banco, degradaria a latência do sistema e criaria inconsistência caso o endpoint externo estivesse instável (Dual-Write Problem). Por outro lado, a equipe é enxuta e busca evitar a introdução de brokers externos complexos sem necessidade comprovada.

## Decisão
Implementar o **Padrão Transactional Outbox** persistido na mesma base de dados MySQL existente através de uma nova tabela `webhook_outbox`.

Principais diretrizes da decisão:
1. **Transacionalidade Estrita:** A inserção do evento na tabela `webhook_outbox` ocorrerá dentro da mesma transação SQL (`tx`) gerenciada pelo Prisma Client no método `changeStatus` de `src/modules/orders/order.service.ts`, através da função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` (`[09:41] Bruno, Diego`). Se a transação falhar ou sofrer rollback, o evento não é gerado; se a transação comitar, o evento fica garantidamente registrado.
2. **Identificador em UUID:** Em consonância com a arquitetura do projeto (`prisma/schema.prisma`), a chave primária da tabela `webhook_outbox` utilizará `String @id @default(uuid()) @db.Char(36)` (`[09:51] Larissa`).
3. **Snapshot do Payload na Inserção:** O payload completo da notificação será serializado em JSON e persistido no momento da criação do registro na outbox (`[09:52] Diego, Larissa`). Isso evita divergências caso o pedido sofra mutações subsequentes antes da entrega física do webhook.
4. **Filtro Prévio de Eventos:** O evento só será inserido na outbox se houver algum webhook ativo cadastrado para o `customerId` com interesse naquele `OrderStatus` específico (`PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED` — `[09:34] Bruno`).
5. **Indexação:** Criação de índices compostos em `(status, next_retry_at)` e `(status, created_at)` para otimizar a leitura do worker.

## Alternativas Consideradas

### 1. Disparo HTTP Síncrono no `OrderService`
- **Descrição:** Executar chamada HTTP (`fetch`/`axios`) diretamente durante a execução de `changeStatus`.
- **Trade-off de Descarte:** Inviabilizado categoricamente ([09:04] Bruno, [09:06] Diego). Um cliente lento aumentaria o tempo de lock das linhas do MySQL, e uma falha de rede do cliente geraria dilema insolúvel: realizar rollback de uma transação de negócio válida ou perder o evento.

### 2. Mensageria Externa / Broker Dedicado (Redis Streams, RabbitMQ ou SQS)
- **Descrição:** Publicar eventos diretamente em um cluster Redis ou fila externa.
- **Trade-off de Descarte:** Descartado por overengineering para o porte atual da equipe e da infraestrutura ([09:07] Larissa, Diego). Introduziria dependência de novo componente distribuído, custos operacionais e não eliminaria o problema de dual-write sem um outbox prévio.

## Consequências

### Positivas
- **Garantia de Consistência:** Eliminação total do Dual-Write Problem sem risco de eventos fantasmas ou pedidos sem notificação.
- **Simplicidade Operacional:** Reuso do banco MySQL gerenciado pelo Prisma já existente no projeto.
- **Isolamento de Falhas:** Indisponibilidades ou lentidões na infraestrutura de clientes não impactam a transação central de pedidos.

### Negativas / Mitigações
- **Overhead de Escrita no Banco:** Escrita adicional de I/O no MySQL a cada transição de status. *Mitigação:* Tabela enxuta, leitura filtrada por índices e limpeza/arquivamento futuro de eventos com mais de 30 dias ([09:08] Diego).
- **Latência Não-Instantânea:** A notificação não ocorre no milissegundo exato do commit, dependendo do intervalo de varredura do worker (mitigado pelo polling de 2 segundos, atendendo à meta de < 10 segundos).
