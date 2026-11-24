# ADR-003: Política de Retry com Backoff Exponencial e Dead Letter Queue (DLQ)

## Status
Aprovado

## Contexto
Endpoints de clientes B2B na internet estão sujeitos a indisponibilidades transitórias (quedas de rede, reinicializações de servidores, manutenções planejadas de infraestrutura ou erros 5xx de aplicação). Se um webhook falhar na primeira tentativa, o sistema deve tentar reentregá-lo sem sobrecarregar o cliente receptor, mas também sem manter mensagens presas indefinidamente na fila principal de outbox.

Conforme discutido na reunião técnica ([09:16] Diego), clientes da empresa já apresentaram períodos de manutenção programada de até duas horas.

## Decisão
Adotar uma política de **5 tentativas com backoff exponencial espaçado** e encaminhamento para uma **Dead Letter Queue (DLQ) persistida em tabela separada**:

1. **Número de Tentativas:** Máximo de 5 tentativas de entrega ([09:15], [09:16] Diego, Larissa).
2. **Escala de Backoff Exponencial:** Os intervalos de espera após cada falha consecutiva serão estritamente:
   - Tentativa 1 (imediata via polling)
   - Tentativa 2: após **1 minuto**
   - Tentativa 3: após **5 minutos**
   - Tentativa 4: após **30 minutos**
   - Tentativa 5: após **2 horas**
   - Tentativa 6 (final): após **12 horas**
   - *Janela total de cobertura:* aproximadamente 14h36m entre o primeiro evento e o descarte ([09:17] Diego).
3. **Timeout de Conexão e Leitura:** Limite rígido de **10 segundos** para a chamada HTTP externa ([09:42] Diego, Sofia). Respostas demoradas serão tratadas como timeout/falha.
4. **Isolamento da DLQ:** Criar a tabela `webhook_dead_letter` no MySQL para armazenar eventos que falharem após as 5 tentativas de retry, registrando o snapshot do payload original, código de erro/status HTTP recebido, stack de erro e timestamps ([09:18] Diego).
5. **Replay Administrativo Controlado:** Disponibilização do endpoint autenticado `POST /admin/webhooks/dead-letter/:id/replay` restrito a operadores com permissão `UserRole.ADMIN`, recolocando o evento na outbox com status `PENDING` e registrando log de auditoria ([09:18] Diego, [09:35], [09:36] Sofia).

## Alternativas Consideradas

### 1. Retry Agressivo em Intervalos Curtos (ex.: 3 tentativas em 30 minutos)
- **Descrição:** Tentar reenviar com intervalos de poucos segundos/minutos e descartar após 3 erros.
- **Trade-off de Descarte:** Rejeitado ([09:16] Diego). Caso um cliente passe por uma janela de manutenção planejada de 2 horas em horário matinal, todas as notificações seriam descartadas em menos de meia hora, gerando atrito e perda irreversível de comunicação.

### 2. Retry Indefinido com Backoff Contínuo
- **Descrição:** O worker tenta entregar a mensagem para sempre com intervalos progressivos até que o cliente retorne 200.
- **Trade-off de Descarte:** Rejeitado ([09:15] Diego). Eventos acumulariam infinitamente na tabela outbox caso um cliente desativasse seu servidor ou alterasse a URL sem avisar, gerando memory leaks e degradação severa das queries de polling.

### 3. Manter Eventos Mortos na Mesma Tabela Outbox (Flag `status = 'FAILED'`)
- **Descrição:** Apenas marcar o registro com status de falha na própria tabela `webhook_outbox`.
- **Trade-off de Descarte:** Rejeitado ([09:18] Diego). A separação física em `webhook_dead_letter` mantém a outbox ativa enxuta e com índices densos focados no fluxo quente de entregas.

## Consequências

### Positivas
- **Alta Resiliência Operacional:** A janela de quase 15 horas tolera manutenções prolongadas de sistemas clientes sem perda de dados.
- **Proteção do Cliente (Throttling):** O backoff progressivo impede que um cliente recém-restabelecido sofra um ataque acidental de negação de serviço (DoS) causado por enxurrada de retries imediatos.
- **Auditoria e Governança:** Eventos na DLQ preservam o diagnóstico exato da falha e permitem recuperação cirúrgica através do endpoint de replay administrativo.

### Negativas / Mitigações
- **Complexidade de Schema:** Necessidade de gerenciar a tabela de DLQ e o campo `next_retry_at` na tabela outbox. *Mitigação:* Estrutura mapeada formalmente no Prisma Client com tipagem estrita.
- **Dependência de Ação Humana na DLQ:** Falhas permanentes não possuem resolução automática além das 5 tentativas. *Mitigação:* Alarmes e logs via Pino facilitam identificação proativa pelo time de suporte.
