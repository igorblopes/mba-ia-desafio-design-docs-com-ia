# ADR-003 — Retry com backoff e Dead-Letter Queue para entregas de webhook

## Status

`Aceito`

- **Data da decisão:** não informada. `TRANSCRICAO.md` registra apenas "quinta-feira, 09:00", sem data de calendário.
- **Participantes:** Diego (Engenheiro Sênior, Plataforma), autor da proposta; Larissa (Tech Lead, conduzindo), que fechou a política em `[09:17]` ("Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h") e a exigência de role em `[09:36]`; Bruno (Engenheiro Pleno, Pedidos), que contestou o número de tentativas em `[09:15]`–`[09:16]`; Marcos (Product Manager), que aceitou a janela de recuperação em `[09:17]`; Sofia (Engenheira de Segurança), que exigiu role `ADMIN` e auditoria em `[09:36]`.
- **Escopo:** como o sistema reage a uma entrega de webhook malsucedida — retentativa, progressão das esperas, destino final das falhas permanentes e reprocessamento administrativo. Não cobre a produção do evento (ADR-001), a leitura da outbox e o polling (ADR-002), a segurança do envio (ADR-004) nem a garantia de entrega e a idempotência (ADR-005).

## Contexto

O ADR-001 decidiu persistir o evento na outbox do MySQL, na mesma transação da mudança de status (CAN-DEC-001). O ADR-002 decidiu que um worker em processo separado, com polling, processará esses eventos (CAN-DEC-003). Ambos param antes do ponto tratado aqui: a entrega depende de uma chamada HTTP a um sistema de terceiro, fora do controle da plataforma, e essa chamada pode não ser bem-sucedida.

Larissa abriu o tema em `[09:14]`: "Vamos pra retry. Se o cliente tá offline, o que a gente faz?" A indisponibilidade externa não é hipótese remota — Diego citou em `[09:16]` um cliente real com "indisponibilidade de duas horas em manutenção planejada" (CAN-RISK-001). A falha pode decorrer de indisponibilidade, lentidão ou outros problemas de comunicação; do conjunto, apenas o estouro de timeout foi qualificado na reunião como falha que enseja retentativa (`[09:42]` Diego — CAN-RNF-012).

Duas reações extremas foram rejeitadas. Descartar o evento na primeira falha desperdiçaria um evento recuperável e contrariaria o motivo de tê-lo persistido transacionalmente. Repetir indefinidamente o deixaria "pendurado pra sempre se o cliente sumiu" (`[09:15]` Diego — CAN-ALT-005), sem encerramento operacional. São necessários, portanto, um limite explícito de recuperação automática e um destino para o que o excede: falhas permanentes separadas do fluxo ativo, com contexto preservado para diagnóstico, e um caminho administrativo e auditável de reprocessamento (`[09:18]` Diego; `[09:36]` Sofia).

O repositório não oferece hoje ponto de partida: não há código de webhook, outbox, worker, DLQ ou replay, e `prisma/schema.prisma` modela apenas usuários, clientes, produtos, pedidos, itens, histórico e a sequência de numeração (CAN-COD-002).

## Drivers da decisão

- **Recuperação automática de indisponibilidades temporárias**, sem intervenção humana (`[09:15]` Diego — CAN-RF-012).
- **Limite finito do ciclo de tentativas** (`[09:15]` Diego — CAN-RNF-010).
- **Janela compatível com indisponibilidades prolongadas**: três tentativas "mataria" o evento em ~30 minutos, insuficiente diante de uma parada de duas horas (`[09:16]` Diego).
- **Outbox principal focada em eventos ativos**: "Mais limpa a leitura da outbox principal" (`[09:18]` Diego).
- **Capacidade de diagnóstico**: a DLQ "fica como evidence pra debug e reprocessamento" (`[09:18]` Diego).
- **Possibilidade de reprocessamento manual** do que falhou em definitivo (`[09:18]` Diego — CAN-RF-009).
- **Controle de acesso administrativo** ao replay (`[09:36]` Sofia — CAN-DEC-010).
- **Rastreabilidade do replay**: "o endpoint de admin tem que logar quem fez o replay, pra auditoria" (`[09:36]` Sofia — CAN-RNF-006).
- **Compatibilidade com a garantia at-least-once** já decidida (`[09:24]` Diego — CAN-DEC-014).
- **Simplicidade operacional adequada à primeira versão**, coerente com o tamanho do time (`[09:07]` Diego).

## Decisão

Foi decidido que:

