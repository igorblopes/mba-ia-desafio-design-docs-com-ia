# Da Reunião ao Documento: Design Docs Gerados por IA

Pacote de documentação técnica derivado de uma transcrição de reunião e do código
existente de um Order Management System (OMS), produzido com apoio de inteligência
artificial e submetido a auditoria documental antes da entrega.

> Este `README.md` substitui o enunciado do desafio, que ocupava o arquivo original. O
> enunciado permanece disponível no repositório base do curso:
> <https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia>.

---

## Sobre o desafio

O ponto de partida foi uma **transcrição de reunião técnica** (`TRANSCRICAO.md`): cinco
participantes — tech lead, PM, dois engenheiros e uma engenheira de segurança — decidem,
em ~55 minutos, como construir uma feature de **webhooks outbound para notificação de
mudanças de status de pedidos**. Nada além da transcrição foi registrado.

O repositório já continha um **Order Management System** funcional em Node.js + TypeScript
(módulos de autenticação, usuários, clientes, produtos e pedidos; MySQL via Prisma; máquina
de estados de pedido com controle transacional de estoque e auditoria de mudanças de status).
A aplicação **não** possui qualquer mecanismo de notificação externa — é exatamente esse
vácuo que a feature discutida pretende preencher.

A tarefa era **produzir design documents** derivados da reunião e do código, em nível
acionável para o time iniciar a implementação. Duas restrições eram absolutas:

- **nenhuma linha de código de aplicação deveria ser implementada ou alterada** — a entrega
  é puramente documental;
- **toda informação registrada deveria ser rastreável** à transcrição ou ao código, sem
  requisitos, decisões ou restrições inventados.

## Objetivo da entrega

A entrega busca demonstrar, na prática, um fluxo de documentação assistida por IA com
disciplina de rastreabilidade:

- **extração estruturada** de informações a partir de uma reunião;
- **distinção explícita** entre fato, decisão fechada, proposta e lacuna em aberto;
- **análise do código existente** para ancorar os pontos de integração;
- **geração assistida por IA** dos documentos finais;
- **revisão humana** crítica de cada saída;
- **rastreabilidade** de cada item à sua origem;
- **consistência** entre os documentos;
- **preservação integral do código original**.

## Resultado

O pacote entregue é composto pelos documentos abaixo. Todos os links são relativos e foram
validados (126 links de arquivo resolvem para arquivos existentes).

