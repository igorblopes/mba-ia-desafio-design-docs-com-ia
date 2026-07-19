# Tracker de Rastreabilidade — Webhooks de Notificação de Mudança de Status de Pedidos

## 1. Metadados

| Campo | Valor |
| ----- | ----- |
| Documento | Tracker de Rastreabilidade — Webhooks de Notificação de Mudança de Status de Pedidos |
| Status | Em revisão |
| Sistema | Order Management System |
| Escopo | PRD, RFC, FDD e ADR-001 a ADR-006 |
| Fonte canônica | [`MATRIZ-FATOS.md`](../MATRIZ-FATOS.md) |
| Fonte primária de negócio | [`TRANSCRICAO.md`](../TRANSCRICAO.md) |
| Fonte primária técnica | Código existente do repositório |
| Responsável | Responsável pela entrega |
| Data | Não informada. `TRANSCRICAO.md` registra apenas "quinta-feira, 09:00", sem data de calendário |

> Autor, aprovador, versão nominal, equipe e data verificável **não constam** das fontes e **não são inventados**.

---

## 2. Objetivo

Este Tracker responde, para cada fato publicado da feature de Webhooks de Notificação de Mudança de Status de Pedidos:

> **De onde veio cada problema, requisito, decisão, alternativa, métrica, risco, regra, limitação ou questão aberta, e em quais documentos finais esse item foi registrado?**

Cada linha permite ao revisor: (1) identificar o fato; (2) localizar sua fonte primária; (3) encontrar seu ID canônico em `MATRIZ-FATOS.md`; (4) saber em quais documentos finais (PRD, RFC, FDD, ADR-001 a ADR-006) o fato aparece; (5) verificar se a cobertura está completa; (6) compreender qualquer ausência ou inconsistência.

O Tracker **comprova** que os documentos foram derivados da reunião (`TRANSCRICAO.md`) e do código, **valida** a cobertura documental, **expõe** ausências e contradições, e **preserva** as questões em aberto sem resolvê-las. Ele **não** substitui a `MATRIZ-FATOS.md`, **não** copia os documentos, **não** cria requisitos ou decisões, e **não** apresenta artefatos futuros como existentes.

---

## 3. Escopo da rastreabilidade

Estão no Tracker os fatos que: (a) aparecem em um documento final; ou (b) deveriam aparecer em um documento final; ou (c) são necessários para provar a rastreabilidade de uma integração com o código existente; ou (d) representam uma ausência relevante identificada na auditoria.

**Fora do Tracker:** as 14 lacunas `CAN-LAC-*` da matriz (marcadas `NÃO PUBLICAR`) não entram como linhas próprias, exceto quando referenciadas para registrar uma pendência documental ou uma questão em aberto já publicada nos documentos (ver §11). Detalhes internos de implementação sem impacto em produto/contrato/segurança/operação também não geram linha.

Cobre-se a feature ponta a ponta: produção transacional do evento (ADR-001), worker/polling (ADR-002), retry/backoff/DLQ/replay (ADR-003), HMAC e secrets (ADR-004), at-least-once e Event ID (ADR-005), reuso dos padrões (ADR-006), além dos contratos de payload/headers e das questões em aberto herdadas.

---

## 4. Convenções

### 4.1 Categorias

Usadas apenas as categorias abaixo; quando uma linha caberia em mais de uma, escolheu-se a natureza principal (ex.: "HMAC-SHA256" → `DECISÃO`; "URL deve ser HTTPS" → `REQUISITO_NÃO_FUNCIONAL`; "HMAC não fornece confidencialidade" → observação da decisão, não requisito).

`PROBLEMA` · `PÚBLICO` · `CENÁRIO_DE_USO` · `REQUISITO_FUNCIONAL` · `REQUISITO_NÃO_FUNCIONAL` · `REGRA_DE_NEGÓCIO` · `DECISÃO` · `ALTERNATIVA_DESCARTADA` · `ITEM_FORA_DE_ESCOPO` · `ITEM_ADIADO` · `MÉTRICA` · `RISCO` · `QUESTÃO_ABERTA` · `EVIDÊNCIA_DE_CÓDIGO` · `INTEGRAÇÃO_FUTURA` · `RESTRIÇÃO` · `PREMISSA`.

### 4.2 Fontes

- **`TRANSCRICAO`** — problema, público, requisito, decisão, alternativa, risco, métrica, restrição, item adiado ou questão aberta discutidos na reunião. Localização obrigatória: `[hh:mm]` + participante (ex.: `[09:20]` Sofia; múltiplas: `[09:20]` Sofia; `[09:44]` Diego).
- **`CODIGO`** — comportamento já existente no repositório. Localização obrigatória: caminho real + símbolo (ex.: `src/modules/orders/order.service.ts` — `OrderService.changeStatus`). Nunca cita arquivo, símbolo, Outbox ou worker inexistentes.
- **`ANALISE`** — lacunas derivadas, relações entre fatos, inconsistências documentais e integrações futuras não explicitadas na reunião nem existentes no código. Não é usada para criar requisitos.

### 4.3 Situações de cobertura

- **`COBERTO`** — presente em todos os documentos em que deveria aparecer, com conteúdo correto e suficiente.
- **`PARCIALMENTE_COBERTO`** — incompleto, presente só em parte dos destinos esperados, resumido de modo insuficiente, ou sem rastreabilidade canônica completa (ex.: `AUSENTE NA MATRIZ`).
- **`NÃO_COBERTO`** — deveria aparecer em um documento final e não foi localizado.
- **`NÃO_APLICÁVEL`** — o documento não é destino adequado para o fato (registrado com `—` na coluna correspondente).
- **`COBERTURA_INCONSISTENTE`** — aparece, mas contradiz a fonte, recebe valores diferentes entre documentos, muda de natureza (decisão × questão aberta) ou é afirmado de forma mais forte que a fonte.

### 4.4 Regras de localização e IDs

- IDs do Tracker: `TRK-NNN`, sequência contínua e única; um fato atômico por linha.
- **ID canônico**: exatamente o ID de `MATRIZ-FATOS.md`. Quando um item relevante não possui ID canônico, usa-se **`AUSENTE NA MATRIZ`** (nenhum ID novo é inventado), e a situação vira `PARCIALMENTE_COBERTO`.
- Colunas PRD/RFC/FDD/ADR: identificador de requisito, nome de seção ou nome do ADR onde o fato aparece; `—` quando o documento não é destino aplicável. Números de linha não são usados (mudam com facilidade).
- Quando um fato aparece em mais de um ADR, cita-se o ADR principal na coluna e os relacionados em Observações.
- Referências de arquivo dos documentos: [PRD](./PRD.md), [RFC](./RFC.md), [FDD](./FDD.md), [ADR-001](./adrs/ADR-001-outbox-no-mysql.md), [ADR-002](./adrs/ADR-002-worker-separado-com-polling.md), [ADR-003](./adrs/ADR-003-retry-backoff-e-dlq.md), [ADR-004](./adrs/ADR-004-hmac-sha256-e-gestao-de-secrets.md), [ADR-005](./adrs/ADR-005-entrega-at-least-once-com-event-id.md), [ADR-006](./adrs/ADR-006-reuso-dos-padroes-do-projeto.md).

> **Nota sobre numeração local × canônica:** o PRD numera seus requisitos (RF-001…RF-018, RNF-001…RNF-018) de forma **independente** dos IDs canônicos. O mapeamento PRD↔canônico está na §10. Nas colunas do Tracker principal, as referências ao PRD usam a numeração **local do PRD**.

---

## 5. Tracker principal

> Colunas: **ID** · **Categoria** · **Item rastreável** (afirmação atômica) · **Fonte primária** · **Localização** · **ID canônico** · **PRD** · **RFC** · **FDD** · **ADR** · **Situação** · **Observações**. `—` = documento não é destino aplicável (`NÃO_APLICÁVEL`).

### 5.1 Problemas e públicos

| ID | Categoria | Item rastreável | Fonte primária | Localização | ID canônico | PRD | RFC | FDD | ADR | Situação | Observações |
|---|---|---|---|---|---|---|---|---|---|---|---|
| TRK-001 | PROBLEMA | Clientes B2B consultam `GET /orders` em polling para detectar mudança de status. | TRANSCRICAO | `[09:00]` Marcos | CAN-PROB-001 | Contexto/Problema | Contexto e motivação | — | ADR-001 | COBERTO | Comportamento atual que a feature elimina; ADR-001 cita como contexto. |
| TRK-002 | PROBLEMA | O polling atual torna a integração dos clientes lenta e cara. | TRANSCRICAO | `[09:00]` Marcos | CAN-PROB-002 | Problema | Contexto e motivação | — | ADR-001 | COBERTO | Consequência de [[CAN-PROB-001]]; motivação central. |
| TRK-003 | PROBLEMA | Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) pediram formalmente ser notificados na mudança de status. | TRANSCRICAO | `[09:00]` Marcos | CAN-PROB-003 | Contexto/Fase 2 | Contexto e motivação | — | ADR-001 | COBERTO | "Tempo real" (<10s) isolado em TRK-082. |
| TRK-004 | PROBLEMA | A Atlas sinalizou possível migração para concorrente se a entrega não ocorrer no prazo. | TRANSCRICAO | `[09:00]` Marcos; `[09:45]` Marcos | CAN-PROB-004 | Problema de negócio/Riscos | Riscos | — | — | COBERTO | Risco correlato em TRK-101 (CAN-RISK-008); prazo em TRK-093. |
| TRK-005 | PÚBLICO | Clientes B2B consumidores das notificações de mudança de status. | TRANSCRICAO | `[09:00]` Marcos | CAN-PUB-001 | Públicos e stakeholders | — | — | — | COBERTO | Destino natural: PRD. |
| TRK-006 | PÚBLICO | Sistema consumidor do cliente que recebe o request HTTP e valida a assinatura HMAC. | TRANSCRICAO | `[09:19]` Sofia; `[09:20]` Sofia | CAN-PUB-002 | Públicos/Experiência | Autenticidade e integridade | — | ADR-004 | COBERTO | Verificação é do lado do cliente. |
| TRK-007 | PÚBLICO | Operador interno com role ADMIN que reprocessa manualmente itens da DLQ. | TRANSCRICAO | `[09:35]` Larissa; `[09:36]` Sofia | CAN-PUB-003 | Públicos/Administrador | Autenticação e autorização | Administração da DLQ | ADR-003 | COBERTO | Ver TRK-019, TRK-050. |
| TRK-008 | PÚBLICO | Usuário autenticado que representa o cliente e gerencia a configuração via API. | TRANSCRICAO | `[09:32]` Marcos; `[09:32]` Larissa | CAN-PUB-004 | Públicos/Operadores | Autenticação e autorização | Autorização | ADR-006 | COBERTO | Cadastro pela API própria, não por painel; ver TRK-057. |
| TRK-009 | PÚBLICO | Engenharia e operações operam e monitoram o worker de entrega. | ANALISE | ANALISE — derivado de CAN-RNF-011; `[09:11]` Diego | AUSENTE NA MATRIZ | Públicos e stakeholders | — | — | — | PARCIALMENTE_COBERTO | Stakeholder listado no PRD sem ID de PÚBLICO na matriz; pendência documental (ver §14 INC-01). |
| TRK-010 | PÚBLICO | Segurança revisa assinatura, geração/armazenamento de secrets e logs antes do lançamento. | TRANSCRICAO | `[09:46]` Sofia | AUSENTE NA MATRIZ | Públicos e stakeholders | — | Implantação e operação | ADR-004 | PARCIALMENTE_COBERTO | Fato sustentado; sem ID de PÚBLICO na matriz (canônico métrico em TRK-092/CAN-MET-012). Pendência: §14 INC-01. |

### 5.2 Requisitos funcionais