1. Uma entrega malsucedida **será elegível para novas tentativas**, executadas automaticamente (CAN-RF-012, CAN-RNF-010).
2. O processo será **limitado a cinco tentativas** — terminologia da reunião, fechada por Larissa em `[09:17]` (CAN-DEC-006, CAN-MET-004). A contagem operacional exata é ambiguidade registrada abaixo.
3. Os **intervalos decididos** são 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas (CAN-MET-005).
4. O **próximo momento elegível** para processamento deverá respeitar essa progressão; um evento em espera não deverá ser reprocessado antes do intervalo aplicável.
5. Uma chamada HTTP que **exceda o timeout de 10 segundos** será tratada como falha e marcada para retry (`[09:42]` Diego — CAN-DEC-026, CAN-MET-008, CAN-RNF-012). Este é o único critério de falha fechado na reunião.
6. **Esgotado o limite**, o evento será removido do fluxo ativo e **registrado na DLQ** (CAN-DEC-007, CAN-RF-013).
7. A DLQ será **persistida em tabela separada** da outbox principal (CAN-DEC-008).
8. A DLQ **deverá preservar informações suficientes** para diagnóstico e reprocessamento (CAN-DEC-009).
9. Haverá **replay manual por endpoint administrativo** (CAN-RF-009).
10. **Somente usuários com role `ADMIN`** poderão executá-lo, reaproveitando o `requireRole` existente (`[09:36]` Larissa — CAN-DEC-010).
11. A **execução do replay deverá ser auditada**, registrando quem a executou (CAN-RNF-006).
12. O replay **recolocará o evento no fluxo de processamento** — "Recoloca na outbox como pendente" (`[09:18]` Diego). A operação de persistência correspondente não é detalhada aqui.
13. Retentativas e replays **fazem parte de uma semântica at-least-once** e **poderão** produzir entregas duplicadas (CAN-RNF-009, CAN-RISK-002). Nenhum comportamento exactly-once é prometido (CAN-ALT-011).

## Ambiguidade sobre "cinco tentativas"

A transcrição **não define** se "cinco tentativas" significa cinco envios totais, incluindo o primeiro, ou cinco retentativas adicionais após o primeiro envio. Diego propôs "5" em `[09:15]`; Larissa fechou "5 tentativas" em `[09:17]`; nenhum dos dois qualificou o primeiro envio.

Há um indício aritmético, **não uma decisão**: os cinco intervalos somam ~14h36min, compatível com a afirmação de Diego em `[09:17]` de "quase 15 horas entre primeira falha e última tentativa" — o que sugere cinco esperas aplicadas *após* a primeira falha. O indício não é conclusivo e não é adotado aqui.

Consequentemente: a ambiguidade fica **registrada como lacuna**; este ADR mantém a terminologia da reunião ("cinco tentativas"); os intervalos **não são alterados** para acomodar qualquer leitura; e o **FDD deverá** esclarecer a contagem operacional antes da implementação.

## Progressão temporal

| Etapa | Espera definida |
| ----- | --------------: |
| 1     |        1 minuto |
| 2     |       5 minutos |
| 3     |      30 minutos |
| 4     |         2 horas |
| 5     |        12 horas |

- Os valores são os **intervalos progressivos definidos na reunião** (`[09:17]` Diego — CAN-MET-005), aceitos por Marcos no mesmo minuto e fechados por Larissa.
- A reunião usou o termo "backoff exponencial" (`[09:15]` Diego; `[09:48]` Larissa — CAN-RNF-010). Os valores decididos são progressivos, mas **não seguem uma razão fixa**; este ADR não os qualifica como progressão exponencial matematicamente exata sem validação.
- **Não há jitter definido** — o tema não foi discutido.
- O total aproximado de ~15h é referência de janela citada em `[09:17]`; **não substitui** os valores individuais, que são a decisão.
- O **FDD deverá** definir como calcular `nextAttemptAt` ou mecanismo equivalente.
- A política **não garante** que a entrega ocorrerá dentro dessa janela.

## Fluxo conceitual da decisão

1. O worker tenta entregar um evento elegível.
2. A entrega não é considerada bem-sucedida.
3. A tentativa e sua evidência são registradas.
4. Se o limite ainda não foi esgotado, o evento recebe novo momento elegível conforme o intervalo aplicável.
5. O evento volta a ser considerado pelo worker no momento apropriado.
6. Quando o limite é esgotado, o evento é encaminhado à DLQ.
7. Um usuário `ADMIN` pode solicitar replay manual.
8. O replay é auditado.
9. O evento retorna ao fluxo de processamento, conforme regras a detalhar no FDD.

