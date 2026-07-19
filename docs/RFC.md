# RFC — Webhooks de Notificação de Mudança de Status de Pedidos

## Metadados

| Campo | Valor |
| ----- | ----- |
| Documento | RFC — Webhooks de Notificação de Mudança de Status de Pedidos |
| Autor | Larissa — Tech Lead (conduziu a reunião técnica) |
| Status | Em revisão |
| Data | Não informada. `TRANSCRICAO.md` registra apenas "quinta-feira, 09:00", sem data de calendário |
| Revisores | Larissa (Tech Lead), Marcos (Product Manager), Bruno (Engenheiro Pleno — time de Pedidos), Diego (Engenheiro Sênior — time de Plataforma) e Sofia (Engenheira de Segurança) — os cinco participantes da reunião técnica registrada em `TRANSCRICAO.md` |
| Sistema | Order Management System |
| Escopo | Webhooks outbound para mudanças de status |
| ADRs relacionados | ADR-001 a ADR-006 |

## Resumo executivo

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — pediram formalmente para serem notificados quando o status de seus pedidos muda. Hoje consultam `GET /orders` repetidamente, o que torna a integração lenta e cara do lado deles (`[09:00]` Marcos). Este RFC consolida a proposta técnica integrada para substituir esse polling por **webhooks outbound**: a plataforma envia, o cliente recebe, sem caminho inbound (`[09:02]` Marcos; `[09:03]` Sofia).

A proposta articula seis decisões já aceitas. O evento é criado por **Transactional Outbox no MySQL existente**, dentro da mesma transação da mudança de status, de modo que status confirmado e evento existente sejam o mesmo fato (ADR-001). A entrega cabe a um **worker em processo separado da API**, que consulta a outbox em **polling de 2 segundos** (ADR-002) — intervalo que atende com folga a janela de "tempo real" definida pelo negócio, abaixo de 10 segundos. Entregas malsucedidas são retentadas em **intervalos de 1m, 5m, 30m, 2h e 12h**, e o que esgota o limite vai para uma **DLQ em tabela dedicada**, com replay manual restrito a administradores (ADR-003).

Cada entrega é assinada com **HMAC-SHA256** sobre o corpo enviado, com **secret própria por endpoint** — nunca global — e **rotação em que a secret anterior permanece válida por 24 horas** antes de ser invalidada (ADR-004). A semântica declarada é **at-least-once**: o mecanismo poderá produzir entregas duplicadas enquanto o evento permanecer no fluxo, e cada evento carrega identidade estável no header **`X-Event-Id`** para o consumidor deduplicar do seu lado; exactly-once foi descartado (ADR-005). A feature é incorporada **reutilizando os padrões existentes** — estrutura modular, `AppError` e middleware de erro central, schemas Zod, Pino, `authenticate`/`requireRole` — sem criar uma segunda arquitetura na mesma aplicação (ADR-006).

A primeira versão não introduz infraestrutura de mensageria: o MySQL já em uso é o substrato dos eventos, escolha coerente com o tamanho do time (`[09:07]` Diego). O documento apresenta a visão integrada e mantém explicitamente abertas as lacunas que pertencem ao FDD.

## Status

`Em revisão`

Este RFC consolida decisões registradas em ADR-001 a ADR-006, todos com status `Aceito`. Não redefine nenhuma delas, não cria decisão arquitetural nova e não resolve as lacunas herdadas. O status permanece `Em revisão` até a auditoria final do próprio RFC.

## Contexto e motivação

Clientes B2B integram-se hoje por consulta repetida a `GET /orders`, verificando periodicamente se algum pedido mudou de estado (CAN-PROB-001). O modelo é lento e caro para o cliente, e a percepção de atraso é estrutural: o intervalo entre a mudança e a descoberta depende da frequência com que o cliente decide perguntar (CAN-PROB-002).

Três clientes pediram formalmente a notificação (CAN-PROB-003). "Tempo real", para eles, foi definido de forma explícita e verificável: qualquer coisa **abaixo de 10 segundos** já atende, e o requisito real é não depender de atualização manual (`[09:02]` Marcos — CAN-MET-001). Há risco comercial associado: a Atlas sinalizou possível migração para o concorrente caso a entrega não ocorra no prazo (`[09:00]` e `[09:45]` Marcos — CAN-RISK-008).

A direção da integração foi delimitada logo no início: os webhooks saem da plataforma para o cliente; o sistema do cliente **não envia eventos de volta** (`[09:02]` Marcos). Trata-se, portanto, de integração **outbound**, o que dispensa desta feature qualquer superfície de recebimento, verificação de assinatura de terceiros ou endpoint público de ingestão.

O detalhamento de negócio — personas, priorização e comunicação aos clientes — pertence ao PRD e não é reproduzido aqui.

## Problema técnico

Notificar exige uma chamada HTTP a um sistema de terceiro, e essa chamada é a origem do problema:

- **Chamadas externas falham e demoram.** O endpoint do cliente pode estar fora do ar, lento ou inacessível, e nada disso está sob controle da plataforma.
- **A chamada não pode ocorrer dentro da transação de pedidos.** A transação de `changeStatus` já é pesada — atualiza pedido, histórico e estoque (`[09:04]` Bruno). Um HTTP no meio dela acoplaria a mudança de status à disponibilidade do cliente, e um cliente lento travaria mudanças de outros pedidos.
- **Mudança de status e criação do evento precisam ser atômicas.** Se o status commitar sem evento, o pedido muda e o cliente nunca é notificado, sem registro do que se perdeu (`[09:40]` Bruno; `[09:41]` Diego — CAN-RISK-005).
- **A API não pode ficar bloqueada pela disponibilidade do cliente.** Contra um cliente fora do ar, a única reação síncrona seria abortar a mudança de status — descartada de imediato (`[09:04]` Bruno).
- **Retentativas produzem duplicidade.** Recuperar falhas implica reenviar; reenviar implica que o cliente pode receber o mesmo evento mais de uma vez (CAN-RISK-002).
- **O cliente precisa validar autenticidade.** Qualquer terceiro pode abrir uma conexão TLS válida contra um endpoint público e enviar um corpo bem formado (`[09:19]` Sofia).
- **Falhas permanentes precisam ser diagnosticáveis.** Sem contexto preservado, um evento que nunca chegou é indistinguível de um evento que nunca existiu.
- **O sistema precisa operar sem nova infraestrutura de mensageria na v1**, dado o tamanho do time (`[09:07]` Diego).

## Objetivos

1. Notificar clientes B2B sobre mudanças de status de seus pedidos por webhook outbound.
2. Garantir atomicidade entre a mudança de status confirmada e a existência do evento.
3. Processar as entregas de forma assíncrona, fora do ciclo da requisição.
4. Manter qualquer chamada HTTP externa fora da transação de mudança de status.
5. Recuperar automaticamente falhas temporárias de entrega, sem intervenção humana.
6. Separar falhas permanentes do fluxo ativo, preservando contexto para diagnóstico.
7. Permitir replay administrativo e auditável do que esgotou a recuperação automática.
8. Permitir ao consumidor validar criptograficamente a origem e a integridade de cada entrega.
9. Permitir ao consumidor deduplicar entregas repetidas por meio de identidade estável do evento.
10. Manter compatibilidade com os padrões arquiteturais já existentes no projeto.
11. Atender à meta de latência inferior a 10 segundos **em condições normais** (CAN-MET-001).

Nenhum destes objetivos promete entrega absoluta. A semântica oferecida é at-least-once, com os limites descritos em *Semântica de entrega*.

## Não objetivos

**Fora de escopo** — não pertence a esta feature:

- Dashboard ou painel visual para o cliente: projeto separado do time de frontend; nesta fase, apenas endpoints (`[09:40]` Larissa — CAN-FORA-002).
- Painel de administração completo da DLQ. O que existe é o endpoint de replay.
- Comunicação inbound por webhook: o cliente recebe, não envia (`[09:02]` Marcos).
- Refatoração geral da aplicação e redesign da autenticação. Nenhuma fonte as exige, e o padrão atual já sustenta a feature (ADR-006).
- Introdução de Redis, Kafka ou infraestrutura externa de mensageria (CAN-ALT-002).

**Adiado** — deixado para fase futura, sem invalidação técnica:

- Alertas por e-mail ao cliente quando o webhook dele falha repetidamente: "Talvez próxima fase, depois que a gente medir o impacto" (`[09:37]` Larissa — CAN-FORA-001).
- Rate limiting de saída: "observar e implementar se virar problema" (`[09:39]` Diego e Larissa — CAN-OPEN-003).
- Múltiplos workers em paralelo: "problema do futuro, não agora" (`[09:13]` Diego — CAN-FORA-005).
- Arquivamento automático das linhas entregues da outbox, citado como ~30 dias e declarado fora do escopo desta feature (`[09:08]` Diego — CAN-FORA-003).
- Endurecimento da autorização do CRUD de configuração: "mais pra frente a gente pode endurecer" (`[09:37]` Sofia — CAN-FORA-004).
- Suporte a eventos além de `order.status_changed`: a taxonomia de `event_type` não foi definida e permanece como evolução futura (CAN-LAC-011).

