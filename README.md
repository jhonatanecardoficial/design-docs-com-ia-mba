# Da Reunião ao Documento: Design Docs Gerados por IA

## 1. Sobre o Desafio

Este desafio técnico do MBA em Engenharia de Software com IA (Full Cycle) teve como objetivo transformar a transcrição bruta de uma reunião técnica interdisciplinar de alinhamento (`TRANSCRICAO.md`) e uma base de código legada em Node.js/TypeScript em um pacote completo, rigoroso e acionável de **Design Docs** para um sistema de Notificações Outbound via Webhooks.

O cenário simula uma situação real de engenharia de software corporativa: uma plataforma de gerenciamento de pedidos (OMS) com persistência em MySQL e Prisma ORM precisa atender clientes B2B que demandam sincronização em tempo real das mudanças de estado dos seus pedidos. O pacote de documentação produzido não inventa regras de negócio arbitrárias, preserva as fronteiras conceituais entre produto, arquitetura e implementação e estabelece rastreabilidade estrita entre cada requisito, decisão e trecho do código existente.

---

## 2. Ferramentas de IA Utilizadas

Durante todo o ciclo de engenharia reversa, análise documental e estruturação técnica, foram empregadas as seguintes ferramentas e recursos de IA:

- **Antigravity AI Agent (DeepMind Engine):** Atuou como orquestrador autônomo e parceiro técnico de engenharia de software para exploração estática do repositório, validação de tipos e análise profunda de código TypeScript/Prisma.
- **Claude 3.5 Sonnet / Gemini Pro via MCP & Subagentes:** Utilizados para síntese de linguagem natural técnica, estruturação de diagramas de sequência em sintaxe Mermaid e conferência cruzada de timestamps na transcrição de áudio transcrito.
- **Git Context & AST Search Utilities:** Ferramentas de inspeção de arquivos e sintaxe de código para extrair contratos de rotas, middlewares (`auth.middleware.ts`, `error.middleware.ts`) e o comportamento atômico de `OrderService.changeStatus`.

---

## 3. Workflow Adotado

Para assegurar consistência, separação rigorosa de responsabilidades e evitar redundâncias desnecessárias entre os documentos, adotou-se um fluxo de trabalho estruturado em 7 etapas consecutivas:

```
[1. Leitura & Mapeamento] ──> [2. 6 ADRs MADR] ──> [3. RFC Arquitetural]
                                                            │
[6. Matriz TRACKER] <── [5. PRD de Produto] <── [4. FDD Detalhado]
         │
[7. README do Processo & Auditoria Final]
```

1. **Inspeção Estática & Mapeamento:** Leitura integral dos 324 turnos de fala da `TRANSCRICAO.md` e análise dos arquivos-chave da aplicação existente (`schema.prisma`, `order.service.ts`, `app-error.ts`, `auth.middleware.ts`, `error.middleware.ts`, `logger/index.ts`).
2. **Definição das Decisões de Arquitetura (ADRs):** Elaboração isolada das 6 ADRs no formato MADR cobrindo as escolhas centrais (Outbox no MySQL, Worker em Polling de 2s, Retry com Backoff e DLQ, Autenticação HMAC-SHA256, Garantia At-Least-Once e Reuso dos Padrões do Projeto).
3. **Elaboração da Proposta Técnica (RFC):** Redação da proposta concisa (2 a 4 páginas) voltada para alinhamento e revisão de arquitetura, com foco no problema B2B, alternativas descartadas (Kafka, Triggers, Exactly-Once), questões em aberto e links diretos para as ADRs.
4. **Detalhamento da Especificação Técnica (FDD):** Aprofundamento do nível técnico de implementação com fluxos Mermaid de sequência, contratos HTTP completos (payloads, headers, status codes), contrato de cabeçalhos outbound, matriz de erros com prefixo `WEBHOOK_*`, estratégias de resiliência e a seção obrigatória de integração com caminhos reais de arquivos.
5. **Consolidação dos Requisitos de Produto (PRD):** Estruturação do documento de produto em 12 seções, contendo personas, metas quantitativas (< 10s de latência, 3 sprints até fim de novembro), 11 requisitos funcionais, requisitos não funcionais e explicitação formal dos itens descartados/fora de escopo (inbound, dashboard, email fallback, expurgo de 30 dias).
6. **Matriz de Rastreabilidade (TRACKER):** Construção da tabela de rastreabilidade com 50 itens conectando cada requisito ou decisão diretamente ao minuto/segundo e nome do falante na transcrição (84% de cobertura) ou ao arquivo real da base de código (16% de cobertura).
7. **Documentação do Processo & Auditoria:** Elaboração deste README detalhando a jornada e validação contra 100% dos critérios de aceite do desafio.