## Dead-Letter Queue

A DLQ representa os eventos que **esgotaram a recuperação automática** e deixaram de ser retentados. Será persistida em **estrutura separada da outbox ativa** (CAN-DEC-008), escolha justificada por Diego em `[09:18]`: manter limpa a leitura da outbox principal e concentrar a evidência para depuração e reprocessamento. A separação evita misturar eventos ativos e permanentemente falhos, simplificando a investigação e a seleção do worker.

A DLQ **deverá reter contexto suficiente** para compreender a falha. Diferenciando decisão de necessidade de implementação:

- **Explicitamente discutido** (`[09:18]` Diego — CAN-DEC-009): payload, motivo da falha e timestamp.
- **Apenas necessidade de implementação, ainda a detalhar:** nomes, tipos, obrigatoriedade e demais campos; a correlação com o evento de origem; a forma de armazenar a evidência da resposta externa.

A tabela **não existe** hoje — Diego citou o nome `webhook_dead_letter` em `[09:18]` como intenção, e `prisma/schema.prisma` não possui model correspondente (CAN-INT-002). O **schema completo não foi definido** (CAN-LAC-001), e **retenção e arquivamento não foram definidos** (CAN-LAC-009): o único valor citado, arquivamento da outbox em ~30 dias, foi declarado fora de escopo em `[09:08]` (CAN-FORA-003).

Estar na DLQ **não significa descarte definitivo**: o evento permanece disponível para replay. Reciprocamente, **replay não significa garantia de sucesso** — o reenvio está sujeito às mesmas falhas que levaram o evento até ali.

## Replay administrativo

O replay existe para **reprocessar manualmente** um evento que esgotou a recuperação automática, tipicamente após a causa da falha ter sido corrigida do lado do cliente. É **restrito à role `ADMIN`** (CAN-DEC-010) — "Mexer em fila de entrega de notificação não é coisa de operador" (`[09:36]` Sofia) — e **deverá registrar quem o executou** (CAN-RNF-006), preservando a rastreabilidade da intervenção manual sobre a fila de entrega.

Como o replay reintroduz o evento no fluxo, **poderá** produzir nova entrega duplicada, coerente com o at-least-once (CAN-RNF-009): consequência aceita, não defeito da política.

O endpoint **não existe** no repositório. Diego citou a forma `POST /admin/webhooks/dead-letter/:id/replay` em `[09:18]` e `[09:35]` (CAN-RF-009); sua criação é integração futura (CAN-INT-003). Quando existir, **deverá** reutilizar `authenticate` e `requireRole('ADMIN')` (CAN-INT-004). O **FDD deverá** definir a semântica completa da operação.

**Não definidos**, e listados adiante como lacunas: replay em lote; aprovação por múltiplos administradores; limite por período; corpo e resposta do endpoint; estado exato criado na outbox; contagem de tentativas após o replay; e se o mesmo item pode ser reprocessado mais de uma vez.

## Classificação das falhas

A reunião definiu **um único** critério de falha: a chamada HTTP que excede 10 segundos é tratada como falha e marcada para retry (`[09:42]` Diego — CAN-RNF-012, CAN-MET-008).

**Não foi definido** o tratamento para: HTTP 2xx (regra de sucesso), 3xx, 4xx, 5xx, erro de DNS, conexão recusada, erro de TLS, payload inválido ou URL desativada (CAN-LAC-007).

Em consequência, este ADR **não escolhe** política de classificação. Não se presume que todo erro seja retentável; não se presume que 4xx seja falha permanente; não se presume que 5xx deva sempre ser retentado. O **FDD deverá** estabelecer uma matriz de classificação de respostas e erros de transporte, indicando quais são elegíveis à política aqui decidida e quais, se alguma, seguem direto para a DLQ.

## Ponto de integração com o sistema existente

**`src/middlewares/auth.middleware.ts` — `authenticate`, `requireRole`** (CAN-COD-007). `authenticate` exige `Authorization: Bearer`, valida o token com `jwt.verify(token, env.JWT_SECRET)` e popula `req.user` com `{ id, email, role }`, emitindo `UnauthorizedError` se o header falta ou o token é inválido. `requireRole(...roles)` é uma fábrica de `RequestHandler` que emite `UnauthorizedError` sem `req.user` e `ForbiddenError('Insufficient permissions')` quando `req.user.role` não está na lista. `AuthUser['role']` define dois papéis — `'ADMIN' | 'OPERATOR'` —, espelhando o enum `UserRole` do schema.