**Alternativa descartada** — avaliada e rejeitada, com registro no ADR correspondente:

- Ordering global e exactly-once (CAN-ALT-007, CAN-ALT-011).
- Disparo síncrono na mudança de status (CAN-ALT-001).

## Escopo funcional

A feature abrange: o cadastro e a gestão, via API autenticada, dos endpoints de webhook de um cliente, incluindo o filtro dos status que cada endpoint deseja receber; a geração e a rotação da secret de assinatura por endpoint; a emissão de um evento `order.status_changed` a cada transição de status válida que interesse a algum endpoint cadastrado; o processamento assíncrono e a entrega HTTP assinada desses eventos; a retentativa automática, o encaminhamento à DLQ e o replay administrativo; e a consulta ao histórico das entregas realizadas.

Não abrange a apresentação visual desses dados, a notificação por outros canais, nem qualquer alteração nas regras de negócio de pedidos, cuja máquina de transições permanece intacta em `src/modules/orders/order.status.ts`.

## Requisitos

### Requisitos funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| RF-01 | O cliente cria um webhook informando URL, lista de status desejados e `customer_id`. | CAN-RF-002 — `[09:31]` Marcos |
| RF-02 | O cliente lista os webhooks de um customer. | CAN-RF-006 — `[09:33]` Bruno |
| RF-03 | O cliente atualiza um webhook. | CAN-RF-004 — `[09:33]` Bruno |
| RF-04 | O cliente remove um webhook. | CAN-RF-005 — `[09:33]` Bruno |
| RF-05 | Cada endpoint assina um subconjunto dos status do pedido (filtro de eventos). | CAN-RF-007 — `[09:33]` Marcos |
| RF-06 | O sistema emite um evento a cada mudança de status de pedido que interesse a algum endpoint. | CAN-RF-001 — `[09:00]`/`[09:43]` |
| RF-07 | A secret é gerada pela plataforma e devolvida na resposta de criação. | CAN-RF-003 — `[09:31]` Marcos |
| RF-08 | O cliente solicita a rotação da secret pela API. | CAN-RF-010 — `[09:21]` Sofia |
| RF-09 | O cliente consulta as últimas 100 entregas de um webhook, com sucesso/falha, payload, response e tempo de resposta. | CAN-RF-008, CAN-MET-009 — `[09:34]` Marcos |
| RF-10 | Entregas malsucedidas são retentadas automaticamente. | CAN-RF-012 — `[09:15]` Diego |
| RF-11 | Esgotadas as tentativas, o evento é encaminhado à DLQ. | CAN-RF-013 — `[09:17]` Larissa |
| RF-12 | Um administrador executa replay manual de um item da DLQ. | CAN-RF-009 — `[09:18]`/`[09:35]` Diego |
| RF-13 | O sistema registra quem executou o replay, para auditoria. | CAN-RNF-006 — `[09:36]` Sofia |
| RF-14 | Cada entrega envia os headers definidos em *Headers da entrega*. | CAN-RF-014, CAN-RF-016, CAN-RF-017 — `[09:44]` |
| RF-15 | Cada entrega envia o payload definido em *Payload do evento*. | CAN-DEC-024 — `[09:43]` Diego |
| RF-16 | O evento gravado na outbox contém o payload renderizado (snapshot) do momento da inserção. | CAN-RF-018, CAN-DEC-023 — `[09:52]` Larissa e Diego |

Os caminhos de RF-08 não foram definidos na reunião e não são inventados aqui; ver *Endpoints da API*.

### Requisitos não funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| RNF-01 | Entrega em menos de 10 segundos em condições normais. | CAN-MET-001 — `[09:02]` Marcos |
| RNF-02 | O worker consulta a outbox a cada 2 segundos. | CAN-MET-002 — `[09:09]` Diego |
| RNF-03 | O polling pode adicionar até aproximadamente 2 segundos de espera **apenas para a descoberta** do evento; não é latência total nem garantia de entrega em 2s. | CAN-MET-003 — `[09:10]` Larissa |
| RNF-04 | Timeout de 10 segundos por chamada HTTP; o estouro é tratado como falha e marcado para retry. | CAN-MET-008, CAN-RNF-012 — `[09:42]` Diego |
| RNF-05 | Payload máximo de 64 KB; acima disso, erro — não trunca. | CAN-MET-007, CAN-RNF-005 — `[09:24]` |
| RNF-06 | A URL do webhook deve ser HTTPS; `http` é recusada em validação. | CAN-RNF-004 — `[09:23]` Sofia |
| RNF-07 | Assinatura HMAC-SHA256 sobre o corpo do request. | CAN-MET-014, CAN-RNF-001 — `[09:20]` Sofia |
| RNF-08 | Uma secret única por endpoint; não há secret global. | CAN-RNF-002 — `[09:21]` Sofia |
| RNF-09 | Rotação com a secret anterior válida por 24 horas em paralelo, depois invalidada. | CAN-MET-006, CAN-RF-011 — `[09:21]` Sofia |
| RNF-10 | Cinco tentativas antes da falha permanente. | CAN-MET-004 — `[09:17]` Larissa |
| RNF-11 | Intervalos de 1m, 5m, 30m, 2h e 12h. | CAN-MET-005 — `[09:17]` Diego |
| RNF-12 | Semântica de entrega at-least-once. | CAN-RNF-009 — `[09:24]` Diego |
| RNF-13 | O worker executa em processo separado da API. | CAN-RNF-011 — `[09:11]` Diego |
| RNF-14 | Não há garantia de ordenação global. | CAN-RNF-013 — `[09:13]` Larissa |
| RNF-15 | A feature integra-se aos padrões existentes do projeto. | CAN-DEC-016 — `[09:30]` Larissa |

## Estado atual do sistema

Comportamentos **confirmados no repositório**:

- Aplicação Node.js com TypeScript (`engines.node` `>=20`), módulos ESM.
- Express 4 em `src/app.ts`, com `express.json({ limit: '1mb' })`, rota `/health` e prefixo global `/api/v1`.
- Prisma 5 sobre **MySQL** (`prisma/schema.prisma`, `datasource db { provider = "mysql" }`).
- Autenticação JWT bearer em `authenticate`, que popula `req.user` com `id`, `email` e `role`.
- Dois papéis: `ADMIN` e `OPERATOR` (enum `UserRole`; `AuthUser['role']`).
- Autorização por papel via `requireRole(...)`, hoje aplicada em uma única rota real: `GET /users/:id`.
- Validação com Zod pelo middleware `validate`, que converte `ZodError` em `ValidationError`.
- Logger Pino em `src/shared/logger/index.ts`, com `redact` cobrindo `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token` e `*.accessToken`.
- `AppError` com `statusCode`, `errorCode` e `details`, e subclasses com códigos em maiúsculas.
- Middleware de erro centralizado, que trata `AppError`, `ZodError` e erros conhecidos do Prisma, respondendo `{ error: { code, message, details? } }`.
- Composição manual de dependências em `buildControllers`, sem container de injeção.
- Mudança de status transacional: `OrderService.changeStatus` executa, dentro de `prisma.$transaction`, a leitura do pedido, a validação da transição, o ajuste de estoque, `tx.order.update` e `tx.orderStatusHistory.create`.
- Histórico de status persistido em `OrderStatusHistory`.
- Regras de estoque em `shouldDebitStock`/`shouldReplenishStock`, com débito em `PENDING → PAID` e reposição ao cancelar a partir de `PAID` ou `PROCESSING`.
- Testes com Vitest e Supertest; `tests/setup.ts` limpa tabelas em `beforeEach`.
- PK em UUID `@db.Char(36)` em todos os models de domínio (`User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory`); a exceção é `OrderNumberSequence`, tabela de sequência com PK `Int @id @default(1)`. Migrations Prisma já inicializadas.

**Ainda não existem** no repositório, e serão criados por esta feature:

- Módulo de webhooks (`src/modules/webhooks`).
- Tabela e model de outbox.
- Worker e o entry-point `src/worker.ts`.
- Tabela e model de dead-letter.
- Tabela e model de histórico de entregas (deliveries).
- Utilitário de assinatura HMAC e qualquer código criptográfico.
- Secrets de webhook, sua geração e sua rotação.
- Headers `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id`.
- Script `worker` em `package.json`.
- Classes de erro com prefixo `WEBHOOK_`.

## Visão geral da solução

A solução separa **produzir o evento** de **entregar o evento**. A produção é síncrona, transacional e barata; a entrega é assíncrona, externa e sujeita a falha. O ponto de articulação entre as duas é a outbox.

```text
Mudança de status
        │
        ▼
OrderService.changeStatus
        │
        ▼
Transação Prisma
 ├── valida transição
 ├── atualiza estoque
 ├── atualiza pedido
 ├── registra histórico
 └── registra evento na Outbox
        │
        ▼
Commit
        │
        ▼
Worker separado
        │
        ▼
Polling da Outbox
        │
        ▼
Preparação do payload
        │
        ▼
Assinatura HMAC-SHA256
        │
        ▼
Entrega HTTP
        │
        ├── sucesso → histórico
        ├── falha → retry
        └── limite esgotado → DLQ
```

