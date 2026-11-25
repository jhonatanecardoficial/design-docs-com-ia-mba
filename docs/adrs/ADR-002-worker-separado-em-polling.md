# ADR-002: Execução de Worker em Processo Separado com Polling a Cada 2 Segundos

## Status
Aprovado

## Contexto
Após a gravação transacional dos eventos na tabela `webhook_outbox` (conforme [ADR-001](ADR-001-outbox-no-mysql.md)), é necessário um mecanismo consumidor responsável por ler os registros pendentes, despachar as requisições HTTP para os endpoints dos clientes, gerenciar retries e atualizar os status de entrega. 

Os clientes B2B exigem que a notificação ocorra em menos de 10 segundos após a mudança de status ([09:02] Marcos). Além disso, a arquitetura deve garantir resiliência contra reinicializações do servidor HTTP principal da API.

## Decisão
Implementar o despachante de webhooks como um **Worker autônomo rodando em processo Node.js separado**, utilizando **polling em loop contínuo a cada 2 segundos**:

1. **Processo Desacoplado:** O worker será inicializado a partir de um novo entrypoint `src/worker.ts` e executado via script dedicado `"npm run worker"` no `package.json` ([09:11] Larissa). Ele não compartilhará memória ou ciclo de vida com `src/server.ts`.
2. **Ciclo de Polling de 2 Segundos:** O worker executará uma busca no MySQL a cada 2.000 ms selecionando eventos com status `PENDING` ordenados por `created_at ASC` em lotes controlados (*batch*) ([09:08], [09:09] Diego).
3. **Instância Dedicada do PrismaClient:** Como roda em processo Node.js independente, o worker instanciará seu próprio `PrismaClient` utilizando a fábrica `createPrismaClient()` (`src/config/database.ts`) conectando à mesma `DATABASE_URL`, sem interferir no pool da API HTTP ([09:11], [09:30] Bruno).
4. **Modelo Single-Worker no MVP:** No estágio inicial, operará uma única instância ativa do worker. Isso garante a ordenação sequencial estrita dos eventos de um mesmo pedido sem a complexidade de locks distribuídos ou particionamento de filas ([09:12] Diego).

## Alternativas Consideradas

### 1. Triggers Nativas do Banco de Dados MySQL
- **Descrição:** Utilizar triggers DDL/DML no MySQL para notificar a aplicação imediatamente após a inserção na outbox.
- **Trade-off de Descarte:** O MySQL não possui mecanismo nativo assíncrono de pub/sub ou listener externo (ao contrário do `LISTEN/NOTIFY` do PostgreSQL) ([09:09] Diego). Disparar rotinas externas via trigger exigiria UDFs inseguras em C/C++ ou escrita em arquivos em disco, introduzindo fragilidade extrema.

### 2. Thread/Timer em Background Dentro da Mesma Instância da API HTTP
- **Descrição:** Executar um `setInterval` no mesmo processo do Express em `src/server.ts`.
- **Trade-off de Descarte:** Rejeitado veementemente ([09:11] Diego). Se a API reiniciar devido a deploy, spike de tráfego ou memory leak, os despachos de webhooks seriam interrompidos abruptamente. Além disso, consumo de CPU/rede pelo worker concorreria diretamente com a resposta a requisições de usuários.

## Consequências

### Positivas
- **Isolamento de Processos:** Falhas na API HTTP não derrubam o processador de webhooks, e lentidões nas chamadas HTTP de webhook não travam o event loop da API de pedidos.
- **Conformidade com SLA:** O intervalo de 2 segundos assegura que, somado ao tempo de handshake HTTP, o webhook é entregue bem abaixo do teto de 10 segundos acordado com os clientes.
- **Previsibilidade de Ordenação:** Single-worker processando por `created_at` garante que transições rápidas (`PAID` -> `PROCESSING` -> `SHIPPED`) cheguem na ordem lógica correta.

### Negativas / Mitigações
- **Custo de Polling em Banco (I/O):** Consultas periódicas executadas a cada 2s mesmo quando a fila está vazia. *Mitigação:* Query ultra leve indexada em `(status, created_at)` retornando `LIMIT` pequeno, com impacto desprezível em instâncias MySQL modernas.
- **Limitação de Concorrência:** O processamento em instância única possui teto de throughput. *Mitigação:* Se o volume de pedidos crescer além da capacidade do worker, adotar-se-á lock pessimista (`SELECT FOR UPDATE SKIP LOCKED`) ou particionamento determinístico por hash de `customerId`/`orderId` no futuro ([09:13] Diego).