> O middleware `requireRole` já suporta restrição de rotas por papel. O futuro endpoint de replay deverá reutilizar `requireRole('ADMIN')`, sem introduzir mecanismo próprio de autorização.

**`src/modules/users/user.routes.ts` — `buildUserRouter`** (CAN-COD-022). Única rota hoje protegida por papel: `router.get('/:id', authenticate, requireRole('ADMIN'), validate({ params: idParamSchema }), controller.getById)`. Fixa o padrão vigente — `authenticate` antes de `requireRole`, validação depois — e é o precedente direto para o replay. **O endpoint de replay ainda não existe.**

**`src/shared/errors/app-error.ts` — `AppError`** (CAN-COD-004). Base com `statusCode`, `errorCode` e `details` opcional.

**`src/shared/errors/http-errors.ts`** (CAN-COD-005). Subclasses de `AppError` com códigos em maiúsculas (`ValidationError`/`VALIDATION_ERROR`, `ForbiddenError`/`FORBIDDEN`, `NotFoundError`/`NOT_FOUND`), e erros de domínio derivando das genéricas: `InvalidStatusTransitionError` estende `ConflictError` (`INVALID_STATUS_TRANSITION`); `InsufficientStockError` estende `UnprocessableEntityError` (`INSUFFICIENT_STOCK`). Erros futuros de DLQ e replay **deverão** seguir a convenção, com o prefixo `WEBHOOK_` acordado em `[09:28]`–`[09:29]`. **Nenhum erro `WEBHOOK_*` existe hoje** (CAN-INT-005).

**`src/middlewares/error.middleware.ts` — `errorMiddleware`** (CAN-COD-006). Trata `AppError` centralizadamente, respondendo `err.statusCode` com `{ error: { code, message, details? } }`; trata `ZodError` e `Prisma.PrismaClientKnownRequestError` (P2002, P2025); no fallback, registra `logger.error({ err, requestId, method, path }, 'Unhandled error in request')` e responde 500 `INTERNAL_SERVER_ERROR`. Erros do replay que estendam `AppError` serão tratados sem alterá-lo.

**`src/shared/logger/index.ts` — `createLogger`, `logger`** (CAN-COD-013). Pino com `redact` e `base: { service, env }`. O replay e as transições para a DLQ **deverão** ser observáveis por ele; a estratégia completa de logs e métricas fica no FDD. Atenção: `secret` **não** consta da redação atual (CAN-INT-007).

**`prisma/schema.prisma` — `datasource db`** (CAN-COD-002). Provider `mysql`; models existentes: `User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory`, `OrderNumberSequence`. **Não existe tabela ou model de DLQ, nem model específico de webhook.** A modelagem é integração futura (CAN-INT-002) e não é desenhada aqui.

## Alternativas consideradas

### Alternativa — Três tentativas

**Descrição** — Limitar a recuperação automática a três tentativas, proposta por Bruno em `[09:16]`: "3 não é melhor? Mais agressivo." (CAN-ALT-004).

**Vantagens** — Menor tempo até classificar uma falha como permanente; menor consumo de recursos; ciclo de vida do evento mais curto.

**Desvantagens e trade-offs** — Janela insuficiente para indisponibilidades mais longas; descarte prematuro de eventos recuperáveis, enviando à DLQ falhas que teriam sucesso mais tarde; mais intervenção manual.

**Motivo do descarte** — Rejeitada por Diego em `[09:16]`: "3 é pouco... a gente retentaria três vezes em 30 minutos e mataria", com o precedente da indisponibilidade de duas horas. Larissa fechou em `[09:16]`: "Cinco fica bom."

### Alternativa — Retry indefinido

**Descrição** — Retentar indefinidamente com backoff, sem teto (`[09:15]` Diego — CAN-ALT-005).

**Vantagens** — Máxima persistência: nenhum evento é abandonado enquanto houver chance de recuperação; nenhuma intervenção manual no caso feliz.

**Desvantagens e trade-offs** — Crescimento ilimitado de eventos pendentes; consumo contínuo de recursos contra endpoints que podem nunca voltar; ausência de encerramento operacional; impossibilidade de distinguir falha temporária de endpoint abandonado, já que ambos permanecem no mesmo estado.

**Motivo do descarte** — Diego em `[09:15]`: "isso traz o problema de evento ficar pendurado pra sempre se o cliente sumiu". O teto finito foi adotado para produzir esse encerramento.

### Alternativa — Manter eventos permanentemente falhos na própria outbox