A propriedade central é que o commit produz simultaneamente o status novo e o evento, e o rollback não produz nenhum dos dois. Tudo o que ocorre depois do commit — descoberta, assinatura, envio, retentativa — acontece fora do caminho da requisição e não pode desfazer nem bloquear a mudança de status já confirmada.

## Arquitetura proposta

### API de configuração de webhooks

Superfície REST autenticada, responsável pelo CRUD dos endpoints de webhook de um customer, pelo filtro de status assinados por endpoint, pela rotação da secret e pela consulta ao histórico de entregas. Seguirá a estrutura modular do projeto em `src/modules/webhooks` (ADR-006).

### Publicação transacional do evento

Responsável por identificar as configurações interessadas na transição, produzir o snapshot do payload, criar o `event_id` e inserir o evento na **mesma transação** da mudança de status. O filtro é aplicado **na inserção, não no envio**: se nenhum webhook do customer assina aquele status, a linha não é inserida (`[09:34]` Bruno e Diego — CAN-DEC-020).

A integração acordada é a função `publishWebhookEvent(tx, order, fromStatus, toStatus)`, que recebe o `Prisma.TransactionClient` já aberto e evita injetar um repository inteiro no `OrderService` (`[09:41]` Bruno e Diego — CAN-DEC-021). Ela **não existe hoje**; criá-la é integração futura (CAN-INT-001).

### Outbox

Responsável por persistir os eventos no MySQL existente, manter identidade e payload e permitir o processamento posterior. É o que torna a retentativa possível: sem evento persistido, não há o que retentar.

Schema completo, conjunto de estados e índices definitivos **não são definidos** aqui. Diego citou índice em status e `created_at` e os estados "pendente, processando, falhou, entregue" (`[09:08]`), sem fechamento (CAN-LAC-001).

### Worker

Processo separado da API (`[09:11]` Diego), responsável por consultar a outbox, selecionar eventos elegíveis, entregar, registrar resultados, reagendar retentativas e encaminhar à DLQ o que esgotar o limite. Usará instância própria de `PrismaClient`, por ser outro processo Node, contra a mesma `DATABASE_URL` (`[09:30]` Bruno — CAN-DEC-017, CAN-DEC-018).

Claim, locking, tamanho de lote, concorrência interna e recuperação após reinício **não são definidos aqui** (CAN-LAC-006, CAN-LAC-014).

### Entrega HTTP

Responsável por preparar o corpo final, calcular a assinatura, montar os headers e executar a chamada HTTPS, com timeout de 10 segundos. A biblioteca ou cliente HTTP **não foi decidido** (CAN-LAC-013).

### Histórico de entregas

Responsável por registrar as tentativas — sucesso ou falha, payload, resposta e tempo de resposta —, sustentando a consulta às últimas 100 entregas de um webhook (`[09:34]` Marcos). Campos e mecanismo de captura da resposta **não foram definidos** (CAN-LAC-005).

### Retry e backoff

Responsável por reagendar entregas malsucedidas segundo a progressão decidida, respeitando o próximo momento elegível e o limite finito de tentativas (ADR-003).

### Dead-Letter Queue

Responsável por armazenar, em **tabela separada da outbox**, os eventos que esgotaram a recuperação automática, preservando contexto para diagnóstico e permitindo replay — payload, motivo da falha e timestamp; nome citado, `webhook_dead_letter` (`[09:18]` Diego). Estar na DLQ não significa descarte definitivo.

### Replay administrativo

Responsável por recolocar um item da DLQ no fluxo, restrito à role `ADMIN` e auditado quanto a quem o executou (`[09:36]` Sofia e Larissa).

### Gestão de secrets

Responsável por manter secret individual por endpoint, gerá-la no cadastro, suportar rotação, manter a anterior válida por 24 horas e invalidá-la depois, além de fornecer o material de assinatura à entrega (ADR-004). Geração, armazenamento e apresentação **não foram definidos** e passarão pela revisão de segurança (CAN-LAC-003, CAN-LAC-004).

## Fluxos principais

### Cadastro de webhook

1. Usuário autenticado solicita a criação de um webhook.
2. O sistema valida os dados e exige URL HTTPS.
3. O sistema associa o webhook ao customer.
4. O sistema gera a secret.
5. O sistema persiste a configuração.
6. O sistema devolve os dados da criação, incluindo a secret (`[09:31]` Marcos).

Permanecem abertas: a localização do `customer_id` (body ou path); o tamanho e o encoding da secret; se a secret é exibida uma única vez; e se há criptografia em repouso.

### Atualização de webhook

1. Usuário autenticado solicita a alteração de um webhook existente.
2. O sistema valida os dados, incluindo a exigência de HTTPS quando a URL muda.
3. O sistema persiste a alteração e responde com o estado atualizado.

Os campos atualizáveis dependem do conjunto de campos da configuração, não fechado (CAN-LAC-002).

### Remoção de webhook

1. Usuário autenticado solicita a remoção.
2. O sistema remove a configuração e confirma a operação.

O comportamento sobre eventos já persistidos e ainda não entregues quando a configuração é removida **não foi definido** e pertence ao FDD.

### Rotação de secret

1. O cliente solicita a rotação pela API.
2. Uma nova secret é criada e torna-se utilizável.
3. A secret anterior permanece válida por 24 horas.
4. O cliente atualiza seus sistemas dentro dessa janela.
5. Após o prazo, a anterior é invalidada — "Depois disso, a antiga morre" (`[09:21]` Sofia).

Não são definidos aqui: o instante em que as 24 horas começam; se a nova secret passa a assinar imediatamente; nem qualquer mecanismo de duas assinaturas simultâneas — nada nas fontes o sustenta.

### Mudança de status e criação do evento

1. O pedido é carregado.
2. A transição é validada.
3. O estoque é atualizado quando a transição o exige.
4. O pedido é atualizado.
5. O histórico de status é criado.
6. As configurações interessadas naquele status são avaliadas.
7. O evento é persistido na outbox, com snapshot do payload e `event_id`.
8. A transação é confirmada.
9. Qualquer falha transacional — inclusive na inserção do evento — provoca rollback de toda a operação.

### Processamento e entrega

1. O worker encontra um evento elegível no ciclo de polling.
2. Carrega a configuração do endpoint de destino.
3. Prepara o corpo final da entrega.
4. Calcula o HMAC-SHA256 sobre os bytes desse corpo.
5. Adiciona os headers definidos.
6. Executa a chamada HTTPS, com timeout de 10 segundos.
7. Registra o resultado da tentativa.

Permanecem abertos: o cliente HTTP; a classificação das respostas; a existência de concorrência interna; a estratégia de claim; e o locking.

### Falha e reagendamento

1. A entrega não é bem-sucedida.
2. A tentativa e sua evidência são registradas.
3. Se o limite não foi esgotado, o evento recebe novo momento elegível conforme o intervalo aplicável.
4. O evento volta a ser considerado pelo worker no momento apropriado.

| Etapa | Espera |
| ----- | -----: |
| 1 | 1 minuto |
| 2 | 5 minutos |
| 3 | 30 minutos |
| 4 | 2 horas |
| 5 | 12 horas |

**Ambiguidade registrada, não resolvida:** a transcrição não define se "cinco tentativas" significa cinco envios totais, incluindo o primeiro, ou cinco retentativas após o primeiro envio. Diego propôs "5" (`[09:15]`); Larissa fechou "5 tentativas" (`[09:17]`); nenhum qualificou o primeiro envio. Há um indício aritmético — os cinco intervalos somam ~14h36min, compatível com "quase 15 horas entre primeira falha e última tentativa" (`[09:17]` Diego) —, não conclusivo e não adotado aqui. O FDD deverá esclarecer a contagem antes da implementação.

### Esgotamento e DLQ

1. O limite de tentativas é esgotado.
2. O evento deixa o fluxo ativo.
3. O evento é registrado na DLQ, em estrutura separada da outbox, com o contexto da falha.
4. O item permanece disponível para replay.

Permanecem abertos: se a passagem é transferência atômica ou cópia; a retenção; e o contador de tentativas.

### Replay administrativo

1. Um usuário com role `ADMIN` solicita o replay de um item da DLQ.
2. O sistema autoriza por `requireRole('ADMIN')`.
3. O sistema registra quem executou o replay, para auditoria.
4. O evento retorna ao fluxo — "Recoloca na outbox como pendente" (`[09:18]` Diego).

Permanecem abertos: o contador de tentativas após o replay; a retenção dos itens; o replay em lote; e **a identidade do evento durante o replay** — se conserva o `event_id` original, recebe um novo ou constitui novo evento lógico correlacionado. A escolha altera o que o consumidor observa e pertence ao FDD.

O replay não garante sucesso: o reenvio está sujeito às mesmas falhas que levaram o evento à DLQ.

### Consulta ao histórico de entregas

1. Usuário autenticado solicita o histórico de um webhook.
2. O sistema retorna as últimas 100 entregas, com sucesso/falha, payload, resposta e tempo de resposta (`[09:34]` Marcos).

