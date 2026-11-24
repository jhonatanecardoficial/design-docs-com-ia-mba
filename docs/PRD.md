# Product Requirements Document (PRD): Notificações Outbound via Webhooks

| Metadado | Detalhe |
| :--- | :--- |
| **Documento** | Product Requirements Document (PRD) |
| **Produto** | Plataforma de Pedidos B2B |
| **Funcionalidade** | Sistema de Notificação de Status de Pedidos por Webhooks Outbound |
| **Status** | Aprovado |
| **Versão** | 1.0.0 |
| **Product Manager** | Marcos |
| **Tech Lead** | Larissa |
| **Data** | 2026-11-24 |
| **Público-Alvo** | Clientes B2B, Engenharia de Software, Operações e Suporte Técnico |

---

## 1. Sumário Executivo & Contexto do Negócio

A plataforma de e-commerce e gestão de pedidos opera com crescente volume de transações corporativas (B2B). Clientes de grande porte com centros de distribuição e frotas próprias necessitam reagir em tempo real a cada atualização no ciclo de vida dos pedidos (por exemplo, quando um pedido é confirmado, separado em estoque, despachado ou entregue).

Atualmente, esses clientes realizam consultas repetidas e exaustivas (polling) aos endpoints de pedidos da API REST a cada poucos segundos. Isso gera consumo ineficiente de banda de rede, latência de sincronização para os clientes e carga desnecessária sobre o banco de dados da plataforma.

Este documento define os requisitos de produto para a introdução do sistema de **Notificações Outbound via Webhooks**, permitindo que os sistemas parceiros cadastrem URLs seguras para receberem eventos em tempo real assim que qualquer alteração de status for processada.

---

## 2. Problema & Motivação

### 2.1. O Problema
Grandes parceiros comerciais enfrentam atrasos operacionais para sincronizar estados de pedidos:
- **Atlas E-commerce:** Relata atrasos na sincronização de fulfillment e separação de mercadorias no centro de distribuição, pois suas rotinas de consulta periódica à API enfrentam gargalos de concorrência.
- **MaxDistribuição:** Mantém rotinas automatizadas de polling com intervalos inferiores a 5 segundos, gerando alto volume de requisições redundantes (HTTP 200 com os mesmos dados) e alertando limites de taxa (*rate limiting*).
- **Nova Cargo Transportes:** Necessita saber instantaneamente quando uma remessa atinge o status `DISPATCHED` para acionar motoristas e roteirizar veículos, sofrendo com esperas que impactam o SLA logístico acordado.

### 2.2. A Solução
Substituir o modelo passivo (*pull/polling*) por um modelo ativo orientado a eventos (*push/webhook*). Toda transição de status confirmada gerará um disparo HTTP direto para o endpoint HTTPS registrado pelo parceiro, contendo os dados essenciais da mudança de estado e assinaturas criptográficas para validação de autenticidade.

---

## 3. Público-Alvo, Personas & Casos de Uso

### 3.1. Personas

| Persona | Papel | Necessidade Principal |
| :--- | :--- | :--- |
| **Desenvolvedor Parceiro B2B** | Engenheiro de Integração de clientes como Atlas, MaxDistribuição e Nova Cargo | Cadastrar URLs seguras, receber eventos em formato padronizado, validar a autenticidade das mensagens via HMAC e contar com rastreabilidade de entregas. |
| **Gerente de Operações Logísticas** | Gestor de Cadeia de Suprimentos externa | Ter a certeza de que despachos e confirmações de entrega chegam ao seu ERP com latência inferior a 10 segundos para não atrasar a expedição. |
| **Administrador / Suporte Interno** | Engenheiro de Suporte e Operações da Plataforma | Monitorar o fluxo de entrega, diagnosticar parceiros com falhas de conectividade e reenviar notificações retidas via Dead Letter Queue. |

### 3.2. Principais Casos de Uso
1. **UC-01 (Cadastro e Ativação de Endpoint):** O desenvolvedor do cliente B2B cadastra a URL HTTPS do seu sistema, gerando um segredo compartilhado para validação das assinaturas.
2. **UC-02 (Recepção Automática de Evento de Pedido):** Ao ocorrer a mudança de status do pedido de `PENDING` para `CONFIRMED`, o parceiro recebe um POST com payload formatado e cabeçalhos de segurança.
3. **UC-03 (Recuperação de Falha Temporária):** O servidor do parceiro fica temporariamente indisponível durante manutenção; nossa plataforma retenta o envio automaticamente seguindo uma escala progressiva de tempo.
4. **UC-04 (Auditoria e Replay de Falhas Persistentes):** Um evento que excedeu 5 tentativas de entrega vai para a fila de erros (DLQ). Após o parceiro corrigir seu serviço, um administrador do sistema executa o reenvio manual do evento.

---

## 4. Objetivos do Produto & Métricas de Sucesso

### 4.1. Objetivos Quantitativos e Metas de Entrega

