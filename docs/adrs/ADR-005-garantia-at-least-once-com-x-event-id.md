# ADR-005: Garantia de Entrega At-Least-Once com Idempotência via Header X-Event-Id

## Status
Aprovado

## Contexto
Em redes distribuídas e assíncronas, falhas parciais de comunicação são inevitáveis. Uma situação clássica ocorre quando o worker envia a requisição de webhook, o servidor do cliente recebe, processa com êxito e emite uma resposta HTTP 200, porém uma oscilação na rota de rede ou lentidão causa um timeout de conexão no lado do worker antes de receber o ACK.

Como consequência da política de resiliência estabelecida no [ADR-003](ADR-003-retry-backoff-exponencial-e-dlq.md), o worker considerará a tentativa falha e executará uma nova tentativa posteriormente. Portanto, a entrega pode ocorrer mais de uma vez para um mesmo evento físico de mudança de status.

## Decisão
Estabelecer a semântica de entrega **At-Least-Once (pelo menos uma vez)** combinada com um identificador universal e imutável de evento transmitido no cabeçalho **`X-Event-Id`** para viabilizar **idempotência no consumidor**:

1. **Semântica At-Least-Once:** A plataforma assume o compromisso de nunca perder um evento gerado na outbox, mesmo que isso resulte na retransmissão de duplicatas em cenários de falha de ACK ([09:24], [09:25] Diego).
2. **UUID Imutável por Evento:** No momento em que a linha é inserida na tabela `webhook_outbox` (transação de `changeStatus`), um UUID versão 4 único é atribuído permanentemente ao evento.
3. **Cabeçalhos de Identificação no Disparo:**
   - **`X-Event-Id`**: Conterá o UUID único daquele evento na outbox, mantendo-se idêntico em todas as eventuais retentativas ([09:25], [09:44] Diego).
   - **`X-Webhook-Id`**: Conterá o ID da assinatura do webhook cadastrado, permitindo ao cliente identificar qual de suas configurações originou a chamada ([09:44] Sofia).
4. **Responsabilidade de Deduplicação:** A responsabilidade de garantir o processamento idempotente é delegada ao receptor (cliente), que deve consultar seu próprio banco/cache e descartar requisições com `X-Event-Id` já processados ([09:25] Diego).
5. **Documentação Explícita:** Essa premissa de integração será documentada com destaque no portal e nas especificações entregues aos clientes ([09:26] Marcos).

## Alternativas Consideradas

### 1. Garantia de Entrega Exactly-Once (Exatamente Uma Vez)
- **Descrição:** Tentar garantir que o cliente receba a notificação uma e exatamente uma vez, sem nenhuma duplicata possível.
- **Trade-off de Descarte:** Rejeitado tecnicamente ([09:25] Diego). No modelo HTTP assíncrono entre sistemas de organizações diferentes, *Exactly-Once* é matematicamente e operacionalmente inviável sem protocolos pesados de consenso distribuído (como Two-Phase Commit - 2PC) que exigiriam que os clientes implementassem padrões proprietários de transação distribuída, inviabilizando integrações ágeis.

### 2. Semântica At-Most-Once (No Máximo Uma Vez)
- **Descrição:** O worker tenta disparar a requisição apenas uma vez; se falhar ou sofrer timeout, não realiza novas tentativas.
- **Trade-off de Descarte:** Rejeitado. Isso transferiria o risco de perda completa de notificações críticas de pedidos para o cliente sob qualquer instabilidade momentânea de rede, violando a exigência dos clientes B2B ([09:00] Marcos).

## Consequências

### Positivas
- **Tolerância a Falhas Confiável:** A aplicação prioriza a completude da entrega em detrimento da complexidade teórica de consenso.
- **Simplicidade e Escalabilidade:** O worker não precisa de coordenação distribuída para checar se o cliente já gravou antes de tentar novamente em caso de falha de socket.
- **Aderência aos Grandes Players:** Mesma semântica adotada por Stripe, GitHub, Shopify e Twilio, sendo um padrão amplamente dominado por equipes de engenharia B2B.

### Negativas / Mitigações
- **Carga de Trabalho no Cliente:** O cliente é obrigado a implementar controle de idempotência (ex.: tabela de eventos processados ou chave única no Redis/DB). *Mitigação:* Fornecer exemplos claros e código de referência no portal de integração demonstrando como filtrar eventos duplicados via `X-Event-Id`.