Paginação e parâmetros além de "últimos 100" não foram definidos (CAN-LAC-008).

## Contratos em alto nível

### Endpoints da API

| Operação | Método e caminho | Autorização | Situação |
| -------- | ---------------- | ----------- | -------- |
| Criar webhook | `POST /api/v1/webhooks` | Autenticado | Proposto |
| Listar webhooks | `GET /api/v1/webhooks` | Autenticado | Proposto |
| Atualizar webhook | `PATCH /api/v1/webhooks/:id` | Autenticado | Proposto |
| Remover webhook | `DELETE /api/v1/webhooks/:id` | Autenticado | Proposto |
| Consultar deliveries | `GET /api/v1/webhooks/:id/deliveries` | Autenticado | Decidido em alto nível |
| Rotacionar secret | Caminho em aberto | Autenticado | Em aberto |
| Reprocessar DLQ | `POST /api/v1/admin/webhooks/dead-letter/:id/replay` | ADMIN | Decidido |

Distinção necessária entre o que foi decidido e o que é proposta de consistência:

- **Decidido na reunião** — a forma `POST /admin/webhooks/dead-letter/:id/replay` (`[09:18]` e `[09:35]` Diego) e a existência de `GET /webhooks/:id/deliveries` (`[09:34]` Marcos). Os métodos do CRUD foram citados por verbo — `POST` para criar, `PATCH` para editar, `DELETE` para remover, `GET` para listar (`[09:31]` Marcos; `[09:33]` Bruno) —, sem caminho completo.
- **Proposto para consistência REST** — os caminhos completos sob `/api/v1`, derivados do prefixo global real de `src/app.ts` e do padrão de agregação de `src/routes/index.ts`. A montagem exata sob o prefixo pertence ao FDD.
- **Em aberto** — o caminho da rotação de secret. A reunião decidiu que existe rotação via API (`[09:21]` Sofia), sem definir rota. Nenhum caminho é inventado aqui.

### Payload do evento

```json
{
  "event_id": "uuid",
  "event_type": "order.status_changed",
  "timestamp": "ISO-8601",
  "order_id": "uuid",
  "order_number": "string",
  "from_status": "PAID",
  "to_status": "PROCESSING",
  "customer_id": "uuid",
  "total_cents": 10000
}
```

O corpo é um **snapshot do momento da mudança**: é renderizado na inserção na outbox e preserva o estado do pedido no instante em que o status mudou, mesmo que o pedido mude depois (`[09:52]` Larissa e Diego). O payload **não inclui `items`**, para não inflar; o cliente que precisar de detalhes consulta `GET /orders/:id` (`[09:43]` Diego). O tamanho é limitado a 64 KB; acima disso, erro, sem truncamento (`[09:24]`).

Apenas os campos acima foram decididos. Versão de contrato, metadata, número da tentativa, identificador da entrega, links, assinatura embutida no corpo e demais campos **não têm sustentação nas fontes** e não são adicionados.

### Headers da entrega

```text
Content-Type: application/json
X-Signature
X-Event-Id
X-Timestamp
X-Webhook-Id
```

- **`Content-Type: application/json`** — o corpo é JSON.
- **`X-Signature`** — assinatura HMAC-SHA256 do corpo, para o cliente verificar origem e integridade (`[09:20]` Sofia).
- **`X-Event-Id`** — identidade estável do evento lógico, base da deduplicação do lado do cliente (`[09:25]` Diego).
- **`X-Timestamp`** — timestamp do envio, para o cliente "detectar replay attack se quiser" (`[09:44]` Diego).
- **`X-Webhook-Id`** — id do cadastro de webhook, para o cliente com vários endpoints saber qual cadastro originou o envio (`[09:44]` Sofia).

Formato, encoding, eventual prefixo do valor da assinatura, formato do timestamp e versionamento de header **não foram definidos** e pertencem ao FDD (CAN-INT-010).

### Semântica das respostas

O único critério fechado na reunião é o timeout: uma chamada que exceda 10 segundos é tratada como falha e marcada para retry (`[09:42]` Diego — CAN-RNF-012).

O que conta como **sucesso** de entrega, e como classificar 2xx, 3xx, 4xx, 5xx, erros de DNS, falhas de TLS, conexão recusada e redirects, **não foi definido** (CAN-LAC-007). Este RFC não escolhe política: não presume que todo erro seja retentável, que 4xx seja permanente ou que 5xx deva sempre ser retentado. A matriz de classificação é questão aberta do FDD.

## Persistência em alto nível

| Entidade futura | Finalidade | Estado |
| --------------- | ---------- | ------ |
| Configuração de webhook | URL, customer, filtros e secrets | A criar |
| Outbox | Eventos pendentes e processamento | A criar |
| Deliveries | Histórico das tentativas | A criar |
| Dead-letter | Eventos com tentativas esgotadas | A criar |

Todas as quatro entidades são novas: `prisma/schema.prisma` modela hoje apenas `User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory` e `OrderNumberSequence`. As novas tabelas usarão **PK em UUID**, seguindo o padrão do projeto — "Tudo é uuid" (`[09:51]` Larissa — CAN-DEC-022) —, e exigirão nova migration Prisma sobre o mecanismo já inicializado.

Bruno levantou em `[09:21]`, com concordância de Sofia, que a configuração armazena "url + secret + customer_id + estado ativo"; o conjunto de campos **não foi fechado** (CAN-LAC-002). Os nomes `webhook_outbox` e `webhook_dead_letter` foram citados como intenção (`[09:06]`, `[09:18]` Diego). Colunas, tipos, obrigatoriedade, índices e enums definitivos **não são definidos aqui** e pertencem ao FDD.

## Semântica de entrega

A plataforma oferece **at-least-once** (`[09:24]` Diego; `[09:26]` e `[09:48]` Larissa).

O mecanismo realizará múltiplas tentativas e **poderá produzir entregas duplicadas enquanto o evento permanecer no fluxo de processamento**. A duplicidade é comportamento esperado, não defeito: o resultado de uma tentativa nem sempre é conclusivo, e um evento processado pelo cliente cuja confirmação se perdeu será reenviado.

A expressão **não significa** que toda entrega sempre ocorrerá. A garantia se lê dentro dos limites do ADR-003: esgotadas as tentativas, o evento vai para a DLQ; a indisponibilidade permanente do endpoint impede a entrega; e o retorno de um item da DLQ ao fluxo depende de ação operacional. A DLQ é o limite da recuperação automática.

Distinção necessária: **evento lógico** é o fato de domínio ocorrido — o pedido mudou de `PAID` para `PROCESSING` —, com identidade própria expressa pelo `event_id` e criada quando o evento entra na outbox; **tentativa de entrega** é cada execução concreta de envio. Um evento pode ter várias tentativas, e as duas não são a mesma entidade.

O `X-Event-Id` identifica o evento lógico e não muda entre retentativas: sem isso, o header não cumpre a função decidida. A deduplicação é **responsabilidade do consumidor** — "ele dedupica pelo event_id do lado dele" (`[09:25]` Diego) —, transferência explicitada por Sofia (`[09:25]`) e a ser documentada aos clientes (`[09:26]` Marcos).

Retries e replay são as duas origens conhecidas de duplicidade. **Exactly-once foi descartado**: exigiria coordenação bilateral entre sistemas independentes, com complexidade desproporcional (`[09:25]` Diego — CAN-ALT-011). O `event_id` **não é mecanismo de autenticação**: um header é forjável; autenticidade é do ADR-004.

## Ordenação

A primeira versão utiliza **um único worker** (`[09:12]` Diego), que processará os eventos pendentes a partir dos mais antigos.

Há **intenção de preservar a ordem relativa dos eventos de um mesmo pedido**, e essa intenção é o que sustenta a expectativa do cliente de observar as transições daquele pedido em sequência. Não há, contudo, **garantia de ordenação global** entre pedidos diferentes: "Não é garantia de ordering global" (`[09:13]` Larissa). Marcos registrou que os clientes nunca pediram ordering global (`[09:14]`).

Três ressalvas explícitas:

- A existência de um único worker **reduz** condições de concorrência, mas **não prova** a preservação da ordem. Trata-se de limitação conhecida, não de garantia formal (CAN-RNF-014).
- **Retries podem afetar a ordem**: um evento posterior pode ser entregue enquanto um evento anterior do mesmo pedido aguarda o intervalo de backoff.
- Reinícios do processo podem afetar eventos em processamento.

A estratégia de claim, a query de seleção, o critério de desempate e a eventual concorrência interna **ainda precisam ser detalhados** e não são definidos aqui (CAN-LAC-006).

## Segurança

### Transporte

HTTPS é obrigatório; URLs `http` serão recusadas em validação: "Se o cliente cadastrar http, recusamos com erro de validação" (`[09:23]` Sofia), item que ela classificou como validação de schema Zod, não decisão arquitetural. Versão mínima de TLS, cipher suites, mTLS e política de redirects não foram discutidas.

### Autenticidade e integridade