| Documento | Finalidade | Link |
| --- | --- | --- |
| PRD | Problema, usuários, escopo, requisitos e métricas | [docs/PRD.md](docs/PRD.md) |
| RFC | Proposta técnica integrada (arquitetura, alternativas, questões em aberto) | [docs/RFC.md](docs/RFC.md) |
| FDD | Desenho detalhado da implementação futura | [docs/FDD.md](docs/FDD.md) |
| ADR-001 | Transactional Outbox no MySQL | [docs/adrs/ADR-001-outbox-no-mysql.md](docs/adrs/ADR-001-outbox-no-mysql.md) |
| ADR-002 | Worker separado com polling | [docs/adrs/ADR-002-worker-separado-com-polling.md](docs/adrs/ADR-002-worker-separado-com-polling.md) |
| ADR-003 | Retry, backoff e DLQ | [docs/adrs/ADR-003-retry-backoff-e-dlq.md](docs/adrs/ADR-003-retry-backoff-e-dlq.md) |
| ADR-004 | HMAC-SHA256 e gestão de secrets | [docs/adrs/ADR-004-hmac-sha256-e-gestao-de-secrets.md](docs/adrs/ADR-004-hmac-sha256-e-gestao-de-secrets.md) |
| ADR-005 | At-least-once e Event ID | [docs/adrs/ADR-005-entrega-at-least-once-com-event-id.md](docs/adrs/ADR-005-entrega-at-least-once-com-event-id.md) |
| ADR-006 | Reuso dos padrões existentes | [docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md](docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| Tracker | Rastreabilidade entre fontes e documentos | [docs/TRACKER.md](docs/TRACKER.md) |
| Análise de evidências | Levantamento inicial (artefato de processo) | [ANALISE-EVIDENCIAS.md](ANALISE-EVIDENCIAS.md) |
| Matriz de fatos | Base canônica de fatos atômicos | [MATRIZ-FATOS.md](MATRIZ-FATOS.md) |
| Transcrição | Fonte primária da reunião | [TRANSCRICAO.md](TRANSCRICAO.md) |

## Estrutura do repositório

```text
.
├── README.md                       (este arquivo — processo da entrega)
├── TRANSCRICAO.md                  (fonte primária — não alterada)
├── ANALISE-EVIDENCIAS.md           (levantamento inicial de evidências)
├── MATRIZ-FATOS.md                 (base canônica de fatos, IDs CAN-*)
├── docs/
│   ├── PRD.md
│   ├── RFC.md
│   ├── FDD.md
│   ├── TRACKER.md
│   └── adrs/
│       ├── README.md
│       ├── ADR-001-outbox-no-mysql.md
│       ├── ADR-002-worker-separado-com-polling.md
│       ├── ADR-003-retry-backoff-e-dlq.md
│       ├── ADR-004-hmac-sha256-e-gestao-de-secrets.md
│       ├── ADR-005-entrega-at-least-once-com-event-id.md
│       └── ADR-006-reuso-dos-padroes-do-projeto.md
├── src/                            (aplicação OMS — não alterada)
├── prisma/                         (schema/migrations — não alterados)
├── tests/                          (testes — não alterados)
└── ... (package.json, configs e demais arquivos do boilerplate)
```

> Os nomes reais dos documentos são `PRD.md`, `RFC.md`, `FDD.md` e `TRACKER.md` (sem sufixo
> de feature). A árvore acima reflete exatamente o que existe no repositório.

## Navegação pelos documentos

**Ordem de leitura sugerida:** PRD → RFC → ADRs → FDD → Tracker. Comece pelo *porquê*
(PRD), passe pela *proposta* (RFC) e pelas *decisões fechadas* (ADRs), desça ao *como*
(FDD) e termine pela *rastreabilidade* (Tracker).

### Documentos principais

- [PRD — Product Requirement Document](docs/PRD.md)
- [RFC — Request for Comments](docs/RFC.md)
- [FDD — Feature Design Document](docs/FDD.md)
- [Tracker de Rastreabilidade](docs/TRACKER.md)

### Architecture Decision Records

- [ADR-001 — Transactional Outbox no MySQL](docs/adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker separado com polling](docs/adrs/ADR-002-worker-separado-com-polling.md)
- [ADR-003 — Retry com backoff e Dead-Letter Queue](docs/adrs/ADR-003-retry-backoff-e-dlq.md)
- [ADR-004 — HMAC-SHA256 e gestão de secrets por endpoint](docs/adrs/ADR-004-hmac-sha256-e-gestao-de-secrets.md)
- [ADR-005 — Entrega at-least-once com identificação estável de evento](docs/adrs/ADR-005-entrega-at-least-once-com-event-id.md)
- [ADR-006 — Reuso dos padrões arquiteturais existentes](docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md)

### Artefatos auxiliares

- [Transcrição da reunião](TRANSCRICAO.md) — fonte primária.
- [Análise de evidências](ANALISE-EVIDENCIAS.md) — levantamento inicial com estados de maturidade.
- [Matriz canônica de fatos](MATRIZ-FATOS.md) — fatos atômicos com IDs `CAN-*`, categoria, estado, origem e destino documental.

## Visão geral da feature

Resumo de alto nível do que a feature **propõe** construir (nada disso existe hoje no código):

- clientes B2B hoje dependem de **polling** contra `GET /orders` para saber se um pedido mudou;
- a proposta permite **cadastrar endpoints HTTPS** por cliente, com filtro dos status de interesse;
- cada **mudança de status** de pedido gera um evento;
- os eventos serão **persistidos por um Transactional Outbox**, na mesma transação da mudança de status;
- um **worker em processo separado** fará a leitura da outbox e as entregas HTTP, fora da transação;
- haverá **retry com backoff, DLQ e replay** administrativo para falhas;
- as requisições serão **assinadas com HMAC-SHA256**, com secret por endpoint e rotação com grace period;
- a semântica de entrega será **at-least-once**;
- duplicidades serão **identificáveis pelo consumidor** através do header `X-Event-Id`.

O detalhamento (modelos, contratos, fluxos, integração com o código) está no
[FDD](docs/FDD.md) e no [RFC](docs/RFC.md); este README não desce a esse nível.

## Processo utilizado

O trabalho seguiu onze passos. A ordem de produção dos documentos (ADRs → RFC → FDD → PRD →
Tracker) segue a recomendação do enunciado: fechar as decisões antes de consolidar a proposta
e detalhar a implementação.

### Passo 1 — Preparação do repositório

Fork do repositório base do curso e clone local; leitura do enunciado (que ocupava o
`README.md` original); inspeção inicial da aplicação OMS e da transcrição. Ficou definido
desde o início que **nenhum código seria implementado**. O fork e o clone estão evidenciados
pelo repositório local e pelo `remote origin`; a **publicação (push) da entrega final
permanece pendente** (ver *Estado final*).

### Passo 2 — Análise inicial

Leitura dirigida da transcrição para identificar o problema, os públicos, os requisitos
funcionais e não funcionais, as decisões fechadas, as alternativas descartadas, as métricas,
os riscos e as questões deixadas em aberto — separando explicitamente o que **entra** do que
foi **descartado** ou **adiado**.

### Passo 3 — Extração de evidências

Produção de [`ANALISE-EVIDENCIAS.md`](ANALISE-EVIDENCIAS.md): cada item recebeu **timestamp**
da fala de origem (`[hh:mm] Nome`), **estado de maturidade** (`DECIDIDO`, `PROPOSTO`,
`DESCARTADO`, `ADIADO`, `EM_ABERTO`, `LACUNA`, etc.) e **separação entre fato e inferência**.
Nesse passo também foram levantados os pontos do código que a feature tocaria.

### Passo 4 — Auditoria do código existente

Mapeamento dos ganchos reais no código, no tempo verbal do **presente** (o que já existe):
a transação de mudança de status em `src/modules/orders/order.service.ts`, a arquitetura
modular (`controller`/`service`/`repository`/`routes`/`schemas`), a autenticação e a
autorização por role (`src/middlewares/auth.middleware.ts`), a validação com Zod, a classe
`AppError` e as classes específicas, o middleware de erro centralizado, o logger Pino, o
`PrismaClient` e os testes existentes.

### Passo 5 — Construção da matriz canônica

Consolidação em [`MATRIZ-FATOS.md`](MATRIZ-FATOS.md): fatos **atômicos**, cada um com **ID
canônico** (`CAN-RF-*`, `CAN-DEC-*`, `CAN-MET-*`, `CAN-RISK-*`, `CAN-LAC-*`, …), **categoria**,
**estado**, **origem** (timestamp ou caminho de código) e **destino documental** (quais
documentos devem consumir o fato). A matriz passou a ser a fonte única normalizada que
alimenta PRD, RFC, FDD, ADRs e Tracker.

### Passo 6 — Produção dos ADRs

Seis ADRs, cada um com contexto, decisão, alternativas consideradas, consequências
(positivas e negativas), riscos e lacunas explicitamente marcadas como abertas. Ao final,
uma **auditoria cruzada** garantiu que os ADRs não se contradizem entre si e que cada
decisão aponta para os fatos canônicos que a sustentam.

### Passo 7 — Produção do RFC

Consolidação da solução ponta a ponta em [`docs/RFC.md`](docs/RFC.md): visão geral,
componentes, fluxos, segurança, tratamento de falhas, riscos e — de forma destacada — as
**questões em aberto** (20 pontos numerados) e as **alternativas descartadas** com o
trade-off que motivou o descarte. O RFC referencia os ADRs correspondentes.

### Passo 8 — Produção do FDD

Desenho técnico em [`docs/FDD.md`](docs/FDD.md): estrutura futura proposta, responsabilidades,
contratos públicos (endpoints, payloads, headers, status codes), persistência conceitual,
integração transacional, worker, matriz de erros `WEBHOOK_*` e estratégia de testes. Inclui a
seção obrigatória **"Integração com o sistema existente"**, que nomeia caminhos reais do código.

### Passo 9 — Produção do PRD

Consolidação de alto nível em [`docs/PRD.md`](docs/PRD.md): problema do cliente, públicos,
requisitos (18 RF + 13 RNF), critérios de aceite, métricas, plano de lançamento e riscos com
mitigação. Produzido depois dos documentos técnicos, como recomenda o enunciado.

### Passo 10 — Construção do Tracker

[`docs/TRACKER.md`](docs/TRACKER.md) liga cada item de cada documento à sua origem (transcrição
com timestamp, ou código com caminho de arquivo) e ao ID canônico da matriz. Registra a
cobertura por documento, por fonte e por categoria, além das inconsistências e das questões
abertas — e é honesto sobre as próprias pendências.

### Passo 11 — Auditoria e publicação

Auditoria documental final (este passo): revisão de links, de Markdown e de consistência de
valores; busca por afirmações indevidas; verificação de que **nenhum código foi alterado** e
de que **não há dados sensíveis**; e preparação do repositório para publicação. Os resultados
estão nas seções *Verificações realizadas* e *Estado final*.

## Uso de inteligência artificial

A IA foi a ferramenta principal de produção — para **ler o código, analisar a transcrição,
estruturar e redigir** os documentos e apoiar a **auditoria**. Ela **não** foi tratada como
fonte de verdade: toda saída foi confrontada com a transcrição, o código, a matriz canônica e
os demais documentos, e **revisada por uma pessoa** antes de ser aceita. As lacunas foram
deliberadamente **preservadas como questões em aberto** em vez de "preenchidas" pela IA.

### Ferramentas utilizadas

| Ferramenta | Finalidade |
| --- | --- |
| Claude / Claude Code | Extração, estruturação, redação e auditoria assistida dos documentos |
| Git | Controle de versões e verificação de mudanças (`git diff`, `git status`) |
| GitHub | Hospedagem do fork do repositório base (publicação da entrega final pendente) |
| Terminal (PowerShell / Git Bash) | Inspeção do repositório e execução das validações |
| `grep` / `ripgrep` | Busca de termos, requisitos, IDs e inconsistências |
| Editor de texto | Leitura e edição dos documentos Markdown |
| Mermaid | Um diagrama de fluxo no FDD |

Não foram usados ChatGPT, Cursor, Copilot ou Gemini nesta entrega, nem scripts automáticos de
geração — por isso não aparecem acima.

### Estratégia de prompting

Os prompts seguiram uma estrutura consistente, pensada para reduzir alucinação:

1. **papel da IA** (arquiteto/analista/auditor, conforme o passo);
2. **delimitação da tarefa** (um documento ou um ADR por vez);
3. **indicação das fontes** (transcrição, código, matriz);
4. **hierarquia de autoridade** entre as fontes;
5. **estrutura obrigatória** de saída;
6. **decisões que deveriam ser preservadas** exatamente como na fonte;
7. **lacunas que deveriam permanecer abertas**;
8. **afirmações proibidas** (o que a IA não podia concluir sozinha);
9. **checklist de auditoria**;
10. **formato de saída** (Markdown, sem metadados de ferramenta).

Foram usados prompts específicos para: análise inicial, auditoria do código, construção da
matriz, cada ADR, auditoria cruzada dos ADRs, RFC, FDD, PRD, Tracker e auditoria final.

Dois exemplos representativos (reduzidos) da estrutura utilizada:

**Prompt de produção de um ADR:**

```text
Papel: arquiteto de software registrando uma decisão já fechada na reunião.
Fonte de verdade, nesta ordem: TRANSCRICAO.md > código > MATRIZ-FATOS.md.
Tarefa: escrever ADR-00X sobre <decisão>, no formato MADR
  (Status, Contexto, Decisão, Alternativas, Consequências).
Regras:
- Cada afirmação deve citar o timestamp da fala ou o caminho do código que a sustenta.
- Registre as alternativas REALMENTE discutidas e o trade-off que levou ao descarte.
- O que não foi decidido na reunião deve aparecer como "questão em aberto", nunca resolvido.
- Proibido afirmar: ordering global garantido; exactly-once; que HMAC cifra/impede replay;
  que um pacote/biblioteca específico será usado; política de erros HTTP; estratégia de lock.
- Nada que a feature ainda vai criar pode ser descrito no presente.
Saída: Markdown puro, sem blocos com atributos de ferramenta.
```

**Prompt de auditoria de um documento:**

```text
Papel: auditor de documentação, cético.
Tarefa: revisar docs/<DOC>.md contra TRANSCRICAO.md, o código e MATRIZ-FATOS.md.
Aponte, com localização exata:
- afirmações sem fonte identificável;
- valores divergentes entre documentos (ex.: polling 2s, timeout 10s, meta <10s, 64KB, 24h);
- artefatos futuros escritos no presente;
- questões abertas que foram indevidamente "resolvidas";
- links quebrados e IDs canônicos inexistentes.
Não corrija reescrevendo: liste os problemas para revisão humana.
```

### Processo de revisão

Cada saída da IA foi revisada contra: a transcrição, o código, a matriz canônica, os ADRs, os
documentos anteriores e o Tracker — além de buscas por termos problemáticos, revisão de links,
revisão do **tempo verbal** (presente para o que existe, futuro para o que será criado) e
verificação da separação entre o que **existe** e o que é **proposto**.

### Cuidados contra alucinações

Mecanismos empregados para manter a documentação ancorada:

- **ordem de autoridade** das fontes (transcrição > código > matriz > documentos);
- **exigência de timestamps** em cada fato da transcrição;
- **exigência de caminhos e símbolos reais** para cada referência de código;
- **matriz canônica** como fonte única normalizada;
- **proibição de decisões sem fonte** identificável;
- **classificação explícita** de propostas, itens adiados e descartados;
- **preservação das questões abertas** em vez de respondê-las;
- **busca ativa por termos de risco** (afirmações fortes demais);
- **auditoria cruzada** entre ADRs e entre documentos;
- **Tracker de rastreabilidade** cobrindo item a item.

## Iterações e ajustes

### Quantas iterações principais

O resultado final não saiu de uma única passada da IA. Cada documento percorreu, no mínimo,
**três iterações principais**: (1) geração assistida a partir da matriz canônica, (2) auditoria
crítica contra a transcrição, o código e a matriz, e (3) correção dirigida pelos achados dessa
auditoria. Os artefatos mais densos — o conjunto de seis ADRs, o RFC e o FDD — exigiram uma
**quarta passada** de auditoria cruzada para eliminar contradições entre documentos (Passo 6,
no caso dos ADRs), e o pacote como um todo passou ainda por uma **auditoria final** (Passo 11).
Na prática, foram de **três a quatro iterações principais por documento** até o estado entregue.

### Onde a IA errou ou foi superficial

Os casos abaixo são os principais momentos em que a saída da IA estava **errada, forte demais
ou rasa** e precisou de correção humana. Todas as correções são verificáveis nos documentos
finais:

- **"latência mínima de 2 segundos"** foi reformulado: o polling de 2s **adiciona até ~2s na
  descoberta** do evento, não é uma latência mínima total (ver `CAN-MET-003`, PRD RNF-002,
  Tracker TRK-084).
- **single-worker não garante ordering global** — a ordem é preservada apenas por `order_id`
  enquanto houver um único worker; documentado como limitação conhecida (ADR-002).
- **UUID é formato, não biblioteca** — adotar UUID (padrão do projeto) **não** obriga o pacote
  `uuid` nem `crypto.randomUUID`; a implementação decide (ADR-005/ADR-006).
- **at-least-once permite duplicidade** e **exactly-once foi explicitamente descartado**; a
  deduplicação é responsabilidade do consumidor via `X-Event-Id` (ADR-005).
- **HMAC assina, não cifra**: não fornece confidencialidade (que depende do HTTPS) e **não
  impede replay sozinho**; `X-Event-Id` **não autentica** a entrega (ADR-004).
- **classificação de respostas HTTP** (o que é sucesso, o que é retentável) permaneceu **aberta**.
- **claim/locking, batch e concorrência** do worker permaneceram **abertos** — `SELECT ... FOR
  UPDATE` e `SKIP LOCKED` aparecem apenas como opções não decididas, nunca como decisão (FDD).
- **armazenamento da secret** e **formato da assinatura** permaneceram **abertos** (RFC Q#10/Q#11).
- **identidade do evento durante o replay** permaneceu **aberta** (RFC Q#8, ADR-005).
- **artefatos futuros** (`webhook_outbox`, `webhook_dead_letter`, `src/worker.ts`,
  `publishWebhookEvent`, erros `WEBHOOK_*`) foram escritos no **futuro**, nunca como código atual.
- **a arquitetura atual não recebeu rótulos não comprovados** — o ADR-006 nega explicitamente
  Clean Architecture, Hexagonal, Ports & Adapters, DDD, CQRS e Event Sourcing, por ausência de
  evidência no código.

## Decisões arquiteturais documentadas

| Tema | Decisão | Documento |
| --- | --- | --- |
| Persistência do evento | Transactional Outbox no MySQL | [ADR-001](docs/adrs/ADR-001-outbox-no-mysql.md) |
| Processamento | Worker em processo separado | [ADR-002](docs/adrs/ADR-002-worker-separado-com-polling.md) |
| Polling | Intervalo de dois segundos | [ADR-002](docs/adrs/ADR-002-worker-separado-com-polling.md) |
| Resiliência | Cinco tentativas, backoff e DLQ | [ADR-003](docs/adrs/ADR-003-retry-backoff-e-dlq.md) |
| Replay | Restrito a ADMIN e auditado | [ADR-003](docs/adrs/ADR-003-retry-backoff-e-dlq.md) |
| Segurança | HTTPS e HMAC-SHA256 | [ADR-004](docs/adrs/ADR-004-hmac-sha256-e-gestao-de-secrets.md) |
| Secrets | Uma por endpoint, rotação com grace de 24 horas | [ADR-004](docs/adrs/ADR-004-hmac-sha256-e-gestao-de-secrets.md) |
| Semântica de entrega | At-least-once | [ADR-005](docs/adrs/ADR-005-entrega-at-least-once-com-event-id.md) |
| Duplicidade | Identificação por `X-Event-Id` | [ADR-005](docs/adrs/ADR-005-entrega-at-least-once-com-event-id.md) |
| Integração | Reuso dos padrões existentes | [ADR-006](docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |

O detalhamento de cada decisão (contexto, alternativas, consequências) está no ADR correspondente.

## Questões que permaneceram abertas

Deliberadamente **não resolvidas** nesta entrega (lista completa e numerada no
[RFC](docs/RFC.md), seção de questões em aberto):

- `customer_id` no body ou no path dos endpoints de configuração;
- se "cinco tentativas" inclui ou não o primeiro envio;
- quais respostas HTTP contam como sucesso de entrega;
- quais falhas são retentáveis;
- estratégia de *claim* das linhas da outbox pelo worker;
- estado intermediário de processamento;
- recuperação após crash do worker no meio de um evento;
- tamanho do *batch* de polling;
- concorrência interna do worker;
- ordenação de um mesmo pedido quando um evento anterior entra em retry;
- se o replay preserva o `event_id`, gera um novo ou cria evento correlacionado;
- algoritmo/entropia de geração da secret;
- armazenamento da secret em repouso;
- formato da assinatura (encoding, prefixo, canonicalização);
- tratamento de replay attack além do `X-Timestamp`;
- retenção de outbox, deliveries e DLQ;
- supervisão do worker em produção;
- cliente HTTP a ser usado pelo worker;
- paginação do endpoint de deliveries além de "últimos 100";
- valores como constantes no código ou variáveis de ambiente.

## Rastreabilidade

O [Tracker](docs/TRACKER.md) é o instrumento central contra alucinação. Ele liga **fontes e
documentos** por meio dos IDs canônicos da matriz, cobrindo requisitos, decisões, alternativas,
riscos, métricas, questões abertas e evidências de código, com cálculo de cobertura reproduzível.

Valores efetivamente registrados no Tracker (não estimados aqui):

- **Cobertura por documento:** PRD 97,0% (98,5% ponderada); RFC 97,5%; FDD 99,1%; ADR-001 a
  ADR-006 com 100% cada.
- **Cobertura por fonte primária:** `TRANSCRICAO` 83,2% (acima do mínimo de 70%); `CODIGO` 10
  linhas (acima do mínimo de 5); `ANALISE` 8,8%.
- **Total:** 125 itens rastreados.

**Pendências declaradas pelo próprio Tracker** (baixa/média gravidade, sem contradizer as
fontes): três IDs canônicos ausentes na matriz (dois públicos e um item de "fora de escopo"),
uma imprecisão de redação no PRD RNF-010 e uma nota de ambiguidade a acrescentar sobre "cinco
tentativas". Nenhuma delas afeta as decisões; o Tracker as classifica como *corrigíveis* e
chega ao veredito de **aprovado com correções**.

## Verificações realizadas

| Verificação | Resultado | Observação |
| --- | --- | --- |
| Documentos esperados presentes | APROVADO | PRD, RFC, FDD, Tracker, 6 ADRs, transcrição, análise e matriz |
| Seis ADRs presentes | APROVADO | ADR-001 a ADR-006, todos com Status `Aceito` |
| Links relativos | APROVADO | 126 links de arquivo validados, nenhum quebrado |
| RFs rastreados | APROVADO | 18 RF no PRD, todos com linha no Tracker |
| RNFs rastreados | APROVADO | 13 RNF no PRD, todos com linha no Tracker |
| IDs do Tracker únicos | APROVADO | Nenhuma linha `TRK-*` duplicada; reaparições são referências cruzadas intencionais |
| IDs canônicos válidos | APROVADO COM RESSALVAS | 3 itens usam `AUSENTE NA MATRIZ` (2 públicos, 1 fora de escopo) — pendência documental baixa |
| Cobertura geral | APROVADO | PRD 97%, RFC 97,5%, FDD 99,1%, ADRs 100% |
| Predominância da transcrição | APROVADO | 83,2% das linhas com fonte `TRANSCRICAO` (mín. 70%) |
| Pelo menos cinco evidências de código | APROVADO | 10 linhas com fonte `CODIGO` e caminho real |
| Questões abertas preservadas | APROVADO | 20 questões numeradas mantidas em aberto no RFC |
| Artefatos futuros no tempo verbal correto | APROVADO | `webhook_outbox`, `src/worker.ts`, etc. sempre no futuro |
| Ausência de políticas inventadas | APROVADO | HTTP, claim, lock, batch, secret storage permanecem abertos |
| Código-fonte não alterado | APROVADO | `git diff -- src prisma tests package.json` retorna vazio |
| Dados sensíveis ausentes | APROVADO | Nenhum secret/token real nos documentos; `.env.example` só com placeholders |
| Markdown válido | APROVADO | Blocos de código balanceados; nenhum bloco com atributo `id=` |
| Repositório pronto para publicação | APROVADO COM RESSALVAS | Conteúdo pronto; push da entrega final ainda pendente |

## Como reproduzir a auditoria

Os comandos abaixo assumem um shell POSIX (Git Bash no Windows, WSL, macOS ou Linux),
executado na raiz do repositório. Cada ocorrência retornada deve ser **analisada em contexto**:
um termo de risco pode ser legítimo quando aparece como alternativa descartada, afirmação
negada ou questão em aberto.

Listar os documentos:

```bash
find docs -maxdepth 2 -type f | sort
```

Contar os ADRs (deve retornar 6):

```bash
find docs/adrs -maxdepth 1 -type f -name "ADR-*.md" | wc -l
```

Verificar que o código não foi alterado (deve retornar vazio):

```bash
git diff -- src prisma tests package.json
```

Listar RFs e RNFs do PRD:

```bash
grep -nE "^### (RF|RNF)-[0-9]+" docs/PRD.md
```

Procurar IDs de Tracker realmente duplicados (linhas de definição; deve retornar vazio):

```bash
grep -oE "^\| TRK-[0-9]+" docs/TRACKER.md | grep -oE "TRK-[0-9]+" | sort | uniq -d
```

> A busca simples `grep -oE "TRK-[0-9]+" docs/TRACKER.md | sort | uniq -d` retorna vários IDs
> porque eles são **referenciados** em notas e no índice de pendências; isso não indica linhas
> duplicadas. Use a versão acima, restrita ao início da linha (`^|`).

Verificar cobertura pendente:

```bash
grep -nE "NÃO_COBERTO|PARCIALMENTE_COBERTO|COBERTURA_INCONSISTENTE" docs/TRACKER.md
```

Procurar termos de risco (analisar cada ocorrência em contexto):

```bash
grep -RniE "latência mínima|single worker garante|ordenação garantida|zero perda|exactly.once garantido|SELECT FOR UPDATE|SKIP LOCKED|secret criptografada|KMS obrigatório|pacote uuid será usado|crypto.randomUUID será usado" README.md docs
```

## Preservação do código original

A restrição de não alterar o código é absoluta. O resultado **real** das verificações desta
auditoria:

- `git diff -- src prisma tests package.json` → **vazio** (nenhuma alteração de código).
- `git status --short` → apenas arquivos **documentais** modificados ou novos
  (`README.md`, `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/TRACKER.md`,
  `ANALISE-EVIDENCIAS.md`, `MATRIZ-FATOS.md` e os seis ADRs em `docs/adrs/`).

Nenhum arquivo em `src/`, `prisma/`, `tests/` ou `package.json` foi tocado. O código serviu
exclusivamente como contexto e referência.

## Limitações da entrega

Limitações registradas de forma honesta (não são defeitos ocultos):

- **nenhuma implementação foi realizada** — a entrega é puramente documental;
- **questões técnicas permanecem abertas** por decisão consciente (ver *Questões que
  permaneceram abertas*);
- a documentação representa **o estado da discussão** capturada na transcrição analisada;
- métricas sem meta explícita permanecem apenas como **indicadores**, não como metas;
- decisões futuras (classificação HTTP, claim, armazenamento de secret, supervisão do worker)
  poderão **exigir novos ADRs** quando fechadas;
- a **operação e supervisão do worker** em produção não foram definidas.

## Estado final

| Item | Situação |
| --- | --- |
| PRD | Presente |
| RFC | Presente |
| FDD | Presente |
| ADRs | 6 encontrados (ADR-001 a ADR-006) |
| Tracker | Presente; veredito interno "aprovado com correções" (pendências baixas) |
| README | Atualizado (este arquivo) |
| Links | 126 relativos validados, nenhum quebrado |
| Rastreabilidade | Cobertura 97–100%; `TRANSCRICAO` 83,2%; 10 evidências de código |
| Código original | Intacto (`git diff` vazio) |
| Dados sensíveis | Ausentes |
| Repositório público | PENDENTE DE VERIFICAÇÃO (visibilidade não confirmada nesta sessão) |
| Push final | PENDENTE (não autorizado nesta sessão) |

## Conclusão

O pacote de design docs está **estruturalmente completo e internamente consistente**: os seis
ADRs, o RFC, o FDD, o PRD e o Tracker derivam de uma matriz canônica ancorada na transcrição e
no código, com rastreabilidade alta e reproduzível. As decisões da reunião estão fielmente
registradas; as lacunas foram **preservadas como questões em aberto** em vez de inventadas; e o
código original permanece **intacto**. As pendências remanescentes são de baixa gravidade,
documentais e já declaradas pelo próprio Tracker. A publicação (commit e push) permanece como
ação manual pendente do autor.