**Descrição** — Dispensar tabela dedicada e marcar o evento como "failed" na própria outbox, alternativa levantada por Larissa em `[09:17]`: "Faz numa tabela separada ou marca como 'failed' na própria outbox?" (CAN-ALT-006).

**Vantagens** — Menos tabelas; modelagem inicial mais simples; nenhuma transferência de linhas entre estruturas, eliminando a classe de risco associada; histórico do evento em um só lugar.

**Desvantagens e trade-offs** — Mistura de eventos ativos e permanentemente falhos; consultas do worker mais complexas, obrigadas a excluir os falhos; degradação operacional conforme os falhos se acumulam; investigação dificultada pela ausência de separação entre o que está vivo e o que terminou.

**Motivo do descarte** — Diego em `[09:18]` optou pela tabela dedicada: "Mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento".

## Consequências

### Consequências positivas

- Recuperação automática de falhas temporárias, sem intervenção humana.
- Limite operacional claro: nenhum evento é retentado indefinidamente.
- Janela progressiva que cobre indisponibilidades curtas e prolongadas.
- Separação entre fluxo ativo e falhas permanentes, mantendo a outbox focada em eventos processáveis.
- Suporte a diagnóstico: a DLQ concentra a evidência da falha.
- Replay controlado, com caminho definido para reprocessamento.
- Auditoria administrativa sobre quem interveio na fila de entrega.
- Compatibilidade com o Outbox (ADR-001) e com o worker (ADR-002), que operam sobre eventos já persistidos.
- Base para métricas futuras de falha e de profundidade da DLQ.

### Consequências negativas

- Uma entrega pode demorar muitas horas até a última tentativa — janela citada de ~15h (`[09:17]`).
- Eventos permanecem armazenados por mais tempo, ativos ou na DLQ.
- Retentativas consomem recursos do worker e do banco contra endpoints possivelmente inoperantes.
- Replay pode gerar duplicidade na ponta do cliente (CAN-RISK-002).
- Passa a ser necessário operar e monitorar a DLQ, com procedimento e responsável.
- A classificação incorreta de falhas pode causar tentativas desnecessárias ou descarte prematuro — risco agravado por a política não estar definida (CAN-LAC-007).
- A tabela da DLQ pode crescer, sem retenção definida (CAN-LAC-009).
- A política é mais complexa que uma única tentativa: introduz agendamento, contador e um segundo destino.
- A execução manual cria responsabilidade operacional: sem alguém observando, itens da DLQ permanecem parados.
- O endpoint administrativo amplia a superfície de segurança da API.

### Consequências neutras ou operacionais

- Criação futura da tabela de DLQ e da respectiva migration (CAN-INT-002).
- Criação futura do endpoint de replay e seu registro na composição de rotas (CAN-INT-003).
- Criação futura das classes de erro `WEBHOOK_*` sobre `AppError` (CAN-INT-005).
- Atualização futura dos testes, incluindo a limpeza das novas tabelas em `tests/setup.ts` (CAN-INT-009).
- Documentação operacional e procedimento de investigação da DLQ.
- Métricas e logs de falha, retentativa e replay, ainda não especificados.
- Política de retenção da DLQ ainda não definida (CAN-LAC-009).
- Notificação ao cliente sobre falhas recorrentes permanece fora de escopo (`[09:37]` Larissa — CAN-FORA-001).

## Riscos e cuidados