---

## 4. Prompts Customizados

Abaixo são apresentados dois dos principais prompts customizados elaborados para orientar a IA na extração de requisitos sem alucinação e na redação dos documentos com linguagem técnica apropriada.

### Prompt 1: Extração e Separação Estrita de Escopo da Transcrição

```markdown
Você é uma Tech Lead sênior e arquiteta de software especializada em sistemas distribuídos e NodeJS/TypeScript.
Sua missão é analisar o arquivo `TRANSCRICAO.md` e a base de código em `src/` e `prisma/` para extrair os fatos técnicos incontestáveis da reunião.

Regras Estritas:
1. Nunca invente ou assuma requisitos que não tenham respaldo explícito nas falas de Larissa, Marcos, Bruno, Diego ou Sofia.
2. Identifique explicitamente o que foi DESCARTADO ou POSTECIPADO (fora de escopo) e quem foi o responsável pela decisão.
3. Para cada decisão arquitetural identificada, extraia:
   - O problema motivador e o contexto.
   - A alternativa vencedora.
   - As alternativas que foram discutidas e descartadas, acompanhadas da justificativa técnica exata dada pelo falante.
   - O timestamp exato [hh:mm] e nome do autor na transcrição.
4. Estruture a resposta diferenciando claramente:
   (a) Decisões de Arquitetura (para ADRs);
   (b) Requisitos de Negócio e Métricas (para PRD);
   (c) Contratos e Integração com Código Existente (para FDD).
```

### Prompt 2: Elaboração do FDD com Integração ao Código Existente

```markdown
Atue como Engenheiro de Software Staff e elabore o documento `docs/FDD.md` (Feature Design Document) para o Sistema de Webhooks Outbound.

O documento deve ser estritamente técnico e acionável para que desenvolvedores iniciem a codificação sem ambiguidades.
Requisitos Mandatórios:
1. Inclua diagramas de sequência detalhados em Mermaid cobrindo:
   - Criação do evento outbox dentro da transação existente de pedidos.
   - Polling de 2 segundos e despacho pelo worker com assinatura HMAC-SHA256.
   - Ciclo de 5 retries com backoff exponencial (1m, 5m, 30m, 2h, 12h) e desvio para a DLQ.
   - Replay manual de registros da DLQ restrito a usuários ADMIN.
2. Forneça contratos HTTP JSON completos (Request Body, Response Body, Headers, Status Codes) para pelo menos 4 endpoints da API e para a notificação outbound (`order.status_changed`).
3. Defina a matriz de erros com o prefixo padronizado `WEBHOOK_*` estendendo a classe base `AppError`.
4. Crie a seção obrigatória "Integração com o sistema existente" citando e descrevendo a interação com pelo menos 4 caminhos reais de arquivos do projeto:
   - `src/modules/orders/order.service.ts` (especificamente o método `changeStatus` e uso de transação `tx`)
   - `prisma/schema.prisma`
   - `src/shared/errors/app-error.ts`
   - `src/middlewares/auth.middleware.ts`
   - `src/middlewares/error.middleware.ts`
   - `src/routes/index.ts`
5. Garanta que nenhuma menção seja feita a arquivos ou dependências inexistentes no projeto.
```

---

## 5. Iterações e Ajustes

Durante as sessões de geração e refinamento assistidas por IA, foram necessários ajustes críticos onde intervenções humanas e correções de prompt foram indispensáveis:

1. **Separação de Fronteiras entre RFC e FDD (Evitando Duplicação de Camada):**
   - *Problema Inicial:* Na primeira proposta de RFC, a IA gerou esquemas completos de Prisma, contratos detalhados de endpoints JSON e queries SQL diretamente no RFC. Isso violava a diretriz de que o RFC deve ser um documento conciso de arquitetura (2 a 4 páginas) focado em alinhamento de propostas, trade-offs e questões em aberto.
   - *Correção Aplicada:* O prompt foi reestruturado para impor um teto de concisão no RFC, movendo todo o detalhamento de contratos, matriz de erros `WEBHOOK_*`, esquemas do banco e código TypeScript para o `docs/FDD.md`, mantendo no RFC apenas os diagramas conceituais de blocos, trade-offs e links cruzados para as ADRs.

2. **Diferenciação Estrita entre "Em Escopo" e "Fora de Escopo":**
   - *Problema Inicial:* A IA listou em versões preliminares do PRD e FDD funcionalidades como "Notificações por E-mail de Fallback", "Dashboard Gráfico de Gerenciamento" e "Expurgo Automático de Logs de 30 Dias" como se fossem entregáveis ativos do MVP.
   - *Correção Aplicada:* Foi realizada conferência com os timestamps `[09:08] Diego`, `[09:37] Larissa` e `[09:40] Larissa` da transcrição, forçando a IA a reposicionar esses quatro tópicos na seção explícita de "Fora de Escopo / Postecipado para Fases Futuras", evitando contaminação do backlog da sprint.

3. **Alinhamento do Mecanismo de Autenticação e Rotação de Segredos:**
   - *Problema Inicial:* A IA sugeriu autenticar as requisições de webhooks enviadas para os clientes parceiros utilizando cabeçalho `Authorization: Bearer <JWT>`.
   - *Correção Aplicada:* Corrigiu-se a diretriz com base nas falas de Sofia (`[09:20]` e `[09:21]`), demonstrando que o padrão de mercado para webhooks outbound é a assinatura criptográfica de integridade de payload via `X-Signature: sha256=<HMAC-SHA256>` utilizando um segredo compartilhado exclusivo por endpoint, com janela de carência de 24 horas na rotação.

---

## 6. Como Navegar a Entrega

A estrutura documental gerada está organizada na pasta `docs/` e segue uma hierarquia de leitura recomendada para diferentes papéis:

```
.
├── README.md                                  # Documentação do processo de produção com IA
├── TRANSCRICAO.md                             # Transcrição original da reunião técnica
└── docs/
    ├── PRD.md                                 # Product Requirements Document (Visão de Produto e Negócio)
    ├── RFC.md                                 # Request for Comments (Proposta Técnica e Alternativas)
    ├── FDD.md                                 # Feature Design Document (Especificação Detalhada de Código)
    ├── TRACKER.md                             # Matriz de Rastreabilidade e Proveniência (50 itens mapeados)
    └── adrs/                                  # Architecture Decision Records (Decisões em formato MADR)
        ├── ADR-001-outbox-no-mysql.md
        ├── ADR-002-worker-separado-em-polling.md
        ├── ADR-003-retry-backoff-exponencial-e-dlq.md
        ├── ADR-004-autenticacao-hmac-sha256-secret-por-endpoint.md
        ├── ADR-005-garantia-at-least-once-com-x-event-id.md
        └── ADR-006-reuso-padroes-existentes-do-projeto.md
```

### Ordem Sugerida de Leitura:
1. **Para Product Managers e Gestores:** Inicie por `docs/PRD.md` para compreender a dor dos clientes B2B (Atlas, MaxDistribuição e Nova Cargo), as métricas de sucesso e os limites de escopo.
2. **Para Arquitetos de Software e Tech Leads:** Leia `docs/RFC.md` para entender a visão sistêmica e consulte as decisões detalhadas na pasta `docs/adrs/`.
3. **Para Engenheiros de Implementação:** Aprofunde-se em `docs/FDD.md` para consultar diagramas de sequência, contratos de API, matriz de erros e a integração com o código TypeScript existente.
4. **Para Auditores e Revisores Técnicos:** Utilize `docs/TRACKER.md` para verificar a rastreabilidade direta de cada item em relação aos diálogos de `TRANSCRICAO.md` e aos arquivos do projeto.