| Métrica / Meta | Alvo Quantitativo | Impacto Esperado |
| :--- | :--- | :--- |
| **Latência de Notificação** | **$\le$ 10 segundos** para 99% das notificações geradas | Garantir reação em tempo hábil para operadores de transporte e logística. |
| **Confiabilidade de Entrega** | **99.9%** de entregas concluídas com sucesso (ou encaminhadas à DLQ para tratamento) | Zero perda de eventos transacionados no banco de dados. |
| **Redução de Polling Ineficiente** | **$\ge$ 70%** de redução no tráfego de requisições `GET /orders` vindas de clientes B2B | Diminuição drástica de custos com I/O de banco e infraestrutura. |
| **Prazo de Lançamento** | **3 sprints de desenvolvimento** (Conclusão até final de novembro de 2026) | Atender à janela de contratação e acordos comerciais com Atlas e MaxDistribuição. |

---

## 5. Escopo do Produto

### 5.1. Em Escopo (MVP)
- Mecanismo outbound transacional baseado em banco de dados (*Transactional Outbox*).
- Disparo de notificações para transições de status de pedidos (`order.status_changed`).
- Assinatura digital criptográfica HMAC-SHA256 em todas as mensagens outbound.
- Rotação segura de segredos com janela de transição de 24 horas.
- Mecanismo de retentativas automáticas (5 tentativas em até 15 horas).
- Dead Letter Queue (DLQ) com capacidade de reprocessamento manual via API administrativa.
- API REST para gerenciamento de endpoints de webhook e consulta de entregas.

### 5.2. Fora de Escopo (Explicitamente Postecipado ou Descartado)

> [!IMPORTANT]
> Os itens a seguir foram deliberadamente excluídos do escopo inicial durante as decisões de arquitetura e produto:
1. **Webhooks de Entrada (Inbound):** Recepção de callbacks ou webhooks de terceiros (como gateways de pagamento). O foco atual é exclusivamente emissão (*outbound*).
2. **Dashboard Gráfico no Frontend:** Não haverá interface visual de administração nesta fase; toda a operação e consumo serão realizados via API REST.
3. **Fallback para E-mail:** Notificações de falha ou conteúdo de pedidos não serão despachados via e-mail corporativo em caso de falha no webhook.
4. **Rotina de Limpeza Automática (Expurgo de 30 dias):** A exclusão automatizada de logs antigos e registros outbox foi adiada para uma fase posterior de governança de dados.

---

## 6. Requisitos Funcionais (RFs)

- **RF-01 (Cadastro de Endpoint de Webhook):** O sistema deve fornecer uma API para cadastrar URLs HTTPS de parceiros associadas a uma descrição, gerando um identificador único (`id`) e um segredo exclusivo.
- **RF-02 (Gerenciamento de Segredo HMAC):** O sistema deve gerar uma chave criptográfica forte (mínimo de 32 bytes hexadecimais) por endpoint cadastrado para uso exclusivo na assinatura das mensagens.
- **RF-03 (Notificação de Transição de Status):** O sistema deve disparar notificações outbound para todos os endpoints ativos sempre que o status de um pedido sofrer alteração confirmada (`order.status_changed`).
- **RF-04 (Assinatura de Integridade Outbound):** O sistema deve assinar cada requisição HTTP outbound via HMAC-SHA256, enviando o resultado no cabeçalho `X-Signature` junto ao cabeçalho `X-Timestamp`.
- **RF-05 (Identificador Único para Idempotência):** O sistema deve incluir em toda notificação o cabeçalho `X-Event-Id` contendo um UUID único, permitindo ao parceiro deduplicar mensagens entregues repetidas vezes.
- **RF-06 (Política de Retentativas com Backoff Exponencial):** Em caso de falha de conexão, timeout ou retorno HTTP não-2xx, o sistema deve retentar o envio em 5 etapas progressivas (1m, 5m, 30m, 2h e 12h).
- **RF-07 (Envio para Dead Letter Queue):** O sistema deve transferir automaticamente para a tabela de Dead Letter Queue todo evento que persistir com falha após a 5ª tentativa.
- **RF-08 (Consulta de Histórico de Entregas):** O sistema deve disponibilizar endpoint para consulta paginada do histórico de tentativas de entrega, informando status HTTP retornado, tempo de resposta e mensagens de erro.
- **RF-09 (Rotação de Segredo com Carência de 24h):** O sistema deve permitir a rotação do segredo de um endpoint, mantendo o segredo anterior funcional por 24 horas para evitar interrupções no parceiro.
- **RF-10 (Replay Manual por Administrador):** O sistema deve permitir que usuários autenticados com papel `ADMIN` executem o replay de eventos registrados na Dead Letter Queue.
- **RF-11 (Desativação e Exclusão de Endpoints):** O sistema deve permitir desativar temporariamente ou remover permanentemente endpoints de webhook cadastrados.

---

## 7. Requisitos Não-Funcionais (RNFs)