| Risco ou cuidado | Impacto | Tratamento ou estado |
| ---------------- | ------- | -------------------- |
| Retry storm contra endpoint degradado | Amplificação da carga sobre um cliente já em dificuldade | Mitigado parcialmente pela progressão crescente (CAN-MET-005); jitter e rate limiting de saída não definidos (CAN-OPEN-003). `Em aberto para detalhamento no FDD` |
| Crescimento do backlog de eventos em espera | Latência acima dos 10s exigidos (CAN-MET-001) | Teto finito limita o acúmulo por evento; métricas de lag não definidas. `Em aberto para detalhamento no FDD` |
| Endpoint permanentemente inválido | Cinco ciclos consumidos até o inevitável | Consequência aceita do teto; tratamento de URL desativada não definido (CAN-LAC-007). `Em aberto para detalhamento no FDD` |
| Entrega duplicada por retry ou replay | Cliente processa o mesmo evento mais de uma vez | Consequência aceita do at-least-once; dedup delegada ao cliente via `X-Event-Id` (ADR-005 — CAN-DEC-015) |
| Replay sem correção da causa da falha | Evento retorna à DLQ; esforço operacional desperdiçado | Replay não garante sucesso; pré-condições não definidas. `Em aberto para detalhamento no FDD` |
| Abuso do endpoint administrativo | Reenvio massivo indevido a clientes | Mitigado por `requireRole('ADMIN')` (CAN-DEC-010); limite por período não definido. `Em aberto para detalhamento no FDD` |
| Falta de auditoria efetiva do replay | Intervenção manual não rastreável | Auditoria é decisão (CAN-RNF-006); formato e destino do registro não definidos. `Em aberto para detalhamento no FDD` |
| Evento perdido na transferência para a DLQ | Notificação desaparece sem registro em nenhuma das estruturas | Atomicidade da transferência não definida. `Em aberto para detalhamento no FDD` |
| Divergência entre outbox e DLQ | Mesmo evento ativo e em dead-letter, ou em nenhuma | Depende de a transferência ser movimento ou cópia — não definido. `Em aberto para detalhamento no FDD` |
| Worker reiniciado durante uma tentativa | Tentativa sem resultado persistido | Claim, lease e recuperação não definidos (CAN-LAC-006, ADR-002). `Em aberto para detalhamento no FDD` |
| Contador de tentativas inconsistente | Retentativas a mais ou a menos que o teto decidido | Agravado pela ambiguidade da contagem. `Em aberto para detalhamento no FDD` |
| Eventos retentados fora de ordem | Cliente observa transições do mesmo pedido fora de sequência | Limitação conhecida: não há garantia formal de ordenação (ADR-002 — CAN-RNF-014) |
| Retenção indefinida da DLQ | Crescimento da tabela sem limite | Não definida (CAN-LAC-009). `Em aberto para detalhamento no FDD` |
| Exposição de payloads sensíveis em logs | Vazamento de dados de pedido ou de `secret` | Pino já redige `authorization`, `*.password` e `*.token`, mas **não** `secret` (CAN-COD-013, CAN-INT-007). `Em aberto para detalhamento no FDD` |
| Resposta externa muito grande armazenada como evidência | Crescimento da DLQ e custo de armazenamento | Tamanho máximo da resposta não definido. `Em aberto para detalhamento no FDD` |
| Ausência de classificação por tipo de falha | Retentar o irretentável, ou descartar o recuperável | Apenas o timeout está definido (CAN-RNF-012); demais critérios são lacuna (CAN-LAC-007). `Em aberto para detalhamento no FDD` |

## Lacunas de implementação

Os pontos abaixo **não são definidos** por esta decisão e não devem ser inferidos a partir dela:

- Significado operacional exato de "cinco tentativas": se inclui ou não o primeiro envio.
- Regra de sucesso de uma entrega (CAN-LAC-007).
- Classificação por status HTTP: 2xx, 3xx, 4xx, 5xx (CAN-LAC-007).
- Classificação por erro de transporte: DNS, TLS, conexão recusada (CAN-LAC-007).
- Uso ou não de jitter na progressão.
- Cálculo exato de `nextAttemptAt` ou mecanismo equivalente.
- Campos definitivos da DLQ, além de payload, motivo e timestamp (CAN-DEC-009, CAN-LAC-001).
- Se a passagem da outbox para a DLQ é transferência atômica ou cópia.
- Preservação ou reset do contador de tentativas no replay.
- Limite de replays por item ou por período.
- Se o replay é único ou pode ser repetido.
- Replay em lote.
- Política de retenção da DLQ e do histórico de entregas (CAN-LAC-009).
- Expiração de eventos por idade.
- Autorização adicional além de `requireRole('ADMIN')`.
- Formato e destino do registro de auditoria do replay (CAN-RNF-006).
- Métricas e alertas de falha, DLQ e replay.
- Tratamento de endpoints desativados e comportamento quando a configuração do webhook é removida.
- Impacto dos retries na ordenação relativa dos eventos de um mesmo pedido (CAN-RNF-014).
- Tratamento de payload acima do limite de 64KB no contexto de retry (CAN-RISK-006).
- Tamanho máximo da resposta externa armazenada como evidência.
- Biblioteca ou cliente HTTP do worker (CAN-LAC-013).

## Critérios de conformidade arquitetural

Uma implementação futura estará em conformidade com este ADR se:

1. Falhas elegíveis seguirem a política de tentativas decidida, e não uma política própria do código.
2. A política possuir **limite finito**, sem caminho para retentativa indefinida.
3. Os **intervalos decididos** (1m, 5m, 30m, 2h, 12h) forem preservados.
4. O esgotamento do limite encaminhar o evento a uma **DLQ em estrutura separada** da outbox.
5. Eventos da DLQ **não permanecerem misturados** ao fluxo ativo nem forem selecionados pelo ciclo normal do worker.
6. O replay for **exclusivamente administrativo**, restrito a `ADMIN`.
7. O replay **registrar quem o executou**.
8. Retentativas e replays respeitarem a semântica **at-least-once**, sem prometer exactly-once.
9. **Nenhuma retentativa ocorrer de forma síncrona** na requisição de mudança de status (ADR-001, CAN-ALT-001).
10. A política de classificação de falhas for **documentada no FDD**, e não decidida silenciosamente no código.
11. Qualquer alteração nos intervalos, no limite ou no destino final exigir **revisão deste ADR**.
12. Desvios relevantes exigirem novo ADR ou atualização formal deste documento.

## Evidências e rastreabilidade

| Tipo | Localização | Evidência sustentada |
| ---- | ----------- | -------------------- |
| TRANSCRICAO | `[09:14]` Larissa; `[09:15]` Diego | Abertura do tema retry por cliente offline; backoff com teto e falha permanente para DLQ (CAN-RF-012, CAN-RNF-010, CAN-RISK-001) |
| TRANSCRICAO | `[09:15]` Bruno; `[09:15]` Diego | "Quantas tentativas?"; sugestão de 5; retry indefinido descartado — "evento ficar pendurado pra sempre se o cliente sumiu" (CAN-ALT-005) |
| TRANSCRICAO | `[09:16]` Bruno; `[09:16]` Diego; `[09:16]` Larissa | Três tentativas propostas e descartadas: "3 é pouco... retentaria três vezes em 30 minutos e mataria"; precedente de indisponibilidade de 2h; "Cinco fica bom" (CAN-ALT-004, CAN-DEC-006) |
| TRANSCRICAO | `[09:17]` Diego; `[09:17]` Marcos; `[09:17]` Larissa | Progressão 1m/5m/30m/2h/12h; "quase 15 horas entre primeira falha e última tentativa"; aceite de negócio; fechamento "Decidido: 5 tentativas" (CAN-MET-004, CAN-MET-005, CAN-DEC-007) |
| TRANSCRICAO | `[09:17]` Larissa; `[09:18]` Diego | Tabela separada vs. marcar "failed" na outbox; escolha por `webhook_dead_letter` com payload, motivo e timestamp; "Mais limpa a leitura da outbox principal" (CAN-ALT-006, CAN-DEC-008, CAN-DEC-009) |
| TRANSCRICAO | `[09:18]` Bruno; `[09:18]` Diego; `[09:19]` Larissa; `[09:35]` Diego | Replay manual por endpoint admin `POST /admin/webhooks/dead-letter/:id/replay`; "Recoloca na outbox como pendente" (CAN-RF-009) |
| TRANSCRICAO | `[09:35]` Larissa; `[09:36]` Sofia; `[09:36]` Larissa | Role `ADMIN` obrigatória; "não é coisa de operador"; auditoria de quem executou; reaproveitar `requireRole` (CAN-DEC-010, CAN-RNF-006) |
| TRANSCRICAO | `[09:42]` Sofia; `[09:42]` Diego | Timeout de 10s tratado como falha e marcado para retry — único critério de falha fechado (CAN-DEC-026, CAN-MET-008, CAN-RNF-012) |
| TRANSCRICAO | `[09:24]` Diego; `[09:25]` Diego | At-least-once com dedup pelo cliente; exactly-once descartado (CAN-DEC-014, CAN-RNF-009, CAN-ALT-011) |
| TRANSCRICAO | `[09:48]` Larissa | Resumo confirmado: "backoff exponencial 1m/5m/30m/2h/12h, total 5 tentativas, depois DLQ persistida em tabela separada" |
| CODIGO | `src/middlewares/auth.middleware.ts` — `authenticate`, `requireRole`, `AuthUser` | JWT bearer; `req.user.role`; papéis `'ADMIN' \| 'OPERATOR'`; `ForbiddenError` por papel insuficiente (CAN-COD-007) |
| CODIGO | `src/modules/users/user.routes.ts` — `buildUserRouter` | Única rota real com `authenticate` + `requireRole('ADMIN')`; precedente do padrão de autorização (CAN-COD-022) |
| CODIGO | `src/shared/errors/app-error.ts` — `AppError` | Base com `statusCode`, `errorCode`, `details` (CAN-COD-004) |
| CODIGO | `src/shared/errors/http-errors.ts` — `ConflictError`, `InvalidStatusTransitionError`, `InsufficientStockError` | Convenção de subclasses e códigos em maiúsculas, a seguir no prefixo `WEBHOOK_` (CAN-COD-005, CAN-INT-005) |
| CODIGO | `src/middlewares/error.middleware.ts` — `errorMiddleware` | Tratamento centralizado de `AppError`; resposta `{ error: { code, message, details? } }`; fallback 500 com `logger.error` (CAN-COD-006) |
| CODIGO | `src/shared/logger/index.ts` — `createLogger`, `logger` | Pino com `redact`; `secret` ausente da lista atual (CAN-COD-013, CAN-INT-007) |
| CODIGO | `prisma/schema.prisma` — `datasource db`, `enum UserRole` | Provider `mysql`; papéis `ADMIN`/`OPERATOR`; ausência de model de DLQ ou de webhook (CAN-COD-002, CAN-INT-002) |
| ADR | [ADR-001](./ADR-001-outbox-no-mysql.md) — Decisão, itens 5 e 6 | Evento persistido transacionalmente; HTTP fora da transação; processamento assíncrono posterior |
| ADR | [ADR-002](./ADR-002-worker-separado-com-polling.md) — Escopo e Responsabilidades do worker | O worker respeita as decisões de retry e DLQ registradas aqui; claim e recuperação permanecem lacuna (CAN-LAC-006) |
| MATRIZ | `MATRIZ-FATOS.md` §3 — índice ADR-003 | Conjunto canônico deste ADR: CAN-DEC-006, CAN-DEC-007, CAN-DEC-008, CAN-DEC-009, CAN-DEC-010, CAN-DEC-026, CAN-RF-009, CAN-RF-012, CAN-RF-013, CAN-RNF-006, CAN-RNF-010, CAN-RNF-012, CAN-ALT-004, CAN-ALT-005, CAN-ALT-006, CAN-MET-004, CAN-MET-005, CAN-MET-008, CAN-COD-022 |
| ANALISE | `ANALISE-EVIDENCIAS.md` §18 — AMB-01 | Divergência 3 vs. 5 registrada e resolvida em 5; a contagem operacional das cinco permanece sem definição |