| ID | Categoria | Item rastreável | Fonte primária | Localização | ID canônico | PRD | RFC | FDD | ADR | Situação | Observações |
|---|---|---|---|---|---|---|---|---|---|---|---|
| TRK-011 | REQUISITO_FUNCIONAL | O sistema emite um webhook outbound quando o status de um pedido muda (evento `order.status_changed`). | TRANSCRICAO | `[09:00]` Marcos; `[09:43]` Diego | CAN-RF-001 | RF-006/RF-007 | RF-06 | Arquitetura futura | ADR-001 | COBERTO | Contrato em TRK-061; entrega pelo worker em TRK-044. |
| TRK-012 | REQUISITO_FUNCIONAL | O cliente cadastra um webhook via `POST`, informando `url`, lista de status e `customer_id`. | TRANSCRICAO | `[09:31]` Marcos; `[09:33]` Bruno | CAN-RF-002 | RF-001 | RF-01 | Contratos HTTP | — | COBERTO | Origem do `customer_id` em TRK-057; body×path em TRK-103. |
| TRK-013 | REQUISITO_FUNCIONAL | A `secret` é gerada pela plataforma e devolvida na resposta de criação. | TRANSCRICAO | `[09:31]` Marcos | CAN-RF-003 | RF-010 | RF-07 | Criar webhook | ADR-004 | COBERTO | Algoritmo/entropia de geração é lacuna (CAN-LAC-003); ver §11. |
| TRK-014 | REQUISITO_FUNCIONAL | O cliente edita um webhook via `PATCH`. | TRANSCRICAO | `[09:33]` Bruno | CAN-RF-004 | RF-003 | RF-03 | Atualizar webhook | — | COBERTO | — |
| TRK-015 | REQUISITO_FUNCIONAL | O cliente remove um webhook via `DELETE`. | TRANSCRICAO | `[09:33]` Bruno | CAN-RF-005 | RF-004 | RF-04 | Remover webhook | — | COBERTO | Comportamento sobre eventos pendentes na remoção é questão aberta (§11). |
| TRK-016 | REQUISITO_FUNCIONAL | O cliente lista os webhooks de um customer via `GET`. | TRANSCRICAO | `[09:33]` Bruno | CAN-RF-006 | RF-002 | RF-02 | Listar webhooks | — | COBERTO | — |
| TRK-017 | REQUISITO_FUNCIONAL | Cada endpoint assina um subconjunto dos status do pedido (filtro de eventos). | TRANSCRICAO | `[09:33]` Marcos; `[09:34]` Bruno | CAN-RF-007 | RF-005 | RF-05 | Filtro de status | ADR-001 | COBERTO | Momento de aplicação do filtro em TRK-058 (CAN-DEC-020). |
| TRK-018 | REQUISITO_FUNCIONAL | Existe `GET /webhooks/:id/deliveries` com histórico (sucesso/falha, payload, response, tempo de resposta). | TRANSCRICAO | `[09:34]` Marcos | CAN-RF-008 | RF-012/RF-013 | RF-09 | Histórico de entregas | — | COBERTO | "Últimas 100" em TRK-090; captura da resposta é lacuna (CAN-LAC-005). |
| TRK-019 | REQUISITO_FUNCIONAL | Existe `POST /admin/webhooks/dead-letter/:id/replay` que recoloca o item da DLQ na outbox como pendente. | TRANSCRICAO | `[09:18]` Diego; `[09:35]` Diego | CAN-RF-009 | RF-016 | RF-12 | Executar replay | ADR-003 | COBERTO | Exige ADMIN (TRK-050) e auditoria (TRK-030); identidade do evento no replay em aberto (§11). |
| TRK-020 | REQUISITO_FUNCIONAL | O cliente solicita a rotação da `secret` via API. | TRANSCRICAO | `[09:21]` Sofia | CAN-RF-010 | RF-011 | RF-08 | Rotacionar secret | ADR-004 | COBERTO | Caminho da rota em aberto (§11). |
| TRK-021 | REQUISITO_FUNCIONAL | Após a rotação, a `secret` anterior permanece válida por 24h em paralelo e depois é invalidada. | TRANSCRICAO | `[09:21]` Sofia | CAN-RF-011 | RNF-008 | RNF-09 | Rotacionar secret | ADR-004 | COBERTO | Valor 24h em TRK-087 (CAN-MET-006); espelha CAN-DEC-013/CAN-RNF-003. |
| TRK-022 | REQUISITO_FUNCIONAL | Entregas que falham são retentadas automaticamente com backoff. | TRANSCRICAO | `[09:15]` Diego | CAN-RF-012 | RF-014 | RF-10 | Retry e classificação | ADR-003 | COBERTO | Teto/progressão em TRK-085/TRK-086; espelha CAN-RNF-010. |
| TRK-023 | REQUISITO_FUNCIONAL | Esgotadas as tentativas, o evento é movido para a DLQ. | TRANSCRICAO | `[09:15]` Diego; `[09:17]` Larissa | CAN-RF-013 | RF-015 | RF-11 | Encaminhar à DLQ | ADR-003 | COBERTO | Espelha CAN-DEC-007; tabela dedicada em TRK-048. |
| TRK-024 | REQUISITO_FUNCIONAL | Cada envio inclui o header `X-Event-Id`, identificador único do evento. | TRANSCRICAO | `[09:25]` Diego; `[09:44]` Diego | CAN-RF-014 | RF-009 | Contrato outbound | ADR-005 | COBERTO | Natureza UUID em TRK-025; também em README (portal). |
| TRK-025 | REQUISITO_FUNCIONAL | O `X-Event-Id` é um UUID gerado no momento em que o evento entra na outbox. | TRANSCRICAO | `[09:25]` Diego; `[09:51]` Larissa | CAN-RF-015 | RNF-010 | Identidade de evento | ADR-005 | COBERTO | PRD RNF-010 diz "no fluxo de entrega"; fonte e RFC/FDD/ADR-005 dizem "na outbox/inserção" — imprecisão de redação (§14 INC-03). |
| TRK-026 | REQUISITO_FUNCIONAL | Cada envio inclui `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`. | TRANSCRICAO | `[09:44]` Diego; `[09:44]` Sofia | CAN-RF-017 | RF-009/CA-010 | Contrato outbound | — | COBERTO | `X-Timestamp` p/ detecção de replay (TRK-102); `X-Webhook-Id` sugerido por Sofia; também README. |
| TRK-027 | REQUISITO_FUNCIONAL | O evento gravado na outbox contém o payload renderizado (snapshot) no momento da inserção. | TRANSCRICAO | `[09:52]` Larissa; `[09:52]` Diego | CAN-RF-018 | RF-008/RN-007 | Contrato outbound | — | COBERTO | Decisão CAN-DEC-023; alternativa descartada em TRK-073. **Sem ADR dedicado** (nota de escopo da matriz). |

### 5.3 Requisitos não funcionais