Cada entrega é assinada com **HMAC-SHA256**, escolhido por ser padrão amplamente suportado — "todo cliente sério tem biblioteca pra isso" (`[09:20]` Sofia). A assinatura vai em `X-Signature` e é verificada pelo consumidor, sem chamada de volta à plataforma.

A assinatura deve ser calculada sobre os **bytes efetivamente enviados**: o corpo não pode ser reserializado depois de assinado, e produtor e consumidor precisam operar sobre a mesma representação. Canonicalização, ordenação de propriedades, encoding e comparação em tempo constante não foram discutidos e pertencem ao FDD.

O HMAC prova posse da secret; **não** identifica pessoa, **não** cifra o payload e **não** oferece confidencialidade — esta depende do HTTPS. Sendo simétrica a secret, não há não repúdio. Sem validação pelo consumidor, a decisão não entrega proteção alguma.

### Gestão de secrets

Uma secret por endpoint, nunca global: "Senão se vaza uma, vaza tudo" (`[09:21]` Sofia). O risco não é hipotético — a plataforma já teve cliente que vazou secret em log de aplicação (`[09:22]` Diego — CAN-RISK-003). O isolamento contém o incidente a um cliente e permite revogar apenas o afetado. A rotação é solicitada via API, com a anterior válida por **24 horas** e invalidada depois. A plataforma precisará armazenar material capaz de assinar — não basta um hash irreversível, como se faz com senha.

Geração, entropia, tamanho, encoding, armazenamento em repouso, exibição única e revogação emergencial **não foram definidos** (CAN-LAC-003, CAN-LAC-004) e passarão pela revisão de segurança de `[09:46]`. Uma secret de endpoint é credencial por cliente, não se confunde com o `JWT_SECRET` global de `src/config/env.ts`, e nada nas fontes sustenta tratá-la como variável de ambiente.

### Proteção de dados sensíveis

O logger Pino existente será reutilizado. Sua lista de `redact` cobre `authorization`, `cookie`, `*.password`, `*.passwordHash`, `*.token` e `*.accessToken` — **`secret` e assinatura não constam**, e a redaction **precisará ser ampliada ou revisada** antes de registrar objetos de webhook (CAN-INT-007). Secrets e assinaturas não devem aparecer em logs, traces ou respostas de erro; atenção ao campo `details` de erros que estendam `AppError`, devolvido ao cliente pelo middleware central. Payloads e respostas de terceiros também podem conter informação sensível.

### Replay attacks

O `X-Timestamp` acompanha cada envio e **poderá auxiliar** o cliente a detectar replay "se quiser" (`[09:44]` Diego). É o que as fontes sustentam.

**O HMAC sozinho não impede replay**: uma mensagem íntegra e corretamente assinada pode ser reenviada por terceiro que a tenha capturado. Nenhuma política anti-replay foi decidida — tolerância de relógio, janela de validade, uso do timestamp no cálculo da assinatura e verificação obrigatória pelo consumidor permanecem indefinidos. O timestamp **não é proteção completa** contra replay, e este RFC não promete que seja.

## Autenticação e autorização

O CRUD de configuração é **autenticado**, reutilizando `authenticate`, e por ora qualquer role autenticada basta: "Por enquanto sim. Mais pra frente a gente pode endurecer" (`[09:37]` Sofia — CAN-FORA-004). Este RFC não endurece o CRUD além do decidido.

O **replay é restrito à role `ADMIN`**, reaproveitando o `requireRole` existente — "Mexer em fila de entrega de notificação não é coisa de operador" (`[09:36]` Sofia; `[09:36]` Larissa). O precedente real está em `buildUserRouter`, hoje a única rota protegida por papel.

Ponto de atenção arquitetural: o **JWT atual representa o usuário operador do sistema, não o cliente**. Por isso o `customer_id` **não deve ser inferido automaticamente do token**: "o customer_id é passado no body ou no path. Não vem do JWT" (`[09:32]` Larissa — CAN-DEC-019). Se será body ou path **permanece em aberto** (CAN-OPEN-001).

Nenhuma role nova é criada: os papéis continuam sendo `ADMIN` e `OPERATOR`.

## Tratamento de falhas

Categorias que a solução distingue:

- **Timeout** — chamada que excede 10 segundos. Tratada como falha e marcada para retry. **Único critério fechado na reunião** (`[09:42]` Diego).
- **Respostas externas** — o código HTTP devolvido pelo endpoint do cliente.
- **Falhas de transporte** — erros antes ou durante o estabelecimento da conexão.
- **Retry** — reagendamento das falhas elegíveis segundo a progressão decidida.
- **DLQ** — destino do que esgota o limite, fora do fluxo ativo.

A reunião **não fechou a matriz completa** de classificação. Permanecem indefinidos: a regra de sucesso (2xx); o tratamento de 3xx e redirects; se 4xx é falha permanente; se 5xx é sempre retentável; erros de DNS; falhas de TLS; e conexão recusada (CAN-LAC-007).

Este RFC registra isso como **questão aberta do FDD** e não inventa política. As consequências de errar são simétricas e reais: classificar como retentável o que é permanente consome cinco ciclos contra um endpoint inválido; classificar como permanente o que é transitório descarta prematuramente eventos recuperáveis.

Falhas na inserção do evento seguem regra diferente e já decidida: propagam e provocam **rollback** de toda a transação, e não devem ser capturadas nem silenciadas.

## Observabilidade

Logs estruturados via Pino, já padrão do projeto, reutilizados na API e no worker (`[09:29]` Bruno — decisão).

As informações abaixo são **necessidades de implementação e propostas**, derivadas das decisões e das lacunas, **não decisões explícitas da reunião**:

- Correlação por `event_id` entre logs, tentativas e itens da DLQ.
- Identificação do `webhook_id` nos registros de entrega.
- Número da tentativa e resultado de cada envio.
- Latência da chamada externa — o tempo de resposta é dado exigido pelo histórico de entregas (`[09:34]` Marcos), o que o torna requisito de contrato, não apenas de observabilidade.
- Profundidade do backlog da outbox e do lag de processamento.
- Profundidade e composição da DLQ.
- Auditoria do replay, que é **decisão** (`[09:36]` Sofia — CAN-RNF-006); formato e destino do registro não foram definidos.

Nenhuma ferramenta de métricas, dashboard, alerta, SLO ou threshold foi discutida ou escolhida; nada disso é proposto aqui.

## Impacto no código existente

| Caminho real | Comportamento atual | Impacto futuro |
| ------------ | ------------------- | -------------- |
| `src/modules/orders/order.service.ts` | `changeStatus` executa, em `prisma.$transaction`, a leitura do pedido, a validação da transição, o ajuste de estoque, `tx.order.update` e `tx.orderStatusHistory.create`. Não cria eventos. | Deverá chamar `publishWebhookEvent(tx, ...)` dentro do mesmo callback transacional, após o histórico; falha na inserção deverá provocar rollback. |
| `src/modules/orders/order.status.ts` | Concentra a máquina de transições e os efeitos de estoque em `canTransition`, `shouldDebitStock`, `shouldReplenishStock`. | Não deverá ser alterado nem duplicado; a outbox será alimentada por transições já validadas por ele. |
| `prisma/schema.prisma` | Provider `mysql`; models `User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory`, `OrderNumberSequence`; PKs `@default(uuid()) @db.Char(36)` nos models de domínio (`OrderNumberSequence` usa PK `Int`); enum `OrderStatus`. | Deverá receber os models de configuração, outbox, deliveries e dead-letter, com PK em UUID, e nova migration. |
| `src/app.ts` | `buildControllers` compõe manualmente repositories, services e controllers; `buildApp` monta Express, `express.json({ limit: '1mb' })`, `/health`, `/api/v1` e `errorMiddleware`. | Deverá compor os componentes de webhook no mesmo ponto, sem container de injeção. |
| `src/routes/index.ts` | `Controllers` tipa os controllers e `buildApiRouter` agrega `/auth`, `/users`, `/customers`, `/products`, `/orders`. | Deverá estender o tipo `Controllers` e agregar o router de webhooks e a rota administrativa de replay. |
| `src/middlewares/auth.middleware.ts` | `authenticate` valida JWT bearer e popula `req.user`; `requireRole(...)` compara `req.user.role` contra `'ADMIN' \| 'OPERATOR'`. | Será reutilizado sem alteração: `authenticate` no CRUD, `requireRole('ADMIN')` no replay. |
| `src/middlewares/validate.middleware.ts` | `validate` valida `body`/`query`/`params` com Zod e converte `ZodError` em `ValidationError`. | Será reutilizado pelos futuros schemas de webhook, incluindo a exigência de HTTPS. |
| `src/shared/errors/app-error.ts` | `AppError` com `statusCode`, `errorCode` e `details` opcional. | Será a base das futuras classes de erro com prefixo `WEBHOOK_`. |
| `src/middlewares/error.middleware.ts` | Trata `AppError`, `ZodError` e Prisma `P2002`/`P2025`; responde `{ error: { code, message, details? } }`; fallback 500 com `logger.error`. | Não deverá ser alterado: tratará os erros `WEBHOOK_*` que estendam `AppError`. Cuidado com `details` carregando dado sensível. |
| `src/shared/logger/index.ts` | Pino com `redact` de `authorization`, `cookie`, `*.password`, `*.passwordHash`, `*.token`, `*.accessToken`; `base: { service, env }`. | Será reutilizado na API e no worker; a lista de `redact` deverá ser revisada para secrets e assinaturas. |
| `src/config/env.ts` | `envSchema` Zod valida `NODE_ENV`, `PORT`, `LOG_LEVEL`, `DATABASE_URL`, `JWT_SECRET`, `JWT_EXPIRES_IN`, com `process.exit(1)` se inválido. Nenhuma variável de webhook. | Poderá receber novas variáveis se a feature precisar; não foi decidido que polling e timeout serão env vars (CAN-INT-008). |
| `src/config/database.ts` | `createPrismaClient()` e o singleton `prisma`, consumido por `src/server.ts`. | A fábrica será reutilizada pelo worker, que abrirá instância própria por ser outro processo. |
| `src/server.ts` | `bootstrap` monta `buildApp({ prisma })`, faz `listen`, registra `SIGINT`/`SIGTERM` e um `shutdown` com `prisma.$disconnect`. | Servirá de referência de lifecycle para o futuro `src/worker.ts`, que não deverá iniciar servidor HTTP. |
| `package.json` | Scripts `dev`, `build`, `start`, `db:migrate`, `db:reset`, `db:seed`, `test`, `test:watch`, `lint`, `format`; `engines.node >= 20`; `uuid` 11.0.3 instalado; sem biblioteca criptográfica externa. | Deverá receber o script `worker`. A disponibilidade de `uuid` e de `node:crypto` é fato, não decisão de uso. |
| `tests/setup.ts` | Conecta o Prisma e limpa tabelas em `beforeEach`, respeitando as relações. | Deverá limpar as novas tabelas na ordem correta. |
| `tests/orders.test.ts` | Cobre criação, transição `PENDING → PAID` com débito de estoque, transição inválida (409), estoque insuficiente (422), reposição em `PAID → CANCELLED`, listagem e regra de exclusão. | Deverá permanecer verde: é a rede de regressão da mudança transacional em `changeStatus`. |