- **RNF-01 (Desempenho e Latência):** O background worker deve coletar eventos pendentes a cada 2 segundos, garantindo que 99% das notificações sejam despachadas em menos de 10 segundos após a confirmação no banco.
- **RNF-02 (Timeout Restritivo de Rede):** O tempo limite de espera por resposta do endpoint do cliente deve ser de no máximo 10 segundos por requisição.
- **RNF-03 (Segurança e Protocolo Obrigatório):** Todas as URLs de webhook registradas devem obrigatoriamente utilizar o protocolo `HTTPS`. Requisições com protocolo não criptografado (`HTTP`) devem ser rejeitadas.
- **RNF-04 (Tamanho Máximo de Mensagem):** O payload JSON despachado não deve ultrapassar o limite arquitetural de 64 KB.
- **RNF-05 (Disponibilidade e Resiliência):** Falhas em endpoints de clientes ou lentidões externas não podem bloquear o processamento normal de pedidos nem afetar outros clientes cadastrados.
- **RNF-06 (Conformidade de Auditoria e Logs):** Todos os eventos de despacho e rotação de credenciais devem ser registrados em formato JSON estruturado (Pino), com mascaramento mandatório de segredos.

---

## 8. Principais Decisões de Produto & Trade-offs

1. **Transactional Outbox no MySQL vs. Fila Externa (RabbitMQ/Kafka):**
   - *Decisão:* Adotar tabela `webhook_outbox` no MySQL aproveitando a transação existente de pedidos.
   - *Trade-off:* Evita a complexidade operacional de introduzir e gerenciar novos clusters de mensageria no MVP, garantindo 100% de consistência transacional atômica com os recursos atuais.
2. **Modelo At-Least-Once vs. Exactly-Once:**
   - *Decisão:* Adotar garantia de entrega *at-least-once* complementada pelo cabeçalho `X-Event-Id`.
   - *Trade-off:* Assume a possibilidade remota de mensagens duplicadas em caso de oscilações de rede, transferindo a responsabilidade de deduplicação ao receptor via identificador único.
3. **Carência de 24h na Rotação de Chaves:**
   - *Decisão:* Manter duas chaves ativas simultaneamente durante 24 horas após solicitação de rotação.
   - *Trade-off:* Adiciona complexidade na validação do receptor e na persistência de credenciais, mas elimina o risco de *downtime* na comunicação com grandes parceiros B2B.

---

## 9. Dependências Organizacionais & Técnicas

- **Dependência de Infraestrutura:** Banco de dados MySQL operando em versão compatível com transações ACID e suporte a índices compostos.
- **Aprovações de Segurança:** Validação da política de HMAC-SHA256 e redaction de logs com a equipe de Governança e Segurança da Informação.
- **Comunicação com Clientes:** Disponibilização da documentação dos contratos de webhook para as equipes técnicas de Atlas E-commerce, MaxDistribuição e Nova Cargo.

---

## 10. Riscos do Produto e Matriz de Mitigação

| Risco Identificado | Probabilidade | Impacto | Plano de Mitigação |
| :--- | :---: | :---: | :--- |
| **Sobrecarga de rede por endpoints lentos ou fora do ar** | Alta | Médio | Configuração de timeout estrito de 10 segundos com `AbortController` e política de retentativa espaçada (backoff). |
| **Vazamento acidental de segredos HMAC em logs** | Baixa | Alto | Configuração de *redaction paths* nativa na biblioteca de logging (Pino), mascarando cabeçalhos de assinatura e campos de segredo. |
| **Gargalo no polling do banco com aumento de pedidos** | Média | Médio | Polling leve a cada 2s com índice composto `[status, next_retry_at]` e limite de lote configurado em 10 registros por ciclo. |
| **Inconsistência por eventos processados fora de ordem** | Média | Baixo | Inclusão de `timestamp` ISO e campos explícitos `from_status` e `to_status` em cada payload. |

---

## 11. Critérios de Aceitação do Produto

- [ ] Parceiros conseguem cadastrar URLs HTTPS e receber chave secreta exclusiva.
- [ ] Alterações de status no pedido (`CONFIRMED`, `DISPATCHED`, `DELIVERED`, `CANCELLED`) acionam o despacho automático de webhooks.
- [ ] Notificações chegam com cabeçalhos `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id`.
- [ ] Falhas no parceiro são retentadas em até 5 tentativas e depois encaminhadas para a DLQ.
- [ ] Administradores conseguem executar o replay de mensagens retidas na DLQ com sucesso.
- [ ] Rotação de segredos permite transição suave sem quebra de mensagens por 24 horas.

---

## 12. Estratégia de Testes, Validação & Rollout

1. **Testes Unitários e de Integração:** Validação da geração de HMAC, montagem de payloads e persistência em banco simulando falhas de transação.
2. **Ambiente de Homologação (Sandbox):** Disponibilização de servidor *echo/mock* com simulação de latência e falhas HTTP 500 para testes de ponta a ponta.
3. **Rollout Piloto:**
   - **Fase 1 (Semana 1):** Validação interna com clientes de teste e monitoramento contínuo de latência.
   - **Fase 2 (Semana 2):** Conexão inicial com a Atlas E-commerce como cliente pioneiro (*early adopter*).
   - **Fase 3 (Semana 3):** Habilitação geral para MaxDistribuição, Nova Cargo e demais parceiros B2B.