| ID | Categoria | Item rastreável | Fonte primária | Localização | ID canônico | PRD | RFC | FDD | ADR | Situação | Observações |
|---|---|---|---|---|---|---|---|---|---|---|---|
| TRK-028 | REQUISITO_NÃO_FUNCIONAL | A `url` do webhook deve ser `https`; `http` é recusada em validação (schema Zod). | TRANSCRICAO | `[09:23]` Sofia | CAN-RNF-004 | RNF-005 | RNF-06 | Segurança/Validação | ADR-004 | COBERTO | Sofia classificou como validação, não decisão arquitetural. |
| TRK-029 | REQUISITO_NÃO_FUNCIONAL | Payload acima do limite é recusado com erro (não é truncado). | TRANSCRICAO | `[09:23]` Sofia; `[09:24]` Larissa | CAN-RNF-005 | RNF-004 | Limite de payload | — | COBERTO | Valor 64 KB em TRK-088; alternativa (truncar) em TRK-072. |
| TRK-030 | REQUISITO_NÃO_FUNCIONAL | O endpoint admin de replay registra quem executou, para auditoria. | TRANSCRICAO | `[09:36]` Sofia | CAN-RNF-006 | RF-018/RN-014 | RF-13 | Executar replay | ADR-003 | COBERTO | Formato/destino do registro não definidos (lacuna). |
| TRK-031 | REQUISITO_NÃO_FUNCIONAL | O evento só existe se a transação de mudança de status commitar; em rollback, não é gravado (atomicidade). | TRANSCRICAO | `[09:06]` Diego; `[09:40]` Bruno; `[09:41]` Diego | CAN-RNF-008 | RF-006 (nota) | Problema técnico | Integração transacional | ADR-001 | COBERTO | Base do outbox (TRK-042/043); risco em TRK-098. |
| TRK-032 | REQUISITO_NÃO_FUNCIONAL | O worker roda como processo separado da instância da API. | TRANSCRICAO | `[09:11]` Diego | CAN-RNF-011 | RNF-013 | RNF-13 | Worker | ADR-002 | COBERTO | Entry-point em TRK-064; PrismaClient próprio em TRK-056. |
| TRK-033 | REQUISITO_NÃO_FUNCIONAL | Chamada HTTP que excede o timeout é tratada como falha e marcada para retry. | TRANSCRICAO | `[09:42]` Diego | CAN-RNF-012 | RNF-003 | Timeout | ADR-003 | COBERTO | Valor 10s em TRK-089; demais critérios de falha (4xx/5xx/DNS) são lacuna (§11). |
| TRK-034 | REQUISITO_NÃO_FUNCIONAL | Não há garantia de ordenação global de eventos. | TRANSCRICAO | `[09:12]` Diego; `[09:13]` Larissa; `[09:14]` Marcos | CAN-RNF-013 | RNF-015 | Ordenação | ADR-002 | COBERTO | Clientes nunca pediram ordering global (`[09:14]` Marcos). |
| TRK-035 | REQUISITO_NÃO_FUNCIONAL | Pretende-se preservar a ordem por `order_id` sob single-worker; é limitação/intenção, não garantia formal. | TRANSCRICAO | `[09:12]` Diego; `[09:13]` Larissa | CAN-RNF-014 | RNF-014/015 | Ordenação | ADR-002 | COBERTO | Efetividade depende de claim/lock/retries/restart (lacuna CAN-LAC-006). |
| TRK-036 | REQUISITO_NÃO_FUNCIONAL | Reutilizar a classe `AppError` e as convenções de erro do projeto no módulo de webhooks. | TRANSCRICAO | `[09:28]` Bruno; `[09:30]` Larissa | CAN-RNF-015 | RNF-016 | Tratamento de erros | ADR-006 | COBERTO | Base em TRK-114/§15; prefixo em TRK-041. |
| TRK-037 | REQUISITO_NÃO_FUNCIONAL | Reutilizar o logger Pino existente na API e no worker. | TRANSCRICAO | `[09:29]` Bruno | CAN-RNF-016 | RNF-016 | Logging e observabilidade | ADR-006 | COBERTO | Atenção à redação de `secret` (TRK-110/TRK-122). |
| TRK-038 | REQUISITO_NÃO_FUNCIONAL | Reutilizar o middleware de erro centralizado (trata `AppError`, Zod e Prisma). | TRANSCRICAO | `[09:29]` Bruno | CAN-RNF-017 | RNF-016 | Tratamento de erros | ADR-006 | COBERTO | "Vai pegar nossos erros sem precisar mudar nada" (`[09:29]` Bruno). |
| TRK-039 | REQUISITO_NÃO_FUNCIONAL | Reutilizar a estrutura modular (controller/service/repository/routes/schemas). | TRANSCRICAO | `[09:27]` Bruno; `[09:30]` Larissa | CAN-RNF-018 | RNF-016 | Organização dos arquivos | ADR-006 | COBERTO | Layout do módulo em TRK-063. |
| TRK-040 | REQUISITO_NÃO_FUNCIONAL | Reutilizar o padrão de schemas de validação Zod. | TRANSCRICAO | `[09:30]` Larissa | CAN-RNF-019 | RNF-016 | Validação | ADR-006 | COBERTO | Base em §15; validação `https` em TRK-028. |
| TRK-041 | REQUISITO_NÃO_FUNCIONAL | Usar o prefixo `WEBHOOK_` nos códigos de erro do módulo. | TRANSCRICAO | `[09:28]` Bruno; `[09:29]` Larissa | CAN-RNF-020 | RNF-016 | Tratamento de erros | ADR-006 | COBERTO | Ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`; classes propostas em TRK-120. |

### 5.4 Decisões arquiteturais

| ID | Categoria | Item rastreável | Fonte primária | Localização | ID canônico | PRD | RFC | FDD | ADR | Situação | Observações |
|---|---|---|---|---|---|---|---|---|---|---|---|
| TRK-042 | DECISÃO | Usar o padrão outbox na base MySQL existente para entrega de webhooks. | TRANSCRICAO | `[09:06]` Diego; `[09:08]` Larissa | CAN-DEC-001 | Restrições | Arquitetura proposta | Resumo/Arquitetura | ADR-001 | COBERTO | Evita infra nova; alternativas em TRK-065/066. |
| TRK-043 | DECISÃO | Gravar o evento de webhook na mesma transação SQL da mudança de status. | TRANSCRICAO | `[09:06]` Diego | CAN-DEC-002 | — | Publicação transacional | Integração transacional | ADR-001 | COBERTO | Atomicidade em TRK-031; integração em TRK-059. |
| TRK-044 | DECISÃO | O worker lê a outbox por polling a cada 2 segundos. | TRANSCRICAO | `[09:09]` Diego; `[09:10]` Larissa | CAN-DEC-003 | — | Worker | Worker | ADR-002 | COBERTO | Valor em TRK-083; alternativa (trigger) em TRK-067. |
| TRK-045 | DECISÃO | O worker lê os eventos pendentes mais antigos em batch pequeno e marca como processados após o envio. | TRANSCRICAO | `[09:08]` Diego; `[09:09]` Diego | CAN-DEC-004 | — | Worker | Outbox (modelo de dados) | ADR-002 | COBERTO | Tamanho do batch e claim/lock são lacunas (CAN-LAC-006/014). |
| TRK-046 | DECISÃO | Adotar single-worker na v1 (sem múltiplos workers em paralelo). | TRANSCRICAO | `[09:12]` Diego; `[09:13]` Larissa | CAN-DEC-005 | RNF-014 | Ordenação | Ordenação | ADR-002 | COBERTO | Efeito na ordem em TRK-035; escala em TRK-081. |
| TRK-047 | DECISÃO | Adotar retry com backoff exponencial e teto de 5 tentativas. | TRANSCRICAO | `[09:15]` Diego; `[09:17]` Larissa | CAN-DEC-006 | RNF-011/RNF-012 | RNF-10/RNF-11 | Retry e classificação | ADR-003 | COBERTO | Espelha CAN-RNF-010; progressão em TRK-086; alternativas em TRK-068/069. |
| TRK-048 | DECISÃO | Persistir a DLQ em tabela dedicada (nome citado: `webhook_dead_letter`). | TRANSCRICAO | `[09:18]` Diego | CAN-DEC-008 | — | Dead-Letter Queue | Dead-Letter Queue | ADR-003 | COBERTO | Alternativa (marcar "failed" na outbox) em TRK-070. |
| TRK-049 | DECISÃO | A DLQ armazena payload, motivo da falha e timestamp. | TRANSCRICAO | `[09:18]` Diego | CAN-DEC-009 | — | Dead-Letter Queue | Dead-Letter Queue | ADR-003 | COBERTO | Esquema completo da DLQ não detalhado (CAN-LAC-001). |
| TRK-050 | DECISÃO | O reprocessamento da DLQ é manual, via endpoint admin, exigindo role `ADMIN`. | TRANSCRICAO | `[09:18]` Diego; `[09:36]` Sofia | CAN-DEC-010 | RF-017 | Autenticação e autorização | Autorização | ADR-003 | COBERTO | Auditoria em TRK-030; role reaproveitada de TRK-111 (ADR-006 relacionado). |
| TRK-051 | DECISÃO | Cada entrega é assinada com HMAC-SHA256 sobre o corpo, enviada no header `X-Signature`. | TRANSCRICAO | `[09:20]` Sofia; `[09:44]` Diego | CAN-DEC-011 | RNF-006 | Segurança/RNF-07 | HMAC-SHA256 | ADR-004 | COBERTO | Espelha CAN-RF-016 (X-Signature), CAN-RNF-001 e CAN-MET-014 (algoritmo); também README. HMAC não dá confidencialidade nem impede replay sozinho. |
| TRK-052 | DECISÃO | Usar uma secret única por endpoint (não há secret global). | TRANSCRICAO | `[09:21]` Sofia | CAN-DEC-012 | RNF-007 | RNF-08 | Gestão de secrets | ADR-004 | COBERTO | Espelha CAN-RNF-002; motivação em TRK-096 ("se vaza uma, vaza tudo"). |
| TRK-053 | DECISÃO | Adotar garantia de entrega at-least-once. | TRANSCRICAO | `[09:24]` Diego; `[09:26]` Larissa | CAN-DEC-014 | RNF-009 | Semântica de entrega | Semântica de entrega | ADR-005 | COBERTO | Espelha CAN-RNF-009; exactly-once descartado em TRK-075. |
| TRK-054 | DECISÃO | Delegar a deduplicação ao cliente, via `X-Event-Id`. | TRANSCRICAO | `[09:25]` Diego; `[09:25]` Sofia | CAN-DEC-015 | RN-011/Experiência | Semântica de entrega | Semântica de entrega | ADR-005 | COBERTO | Padrão citado: Stripe/GitHub; também README. |
| TRK-055 | DECISÃO | Reuso máximo das convenções existentes do projeto no módulo de webhooks. | TRANSCRICAO | `[09:30]` Larissa | CAN-DEC-016 | RNF-016 | RNF-15 | Premissas e restrições | ADR-006 | COBERTO | Itens específicos em TRK-036 a TRK-041. |
| TRK-056 | DECISÃO | O worker abre um `PrismaClient` próprio, na mesma `DATABASE_URL` da API. | TRANSCRICAO | `[09:29]` Diego; `[09:30]` Bruno | CAN-DEC-017 | — | Worker | Implantação e operação | ADR-002 | COBERTO | Consolida CAN-DEC-017 e CAN-DEC-018 (mesma base); base em TRK-112. |
| TRK-057 | DECISÃO | O `customer_id` é passado no body ou no path, não é derivado do JWT. | TRANSCRICAO | `[09:32]` Bruno; `[09:32]` Larissa | CAN-DEC-019 | Cenários (aberto) | Autenticação e autorização | Autorização | ADR-006 | COBERTO | JWT atual é do operador; body×path em aberto (TRK-103). |
| TRK-058 | DECISÃO | O filtro de status é aplicado na inserção na outbox, não no envio. | TRANSCRICAO | `[09:34]` Bruno; `[09:34]` Diego | CAN-DEC-020 | RN-003 | Publicação transacional | Integração transacional | ADR-001 | COBERTO | Se nenhum webhook assina o status, não insere. |
| TRK-059 | DECISÃO | A integração ocorre via `publishWebhookEvent(tx, order, fromStatus, toStatus)`, recebendo o `tx` atual. | TRANSCRICAO | `[09:41]` Bruno; `[09:41]` Diego | CAN-DEC-021 | Dependências | Publicação transacional | Publicador transacional | ADR-001 | COBERTO | Função pura evita injetar o repository; ponto de código em TRK-106/TRK-116. |
| TRK-060 | DECISÃO | As novas tabelas de webhook usam PK em UUID, seguindo o padrão do projeto. | TRANSCRICAO | `[09:51]` Larissa | CAN-DEC-022 | — | Persistência em alto nível | Modelo de dados conceitual | ADR-006 | COBERTO | Alternativa (auto-incremental) em TRK-074; padrão em TRK-108. |
| TRK-061 | DECISÃO | O payload é JSON com `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents`. | TRANSCRICAO | `[09:43]` Diego; `[09:44]` Bruno | CAN-DEC-024 | RF-008/CA-009 | Payload do evento | Contrato outbound | — | COBERTO | Taxonomia de `event_type` além de status_changed é lacuna (CAN-LAC-011). **Sem ADR dedicado**; também README. |
| TRK-062 | DECISÃO | O payload não inclui os `items` do pedido; detalhes via `GET /orders/:id`. | TRANSCRICAO | `[09:43]` Diego | CAN-DEC-025 | RN-008/Não objetivos | Payload do evento | Contrato outbound | — | COBERTO | Payload enxuto. **Sem ADR dedicado**; também README. |
| TRK-063 | DECISÃO | Criar o módulo `src/modules/webhooks` (controller, service, repository, routes, schemas). | TRANSCRICAO | `[09:27]` Bruno; `[09:28]` Diego | CAN-DEC-027 | — | Arquitetura proposta | Organização dos arquivos | ADR-006 | COBERTO | Segue o padrão de módulos existente. |
| TRK-064 | DECISÃO | Criar o entry-point separado `src/worker.ts` com script `npm run worker`. | TRANSCRICAO | `[09:11]` Larissa; `[09:28]` Bruno | CAN-DEC-028 | Dependências | Worker | Worker/Organização dos arquivos | ADR-002 | COBERTO | Modelo em TRK-112; supervisão em produção é lacuna (CAN-LAC-010). |

### 5.5 Alternativas descartadas

| ID | Categoria | Item rastreável | Fonte primária | Localização | ID canônico | PRD | RFC | FDD | ADR | Situação | Observações |
|---|---|---|---|---|---|---|---|---|---|---|---|
| TRK-065 | ALTERNATIVA_DESCARTADA | Disparo síncrono do webhook dentro do service de orders na mudança de status. | TRANSCRICAO | `[09:03]` Larissa; `[09:06]` Diego | CAN-ALT-001 | Não objetivos | Alternativas consideradas | Fora de escopo | ADR-001 | COBERTO | "Síncrono está fora de questão" (`[09:06]` Diego). |
| TRK-066 | ALTERNATIVA_DESCARTADA | Redis Streams ou infra de fila dedicada. | TRANSCRICAO | `[09:07]` Larissa; `[09:07]` Diego | CAN-ALT-002 | Não objetivos | Alternativas consideradas | Fora de escopo | ADR-001 | COBERTO | Overengineering para time pequeno; relacionado a ADR-002. |
| TRK-067 | ALTERNATIVA_DESCARTADA | Trigger de banco para notificar o worker de forma reativa. | TRANSCRICAO | `[09:09]` Bruno; `[09:09]` Diego | CAN-ALT-003 | — | Alternativas consideradas | — | ADR-002 | COBERTO | MySQL não tem listener nativo (NOTIFY/LISTEN); polling de 2s atende. |
| TRK-068 | ALTERNATIVA_DESCARTADA | 3 tentativas de retry. | TRANSCRICAO | `[09:16]` Bruno; `[09:16]` Diego | CAN-ALT-004 | — | Alternativas consideradas | — | ADR-003 | COBERTO | Mataria o evento em ~30min; escolhido 5 (TRK-047). |
| TRK-069 | ALTERNATIVA_DESCARTADA | Retry indefinido com backoff. | TRANSCRICAO | `[09:15]` Diego | CAN-ALT-005 | — | Alternativas consideradas | — | ADR-003 | COBERTO | Evento ficaria pendurado para sempre se o cliente sumisse. |
| TRK-070 | ALTERNATIVA_DESCARTADA | DLQ como marcação "failed" na própria outbox (sem tabela separada). | TRANSCRICAO | `[09:17]` Larissa; `[09:18]` Diego | CAN-ALT-006 | — | Alternativas (nota) | — | ADR-003 | COBERTO | Tabela separada mantém a outbox limpa; escolhido TRK-048. |
| TRK-071 | ALTERNATIVA_DESCARTADA | Múltiplos workers em paralelo / ordering global desde já. | TRANSCRICAO | `[09:12]` Diego; `[09:13]` Larissa | CAN-ALT-007 | Não objetivos | Alternativas consideradas | Fora de escopo | ADR-002 | COBERTO | Perderia a garantia de ordem; escala futura em TRK-081. |
| TRK-072 | ALTERNATIVA_DESCARTADA | Truncar o payload acima do limite de tamanho. | TRANSCRICAO | `[09:23]` Sofia; `[09:24]` Diego | CAN-ALT-008 | — | Alternativas (nota)/RNF-05 | Limite de payload | — | COBERTO | Escolhido: erro com limite de 64 KB (TRK-029/TRK-088). |
| TRK-073 | ALTERNATIVA_DESCARTADA | Guardar apenas `order_id` na outbox e renderizar o payload no envio. | TRANSCRICAO | `[09:51]` Bruno; `[09:52]` Larissa | CAN-ALT-009 | — | Alternativas (nota)/Payload | Contrato outbound | — | COBERTO | Perderia o estado no instante da mudança; escolhido snapshot (TRK-027). |
| TRK-074 | ALTERNATIVA_DESCARTADA | ID auto-incremental para a outbox. | TRANSCRICAO | `[09:51]` Diego; `[09:51]` Larissa | CAN-ALT-010 | — | Persistência (nota) | Modelo de dados | ADR-006 | COBERTO | Não segue o padrão UUID; escolhido UUID (TRK-060). |
| TRK-075 | ALTERNATIVA_DESCARTADA | Garantia de entrega exactly-once. | TRANSCRICAO | `[09:25]` Diego | CAN-ALT-011 | Não objetivos | Alternativas/Semântica | Semântica de entrega | ADR-005 | COBERTO | Exigiria coordenação bilateral; escolhido at-least-once (TRK-053). |

### 5.6 Itens fora de escopo e adiados

| ID | Categoria | Item rastreável | Fonte primária | Localização | ID canônico | PRD | RFC | FDD | ADR | Situação | Observações |
|---|---|---|---|---|---|---|---|---|---|---|---|
| TRK-076 | ITEM_FORA_DE_ESCOPO | Webhooks inbound: a plataforma só envia, não recebe eventos de volta. | TRANSCRICAO | `[09:02]` Marcos; `[09:03]` Sofia | AUSENTE NA MATRIZ | Fora de escopo | Não objetivos | — | — | PARCIALMENTE_COBERTO | Item de escopo publicado em PRD e RFC sem ID canônico (CAN-FORA) na matriz; pendência documental (§14 INC-02). |
| TRK-077 | ITEM_ADIADO | Notificar o cliente por email quando o webhook falha N vezes seguidas. | TRANSCRICAO | `[09:37]` Marcos; `[09:37]` Larissa | CAN-FORA-001 | Não objetivos/Adiados | Adiado | Fora de escopo | — | COBERTO | "Talvez próxima fase, após medir impacto." |
| TRK-078 | ITEM_FORA_DE_ESCOPO | Dashboard/painel visual para o cliente ver seus webhooks. | TRANSCRICAO | `[09:39]` Marcos; `[09:40]` Larissa | CAN-FORA-002 | Não objetivos | Não objetivos | Fora de escopo | — | COBERTO | Projeto separado do time de frontend; nesta fase só endpoints. |
| TRK-079 | ITEM_ADIADO | Arquivamento de linhas entregues da outbox após ~30 dias. | TRANSCRICAO | `[09:08]` Diego | CAN-FORA-003 | Itens adiados | Adiado | Fora de escopo | ADR-001 | COBERTO | Valor ~30 dias (CAN-MET-010) consolidado aqui; "fora do escopo dessa feature". |
| TRK-080 | ITEM_ADIADO | Endurecer a autorização do CRUD de configuração (restringir por role). | TRANSCRICAO | `[09:36]` Marcos; `[09:37]` Sofia | CAN-FORA-004 | Itens adiados | Adiado | Fora de escopo | ADR-004 | COBERTO | Por ora qualquer role autenticada; "mais pra frente a gente pode endurecer" (relacionado ADR-006). |
| TRK-081 | ITEM_ADIADO | Escalar para múltiplos workers (particionar por `order_id` ou lock pessimista). | TRANSCRICAO | `[09:13]` Diego | CAN-FORA-005 | Itens adiados | Adiado | Fora de escopo | ADR-002 | COBERTO | "Problema do futuro, não agora"; limitação atual em TRK-035. |

### 5.7 Métricas e limites

| ID | Categoria | Item rastreável | Fonte primária | Localização | ID canônico | PRD | RFC | FDD | ADR | Situação | Observações |
|---|---|---|---|---|---|---|---|---|---|---|---|
| TRK-082 | MÉTRICA | Definição de "tempo real" = latência abaixo de 10 segundos. | TRANSCRICAO | `[09:02]` Marcos | CAN-MET-001 | RNF-001 | RNF-01 | Resumo/Configurações | — | COBERTO | Requisito espelhado em CAN-RNF-007; não é garantia absoluta em qualquer cenário. |
| TRK-083 | MÉTRICA | Intervalo de polling do worker = 2 segundos. | TRANSCRICAO | `[09:09]` Diego | CAN-MET-002 | RNF-002 | RNF-02 | Configurações | ADR-002 | COBERTO | Decisão em TRK-044. |
| TRK-084 | MÉTRICA | O polling adiciona até ~2s de espera **apenas na descoberta** do evento (não é latência mínima total). | TRANSCRICAO | `[09:10]` Larissa | CAN-MET-003 | RNF-002 | Semântica do polling (RFC/ADR-002) | — | ADR-002 | COBERTO | Reformulação canônica de `[09:10]`; tempo total inclui banco/HTTP, dentro de <10s. |
| TRK-085 | MÉTRICA | Teto de tentativas de retry = 5, antes de mover para a DLQ. | TRANSCRICAO | `[09:15]` Diego; `[09:17]` Larissa | CAN-MET-004 | RNF-011 | RNF-10 | Retry e classificação | ADR-003 | COBERTO | Contagem "5 totais × 5 retries" preservada como aberta (§11). |
| TRK-086 | MÉTRICA | Progressão do backoff = 1m / 5m / 30m / 2h / 12h (~15h no total). | TRANSCRICAO | `[09:17]` Diego | CAN-MET-005 | RNF-012 | RNF-11 | Retry e classificação | ADR-003 | COBERTO | Cobre janela de indisponibilidade de até ~15h. |
| TRK-087 | MÉTRICA | Grace period de rotação de secret = 24 horas. | TRANSCRICAO | `[09:21]` Sofia | CAN-MET-006 | RNF-008 | RNF-09 | Rotacionar secret | ADR-004 | COBERTO | Comportamento em TRK-021; instante inicial das 24h em aberto (§11). |
| TRK-088 | MÉTRICA | Limite de tamanho de payload = 64 KB (acima disso, erro). | TRANSCRICAO | `[09:24]` Diego; `[09:24]` Larissa | CAN-MET-007 | RNF-004 | RNF-05 | Limite de payload | — | COBERTO | Comportamento em TRK-029. |
| TRK-089 | MÉTRICA | Timeout de chamada HTTP = 10 segundos. | TRANSCRICAO | `[09:42]` Diego | CAN-MET-008 | RNF-003 | RNF-04 | Timeout | ADR-003 | COBERTO | Consequência em TRK-033; único critério de falha fechado. |
| TRK-090 | MÉTRICA | Tamanho do histórico de entregas = últimas 100 entregas. | TRANSCRICAO | `[09:34]` Marcos | CAN-MET-009 | RNF-018 | RF-09 | Histórico de entregas | — | COBERTO | Paginação além disso é lacuna (CAN-LAC-008). |
| TRK-091 | MÉTRICA | Estimativa de prazo = 3 sprints (inclui a revisão de segurança). | TRANSCRICAO | `[09:46]` Larissa; `[09:47]` Larissa | CAN-MET-011 | Restrições | Estratégia de implementação | — | — | COBERTO | Reserva de segurança em TRK-092. |
| TRK-092 | MÉTRICA | Reserva para revisão de segurança = pelo menos 2 dias úteis antes do deploy. | TRANSCRICAO | `[09:46]` Sofia | CAN-MET-012 | Públicos/Fase 1 | Estratégia de implementação | Implantação e operação | ADR-004 | COBERTO | Sofia quer revisar HMAC e geração de secret. |
| TRK-093 | MÉTRICA | Prazo de negócio (Atlas) = fim de novembro. | TRANSCRICAO | `[09:45]` Marcos | CAN-MET-013 | Contexto/Restrições | Contexto e motivação | — | — | COBERTO | Risco comercial em TRK-004/TRK-101. |

### 5.8 Riscos

| ID | Categoria | Item rastreável | Fonte primária | Localização | ID canônico | PRD | RFC | FDD | ADR | Situação | Observações |
|---|---|---|---|---|---|---|---|---|---|---|---|
| TRK-094 | RISCO | Cliente offline/indisponível por período longo deixa eventos pendentes. | TRANSCRICAO | `[09:14]` Larissa; `[09:16]` Diego | CAN-RISK-001 | R-01 | Riscos e mitigações | — | ADR-003 | COBERTO | Mitigação: backoff + 5 tentativas + DLQ. |
| TRK-095 | RISCO | Cliente recebe o mesmo evento mais de uma vez (at-least-once). | TRANSCRICAO | `[09:24]` Diego; `[09:25]` Bruno | CAN-RISK-002 | R-02 | Riscos e mitigações | Riscos técnicos | ADR-005 | COBERTO | Mitigação: `X-Event-Id` para dedup no cliente. |
| TRK-096 | RISCO | Vazamento de secret (já ocorreu: cliente vazou secret em log). | TRANSCRICAO | `[09:21]` Sofia; `[09:22]` Diego | CAN-RISK-003 | R-04 | Riscos e mitigações | Riscos técnicos | ADR-004 | COBERTO | Mitigação: secret por endpoint + rotação com grace de 24h. |
| TRK-097 | RISCO | Perda de ordenação se escalar para múltiplos workers. | TRANSCRICAO | `[09:12]` Diego; `[09:13]` Larissa | CAN-RISK-004 | R-09 | Riscos e mitigações | — | ADR-002 | COBERTO | Mitigação atual: single-worker (TRK-046); futuro em TRK-081. |
| TRK-098 | RISCO | Falha na inserção do evento poderia gerar status alterado sem evento (inconsistência). | TRANSCRICAO | `[09:40]` Bruno; `[09:41]` Diego | CAN-RISK-005 | — | Problema técnico/Riscos | Integração transacional | ADR-001 | COBERTO | Mitigação: inserção na mesma transação com rollback (TRK-031). |
| TRK-099 | RISCO | Payload excessivamente grande (ex.: 500 KB). | TRANSCRICAO | `[09:23]` Sofia; `[09:24]` Diego | CAN-RISK-006 | R-11 | Riscos e mitigações | Limite de payload | — | COBERTO | Mitigação: limite de 64 KB com erro (TRK-029/TRK-088). |
| TRK-100 | RISCO | Bombardeio do cliente com muitas chamadas (ex.: 50 status/min). | TRANSCRICAO | `[09:38]` Diego; `[09:39]` Larissa | CAN-RISK-007 | Não objetivos | Riscos e mitigações | — | — | COBERTO | Sem mitigação definida — observar e decidir (TRK-105). |
| TRK-101 | RISCO | Perda comercial: Atlas migrar para concorrente se não entregar no prazo. | TRANSCRICAO | `[09:00]` Marcos; `[09:45]` Marcos | CAN-RISK-008 | R-14 | Riscos e mitigações | — | — | COBERTO | Mitigação: prazo de 3 sprints / fim de novembro (TRK-091/TRK-093). |
| TRK-102 | RISCO | Replay attack contra o endpoint do cliente. | TRANSCRICAO | `[09:44]` Diego | CAN-RISK-009 | — | Segurança/Replay attacks | Replay attacks | ADR-004 | COBERTO | Mitigação parcial: `X-Timestamp` permite o cliente detectar "se quiser"; HMAC sozinho não impede replay. |

### 5.9 Questões em aberto

| ID | Categoria | Item rastreável | Fonte primária | Localização | ID canônico | PRD | RFC | FDD | ADR | Situação | Observações |
|---|---|---|---|---|---|---|---|---|---|---|---|
| TRK-103 | QUESTÃO_ABERTA | O `customer_id` deve ser passado no body ou no path dos endpoints de configuração? | TRANSCRICAO | `[09:32]` Larissa; `[09:33]` Bruno | CAN-OPEN-001 | Questões em aberto | Questões em aberto | Questões em aberto | — | COBERTO | Definido que NÃO vem do JWT (TRK-057); localização exata em aberto. Não resolver. |
| TRK-104 | QUESTÃO_ABERTA | Nome do arquivo de processamento do worker: `webhook.worker.ts` ou `webhook.processor.ts`? | TRANSCRICAO | `[09:28]` Bruno | CAN-OPEN-002 | — | — | Questões em aberto/Organização | ADR-006 | COBERTO | Baixo risco; decisão de organização (CAN-DEC-029). Não resolver. |
| TRK-105 | QUESTÃO_ABERTA | Haverá rate limiting de saída para o cliente (ex.: 50 mudanças/min)? | TRANSCRICAO | `[09:38]` Diego; `[09:39]` Larissa | CAN-OPEN-003 | Não objetivos/Adiados | Questões em aberto | — | — | COBERTO | "Observar e implementar se virar problema"; risco em TRK-100. Não resolver. |

### 5.10 Evidências de código

| ID | Categoria | Item rastreável | Fonte primária | Localização | ID canônico | PRD | RFC | FDD | ADR | Situação | Observações |
|---|---|---|---|---|---|---|---|---|---|---|---|
| TRK-106 | EVIDÊNCIA_DE_CÓDIGO | A mudança de status já executa, em `$transaction`, validação, estoque, `order.update` e `orderStatusHistory.create`. | CODIGO | `src/modules/orders/order.service.ts` — `OrderService.changeStatus` | CAN-COD-001 | Dependências | Impacto no código existente | Integração transacional | ADR-001 | COBERTO | Ponto onde `publishWebhookEvent(tx, ...)` **será** chamado (futuro, TRK-116). Hoje **não** publica eventos. |
| TRK-107 | EVIDÊNCIA_DE_CÓDIGO | A máquina de transições e efeitos de estoque estão em `canTransition`, `shouldDebitStock`, `shouldReplenishStock`. | CODIGO | `src/modules/orders/order.status.ts` — `canTransition`, `shouldDebitStock`, `shouldReplenishStock` | CAN-COD-003 | RN-004 | Impacto no código existente | Integração transacional | ADR-001 | COBERTO | Fonte da verdade dos eventos; módulo de webhook **não** deve duplicá-la. Sustenta RN-004 (transição inválida não gera evento). |
| TRK-108 | EVIDÊNCIA_DE_CÓDIGO | O schema Prisma usa provider `mysql`, enum `OrderStatus` e PKs `@default(uuid()) @db.Char(36)`; **sem** models de webhook. | CODIGO | `prisma/schema.prisma` — `datasource db`, `enum OrderStatus`, models `Order`/`OrderStatusHistory`/`Product` | CAN-COD-002 | Dependências | Estado atual do sistema | Estado atual do sistema | ADR-006 | COBERTO | Enum alimenta `from_status`/`to_status`; padrão UUID para novas tabelas (TRK-060). |
| TRK-109 | EVIDÊNCIA_DE_CÓDIGO | Autenticação JWT bearer (`authenticate`) e `requireRole(...)` comparando `req.user.role` (`ADMIN`/`OPERATOR`). | CODIGO | `src/middlewares/auth.middleware.ts` — `authenticate`, `requireRole` | CAN-COD-007 | Dependências | Estado atual do sistema | Autenticação e autorização | ADR-006 | COBERTO | `requireRole('ADMIN')` no replay (TRK-050); `authenticate` no CRUD (TRK-119). |
| TRK-110 | EVIDÊNCIA_DE_CÓDIGO | O logger Pino redige `authorization`, `cookie`, `*.password`, `*.passwordHash`, `*.token`, `*.accessToken`; `secret` **não** está na lista. | CODIGO | `src/shared/logger/index.ts` — `redactPaths`, `createLogger` | CAN-COD-013 | RNF-017 | Segurança/Impacto no código | Logging e observabilidade | ADR-004 | COBERTO | Fato observável; proposta de acrescentar `secret` é TRK-122 (futuro). |
| TRK-111 | EVIDÊNCIA_DE_CÓDIGO | Rota real protegida por `requireRole('ADMIN')` (precedente do endpoint de replay). | CODIGO | `src/modules/users/user.routes.ts` — `buildUserRouter` | CAN-COD-022 | Dependências | Autenticação e autorização | Administração da DLQ | ADR-003 | COBERTO | `GET /users/:id` é hoje a única rota protegida por papel; relacionado ADR-006. |
| TRK-112 | EVIDÊNCIA_DE_CÓDIGO | Entry-point da API com `buildApp`, `listen` e shutdown por `SIGINT`/`SIGTERM` com `prisma.$disconnect`. | CODIGO | `src/server.ts` — `bootstrap` | CAN-COD-016 | — | Impacto no código existente | Worker/Implantação | ADR-002 | COBERTO | Modelo de lifecycle para `src/worker.ts` (futuro, TRK-121). |
| TRK-113 | EVIDÊNCIA_DE_CÓDIGO | `package.json` traz `dev/build/start/db:*/test`, `engines.node >=20` e `uuid` 11.0.3; **sem** lib de HMAC e **sem** script `worker`. | CODIGO | `package.json` — `scripts`, `dependencies`, `engines` | CAN-COD-019 | — | Impacto no código existente | Estado atual do sistema | — | COBERTO | `uuid` e `node:crypto` são disponibilidade, **não** decisão de uso (propostas em TRK-125). |
| TRK-114 | EVIDÊNCIA_DE_CÓDIGO | `buildApp`/`buildControllers` compõem Express manualmente, com `express.json({ limit: '1mb' })` e prefixo `/api/v1`. | CODIGO | `src/app.ts` — `buildApp`, `buildControllers` | CAN-COD-017 | — | Impacto no código existente | Arquitetura futura | ADR-006 | COBERTO | Ponto de registro de `WebhookController` (futuro, TRK-118). |
| TRK-115 | EVIDÊNCIA_DE_CÓDIGO | Testes com Vitest/Supertest; `setup.ts` limpa tabelas em `beforeEach`; factories criam user/customer/product. | CODIGO | `tests/setup.ts`, `tests/helpers/factories.ts` — `getTestApp`, `bootstrapAuthenticatedUser` | CAN-COD-021 | Dependências | Estratégia de testes | Estratégia de testes | ADR-006 | COBERTO | `setup.ts` precisará limpar as novas tabelas (futuro, TRK-124). |

### 5.11 Integrações futuras (propostas / decididas, ainda não implementadas)

| ID | Categoria | Item rastreável | Fonte primária | Localização | ID canônico | PRD | RFC | FDD | ADR | Situação | Observações |
|---|---|---|---|---|---|---|---|---|---|---|---|
| TRK-116 | INTEGRAÇÃO_FUTURA | **Futuro:** chamar `publishWebhookEvent(tx, ...)` dentro da transação de `changeStatus`, com rollback se a inserção falhar. | ANALISE | ANALISE §15 INT-01; `src/modules/orders/order.service.ts` — `changeStatus`; `[09:41]` Bruno/Diego | CAN-INT-001 | Dependências | Publicação transacional | Publicador transacional | ADR-001 | COBERTO | Base de código em TRK-106; decisão em TRK-059. Não existe hoje. |
| TRK-117 | INTEGRAÇÃO_FUTURA | **Futuro:** modelar as novas tabelas Prisma (outbox, `webhook_dead_letter`, configuração, deliveries). | ANALISE | ANALISE §15 INT-02; `prisma/schema.prisma`; `[09:06]`/`[09:18]` Diego; `[09:34]` Marcos | CAN-INT-002 | Dependências | Persistência em alto nível | Modelo de dados conceitual | — | COBERTO | Campos completos são lacunas (CAN-LAC-001/002/005). A criar. |
| TRK-118 | INTEGRAÇÃO_FUTURA | **Futuro:** registrar o router `/webhooks` e a rota admin de replay na composição de rotas. | ANALISE | ANALISE §15 INT-03; `src/routes/index.ts`, `src/app.ts`; `[09:33]` Bruno; `[09:35]` Diego | CAN-INT-003 | — | Arquitetura proposta | Contratos HTTP da API | ADR-006 | COBERTO | Base em TRK-114. A criar. |
| TRK-119 | INTEGRAÇÃO_FUTURA | **Futuro:** aplicar `authenticate` no CRUD e `requireRole('ADMIN')` no replay. | ANALISE | ANALISE §15 INT-04; `src/middlewares/auth.middleware.ts`; `[09:36]` Sofia | CAN-INT-004 | — | Autenticação e autorização | Autorização | ADR-006 | COBERTO | Base em TRK-109/TRK-111. |
| TRK-120 | INTEGRAÇÃO_FUTURA | **Futuro:** definir as classes de erro `WEBHOOK_*` sobre `AppError`. | ANALISE | ANALISE §15 INT-05; `src/shared/errors/*`; `[09:28]`/`[09:29]` Bruno | CAN-INT-005 | — | Estado atual do sistema | Tratamento de erros | ADR-006 | COBERTO | Middleware existente já as trata; prefixo em TRK-041. Não existem hoje. |
| TRK-121 | INTEGRAÇÃO_FUTURA | **Futuro:** criar `src/worker.ts` como entry separado, com `PrismaClient` próprio e script `npm run worker`. | ANALISE | ANALISE §15 INT-06; `src/server.ts`, `src/config/database.ts`; `[09:11]`/`[09:28]`/`[09:30]` Bruno | CAN-INT-006 | Dependências | Impacto no código existente | Worker | ADR-002 | COBERTO | Decisões em TRK-056/TRK-064; supervisão em produção é lacuna (CAN-LAC-010). |
| TRK-122 | INTEGRAÇÃO_FUTURA | **Futuro:** acrescentar `secret`/assinatura à redação do Pino, além de reutilizar o logger no worker. | ANALISE | ANALISE §15 INT-07; `src/shared/logger/index.ts`; `[09:29]` Bruno | CAN-INT-007 | RNF-017 | Segurança | Logging e observabilidade | ADR-004 | COBERTO | Ponto de atenção derivado de TRK-110; a tratar antes do lançamento (relacionado ADR-006). |
| TRK-123 | INTEGRAÇÃO_FUTURA | **Em aberto:** decidir se intervalo de polling e timeout serão constantes no código ou variáveis de ambiente. | ANALISE | ANALISE §15 INT-08; `src/config/env.ts`; `[09:09]`/`[09:42]` Diego | CAN-INT-008 | — | Questões em aberto/Impacto no código | Configurações | — | PARCIALMENTE_COBERTO | Nenhuma env nova foi decidida na reunião (TRK-113); decisão pendente, registrada como aberta em RFC/FDD. |
| TRK-124 | INTEGRAÇÃO_FUTURA | **Futuro:** prever testes de webhook e atualizar `tests/setup.ts` para limpar as novas tabelas. | ANALISE | ANALISE §15 INT-09; `tests/*`; `[09:46]` Larissa | CAN-INT-009 | Dependências | Estratégia de testes | Estratégia de testes | — | COBERTO | Base em TRK-115. |
| TRK-125 | INTEGRAÇÃO_FUTURA | **Futuro:** especificar util de assinatura HMAC-SHA256 usando `node:crypto`. | ANALISE | ANALISE §15 INT-10; `package.json`; `[09:20]` Sofia | CAN-INT-010 | — | Headers (nota)/Segurança | Contrato outbound/Assinador HMAC | ADR-004 | COBERTO | Não há utilitário de HMAC hoje (TRK-113); formato/encoding em aberto (§11). |

---

## 6. Cobertura por documento

> **Método (reprodutível a partir da §5):** *Itens identificáveis* = linhas do Tracker cujo destino esperado inclui o documento (coluna do documento ≠ `—`). *Cobertos* = situação `COBERTO`; *Parciais* = `PARCIALMENTE_COBERTO`. Para os ADRs, *itens identificáveis* = IDs canônicos roteados ao ADR em `MATRIZ-FATOS.md` §3, todos rastreados na §5. `Cobertura = cobertos ÷ identificáveis × 100`. `Cobertura ponderada = (cobertos + 0,5 × parciais) ÷ identificáveis × 100`.

| Documento | Itens identificáveis | Itens rastreados | Cobertos | Parciais | Não cobertos | Cobertura | Cobertura ponderada |
|---|---:|---:|---:|---:|---:|---:|---:|
| PRD | 99 | 99 | 96 | 3 | 0 | 97,0% | 98,5% |
| RFC | 118 | 118 | 115 | 3 | 0 | 97,5% | 98,7% |
| FDD | 106 | 106 | 105 | 1 | 0 | 99,1% | 99,5% |
| ADR-001 | 9 | 9 | 9 | 0 | 0 | 100% | 100% |
| ADR-002 | 16 | 16 | 16 | 0 | 0 | 100% | 100% |
| ADR-003 | 19 | 19 | 19 | 0 | 0 | 100% | 100% |
| ADR-004 | 15 | 15 | 15 | 0 | 0 | 100% | 100% |
| ADR-005 | 7 | 7 | 7 | 0 | 0 | 100% | 100% |
| ADR-006 | 25 | 25 | 25 | 0 | 0 | 100% | 100% |

**Parciais por documento:** PRD → TRK-009, TRK-010, TRK-076; RFC → TRK-010, TRK-076, TRK-123; FDD → TRK-123. Todos são pendências documentais **corrigíveis** (IDs ausentes na matriz ou decisão de configuração em aberto), não contradições com as fontes.

> A cobertura próxima de 100% reflete que PRD/RFC/FDD e os seis ADRs foram derivados **diretamente da matriz canônica**. O valor da auditoria está nos parciais e nas pendências (§12/§14), não na nota agregada.

---

## 7. Cobertura por fonte primária

> `Percentual = quantidade da fonte ÷ total de linhas do Tracker × 100`. Cada linha conta em **uma** fonte (a mais primária). ADR, PRD, RFC e FDD **não** são fontes primárias; a matriz é índice canônico, não fonte primária.

| Fonte primária | Quantidade | Percentual | Observação |
|---|---:|---:|---|
| TRANSCRICAO | 104 | 83,2% | Acima do mínimo de 70%. Problemas, públicos, requisitos, decisões, alternativas, métricas, riscos e questões abertas. |
| CODIGO | 10 | 8,0% | Acima do mínimo de 5. Comportamentos existentes (TRK-106 a TRK-115); nenhum artefato futuro citado como existente. |
| ANALISE | 11 | 8,8% | Integrações futuras (TRK-116 a TRK-125) e um stakeholder derivado (TRK-009). |
| **Total** | **125** | **100%** | — |

---

## 8. Cobertura por categoria

| Categoria | Itens | Cobertos | Parciais | Não cobertos | Cobertura |
|---|---:|---:|---:|---:|---:|
| PROBLEMA | 4 | 4 | 0 | 0 | 100% |
| PÚBLICO | 6 | 4 | 2 | 0 | 66,7% |
| REQUISITO_FUNCIONAL | 17 | 17 | 0 | 0 | 100% |
| REQUISITO_NÃO_FUNCIONAL | 14 | 14 | 0 | 0 | 100% |
| DECISÃO | 23 | 23 | 0 | 0 | 100% |
| ALTERNATIVA_DESCARTADA | 11 | 11 | 0 | 0 | 100% |
| ITEM_FORA_DE_ESCOPO | 2 | 1 | 1 | 0 | 50% |
| ITEM_ADIADO | 4 | 4 | 0 | 0 | 100% |
| MÉTRICA | 12 | 12 | 0 | 0 | 100% |
| RISCO | 9 | 9 | 0 | 0 | 100% |
| QUESTÃO_ABERTA | 3 | 3 | 0 | 0 | 100% |
| EVIDÊNCIA_DE_CÓDIGO | 10 | 10 | 0 | 0 | 100% |
| INTEGRAÇÃO_FUTURA | 10 | 9 | 1 | 0 | 90% |
| **Total** | **125** | **121** | **4** | **0** | **96,8%** |

**Leituras da distribuição:** decisões, alternativas, requisitos, métricas e riscos com cobertura integral. A menor cobertura está em **PÚBLICO** (dois stakeholders do PRD sem ID canônico na matriz) e em **ITEM_FORA_DE_ESCOPO** (webhooks inbound sem ID canônico). Nenhuma questão aberta foi esquecida; nenhum risco principal ficou fora.

---

## 9. Cobertura das decisões arquiteturais

| ADR | Decisão principal | IDs do Tracker | Alternativas rastreadas | Consequências/riscos rastreados | Situação |
|---|---|---|---|---|---|
| [ADR-001](./adrs/ADR-001-outbox-no-mysql.md) | Transactional Outbox no MySQL; evento na mesma transação; rollback conjunto; filtro na inserção; snapshot; HTTP fora da transação | TRK-042, TRK-043, TRK-058, TRK-059, TRK-031, TRK-027 | TRK-065 (síncrono), TRK-066 (Redis/fila) | TRK-098 (status sem evento), TRK-079 (crescimento outbox) | COBERTO |
| [ADR-002](./adrs/ADR-002-worker-separado-com-polling.md) | Worker em processo separado; polling 2s; single-worker; ordem por `order_id` sem ordering global; múltiplos workers adiados; Prisma próprio | TRK-032, TRK-044, TRK-046, TRK-034, TRK-035, TRK-056, TRK-064, TRK-045 | TRK-067 (trigger), TRK-071 (múltiplos workers) | TRK-097 (perda de ordem), TRK-081 (escala adiada) | COBERTO |
| [ADR-003](./adrs/ADR-003-retry-backoff-e-dlq.md) | 5 tentativas; backoff 1m/5m/30m/2h/12h; DLQ em tabela dedicada; replay admin (ADMIN) auditado; timeout 10s | TRK-047, TRK-085, TRK-086, TRK-023, TRK-048, TRK-049, TRK-019, TRK-050, TRK-030, TRK-089, TRK-033 | TRK-068 (3 tentativas), TRK-069 (indefinido), TRK-070 (failed na outbox) | TRK-094 (cliente offline), TRK-100 (retry storm) | COBERTO |
| [ADR-004](./adrs/ADR-004-hmac-sha256-e-gestao-de-secrets.md) | HMAC-SHA256; secret por endpoint; rotação com grace de 24h; HTTPS; proteção de logs | TRK-051, TRK-052, TRK-020, TRK-021, TRK-087, TRK-028, TRK-013, TRK-122 | — (contrapontos analíticos: secret global, sem assinatura, rotação imediata — registrados no ADR-004) | TRK-096 (vazamento de secret), TRK-102 (replay attack) | COBERTO |
| [ADR-005](./adrs/ADR-005-entrega-at-least-once-com-event-id.md) | Entrega at-least-once; duplicidade possível; UUID; `X-Event-Id`; dedup pelo cliente; exactly-once descartado | TRK-053, TRK-095, TRK-025, TRK-024, TRK-054 | TRK-075 (exactly-once) | TRK-095 (evento duplicado) | COBERTO |
| [ADR-006](./adrs/ADR-006-reuso-dos-padroes-do-projeto.md) | Estrutura modular; composição/rotas; `authenticate`/`requireRole`; Zod; `AppError` + middleware; Pino; testes; PK UUID; regras de pedidos preservadas | TRK-055, TRK-063, TRK-039, TRK-036, TRK-038, TRK-040, TRK-037, TRK-041, TRK-060, TRK-119, TRK-107 | TRK-074 (ID auto-incremental) | TRK-110/TRK-122 (redação de secret insuficiente hoje) | COBERTO |

Todas as decisões principais dos seis ADRs estão rastreadas, com pelo menos uma alternativa e uma consequência/risco por ADR. Decisões **sem ADR dedicado** (snapshot do payload — TRK-027/CAN-DEC-023; contrato do payload — TRK-061/TRK-062) estão detalhadas no FDD/RFC, conforme a nota de escopo da matriz.

---

## 10. Cobertura dos requisitos do PRD

> A numeração do PRD é **local** e independente dos IDs canônicos. Todo `RF-NNN` e `RNF-NNN` do PRD possui ID de Tracker.

### 10.1 Requisitos funcionais do PRD

| ID do PRD | Título | ID do Tracker | ID canônico | RFC | FDD | Situação |
|---|---|---|---|---|---|---|
| RF-001 | Criar configuração de webhook | TRK-012, TRK-013 | CAN-RF-002, CAN-RF-003 | RF-01/RF-07 | Criar webhook | COBERTO |
| RF-002 | Listar configurações | TRK-016 | CAN-RF-006 | RF-02 | Listar webhooks | COBERTO |
| RF-003 | Atualizar configuração | TRK-014 | CAN-RF-004 | RF-03 | Atualizar webhook | COBERTO |
| RF-004 | Remover configuração | TRK-015 | CAN-RF-005 | RF-04 | Remover webhook | COBERTO |
| RF-005 | Selecionar status de interesse | TRK-017 | CAN-RF-007 | RF-05 | Filtro de status | COBERTO |
| RF-006 | Criar evento após mudança de status | TRK-011 | CAN-RF-001 | RF-06 | Criar evento na outbox | COBERTO |
| RF-007 | Entregar evento ao endpoint | TRK-011, TRK-044 | CAN-RF-001, CAN-DEC-003 | RF-06 | Processar evento | COBERTO |
| RF-008 | Enviar payload definido | TRK-061, TRK-027 | CAN-DEC-024, CAN-RF-018 | Payload do evento | Contrato outbound | COBERTO |
| RF-009 | Enviar headers definidos | TRK-024, TRK-026, TRK-051 | CAN-RF-014, CAN-RF-017, CAN-DEC-011 | RF-14/Headers | Contrato outbound | COBERTO |
| RF-010 | Gerar secret por endpoint | TRK-013, TRK-052 | CAN-RF-003, CAN-DEC-012 | RF-07/RNF-08 | Criar webhook | COBERTO |
| RF-011 | Rotacionar secret | TRK-020, TRK-021 | CAN-RF-010, CAN-RF-011 | RF-08/RNF-09 | Rotacionar secret | COBERTO |
| RF-012 | Registrar tentativas de entrega | TRK-018 | CAN-RF-008 | RF-09 | Histórico de entregas | COBERTO |
| RF-013 | Consultar as últimas 100 entregas | TRK-018, TRK-090 | CAN-RF-008, CAN-MET-009 | RF-09 | Histórico de entregas | COBERTO |
| RF-014 | Repetir entregas com falha | TRK-022, TRK-086 | CAN-RF-012, CAN-MET-005 | RF-10 | Retry | COBERTO |
| RF-015 | Encaminhar evento à DLQ | TRK-023 | CAN-RF-013 | RF-11 | Encaminhar à DLQ | COBERTO |
| RF-016 | Executar replay administrativo | TRK-019 | CAN-RF-009 | RF-12 | Executar replay | COBERTO |
| RF-017 | Restringir replay à role ADMIN | TRK-050 | CAN-DEC-010 | Autorização | Autorização | COBERTO |
| RF-018 | Registrar o executor do replay | TRK-030 | CAN-RNF-006 | RF-13 | Executar replay | COBERTO |

### 10.2 Requisitos não funcionais do PRD

| ID do PRD | Título | ID do Tracker | ID canônico | RFC | FDD | Situação |
|---|---|---|---|---|---|---|
| RNF-001 | Menos de 10 segundos em condições normais | TRK-082 | CAN-MET-001 (CAN-RNF-007) | RNF-01 | Configurações | COBERTO |
| RNF-002 | Polling de 2 segundos | TRK-083, TRK-084 | CAN-MET-002, CAN-MET-003 | RNF-02/03 | Configurações | COBERTO |
| RNF-003 | Timeout de 10 segundos | TRK-089, TRK-033 | CAN-MET-008, CAN-RNF-012 | RNF-04 | Timeout | COBERTO |
| RNF-004 | Payload máximo de 64 KB | TRK-088, TRK-029 | CAN-MET-007, CAN-RNF-005 | RNF-05 | Limite de payload | COBERTO |
| RNF-005 | HTTPS obrigatório | TRK-028 | CAN-RNF-004 | RNF-06 | Segurança/Validação | COBERTO |
| RNF-006 | HMAC-SHA256 | TRK-051 | CAN-DEC-011 (CAN-RNF-001/CAN-MET-014) | RNF-07 | HMAC-SHA256 | COBERTO |
| RNF-007 | Secret por endpoint | TRK-052 | CAN-DEC-012 (CAN-RNF-002) | RNF-08 | Gestão de secrets | COBERTO |
| RNF-008 | Grace period de 24 horas | TRK-087, TRK-021 | CAN-MET-006, CAN-RF-011 | RNF-09 | Rotacionar secret | COBERTO |
| RNF-009 | Entrega at-least-once | TRK-053 | CAN-DEC-014 (CAN-RNF-009) | Semântica | Semântica de entrega | COBERTO |
| RNF-010 | Identificador UUID | TRK-025 | CAN-RF-015 | Semântica | Identidade de evento | COBERTO¹ |
| RNF-011 | Cinco tentativas | TRK-085, TRK-047 | CAN-MET-004, CAN-DEC-006 | RNF-10 | Retry | COBERTO |
| RNF-012 | Intervalos 1m/5m/30m/2h/12h | TRK-086 | CAN-MET-005 | RNF-11 | Retry | COBERTO |
| RNF-013 | Worker separado | TRK-032 | CAN-RNF-011 | RNF-13 | Worker | COBERTO |
| RNF-014 | Um worker na primeira versão | TRK-046 | CAN-DEC-005 | Ordenação | Ordenação | COBERTO |
| RNF-015 | Sem ordering global | TRK-034, TRK-035 | CAN-RNF-013, CAN-RNF-014 | RNF-14 | Ordenação | COBERTO |
| RNF-016 | Compatibilidade com padrões existentes | TRK-055 | CAN-DEC-016 | RNF-15 | Premissas | COBERTO |
| RNF-017 | Proteção de secrets/assinaturas em logs | TRK-110, TRK-122 | CAN-COD-013, CAN-INT-007 | Segurança | Logging | COBERTO² |
| RNF-018 | Histórico limitado às últimas 100 entregas | TRK-090 | CAN-MET-009 | RF-09 | Histórico de entregas | COBERTO |

¹ Redação do PRD ("no fluxo de entrega") diverge levemente da fonte/RFC/FDD ("na outbox/inserção") — ver §14 INC-03.
² Requisito documentado, porém com **lacuna de implementação conhecida**: a redação atual do Pino **não** cobre `secret` (TRK-110); ampliação registrada como integração futura (TRK-122).

### 10.3 Regras de negócio e critérios de aceite

As regras de negócio do PRD (RN-001 a RN-016) são **restatements** de fatos já rastreados e não geram linha própria: RN-001→TRK-028; RN-002→TRK-052; RN-003→TRK-058; RN-004→TRK-107; RN-005→TRK-021; RN-006→TRK-061; RN-007→TRK-027; RN-008→TRK-062; RN-009→TRK-029/088; RN-010→TRK-095; RN-011→TRK-054; RN-012→TRK-024/025; RN-013→TRK-050; RN-014→TRK-030; RN-015→TRK-023; RN-016→questão aberta (§11, classificação de falhas HTTP).

Os critérios de aceite CA-001 a CA-025 **testam** requisitos já rastreados e não geram linha própria. **Nenhum** critério de aceite introduz comportamento novo, política HTTP, replay, armazenamento de secret, paginação, locking ou concorrência não sustentados — o PRD encerra a seção declarando explicitamente que status HTTP de sucesso, política 4xx/5xx, preservação do `event_id` no replay, paginação e armazenamento da secret **permanecem em aberto**. Portanto **não há** inconsistência de critério de aceite a registrar em §14.

---

## 11. Cobertura das questões em aberto

> O Tracker **preserva** as questões em aberto e **não** responde a nenhuma delas. Muitas correspondem a lacunas `CAN-LAC-*` (marcadas `NÃO PUBLICAR` na matriz) que os documentos finais **corretamente** expuseram como questões abertas. Situação: `PRESENTE EM TODOS OS DESTINOS NECESSÁRIOS`, `AUSENTE EM DOCUMENTO RELEVANTE`, `RESOLVIDA SEM EVIDÊNCIA`, `NÃO APLICÁVEL AO PRD`, `NÃO APLICÁVEL A ADR`.

| ID canônico | Questão | Origem | PRD | RFC | FDD | ADR relacionado | Situação |
|---|---|---|---|---|---|---|---|
| CAN-OPEN-001 | `customer_id` no body ou no path dos endpoints de configuração | `[09:32]` Larissa | Q#1 | Q#1 | Q#1 | ADR-006 | PRESENTE EM TODOS OS DESTINOS NECESSÁRIOS |
| CAN-LAC-007 | Qual resposta HTTP é sucesso de entrega | `[09:42]` Diego | RN-016/Q#10 | Q#2 | Q#4/Matriz de classificação | ADR-003 | PRESENTE EM TODOS OS DESTINOS NECESSÁRIOS |
| CAN-LAC-007 | Quais falhas HTTP (2xx/3xx/4xx/5xx, DNS, TLS, conexão recusada) são retentáveis | `[09:42]` Diego | Q#11 | Q#3 | Q#5 | ADR-003 | PRESENTE EM TODOS OS DESTINOS NECESSÁRIOS |
| ADR-003 (ambiguidade) | "Cinco tentativas" = cinco envios totais ou cinco retries | `[09:15]`/`[09:17]` | — | Q#19 | Q#3 | ADR-003 | AUSENTE EM DOCUMENTO RELEVANTE (PRD não sinaliza a ambiguidade da contagem — §14 INC-05) |
| CAN-LAC-006 | Estratégia de claim/lock das linhas da outbox (inclusive em restart/crash) | `[09:12]` Diego | Q(interna) | Q#4/16 | Q#6/8 | ADR-002 | NÃO APLICÁVEL AO PRD (implementação); presente em RFC/FDD |
| CAN-LAC-014 | Tamanho do batch de polling e concorrência interna | `[09:08]` Diego | — | Q#5/6 | Q#9/10 | ADR-002 | NÃO APLICÁVEL AO PRD; presente em RFC/FDD |
| CAN-RNF-014 | Ordenação por pedido quando um evento anterior entra em retry | `[09:12]` Diego | — | Q#7 | Q#11/12 | ADR-002 | PRESENTE EM TODOS OS DESTINOS NECESSÁRIOS |
| FDD (Opção A/B) | Um evento por endpoint (A) ou um evento lógico com múltiplas deliveries (B) | ANALISE (derivação do FDD) | — | — | Q#2/Distinção evento×entrega | ADR-005 | NÃO APLICÁVEL AO PRD; presente e explícito no FDD |
| CAN-RF-009 / ADR-005 | O replay preserva o `event_id`, gera novo, ou cria evento correlacionado | `[09:18]` Diego | Q#12 | Q#8 | Q#13/Identidade de evento | ADR-005 | PRESENTE EM TODOS OS DESTINOS NECESSÁRIOS |
| CAN-LAC-003 | Geração da secret (algoritmo, entropia, tamanho, encoding) | `[09:46]` Sofia | Q#4/5 | Q#9 | Q#14 | ADR-004 | PRESENTE EM TODOS OS DESTINOS NECESSÁRIOS |
| CAN-LAC-004 | Armazenamento da secret em repouso (cifrado?) | ANALISE | Q#4 | Q#10 | Q#15 | ADR-004 | PRESENTE EM TODOS OS DESTINOS NECESSÁRIOS |
| CAN-LAC-003/004 | A secret é exibida uma única vez / pode ser recuperada | `[09:46]` Sofia | Q#3 | Cadastro de webhook (aberto) | Q#16 | ADR-004 | PRESENTE EM TODOS OS DESTINOS NECESSÁRIOS |
| CAN-INT-010 | Formato/encoding/prefixo da assinatura `X-Signature`; canonicalização do corpo | `[09:20]` Sofia | — | Q#11 | Q#17 | ADR-004 | NÃO APLICÁVEL AO PRD; presente em RFC/FDD |
| CAN-RISK-009 | Uso do timestamp na assinatura; política anti-replay | `[09:44]` Diego | — | Q#12 | Q#18/19 | ADR-004 | NÃO APLICÁVEL AO PRD; presente em RFC/FDD |
| CAN-INT-008 | Configurações (polling/timeout) serão constantes ou env vars | ANALISE §15 INT-08 | — | Q#13 | Q#20/Configurações | ADR-002/006 | PRESENTE (TRK-123, parcial) |
| CAN-LAC-009 | Retenção de outbox, deliveries e DLQ | `[09:08]` Diego | Q#14 | Q#14 | Q#21 | ADR-003 | PRESENTE EM TODOS OS DESTINOS NECESSÁRIOS |
| CAN-LAC-010 | Supervisão do worker em produção | `[09:11]` Diego | R-06 | Q#15 | Q#22 | ADR-002 | PRESENTE EM TODOS OS DESTINOS NECESSÁRIOS |
| CAN-LAC-013 | Cliente HTTP a ser usado pelo worker | ANALISE (auditoria) | — | Q#17 | Q#23 | ADR-002/003 | NÃO APLICÁVEL AO PRD; presente em RFC/FDD |
| — (aberto no doc) | Tratamento de endpoint removido/alterado com eventos pendentes | ANALISE | R-16/Q#7/19 | Remoção de webhook (aberto) | Q#24/25 | ADR-003 | PRESENTE EM TODOS OS DESTINOS NECESSÁRIOS |
| CAN-LAC-008 | Paginação/parâmetros de deliveries além de "últimas 100" | `[09:34]` Marcos | Q#8/9 | Q#20 | Q#29 | — | PRESENTE EM TODOS OS DESTINOS NECESSÁRIOS |
| CAN-OPEN-003 | Rate limiting de saída | `[09:38]` Diego | Não objetivos | Q#18 | — | — | NÃO APLICÁVEL A ADR; presente em PRD e RFC (TRK-105) |
| — (aberto no doc) | Visibilidade da DLQ para o cliente | ANALISE | Q#13/22 | — | — | ADR-003 | AUSENTE EM DOCUMENTO RELEVANTE (só no PRD; RFC/FDD não retomam) — pendência de baixa gravidade |
| — (aberto no doc) | Caminho da rota de rotação de secret | `[09:21]` Sofia | Q#2 | Endpoints (em aberto) | Q#28 | ADR-004 | PRESENTE EM TODOS OS DESTINOS NECESSÁRIOS |
| CAN-OPEN-002 | Nome do arquivo do worker (`webhook.worker.ts`/`webhook.processor.ts`) | `[09:28]` Bruno | — | — | Q(organização) | ADR-006 | NÃO APLICÁVEL AO PRD; presente no FDD (TRK-104) |

**Verificação:** nenhuma das questões acima é respondida por este Tracker. Todas permanecem abertas nos documentos, como deve ser. Duas observações de completude: (a) a ambiguidade "cinco tentativas" está aberta em RFC/FDD/ADR-003 mas **não é sinalizada** no PRD (§14 INC-05); (b) a "visibilidade da DLQ pelo cliente" aparece só no PRD e não é retomada em RFC/FDD — ambas de baixa gravidade e sem contradição de valor.

---

## 12. Itens parcialmente cobertos

| ID do Tracker | Item | Documento incompleto ou ausente | Problema | Correção documental necessária |
|---|---|---|---|---|
| TRK-009 | Engenharia e operações como público que opera o worker | `MATRIZ-FATOS.md` (sem ID de PÚBLICO) | Stakeholder listado no PRD (Públicos e stakeholders) sem ID canônico `CAN-PUB-*`; rastreabilidade derivada de CAN-RNF-011 | Acrescentar `CAN-PUB-005` (ou equivalente) à matriz para "Engenharia e operações" e vincular ao PRD |
| TRK-010 | Segurança como público que revisa antes do lançamento | `MATRIZ-FATOS.md` (sem ID de PÚBLICO) | Stakeholder do PRD sustentado por `[09:46]` Sofia, mas com canônico apenas métrico (CAN-MET-012), sem PÚBLICO | Acrescentar `CAN-PUB-006` (ou equivalente) para "Segurança" na matriz |
| TRK-076 | Webhooks inbound fora de escopo | `MATRIZ-FATOS.md` (sem `CAN-FORA`) | Item de escopo publicado em PRD (Fora de escopo) e RFC (Não objetivos), sustentado por `[09:02]` Marcos, sem ID canônico | Registrar `CAN-FORA-006` (ou equivalente) "webhooks inbound" na matriz |
| TRK-123 | Configuração de polling/timeout: constante × env var | RFC (Q#13) / FDD (Configurações) | Decisão de configuração **em aberto**; nenhuma env nova foi decidida na reunião (TRK-113) | Nenhuma no momento — **não resolver**; registrar a decisão na matriz/FDD quando fechada |
| TRK-025 | `X-Event-Id` como UUID gerado na inserção na outbox | PRD (RNF-010) | Redação "no fluxo de entrega" é menos precisa que "na outbox/inserção" (RFC/FDD/ADR-005) | Alinhar a redação do PRD RNF-010 para "gerado na inserção na outbox" (ver §14 INC-03) |

Nenhum item foi marcado coberto apenas por semelhança de palavra: todos os `COBERTO` da §5 têm conteúdo suficiente no documento indicado.

---

## 13. Itens não cobertos

| ID canônico | Item | Destino esperado | Gravidade | Ação necessária |
|---|---|---|---|---|
| — | *(Nenhum requisito obrigatório, decisão principal de ADR, métrica sustentada, risco principal ou questão aberta publicada ficou sem rastreabilidade.)* | — | — | — |

**Resultado:** **não há itens `NÃO_COBERTO` críticos.** Todos os RF/RNF do PRD, todas as decisões principais dos seis ADRs, as métricas sustentadas, os riscos e as questões abertas publicadas estão rastreados na §5. As 14 lacunas `CAN-LAC-*` da matriz permanecem intencionalmente `NÃO PUBLICAR` e foram **preservadas como questões em aberto** nos documentos (§11) — não constituem "itens não cobertos", e sim ausências de definição corretamente mantidas abertas. Os demais apontamentos são as pendências de **baixa/média** gravidade da §12 e da §14: IDs canônicos ausentes na matriz e imprecisões de redação, que não contradiziam as fontes; e duas descrições do código existente que o código não sustentava (INC-06, PK em UUID "em todos os models"; INC-07, base de herança dos erros `WEBHOOK_*`), **ambas já corrigidas** nos documentos.

---

## 14. Inconsistências entre documentos

| ID | Tema | Documentos envolvidos | Inconsistência | Fonte correta | Correção recomendada |
|---|---|---|---|---|---|
| INC-01 | Públicos "Engenharia e operações" e "Segurança" | PRD × `MATRIZ-FATOS.md` | Ambos constam nos "Públicos e stakeholders" do PRD, mas a matriz só tem 4 IDs `CAN-PUB-*` e não cobre esses dois stakeholders | PRD / transcrição (`[09:11]` Diego; `[09:46]` Sofia) | Acrescentar `CAN-PUB-005`/`CAN-PUB-006` à matriz (pendência da matriz, não do PRD) — gravidade **MÉDIA** |
| INC-02 | Webhooks inbound fora de escopo | PRD × RFC × `MATRIZ-FATOS.md` | "A plataforma não recebe webhooks" (`[09:02]` Marcos) está em PRD e RFC, mas sem `CAN-FORA` na matriz | PRD / RFC / transcrição | Registrar item de escopo na matriz — gravidade **BAIXA** |
| INC-03 | Momento de geração do `X-Event-Id` | PRD (RNF-010) × RFC × FDD × ADR-005 | PRD diz "gerado quando o evento entra no **fluxo de entrega**"; RFC/FDD/ADR-005/CAN-RF-015 dizem "gerado na **inserção na outbox**" (lado da produção). Como o ID **precede** qualquer tentativa, a formulação do PRD é imprecisa | `TRANSCRICAO.md` `[09:25]` Diego / CAN-RF-015 | Ajustar o PRD RNF-010 para "gerado na inserção na outbox" — gravidade **BAIXA** |
| INC-04 | Nomes dos arquivos dos documentos | Processo × repositório | Instruções de processo citam nomes longos (`PRD-webhooks-notificacao-pedidos.md`, etc.); os arquivos reais são `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`. **Os links internos entre os documentos usam os nomes reais e resolvem** (nenhum link quebrado) | Repositório (nomes reais) | Observação apenas; nenhuma correção de link necessária — gravidade **BAIXA** |
| INC-05 | Sinalização da ambiguidade "cinco tentativas" | PRD × RFC × FDD × ADR-003 | RFC (§Falha), FDD (§Retry) e ADR-003 registram explicitamente que "cinco tentativas" pode significar 5 envios totais ou 5 retries e mantêm a contagem **aberta**; o PRD usa "cinco tentativas" (RNF-011/CA-017) **sem** sinalizar a mesma ambiguidade. Não é contradição de valor (todos dizem 5) | RFC/FDD/ADR-003 (contagem operacional em aberto) | Acrescentar nota cruzada no PRD apontando a ambiguidade de contagem — gravidade **BAIXA** |
| INC-06 | "PK em UUID em **todos** os models" | FDD × RFC × TRACKER × ADR-005 × ADR-006 vs. **código** | Descrito como fato de código verificado, mas `prisma/schema.prisma` tem `OrderNumberSequence` com PK `Int @id @default(1)`. A frase "Tudo é uuid" (`[09:51]` Larissa) é válida como **decisão** para as tabelas novas, não como descrição exaustiva do schema atual | `prisma/schema.prisma` (código) | **CORRIGIDO**: as afirmações passaram a dizer "models de domínio", com a exceção `OrderNumberSequence` explicitada. CAN-DEC-022 (UUID nas tabelas novas) permanece inalterada — gravidade **MÉDIA** |
| INC-07 | Base de herança dos erros `WEBHOOK_*` | FDD (§Integração, §Convenções, §Tratamento de erros) vs. **código** e transcrição `[09:29]` | O FDD mapeava `WEBHOOK_NOT_FOUND`→`NotFoundError`, `WEBHOOK_REPLAY_NOT_ALLOWED`→`ForbiddenError` e `WEBHOOK_INVALID_FILTER`→`ValidationError`. Essas classes **fixam o `errorCode`** no construtor e não o expõem como parâmetro, e `AppError.errorCode` é `readonly` — a herança emitiria `NOT_FOUND`/`FORBIDDEN`/`VALIDATION_ERROR`, anulando o prefixo `WEBHOOK_` decidido por Larissa em `[09:29]`. Só `BadRequestError`, `ConflictError` e `UnprocessableEntityError` aceitam `code`, e é por isso que `InvalidStatusTransitionError`/`InsufficientStockError` funcionam | `src/shared/errors/http-errors.ts` + `app-error.ts` (código); `[09:29]` Larissa | **CORRIGIDO**: o mapeamento passou a usar `BadRequestError`/`ConflictError` onde há `code`, e `AppError` direto para 403/404. Registrado ainda que a recusa por papel no replay produz `FORBIDDEN` via `requireRole`, não um código `WEBHOOK_*` — gravidade **MÉDIA** |

**Não** foram encontradas contradições de **valor** entre os documentos nos temas críticos: quantidade de tentativas (5), intervalos (1m/5m/30m/2h/12h), polling (2s), timeout (10s), payload máximo (64 KB), últimas 100 entregas, HMAC-SHA256, HTTPS, secret por endpoint, grace de 24h, at-least-once, `X-Event-Id`/UUID, ordenação (sem ordering global), replay (role ADMIN), três sprints e revisão de segurança são **idênticos** em PRD, RFC, FDD e ADRs. INC-01 a INC-05 são de **rastreabilidade/redação**. INC-06 e INC-07 eram de natureza distinta — **descrições do código existente que o código não sustentava** —, ambas já corrigidas nos documentos; nenhuma delas alterava uma decisão da reunião.

### 14.1 Verificação de links

Validados os links relativos entre PRD, RFC, FDD, os seis ADRs e este Tracker:

- **PRD → RFC/FDD/ADRs** (`./RFC.md`, `./FDD.md`, `./adrs/ADR-00X-...md`): resolvem. ✔
- **RFC → ADRs** e **RFC → PRD/FDD**: resolvem. ✔
- **FDD → RFC e ADRs**: resolvem. ✔
- **ADRs → ADRs irmãos** (`./ADR-00X-...md`) e **ADRs → código**: resolvem; os seis arquivos existem em `docs/adrs/`. ✔
- **Tracker → PRD/RFC/FDD/ADRs/matriz/transcrição**: usados nos nomes reais dos arquivos. ✔

**Nenhum link quebrado.** Não foi inventado nome de arquivo para corrigir link algum; os nomes reais foram verificados no repositório (§15). A única observação de nomenclatura é INC-04 (nomes curtos `PRD.md`/`RFC.md`/`FDD.md` × nomes longos citados em instruções de processo), sem impacto sobre os links.

---

## 15. Referências de código auditadas

> Para cada evidência: (1) arquivo existe; (2) símbolo existe; (3) comportamento confere; (4) documentos que a citam; (5) nada de integração futura apresentado como comportamento atual. Os arquivos abaixo foram **reabertos e verificados diretamente** nesta auditoria.

| Caminho real | Símbolo | Comportamento confirmado | Citado em | Status |
|---|---|---|---|---|
| `src/modules/orders/order.service.ts` | `OrderService.changeStatus`; `TxClient = Prisma.TransactionClient` | `$transaction` com `findUnique`, `canTransition`, débito/reposição de estoque, `tx.order.update`, `tx.orderStatusHistory.create`. **Não** cria eventos nem publica webhook | PRD, RFC, FDD, ADR-001/005/006 | ✔ Verificado |
| `src/modules/orders/order.status.ts` | `canTransition`, `shouldDebitStock`, `shouldReplenishStock` | Máquina de transições `PENDING→PAID→PROCESSING→SHIPPED→DELIVERED` + `CANCELLED`; débito em `PENDING→PAID`; reposição ao cancelar de `PAID`/`PROCESSING` | RFC, FDD, ADR-001 | ✔ Verificado |
| `src/shared/logger/index.ts` | `redactPaths`, `createLogger` | `redact` cobre `authorization`, `cookie`, `*.password`, `*.passwordHash`, `*.token`, `*.accessToken`; **`secret` e assinatura ausentes** | RFC, FDD, ADR-004/006 | ✔ Verificado |
| `src/middlewares/auth.middleware.ts` | `authenticate`, `requireRole`, `AuthUser` | JWT bearer; `req.user` = `{id,email,role}`; `requireRole` emite `ForbiddenError` por papel; papéis `'ADMIN' \| 'OPERATOR'` | RFC, FDD, ADR-003/006 | ✔ Verificado |
| `src/modules/users/user.routes.ts` | `buildUserRouter` | `GET /:id` com `authenticate` + `requireRole('ADMIN')` — única rota real protegida por papel; precedente do replay | RFC, FDD, ADR-003/006 | ✔ Verificado |
| `prisma/schema.prisma` | `datasource db`, `enum OrderStatus`, models | Provider `mysql`; models `User/Customer/Product/Order/OrderItem/OrderStatusHistory/OrderNumberSequence`; PKs `@default(uuid()) @db.Char(36)` nos seis models de domínio — `OrderNumberSequence` é a exceção, com PK `Int @id @default(1)`. **Nenhum** model de webhook/outbox/DLQ/deliveries | RFC, FDD, ADR-001/006 | ✔ Verificado |
| `package.json` | `scripts`, `dependencies`, `engines` | `engines.node >=20`; `uuid` 11.0.3; **sem** lib de HMAC (`node:crypto` builtin); scripts `dev/build/start/db:*/test/lint/format`; **sem** script `worker` | RFC, FDD, ADR-002/004/005 | ✔ Verificado |

**Artefatos futuros — confirmados como inexistentes hoje** (tratados na §5.11 com linguagem de futuro/proposto): módulo `src/modules/webhooks`, Outbox, Deliveries, DLQ, worker/`src/worker.ts`, `publishWebhookEvent`, `WebhookService`/`WebhookRepository`/`WebhookController`, schemas de webhook, erros `WEBHOOK_*`, código de HMAC/secrets, headers `X-Event-Id`/`X-Signature`/`X-Timestamp`/`X-Webhook-Id`, script `worker`. Nenhuma linha do Tracker os descreve no presente.

> **Limitação de auditoria (transparência):** os demais arquivos citados nos documentos e nas linhas de código evidência secundárias — `src/app.ts`, `src/routes/index.ts`, `src/server.ts`, `src/config/database.ts`, `src/config/env.ts`, `src/middlewares/validate.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/shared/errors/*`, `tests/setup.ts`, `tests/orders.test.ts`, `tests/helpers/factories.ts` — têm **existência confirmada** pela listagem do repositório, e seus símbolos são **corroborados de forma consistente** pelas tabelas de evidências do PRD/RFC/FDD/ADRs; nesta auditoria eles **não foram reabertos individualmente**. As sete linhas com fonte `CODIGO` de maior peso (acima) foram verificadas diretamente.

---

## 16. Verificações de qualidade

| Verificação | Resultado | Evidência ou observação |
|---|---|---|
| IDs do Tracker são únicos? | SIM | TRK-001 a TRK-125, sequência contínua, sem repetição. |
| IDs canônicos existem na matriz? | SIM | Todos conferidos em `MATRIZ-FATOS.md` §2; 3 linhas usam `AUSENTE NA MATRIZ` (TRK-009, TRK-010, TRK-076), marcadas parciais. |
| Todos os RFs do PRD foram rastreados? | SIM | RF-001 a RF-018 mapeados na §10.1. |
| Todos os RNFs do PRD foram rastreados? | SIM | RNF-001 a RNF-018 mapeados na §10.2. |
| Todas as decisões principais dos ADRs foram rastreadas? | SIM | §9: ADR-001 a ADR-006 com decisão principal, alternativa e consequência. |
| Alternativas principais foram rastreadas? | SIM | 11 alternativas (TRK-065 a TRK-075), incluindo síncrono, Redis, trigger, 3 tentativas, indefinido, secret global (contraponto no ADR-004), exactly-once, múltiplos workers. |
| Métricas possuem fonte? | SIM | 12 métricas (TRK-082 a TRK-093), todas `TRANSCRICAO` com `[hh:mm]` + participante. |
| Riscos principais foram rastreados? | SIM | 9 riscos (TRK-094 a TRK-102). |
| Questões abertas foram rastreadas? | SIM | §5.9 (3 canônicas) + §11 (tabela dedicada, ~24 questões preservadas). |
| Existem ao menos 5 evidências de código? | SIM | 10 (TRK-106 a TRK-115). |
| A transcrição representa ≥70% das linhas? | SIM | 104/125 = 83,2%. |
| A cobertura geral atinge ≥80%? | SIM | Integral 96,8%; ponderada 98,4%. |
| Existem itens críticos não cobertos? | NÃO | §13 vazia de itens críticos. |
| Há linhas não atômicas? | NÃO | Uma afirmação por linha; segurança/retry/at-least-once/worker separados conforme §12 do enunciado. |
| Há duplicidades? | NÃO | Facetas duplicadas da matriz (RF/RNF/DEC/MET do mesmo fato) consolidadas em uma linha, com espelhos em Observações. |
| Há fontes incorretas? | NÃO | Só `TRANSCRICAO`/`CODIGO`/`ANALISE`; ADR/PRD/RFC/FDD não usados como fonte primária. |
| Há timestamps sem participante? | NÃO | Todas as linhas `TRANSCRICAO` trazem `[hh:mm]` + nome. |
| Há caminhos inexistentes? | NÃO | Caminhos `CODIGO` verificados (§15). |
| Há símbolos inexistentes? | NÃO | Símbolos das 7 evidências principais verificados diretamente. |
| Há links quebrados? | NÃO | §14.1. |
| Há artefatos futuros como existentes? | NÃO | §5.11 e §15 usam linguagem de futuro/proposto. |
| Há lacunas resolvidas silenciosamente? | NÃO | As 14 `CAN-LAC-*` permanecem abertas (§11); nenhuma política inventada. |
| Há políticas HTTP inventadas? | NÃO | Só o timeout de 10s está fechado; 2xx/3xx/4xx/5xx/DNS/TLS permanecem em aberto. |
| Polling está corretamente descrito? | SIM | TRK-083/084: 2s de intervalo e até ~2s de espera **na descoberta**, não latência mínima total. |
| Ordering está corretamente limitado? | SIM | TRK-034 (sem ordering global) e TRK-035 (intenção por `order_id`, não garantia). |
| UUID está corretamente classificado? | SIM | TRK-025/060: UUID é o formato decidido; biblioteca é detalhe de implementação (TRK-125). |
| At-least-once está corretamente descrito? | SIM | TRK-053/095: duplicidades possíveis; nenhuma promessa de entrega inevitável. |
| HMAC está corretamente limitado? | SIM | TRK-051: autenticidade/integridade; **não** confidencialidade nem anti-replay isolado. |
| Os documentos estão prontos para auditoria final? | PARCIALMENTE | Estruturalmente prontos; restam as pendências corrigíveis de §12/§14 (IDs ausentes na matriz e redações). |

---

## 17. Resultado final

### 17.1 Resumo quantitativo

| Métrica | Resultado | Critério | Situação |
|---|---|---|---|
| Total de linhas do Tracker | 125 | — | — |
| Cobertura integral geral | 96,8% (121/125) | ≥ 80% | ATENDE |
| Cobertura ponderada geral | 98,4% ((121 + 0,5×4)/125) | — | ATENDE |
| Itens parcialmente cobertos | 4 (TRK-009/010/076/123) | — | ATENDE |
| Itens não cobertos | 0 | 0 crítico | ATENDE |
| Linhas com fonte `TRANSCRICAO` | 104 (83,2%) | ≥ 70% | ATENDE |
| Linhas com fonte `CODIGO` | 10 | ≥ 5 | ATENDE |
| Linhas com fonte `ANALISE` | 11 (8,8%) | — | ATENDE |
| Requisitos funcionais rastreados (PRD) | 18/18 | todos | ATENDE |
| Requisitos não funcionais rastreados (PRD) | 18/18 | todos | ATENDE |
| Decisões principais dos ADRs rastreadas | 6/6 | todas | ATENDE |
| Questões abertas rastreadas | ✔ (§5.9 + §11) | todas as relevantes | ATENDE |
| Inconsistências registradas | 5 (todas BAIXA/MÉDIA) | sem contradição material | ATENDE |
| Links quebrados | 0 | 0 | ATENDE |
| Referências de código inválidas | 0 | 0 | ATENDE |
| IDs canônicos ausentes | 3 usos de `AUSENTE NA MATRIZ` | — | PRECISA DE REVISÃO (matriz) |

**Total por categoria:** PROBLEMA 4 · PÚBLICO 6 · REQUISITO_FUNCIONAL 17 · REQUISITO_NÃO_FUNCIONAL 14 · DECISÃO 23 · ALTERNATIVA_DESCARTADA 11 · ITEM_FORA_DE_ESCOPO 2 · ITEM_ADIADO 4 · MÉTRICA 12 · RISCO 9 · QUESTÃO_ABERTA 3 · EVIDÊNCIA_DE_CÓDIGO 10 · INTEGRAÇÃO_FUTURA 10 = **125**.

**Total por fonte:** TRANSCRICAO 104 · CODIGO 10 · ANALISE 11 = **125**.

### 17.2 Pendências documentais

Correções necessárias nos documentos anteriores (registradas, **não** aplicadas por este Tracker):

1. **`MATRIZ-FATOS.md`** — acrescentar IDs de PÚBLICO para "Engenharia e operações" e "Segurança" (INC-01) e um ID `CAN-FORA` para "webhooks inbound" (INC-02).
2. **PRD** — ajustar a redação de RNF-010 para "gerado na inserção na outbox" (INC-03); acrescentar nota cruzada sobre a ambiguidade da contagem "cinco tentativas" (INC-05); opcionalmente retomar a "visibilidade da DLQ pelo cliente" em RFC/FDD (§11).
3. **Processo/nomenclatura** — alinhar as referências de nome de arquivo (`PRD.md`/`RFC.md`/`FDD.md`) usadas em instruções externas; os links internos já estão corretos (INC-04).
4. **Decisão futura (não resolver agora)** — CAN-INT-008: definir se polling/timeout serão constantes ou env vars, e registrar quando fechado (TRK-123).

Nenhuma dessas pendências contradiz a transcrição ou o código; todas são corrigíveis sem alterar decisões.

### 17.3 Veredito

> ## `PASSO 10 APROVADO COM CORREÇÕES`

**Justificativa objetiva:** o Tracker está estruturalmente correto e completo — 125 linhas atômicas, IDs únicos e contínuos, fontes válidas com localização verificável, e **cobertura suficiente** (integral 96,8%, ponderada 98,4%, acima do mínimo de 80%). Todos os RFs e RNFs do PRD estão rastreados, as decisões principais dos seis ADRs estão cobertas com alternativas e consequências, a transcrição representa 83,2% das linhas (≥70%) e há 10 evidências de código (≥5). **Não** há itens críticos não cobertos, **nem** contradições materiais de valor entre PRD, RFC, FDD e ADRs, **nem** artefatos futuros apresentados como existentes, **nem** lacunas resolvidas silenciosamente, **nem** percentuais manipulados.

Restam **pendências documentais corrigíveis** — três IDs canônicos ausentes na matriz (dois públicos e um item de escopo), uma imprecisão de redação no PRD (RNF-010) e uma nota de ambiguidade a acrescentar (cinco tentativas) — todas de gravidade baixa/média e sem impacto sobre as decisões. Por existirem correções pendentes, ainda que menores, o resultado é **aprovado com correções**, e não aprovação plena.

---

> **Notas finais.** Este Tracker não resolve nenhuma questão em aberto, não introduz requisitos ou decisões, não altera fatos para elevar percentuais e não aplica automaticamente as correções apontadas — que devem ser tratadas no PRD, RFC, FDD, ADRs e `MATRIZ-FATOS.md` pelas pessoas responsáveis. Todas as métricas são reproduzíveis a partir das tabelas das §5 a §8.