## Alternativas consideradas

### Chamada síncrona na mudança de status

**Benefício** — Nenhuma tabela nova, nenhum processo adicional, sem defasagem entre mudança e envio. **Custo** — Bloqueia uma transação já pesada pelo tempo de resposta de terceiros; acopla mudanças de status à disponibilidade do cliente; sem persistência prévia, não há o que retentar. **Descarte** — "Síncrono está fora de questão" (`[09:06]` Diego). Ver [ADR-001](./adrs/ADR-001-outbox-no-mysql.md).

### Redis Streams ou fila externa

**Benefício** — Mensageria nativa, com consumo reativo e recursos de distribuição. **Custo** — Infraestrutura adicional a provisionar, operar e monitorar por um time pequeno; e a publicação na fila não participaria da transação SQL, reintroduzindo o problema a resolver. **Descarte** — Overengineering: "Outbox no MySQL existente resolve" (`[09:07]` Diego). Ver [ADR-001](./adrs/ADR-001-outbox-no-mysql.md) e [ADR-002](./adrs/ADR-002-worker-separado-com-polling.md).

### Trigger de banco

**Possibilidade percebida** — Notificar o worker reativamente, evitando consultas em vazio (`[09:09]` Bruno). **Limitação** — O MySQL não tem listener nativo equivalente a `NOTIFY`/`LISTEN`; trigger executa SQL e não notifica processo externo, exigindo improvisos descritos como "esquisito". **Descarte** — "Polling de 2 segundos atende o requisito de 'abaixo de 10 segundos' tranquilo" (`[09:09]` Diego). Ver [ADR-002](./adrs/ADR-002-worker-separado-com-polling.md).

### Três tentativas

**Benefício** — Falha permanente identificada mais rápido; ciclo de vida mais curto. **Limitação** — "3 é pouco... a gente retentaria três vezes em 30 minutos e mataria", insuficiente diante do precedente de indisponibilidade planejada de duas horas (`[09:16]` Diego). **Descarte** — "Cinco fica bom" (`[09:16]` Larissa). Ver [ADR-003](./adrs/ADR-003-retry-backoff-e-dlq.md).

### Retry indefinido

**Benefício** — Nenhum evento abandonado enquanto houver chance de recuperação. **Risco** — "Evento ficar pendurado pra sempre se o cliente sumiu" (`[09:15]` Diego); crescimento ilimitado e ausência de encerramento operacional. **Descarte** — Adotado teto finito. Ver [ADR-003](./adrs/ADR-003-retry-backoff-e-dlq.md).

### Exactly-once

**Benefício aparente** — Nenhuma duplicidade; consumidor dispensado de deduplicar. **Coordenação necessária** — Exigiria confirmação bilateral entre sistemas independentes; a plataforma não participa da transação interna do cliente. **Descarte** — "At-least-once com event_id resolve 99% dos casos" (`[09:25]` Diego), com Stripe e GitHub citados como precedente. Ver [ADR-005](./adrs/ADR-005-entrega-at-least-once-com-event-id.md).

### Múltiplos workers

**Benefício** — Maior throughput e ausência de ponto único de processamento. **Complexidade** — Exigiria locking, particionamento por `order_id` ou coordenação entre consumidores, afetando a ordem pretendida. **Adiamento** — "Isso é problema do futuro, não agora" (`[09:13]` Diego). Adiado, não invalidado: a adoção exige nova decisão. Ver [ADR-002](./adrs/ADR-002-worker-separado-com-polling.md).

Outras alternativas registradas nos ADRs, sem repetição aqui: DLQ como marcação "failed" na própria outbox (CAN-ALT-006), truncar payload acima do limite (CAN-ALT-008), renderizar o payload no envio em vez do snapshot (CAN-ALT-009) e ID auto-incremental para a outbox (CAN-ALT-010).

## Riscos e mitigações

| Risco | Impacto | Mitigação ou estado | Fonte |
| ----- | ------- | ------------------- | ----- |
| Worker parado sem detecção | Eventos acumulam; nenhum cliente é notificado, sem sinal visível | Sem supervisão nem health check definidos. `Em aberto para o FDD` | CAN-LAC-010; ADR-002 |
| Crescimento do backlog acima da capacidade do worker | Latência acima dos 10s exigidos | Single worker é decisão da v1; escala adiada; métricas de lag não definidas. `Em aberto para o FDD` | CAN-FORA-005; ADR-002 |
| Entrega duplicada por retry ou replay | Cliente processa o mesmo evento mais de uma vez | Consequência aceita do at-least-once; dedup pelo cliente via `X-Event-Id` | CAN-RISK-002; ADR-005 |
| Secret vazada | Forja de mensagens naquele endpoint enquanto valer | Secret por endpoint contém o raio; rotação com grace de 24h. Armazenamento e redaction `Em aberto para o FDD` | CAN-RISK-003 — `[09:22]` Diego; ADR-004 |
| Cliente indisponível por período longo | Eventos pendentes acumulam e podem terminar na DLQ | Backoff progressivo cobrindo janela de ~15h, teto de tentativas e DLQ | CAN-RISK-001 — `[09:16]` Diego; ADR-003 |
| Crescimento indefinido da outbox | Degradação de leitura e ocupação de disco | Arquivamento (~30 dias) citado e adiado; retenção não definida. `Em aberto para o FDD` | CAN-FORA-003 — `[09:08]` Diego |
| Perda de ordem entre eventos de um mesmo pedido | Cliente observa transições fora de sequência | Intenção sob single worker, sem garantia formal; retries afetam a ordem. `Em aberto para o FDD` | CAN-RNF-014; ADR-002 |
| Retry storm contra endpoint degradado | Amplificação de carga sobre cliente já em dificuldade | Mitigado parcialmente pela progressão crescente; jitter e rate limiting não definidos. `Em aberto para o FDD` | CAN-OPEN-003; ADR-003 |
| Payload acima do limite | Evento não enviado | Limite de 64 KB com erro, sem truncamento; tratamento no contexto de retry `Em aberto para o FDD` | CAN-RISK-006 — `[09:24]` |
| Replay indevido ou massivo | Reenvio em massa a clientes | Restrito a `requireRole('ADMIN')` e auditado; limite por período `Em aberto para o FDD` | CAN-DEC-010 — `[09:36]` Sofia |
| Classificação incorreta de falhas | Retentar o irretentável ou descartar o recuperável | Apenas o timeout está definido; matriz completa `Em aberto para o FDD` | CAN-LAC-007 |
| Armazenamento inadequado das secrets | Vazamento em base, backup ou log | Criptografia em repouso não decidida; `secret` ausente da redaction atual. `Em aberto para o FDD` | CAN-LAC-004, CAN-INT-007 |
| Concorrência futura sobre as mesmas linhas | Duplo envio; ordem afetada; linhas sem dono após restart | Claim, lock e recuperação não definidos. `Em aberto para o FDD` | CAN-LAC-006 |
| Perda comercial (Atlas migrar) | Perda de cliente relevante | Prazo de três sprints com revisão de segurança incluída | CAN-RISK-008 — `[09:45]` Marcos |

