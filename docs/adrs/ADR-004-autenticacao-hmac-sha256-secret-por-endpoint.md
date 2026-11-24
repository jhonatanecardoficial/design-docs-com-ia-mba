# ADR-004: Autenticação por HMAC-SHA256 com Secret Única por Endpoint e Suporte a Rotação

## Status
Aprovado

## Contexto
O envio de webhooks trafega dados sensíveis de pedidos (identificadores, status, valores monetários) através da internet pública em direção a servidores externos pertencentes a clientes B2B. A aplicação receptora necessita de um mecanismo confiável para atestar duas garantias fundamentais de segurança:
1. **Autenticidade:** Certeza de que a mensagem foi originada comprovadamente pela nossa plataforma.
2. **Integridade:** Certeza de que o corpo da requisição (*payload*) não foi alterado ou forjado por agentes maliciosos em trânsito (*Man-in-the-Middle*).

Adicionalmente, se as credenciais de um cliente forem comprometidas acidentalmente (por exemplo, vazadas em logs de terceiros, conforme histórico relatado em [09:22] Diego), o impacto deve ser contido e a plataforma deve permitir a troca do segredo sem interrupção de serviço.

## Decisão
Adotar o padrão da indústria de assinatura simétrica **HMAC-SHA256 gerada por endpoint com suporte a rotação com período de transição (*grace period*) de 24 horas**:

1. **Algoritmo de Assinatura:** Uso de HMAC com função hash SHA-256 (`crypto.createHmac('sha256', secret)`) calculada sobre o *raw body* exato em formato JSON ([09:20] Sofia).
2. **Transporte da Assinatura:** A assinatura hexadecimal gerada será transmitida no cabeçalho HTTP customizado `X-Signature`.
3. **Secret Única por Endpoint:** Cada endpoint cadastrado por um cliente possuirá uma chave secreta (`secret`) exclusiva gerada aleatoriamente no momento do cadastro ([09:21] Sofia). Não haverá chave global de plataforma.
4. **Mecanismo de Rotação com Grace Period:**
   - Criação do endpoint `POST /webhooks/:id/rotate-secret` para solicitar um novo segredo.
   - Ao rotacionar, a secret anterior é mantida em estado secundário válido por um período estrito de **24 horas** ([09:21] Sofia). Durante essa janela, o cliente tem tempo hábil para reconfigurar seus ambientes sem perder eventos. Após 24h, a chave antiga é revogada definitivamente.
5. **Prevenção a Ataques de Replay:** Envio do cabeçalho `X-Timestamp` contendo a data/hora em ISO 8601 da expedição, permitindo ao receptor validar a janela de tolerância de frescor da requisição ([09:44] Diego).
6. **Obrigatoriedade de TLS (HTTPS):** URLs cadastradas devem obrigatoriamente utilizar o protocolo `https://`. Protocolos inseguros (`http://`) serão rejeitados na camada de schema Zod ([09:23] Sofia).
7. **Teto de Carga (Payload Limit):** Limite máximo de **64 KB** por evento. Payloads superiores serão bloqueados com erro explícito de validação ([09:24] Diego, Larissa).

## Alternativas Consideradas

### 1. Secret Global Única para Toda a Plataforma
- **Descrição:** Usar uma única chave mestra compartilhada entre todos os clientes da empresa.
- **Trade-off de Descarte:** Rejeitado categoricamente pela Engenharia de Segurança ([09:21] Sofia). Se um único cliente vazar sua chave em repositório público ou log de aplicação, a segurança de todos os demais clientes ficaria vulnerável.

### 2. Assinatura Assimétrica com Chaves Públicas/Privadas (RSA ou Ed25519)
- **Descrição:** A plataforma assina com chave privada e publica chaves públicas via JWKS/certificados.
- **Trade-off de Descarte:** Rejeitado por complexidade de integração desproporcional. A maioria esmagadora das integrações de webhooks no mercado B2B (ex.: Stripe, Shopify, GitHub) opera com HMAC simétrico, cuja validação requer poucas linhas de código em qualquer linguagem.

### 3. Validação Apenas por IP de Origem (Allowlist de Firewall)
- **Descrição:** O cliente cadastra os IPs públicos dos servidores de saída da empresa.
- **Trade-off de Descarte:** Frágil e não garante integridade do payload contra ataques em trânsito. Adicionalmente, em ambientes de nuvem elástica, IPs de saída podem variar e demandar manutenção constante de regras de rede.

## Consequências

### Positivas
- **Padrão Canônico de Mercado:** Compatível com ferramentas e bibliotecas populares de integração utilizadas por clientes B2B.
- **Isolamento Criptográfico:** O vazamento de uma secret compromete unicamente as notificações daquele endpoint específico, sem efeito cascata.
- **Zero Downtime em Rotação:** O período de tolerância de 24 horas garante flexibilidade operacional aos clientes durante a troca preventiva ou emergencial de chaves.

### Negativas / Mitigações
- **Responsabilidade do Cliente:** O cliente precisa implementar o cálculo e a verificação do HMAC no seu middleware receptor. *Mitigação:* Fornecer documentação de referência detalhada e snippets de código nos exemplos de integração.
- **Armazenamento de Estado de Rotação:** Necessidade de persistir colunas adicionais para `secret`, `previous_secret` e `previous_secret_expires_at` no banco de dados.