## Decisões relacionadas

As decisões abaixo não são repetidas aqui. ADR-001 e ADR-002 já existem em `docs/adrs/`; ADR-005 e ADR-006 estão **planejados e ainda não foram escritos**, com os nomes de arquivo já referenciados pelos ADRs anteriores.

- [ADR-001 — Transactional Outbox no MySQL para eventos de webhook](./ADR-001-outbox-no-mysql.md) — produção e persistência atômica do evento que esta política retenta.
- [ADR-002 — Worker separado com polling para processamento da outbox](./ADR-002-worker-separado-com-polling.md) — quem executa a entrega, o polling e a seleção dos eventos elegíveis.
- [ADR-005 — Entrega at-least-once com Event ID](./ADR-005-entrega-at-least-once-com-event-id.md) — garantia de entrega e idempotência delegada ao cliente, semântica sob a qual retries e replays operam.
- [ADR-006 — Reuso dos padrões do projeto](./ADR-006-reuso-dos-padroes-do-projeto.md) — convenções de erro, logger e estrutura modular reutilizadas pelo replay.

## Notas de implementação

- A tentativa e seu resultado **deverão** ser registráveis: sem evidência persistida da falha, a DLQ não cumpre sua função de diagnóstico.
- O agendamento do próximo momento elegível **deverá** ser persistente, e não mantido em memória do worker: um reinício não pode reiniciar nem perder a progressão.
- O replay **deverá** ser auditável, identificando o executor.
- A falha de um evento **não deverá** encerrar permanentemente o worker nem interromper o processamento dos demais.
- As transições entre fluxo ativo e DLQ **não deverão** perder o evento: em nenhum instante ele pode deixar de existir nas duas estruturas.
- O esgotamento do limite **não deverá** resultar em exclusão silenciosa do evento.

## Histórico

| Data | Alteração | Autor |
| ---- | --------- | ----- |
| Não informada | Decisão tomada na reunião técnica de webhooks (`[09:14]`–`[09:19]`; role `ADMIN` e auditoria confirmadas em `[09:35]`–`[09:36]`; timeout em `[09:42]`; resumo em `[09:48]`) | Diego, Larissa, Bruno, Marcos, Sofia |
| Não informada | Redação inicial do ADR a partir de `TRANSCRICAO.md`, do código e de `MATRIZ-FATOS.md` | Responsável pela entrega |
</content>
</invoke>