## Limitações conhecidas

Trade-offs aceitos ou pontos pendentes desta primeira versão, não falhas da proposta:

- **Um único worker**, sem redundância inicial: o processamento é ponto único de parada, trade-off aceito para simplificar a v1.
- **Sem ordering global**; a ordem por pedido é intenção sob single worker, não garantia formal.
- **Duplicidades são possíveis** e esperadas; **exactly-once não é oferecido**.
- **Sem rate limiting de saída** nesta fase: observar e decidir depois.
- **Sem dashboard visual** e **sem alertas por e-mail**: adiados ou fora de escopo.
- **Sem arquivamento automático** da outbox; **retenção indefinida** de outbox, deliveries e DLQ.
- **Classificação de respostas HTTP em aberto**: só o timeout está fechado.
- **Claim e locking em aberto**, com efeito sobre duplicidade e ordem.
- **Armazenamento da secret em aberto**, incluindo criptografia em repouso.
- **Supervisão do worker em produção em aberto**: não há ferramenta escolhida.

## Questões em aberto

| # | Questão | Referência |
| - | ------- | ---------- |
| 1 | O `customer_id` vai no body ou no path dos endpoints de configuração? | CAN-OPEN-001 — `[09:32]` Larissa |
| 2 | O que conta como sucesso de entrega? | CAN-LAC-007 |
| 3 | Quais erros são retentáveis (2xx/3xx/4xx/5xx, DNS, TLS, conexão recusada, redirects)? | CAN-LAC-007 |
| 4 | Como será feito o claim das linhas da outbox pelo worker? | CAN-LAC-006 |
| 5 | Qual será o tamanho do batch de polling? "Batch pequeno" foi citado sem número. | CAN-LAC-014 — `[09:08]` Diego |
| 6 | Haverá concorrência interna no worker, ou o processamento é sequencial? | CAN-LAC-014 |
| 7 | Como tratar a ordenação de um mesmo pedido quando um evento anterior entra em retry? | CAN-RNF-014 |
| 8 | O replay preserva o `event_id`, gera um novo, ou cria evento lógico correlacionado? | ADR-005 — CAN-RF-009 |
| 9 | Como a secret será gerada (algoritmo, entropia, tamanho, encoding)? | CAN-LAC-003 — `[09:46]` Sofia |
| 10 | Como a secret será armazenada em repouso? | CAN-LAC-004 |
| 11 | Qual será o formato da assinatura (encoding, eventual prefixo, canonicalização)? | CAN-INT-010 |
| 12 | Como será tratado replay attack, além de enviar `X-Timestamp`? | CAN-RISK-009 — `[09:44]` Diego |
| 13 | Intervalo de polling, timeout e demais valores serão constantes no código ou env vars? | CAN-INT-008 |
| 14 | Qual será a retenção de outbox, deliveries e DLQ? | CAN-LAC-009 |
| 15 | Como o worker será supervisionado em produção? | CAN-LAC-010 |
| 16 | Como tratar o reinício do worker durante o processamento de um evento? | CAN-LAC-006 |
| 17 | Qual cliente HTTP o worker usará? | CAN-LAC-013 |
| 18 | Rate limiting de saída será necessário no futuro? | CAN-OPEN-003 — `[09:39]` |
| 19 | "Cinco tentativas" inclui ou não o primeiro envio? | ADR-003 — `[09:15]`/`[09:17]` |
| 20 | Paginação e parâmetros do endpoint de deliveries além de "últimos 100"? | CAN-LAC-008 |

Nenhuma destas questões é respondida por este RFC. Todas pertencem ao FDD, exceto onde indicado que dependem da revisão de segurança.

## Estratégia de implementação

Organização em fases, sem datas absolutas. A estimativa registrada é de **três sprints, incluindo a revisão de segurança** (`[09:46]`–`[09:47]` Larissa), com **pelo menos dois dias úteis reservados para a revisão de segurança antes do deploy** (`[09:46]` Sofia).

**Fase 1 — Persistência.** Modelagem e migration das entidades de configuração, outbox, deliveries e dead-letter, com PK em UUID.

**Fase 2 — Integração transacional.** Publicação do evento dentro da transação de `changeStatus`; avaliação das configurações interessadas (filtro na inserção); snapshot do payload; geração do `event_id`.

**Fase 3 — Worker.** Entry-point do processo separado; loop de polling; entrega HTTP; registro dos resultados.

**Fase 4 — Resiliência.** Retry com a progressão decidida; encaminhamento à DLQ; replay administrativo.

**Fase 5 — APIs e segurança.** CRUD de configuração; endpoint de deliveries; rotação de secret; assinatura HMAC; autorização.

**Fase 6 — Testes e segurança.** Cobertura de testes; revisão de segurança da Sofia; preparação da operação.

As fases descrevem dependência técnica, não alocação de sprint: a modelagem precede a integração transacional, que precede o consumo, que precede a resiliência sobre esse consumo. O mapeamento fase-a-sprint pertence ao Tracker.

## Estratégia de testes

Categorias a cobrir, sem casos completos, que pertencem ao FDD:

- CRUD de configuração de webhook.
- Autorização: CRUD autenticado; replay restrito a ADMIN.
- Validação de URL HTTPS, com recusa de `http`.
- Filtro de status assinados, incluindo o caso em que nenhum webhook assina o status.
- Atomicidade da criação do evento na transação de mudança de status.
- Rollback conjunto quando a inserção do evento falha.
- Snapshot: o payload reflete o estado do momento da mudança.
- Payload: campos e ausência de `items`.
- Headers da entrega.
- Assinatura HMAC: payload modificado e secret incorreta.
- Rotação de secret.
- Grace period: secret anterior dentro da janela e depois de expirada.
- Worker: seleção e processamento de eventos elegíveis.
- Timeout de 10 segundos tratado como falha.
- Retry e progressão dos intervalos.
- Encaminhamento à DLQ ao esgotar o limite.
- Replay administrativo.
- Auditoria do replay.
- Semântica at-least-once.
- Estabilidade do `event_id` entre retentativas.
- Limite de 64 KB.
- Histórico de entregas.
- Regressão do domínio de pedidos: `tests/orders.test.ts` deve permanecer verde.

Os testes seguirão o padrão existente de Vitest e Supertest, com `tests/setup.ts` estendido às novas tabelas. Testar o worker poderá exigir estratégia adicional, por ser processo separado; a forma não está definida.

## Rollout e operação

Sequência em alto nível:

1. Aplicar a migration das novas tabelas **antes** da ativação da feature.
2. Implantar a API com o módulo de webhooks.
3. Implantar o worker como processo separado.
4. Garantir o **worker ativo antes de habilitar clientes**: sem consumidor da outbox, eventos apenas se acumulam.
5. Concluir a revisão de segurança antes do deploy, com os dois dias úteis reservados.
6. Ativar gradualmente por cadastro: um cliente só passa a receber quando seu endpoint é cadastrado — o próprio CRUD é o mecanismo de rollout incremental.
7. Acompanhar backlog da outbox, falhas e profundidade da DLQ.

Permanecem abertos: existência de feature flag; orquestração e supervisão do processo; health check e readiness do worker; procedimento de rollback; eventual backfill de eventos anteriores à ativação; e ferramenta de deploy (CAN-LAC-010, CAN-LAC-012).

## Critérios de aprovação

Este RFC estará apto a orientar o FDD quando:

1. As decisões dos seis ADRs estiverem consolidadas e mutuamente consistentes.
2. Os fluxos ponta a ponta estiverem documentados.
3. Os contratos estiverem descritos em alto nível, sem antecipar o detalhamento do FDD.
4. Os impactos no código existente estiverem mapeados sobre caminhos reais.
5. As alternativas consideradas estiverem registradas com o motivo do descarte ou do adiamento.
6. Os riscos estiverem registrados, com mitigação ou estado explícito.
7. As lacunas permanecerem explícitas, sem resolução silenciosa.
8. Nenhuma decisão inventada tiver sido introduzida.
9. Nenhum artefato futuro estiver descrito como existente.
10. Não houver contradição com a transcrição, com o código ou com os ADRs.
11. O documento for suficiente para orientar a elaboração do FDD.

## ADRs relacionados

| ADR | Decisão |
| --- | ------- |
| [ADR-001](./adrs/ADR-001-outbox-no-mysql.md) | Outbox no MySQL |
| [ADR-002](./adrs/ADR-002-worker-separado-com-polling.md) | Worker separado e polling |
| [ADR-003](./adrs/ADR-003-retry-backoff-e-dlq.md) | Retry e DLQ |
| [ADR-004](./adrs/ADR-004-hmac-sha256-e-gestao-de-secrets.md) | HMAC e secrets |
| [ADR-005](./adrs/ADR-005-entrega-at-least-once-com-event-id.md) | At-least-once e Event ID |
| [ADR-006](./adrs/ADR-006-reuso-dos-padroes-do-projeto.md) | Padrões existentes |

Decisões sem ADR dedicado, por definição do escopo de seis ADRs: **snapshot do payload na inserção** (CAN-DEC-023, CAN-RF-018) e **contrato do payload** (CAN-DEC-024, CAN-DEC-025), detalhados no FDD; o trade-off snapshot versus render-no-envio (CAN-ALT-009) é registrado aqui e no FDD.

## Evidências e rastreabilidade

| Tipo | Localização | Informação sustentada |
| ---- | ----------- | --------------------- |
| TRANSCRICAO | `[09:00]`, `[09:02]` Marcos; `[09:03]` Sofia | Polling atual lento e caro; pedido de Atlas, MaxDistribuição e Nova Cargo; "tempo real" = <10s; integração outbound (CAN-PROB-001 a 004, CAN-MET-001, CAN-RISK-008) |
| TRANSCRICAO | `[09:04]` Bruno; `[09:06]`, `[09:07]` Diego; `[09:07]`, `[09:08]` Larissa | Transação já pesada; "Síncrono está fora de questão"; outbox na mesma transação SQL; "Outbox no MySQL existente resolve"; índices, estados e batch citados sem fechamento; arquivamento ~30d fora de escopo (CAN-DEC-001, CAN-DEC-002, CAN-RNF-008, CAN-ALT-001, CAN-ALT-002, CAN-LAC-001, CAN-LAC-014, CAN-FORA-003) |
| TRANSCRICAO | `[09:09]` Bruno e Diego; `[09:10]` Marcos e Larissa | Trigger de banco descartada (MySQL sem listener nativo); polling de 2s; espera adicional aceita (CAN-ALT-003, CAN-DEC-003, CAN-MET-002, CAN-MET-003) |
| TRANSCRICAO | `[09:11]`–`[09:14]` Diego, Larissa, Bruno, Marcos; `[09:30]` Bruno | Worker em processo separado; `src/worker.ts` e script `npm run worker`; `PrismaClient` próprio e mesma `DATABASE_URL`; single worker; ordem por pedido como intenção, sem ordering global (CAN-RNF-011, CAN-DEC-005, CAN-DEC-017, CAN-DEC-018, CAN-DEC-028, CAN-RNF-013, CAN-RNF-014, CAN-FORA-005) |
| TRANSCRICAO | `[09:15]`–`[09:18]` Diego, Bruno, Marcos, Larissa | 3 tentativas e retry indefinido descartados; "Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h"; DLQ em tabela separada com payload, motivo e timestamp; replay por endpoint admin (CAN-DEC-006 a 009, CAN-MET-004, CAN-MET-005, CAN-ALT-004 a 006, CAN-RF-009) |
| TRANSCRICAO | `[09:19]`–`[09:24]` Sofia; `[09:22]`, `[09:24]` Diego | Autenticidade e integridade; HMAC-SHA256; secret por endpoint ("se vaza uma, vaza tudo"); rotação com grace de 24h; vazamento real em log; HTTPS obrigatório; 64 KB com erro, sem truncar (CAN-DEC-011 a 013, CAN-RNF-001 a 005, CAN-MET-006, CAN-MET-007, CAN-RISK-003, CAN-ALT-008) |
| TRANSCRICAO | `[09:24]`–`[09:26]` Diego, Bruno, Sofia, Marcos, Larissa | At-least-once; `X-Event-Id` UUID gerado na entrada da outbox; dedup pelo cliente; exactly-once descartado (CAN-DEC-014, CAN-DEC-015, CAN-RF-014, CAN-RF-015, CAN-ALT-011) |
| TRANSCRICAO | `[09:27]`–`[09:30]` Bruno, Diego, Larissa | Módulo `src/modules/webhooks`; prefixo `WEBHOOK_`; Pino e middleware de erro reutilizados; "reuso máximo do que já existe" (CAN-DEC-016, CAN-DEC-027, CAN-RNF-015 a CAN-RNF-020) |
| TRANSCRICAO | `[09:31]`–`[09:37]` Marcos, Bruno, Larissa, Sofia | CRUD por verbo; secret gerada e devolvida na criação; `customer_id` no body ou path, não do JWT; filtro na inserção; deliveries com últimos 100; replay com role ADMIN e auditoria; CRUD com qualquer role por ora; e-mail adiado (CAN-RF-002 a 008, CAN-DEC-010, CAN-DEC-019, CAN-DEC-020, CAN-RNF-006, CAN-MET-009, CAN-OPEN-001, CAN-FORA-001, CAN-FORA-004) |
| TRANSCRICAO | `[09:38]`–`[09:44]` Diego, Larissa, Marcos, Bruno, Sofia | Rate limiting em aberto; dashboard fora de escopo; inserção na mesma transação com rollback; `publishWebhookEvent(tx, order, fromStatus, toStatus)`; timeout de 10s; payload e headers (CAN-OPEN-003, CAN-RISK-007, CAN-FORA-002, CAN-RISK-005, CAN-DEC-021, CAN-DEC-024 a 026, CAN-MET-008, CAN-RF-016, CAN-RF-017) |
| TRANSCRICAO | `[09:45]`–`[09:47]`, `[09:51]`–`[09:52]` Marcos, Larissa, Sofia, Diego, Bruno | Prazo Atlas fim de novembro; três sprints; dois dias úteis de revisão de segurança; "Tudo é uuid"; snapshot na inserção (CAN-MET-011 a 013, CAN-DEC-022, CAN-DEC-023, CAN-RF-018, CAN-ALT-009, CAN-ALT-010) |
| CODIGO | `src/modules/orders/order.service.ts` — `OrderService.changeStatus`, `TxClient = Prisma.TransactionClient`; `src/modules/orders/order.status.ts` — `canTransition`, `shouldDebitStock`, `shouldReplenishStock` | Transação Prisma com validação, estoque, `tx.order.update` e `tx.orderStatusHistory.create`; nenhuma publicação de evento; máquina de transições a não duplicar (CAN-COD-001, CAN-COD-003, CAN-INT-001) |
| CODIGO | `prisma/schema.prisma` — `datasource db`, models `Order`, `OrderStatusHistory`, enum `OrderStatus`; `prisma/migrations/` | Provider `mysql`; PKs `@default(uuid()) @db.Char(36)`; ausência de models de webhook, outbox, deliveries e DLQ; migrations já inicializadas (CAN-COD-002, CAN-COD-020, CAN-COD-023) |
| CODIGO | `src/app.ts` — `buildApp`, `buildControllers`; `src/routes/index.ts` — `buildApiRouter`, `Controllers` | Composição manual; prefixo `/api/v1`; agregação por domínio; nenhum componente de webhook registrado (CAN-COD-017, CAN-COD-018, CAN-INT-003) |
| CODIGO | `src/middlewares/auth.middleware.ts` — `authenticate`, `requireRole`, `AuthUser`; `src/modules/users/user.routes.ts` — `buildUserRouter` | JWT bearer; papéis `ADMIN`/`OPERATOR`; precedente real de rota protegida por papel (CAN-COD-007, CAN-COD-022, CAN-INT-004) |
| CODIGO | `src/middlewares/validate.middleware.ts` — `validate`; `src/shared/errors/app-error.ts` — `AppError`; `src/middlewares/error.middleware.ts` — `errorMiddleware` | Zod convertendo `ZodError` em `ValidationError`; erros com `statusCode`/`errorCode`/`details`; envelope `{ error: { code, message, details? } }` (CAN-COD-004, CAN-COD-006, CAN-COD-008, CAN-INT-005) |
| CODIGO | `src/shared/logger/index.ts` — `redactPaths`, `createLogger` | Pino com `redact` de `authorization`, `cookie`, `*.password`, `*.passwordHash`, `*.token`, `*.accessToken`; `secret` e assinatura **ausentes** (CAN-COD-013, CAN-INT-007) |
| CODIGO | `src/config/env.ts` — `envSchema`, `env`; `src/config/database.ts` — `createPrismaClient`, `prisma`; `src/server.ts` — `bootstrap`; `package.json` — `scripts`, `engines` | Configuração Zod sem variáveis de webhook; fábrica e singleton de Prisma; lifecycle `SIGINT`/`SIGTERM` com `$disconnect`; sem script `worker`; `uuid` e `node >= 20` como disponibilidade, não decisão (CAN-COD-014 a 016, CAN-COD-019, CAN-INT-008) |
| CODIGO | `tests/setup.ts`; `tests/orders.test.ts` | Limpeza por `beforeEach` respeitando relações; cobertura de transições, estoque e regras de pedido (CAN-COD-021, CAN-INT-009) |
| ADR | [ADR-001](./adrs/ADR-001-outbox-no-mysql.md) a [ADR-006](./adrs/ADR-006-reuso-dos-padroes-do-projeto.md) | Decisões arquiteturais aceitas consolidadas por este RFC |
| MATRIZ | `MATRIZ-FATOS.md` §2 e §3 | IDs canônicos citados neste documento e seus destinos |
| ANALISE | `ANALISE-EVIDENCIAS.md` §15, §18, §19 | Índice auxiliar de integrações futuras, ambiguidades e lacunas |
