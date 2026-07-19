# ADR-001 — Transactional Outbox no MySQL para eventos de webhook

## Status

`Aceito`

- **Data da decisão:** não informada. `TRANSCRICAO.md` registra apenas "quinta-feira, 09:00", sem data de calendário.
- **Participantes:** Diego (Engenheiro Sênior, Plataforma), Larissa (Tech Lead, conduzindo) e Bruno (Engenheiro Pleno, Pedidos). Fechada por Larissa em `[09:08]` ("Tá decidido então: outbox em MySQL").
- **Escopo:** como o evento de mudança de status é produzido e persistido atomicamente junto à própria mudança de status. Não cobre leitura e entrega, retry/DLQ, segurança do envio nem contrato do payload.

## Contexto

Três clientes B2B pediram formalmente para serem notificados quando o status de seus pedidos muda; hoje consultam `GET /orders` em polling, o que torna a integração deles lenta e cara (`[09:00]` Marcos — CAN-PROB-001, CAN-PROB-002). "Tempo real", para eles, é latência abaixo de 10 segundos (`[09:02]` Marcos — CAN-MET-001). O webhook é outbound: a plataforma envia, o cliente não envia de volta (`[09:02]` Marcos; `[09:03]` Sofia).

A notificação depende de uma chamada HTTP a um sistema de terceiro, que pode falhar ou demorar. O fluxo que a origina já é transacional: `OrderService.changeStatus` executa, dentro de `prisma.$transaction`, a leitura do pedido, a validação da transição, o ajuste de estoque, a atualização do status e a gravação do histórico (CAN-COD-001) — transação que Bruno descreveu como já pesada (`[09:04]`).

Daí duas exposições opostas. Executar o HTTP dentro da transação acopla a mudança de status à disponibilidade do cliente: um cliente lento travaria mudanças de outros pedidos, e um cliente fora do ar deixaria como única reação o rollback da mudança de status, que Bruno descartou de imediato (`[09:04]`). Registrar o evento fora da transação cria a exposição inversa: status commitado sem evento, um pedido que muda sem que o cliente seja notificado (`[09:40]` Bruno; `[09:41]` Diego — CAN-RISK-005).

Não existe hoje no repositório qualquer mecanismo assíncrono, fila, outbox, worker ou código de webhook; `prisma/schema.prisma` modela apenas usuários, clientes, produtos, pedidos, itens, histórico e a sequência de numeração (CAN-COD-002). O time é pequeno, e infraestrutura dedicada foi considerada desproporcional para a primeira versão (`[09:07]` Diego).

## Drivers da decisão

- **Atomicidade** entre a mudança de status confirmada e a existência do evento (`[09:06]` Diego; CAN-RNF-008).
- **Confiabilidade**: nenhum status commitado pode ficar sem evento (`[09:40]` Bruno; CAN-RISK-005).
- **Desacoplamento da disponibilidade externa**: a transação não pode depender do cliente estar no ar (`[09:04]` Bruno).
- **Uso da infraestrutura existente**: "Outbox no MySQL existente resolve" (`[09:07]` Diego; CAN-DEC-001).
- **Baixa complexidade operacional inicial**, dado o tamanho do time (`[09:07]` Diego).
- **Compatibilidade com a transação Prisma existente**, que já agrupa pedido, estoque e histórico (CAN-COD-001).
- **Possibilidade de retry assíncrono posterior**, viabilizada pela persistência prévia do evento (`[09:06]` Diego).

## Decisão

Foi decidido adotar o padrão **Transactional Outbox** para a produção dos eventos de webhook (CAN-DEC-001):

1. Os eventos serão persistidos em tabela de outbox no **MySQL já utilizado pela aplicação** (`prisma/schema.prisma`, `datasource db { provider = "mysql" }`). Não será introduzida infraestrutura de mensageria na primeira versão.
2. A criação do evento **deverá ocorrer dentro da mesma transação SQL** da mudança de status (CAN-DEC-002).
3. O evento **só existirá se a transação for confirmada**; em rollback, não é gravado (CAN-RNF-008).
4. Uma **falha na inserção na outbox deverá causar rollback** de toda a operação — pedido, estoque, histórico e evento. Bruno: "Não pode ter caso de status mudar e evento não sair" (`[09:40]`); Diego: "Se ficar fora da transação, perde a garantia toda" (`[09:41]`).
5. A **chamada HTTP ao cliente não ocorrerá dentro da transação**; o disparo síncrono foi descartado (`[09:06]` Diego — CAN-ALT-001).
6. Um **componente assíncrono posterior** processará os eventos persistidos e executará a entrega; seu desenho pertence ao ADR-002.

Duas propriedades da inserção foram fechadas na reunião e ficam registradas como fronteira desta decisão, sem detalhamento:

- **Filtro na inserção** (CAN-DEC-020): o filtro de status assinados é aplicado na inserção, não no envio — se nenhum webhook do customer quer aquele status, a linha não é inserida (`[09:34]` Bruno; `[09:34]` Diego).
- **Snapshot do payload** (CAN-DEC-023, CAN-RF-018): o payload será materializado no momento da inserção, preservando o estado do pedido no instante em que o status mudou, em vez de ser renderizado no envio (`[09:52]` Larissa: "o evento ainda reflete o estado de quando o status mudou"; `[09:52]` Diego: "snapshot na inserção"). Conforme a nota de escopo de `MATRIZ-FATOS.md`, essa decisão não tem ADR dedicado e seu detalhamento pertence ao FDD.

Nenhum campo de tabela, estado interno ou detalhe do processador é definido por este ADR.

## Fluxo conceitual da decisão

1. Uma mudança de status válida é solicitada.
2. A aplicação utiliza a transação já existente em `OrderService.changeStatus`.
3. O pedido, o estoque e o histórico são atualizados.
4. O evento correspondente é persistido na outbox **dentro da mesma transação**.
5. A transação é confirmada.
6. Um processador assíncrono lê o evento persistido e o entrega posteriormente.

Se qualquer operação transacional falhar — inclusive a inserção na outbox — nenhuma alteração é confirmada, e o pedido permanece no status anterior sem evento gerado. A entrega HTTP não faz parte desse commit: ocorre depois, fora da transação, e sua falha não afeta a mudança de status já confirmada.

## Ponto de integração com o sistema existente

**`src/modules/orders/order.service.ts` — `OrderService.changeStatus`** (CAN-COD-001). O método atualmente abre a transação via `this.prisma.$transaction`, lê o pedido com `tx.order.findUnique`, valida a transição, ajusta estoque por `debitStock`/`replenishStock`, executa `tx.order.update`, grava `tx.orderStatusHistory.create` e retorna o pedido atualizado. É o ponto documentalmente coerente para a futura publicação na outbox: após a gravação do histórico e antes do fim do callback transacional.

Explicitamente: a transação **já existe**; a inserção na outbox **ainda não existe**; a função `publishWebhookEvent(tx, order, fromStatus, toStatus)` — assinatura acordada em `[09:41]` por Bruno e Diego (CAN-DEC-021) — **ainda não existe** no repositório, e criá-la é integração futura (CAN-INT-001). Não há hoje nenhum código de webhook, outbox ou worker; a integração descrita é futura.

> O método atualmente executa as alterações de domínio dentro de uma transação Prisma. A futura inserção na outbox deverá utilizar o mesmo `Prisma.TransactionClient` (já aliasado como `TxClient` no arquivo), sem iniciar uma transação independente.

**`prisma/schema.prisma`** (CAN-COD-002, CAN-COD-020). O datasource é `mysql`; os models existentes (`Order`, `OrderStatusHistory`, `Product`) usam PK `String @id @default(uuid()) @db.Char(36)`; o enum `OrderStatus` define os valores que alimentarão `from_status`/`to_status`. **Não existem models de webhook ou de outbox.** A modelagem das novas tabelas é integração futura (CAN-INT-002) e não é desenhada aqui.

**`src/modules/orders/order.status.ts`** (CAN-COD-003). `canTransition`, `shouldDebitStock` e `shouldReplenishStock` concentram a máquina de transições e os efeitos de estoque. O módulo de webhook **não deverá redefinir nem duplicar essas regras**: a outbox é alimentada por transições já validadas por esse código.

## Alternativas consideradas

### Alternativa — Chamada HTTP síncrona dentro da mudança de status

**Descrição** — Disparar a chamada HTTP ao endpoint do cliente diretamente no service de orders, durante a mudança de status (`[09:03]` Larissa — CAN-ALT-001).

**Vantagens** — Simplicidade inicial: nenhuma tabela nova, nenhum processamento assíncrono, nenhum processo adicional; caminho único, sem defasagem entre a mudança e o envio.

**Desvantagens e trade-offs**

- Bloqueio da transação enquanto a chamada externa não retorna, sobre uma transação já pesada (`[09:04]` Bruno).
- Dependência da disponibilidade do cliente: um cliente lento travaria mudanças de status de outros pedidos (`[09:04]` Bruno).
- Aumento da latência da operação de negócio, proporcional ao tempo de resposta de terceiros.
- Dificuldade de recuperação: sem persistência prévia do evento, não há o que retentar.
- Risco de inconsistência: contra um cliente fora do ar, a única reação seria abortar a mudança de status (`[09:04]` Bruno).

**Motivo do descarte** — Rejeitada explicitamente: "Síncrono está fora de questão" (`[09:06]` Diego). O outbox foi adotado por desacoplar o commit do pedido da chamada externa.

### Alternativa — Redis Streams ou fila externa

**Descrição** — Publicar o evento em Redis Streams ou em fila dedicada, consumida por um worker (`[09:07]` Larissa — CAN-ALT-002).

**Vantagens** — Recursos nativos de mensageria (consumer groups, acks, retenção) sem construção própria; melhor desacoplamento de consumo em volumes maiores.

**Desvantagens e trade-offs**

- Nova infraestrutura a provisionar, versionar e manter.
- Operação e monitoramento adicionais, para um time descrito como pequeno (`[09:07]` Diego).
- Perda da atomicidade nativa: a publicação na fila não participa da transação SQL do pedido, reintroduzindo o problema que a decisão precisa resolver.
- Complexidade desproporcional para a primeira versão.

**Motivo do descarte** — Considerada overengineering no contexto atual: "Subir Redis Cluster pra isso é overengineering. Outbox no MySQL existente resolve" (`[09:07]` Diego).

> A alternativa de **trigger de banco** (`[09:09]` Bruno; `[09:09]` Diego — CAN-ALT-003) foi discutida como forma de **notificar o worker**, não como mecanismo de publicação; pertence ao ADR-002.

## Consequências

### Consequências positivas

- Atomicidade real: um commit produz status e evento, um rollback não produz nenhum (CAN-RNF-008).
- Redução do risco de perda do evento após mudança de status confirmada (CAN-RISK-005).
- Desacoplamento da disponibilidade do cliente: o status commita mesmo com o endpoint fora do ar.
- Reutilização do MySQL existente, sem provisionar infraestrutura nova.
- Viabilização do processamento assíncrono e do retry: o evento persistido é o que torna a retentativa possível.
- Base para as decisões de retry e DLQ, que operam sobre eventos já persistidos.
- Integração natural com a transação atual, que já usa `Prisma.TransactionClient`.

### Consequências negativas

- Uma escrita adicional passa a integrar a transação, já descrita como pesada (`[09:04]` Bruno).
- Aumento da complexidade do modelo de persistência: o domínio de pedidos passa a carregar uma tabela de integração.
- Crescimento contínuo da tabela de outbox, sem política de limpeza nesta fase.
- Necessidade de um componente assíncrono em execução; sem ele, os eventos apenas se acumulam.
- Necessidade futura de política de retenção, hoje inexistente.
- Possibilidade de duplicidade na entrega, coerente com o at-least-once de ADR-005 (CAN-RNF-009, CAN-DEC-014).
- Necessidade de monitoramento do backlog e do processamento, ainda não especificado.
- Dependência operacional do MySQL, que passa a armazenar pedidos e eventos: uma degradação do banco afeta os dois domínios ao mesmo tempo.

### Consequências neutras ou operacionais

- Migration Prisma nova será necessária; o mecanismo já está inicializado (CAN-COD-023, CAN-INT-002).
- Novos testes deverão cobrir o comportamento transacional, em especial o rollback conjunto.
- `tests/setup.ts`, que hoje limpa tabelas em `beforeEach`, deverá limpar as novas tabelas (CAN-COD-021, CAN-INT-009).
- Será necessário executar e supervisionar um worker; a orquestração em produção não está definida (CAN-LAC-010).
- Retenção e arquivamento permanecem fora do escopo inicial (CAN-FORA-003).

## Riscos e cuidados

| Risco ou cuidado | Impacto | Tratamento ou estado |
|---|---|---|
| Crescimento indefinido da outbox | Degradação de leitura e ocupação de disco | Arquivamento (~30 dias) citado em `[09:08]` por Diego e adiado (CAN-FORA-003, CAN-MET-010). `Em aberto para detalhamento no FDD` |
| Aumento do tempo da transação pela escrita adicional | Maior janela de locks em `orders`, `order_status_history` e `products` | Mitigado por manter o HTTP fora da transação (CAN-ALT-001 descartada); impacto não quantificado. `Em aberto para detalhamento no FDD` |
| Evento entregue mais de uma vez | Cliente processa o mesmo evento repetidamente | Consequência aceita do at-least-once; dedup delegada ao cliente via `X-Event-Id` (ADR-005 — CAN-DEC-014, CAN-DEC-015) |
| Evento preso em estado intermediário por falha do worker | Notificação não entregue apesar do commit | Depende de claim e recuperação, não definidos (CAN-LAC-006). `Em aberto para detalhamento no FDD` |
| Ausência de estratégia definida de claim/lock | Duplo envio ou linha nunca reprocessada | Não definido (CAN-LAC-006). `Em aberto para detalhamento no FDD` |
| Reinício do worker durante o processamento | Linhas em processamento sem dono | Não definido (CAN-LAC-006). `Em aberto para detalhamento no FDD` |
| Dependência excessiva do MySQL, que serve pedidos e eventos | Falha do banco afeta domínio e integração juntos | Trade-off aceito para evitar infraestrutura nova (`[09:07]` Diego — CAN-DEC-001) |
| Limpeza e retenção adiadas | Dívida operacional desde o primeiro dia | Adiado explicitamente (CAN-FORA-003); retenção de DLQ e histórico indefinida (CAN-LAC-009). `Em aberto para detalhamento no FDD` |

## Lacunas de implementação

Os pontos abaixo **não são definidos** por esta decisão:

- Schema completo da outbox, incluindo colunas e índices definitivos — Diego mencionou intenção de índice em status e `created_at` em `[09:08]`, sem definição formal (CAN-LAC-001).
- Estados internos do evento: "pendente, processando, falhou, entregue" foram citados em `[09:08]`, sem conjunto nem transições fechados (CAN-LAC-001).
- Estratégia de claim das linhas pelo worker (CAN-LAC-006).
- Locking e recuperação de eventos em processamento após reinício (CAN-LAC-006).
- Tamanho do batch e paralelismo: "batch pequeno" foi citado em `[09:08]` sem número (CAN-LAC-014).
- Política de retenção e arquivamento das linhas entregues (CAN-FORA-003, CAN-LAC-009).
- Plano de migração e rollout das novas tabelas (CAN-LAC-012).
- Detalhes do snapshot: campos, serialização e limites pertencem ao FDD (CAN-DEC-023, CAN-RF-018).
- Forma de armazenar o filtro de status e, por consequência, o momento exato da consulta aos webhooks inscritos na inserção (CAN-LAC-002).

## Critérios de conformidade arquitetural

Uma implementação futura estará em conformidade com este ADR se:

1. A inserção na outbox ocorrer **no mesmo callback transacional** da mudança de status, recebendo o `Prisma.TransactionClient` em uso e não abrindo transação independente.
2. **Não houver chamada HTTP externa** dentro da transação de mudança de status.
3. A falha na inserção do evento **impedir o commit** da mudança de status, do estoque e do histórico.
4. O evento **não for persistido** quando a transição for rejeitada por `canTransition` ou pelas demais validações de `changeStatus`.
5. O módulo de webhooks **não duplicar** a máquina de estados de `src/modules/orders/order.status.ts`.
6. A persistência utilizar o **MySQL existente**, sem novo armazenamento para os eventos.
7. Qualquer desvio dos critérios acima exigir novo ADR ou alteração deste documento.

## Evidências e rastreabilidade

| Tipo | Localização | Evidência sustentada |
|---|---|---|
| TRANSCRICAO | `[09:04]` Bruno | Transação já pesada; HTTP síncrono travaria mudanças de status; rollback por cliente fora do ar é inaceitável (CAN-ALT-001) |
| TRANSCRICAO | `[09:06]` Diego | "Síncrono está fora de questão"; padrão outbox: mesma transação SQL, commit garante o evento, rollback o remove (CAN-DEC-001, CAN-DEC-002, CAN-RNF-008) |
| TRANSCRICAO | `[09:07]` Larissa; `[09:07]` Diego | Redis Streams exigiria infra nova; "Outbox no MySQL existente resolve" (CAN-ALT-002) |
| TRANSCRICAO | `[09:08]` Larissa; `[09:08]` Diego | Fechamento da decisão; índices e estados citados; arquivamento ~30 dias fora de escopo (CAN-DEC-001, CAN-LAC-001, CAN-FORA-003) |
| TRANSCRICAO | `[09:34]` Bruno; `[09:34]` Diego | Filtro de status aplicado na inserção, não no envio (CAN-DEC-020) |
| TRANSCRICAO | `[09:40]` Bruno; `[09:41]` Diego | Inserção na mesma transação de `changeStatus`, com rollback conjunto (CAN-RISK-005, CAN-RNF-008) |
| TRANSCRICAO | `[09:41]` Bruno; `[09:41]` Diego | Integração por `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebendo o `tx` atual (CAN-DEC-021) |
| TRANSCRICAO | `[09:52]` Larissa; `[09:52]` Diego | Snapshot do payload na inserção, preservando o estado do pedido (CAN-DEC-023, CAN-RF-018) |
| CODIGO | `src/modules/orders/order.service.ts` — `OrderService.changeStatus`, `TxClient = Prisma.TransactionClient` | Transação Prisma existente com leitura, validação, estoque, `order.update` e `orderStatusHistory.create`; ponto futuro de extensão (CAN-COD-001, CAN-INT-001) |
| CODIGO | `prisma/schema.prisma` — `datasource db`, models `Order`, `OrderStatusHistory` | Provider `mysql`; PKs `@default(uuid()) @db.Char(36)`; ausência de models de webhook e outbox (CAN-COD-002, CAN-COD-020) |
| CODIGO | `src/modules/orders/order.status.ts` — `canTransition`, `shouldDebitStock`, `shouldReplenishStock` | Máquina de transições que o módulo de webhook não deve duplicar (CAN-COD-003) |
| CODIGO | `prisma/migrations/`, `tests/setup.ts` | Migrations já inicializadas e limpeza por `beforeEach`, ambas a estender (CAN-COD-023, CAN-COD-021, CAN-INT-009) |
| MATRIZ | `MATRIZ-FATOS.md` §3 — índice ADR-001 | Conjunto canônico deste ADR: CAN-DEC-001, CAN-DEC-002, CAN-DEC-020, CAN-DEC-021, CAN-RNF-008, CAN-ALT-001, CAN-ALT-002, CAN-COD-001, CAN-INT-001 |

## Decisões relacionadas

Os ADRs abaixo estão planejados e **ainda não foram escritos** — `docs/adrs/` contém hoje apenas o `README.md`. Suas decisões não são repetidas aqui.

- [ADR-002 — Worker separado com polling](./ADR-002-worker-separado-com-polling.md) — leitura da outbox, polling, single-worker, ordenação e descarte de trigger de banco.
- [ADR-003 — Retry, backoff e DLQ](./ADR-003-retry-backoff-e-dlq.md) — retentativa, dead-letter e replay administrativo.
- [ADR-005 — Entrega at-least-once com Event ID](./ADR-005-entrega-at-least-once-com-event-id.md) — garantia de entrega e idempotência delegada ao cliente.
- [ADR-006 — Reuso dos padrões do projeto](./ADR-006-reuso-dos-padroes-do-projeto.md) — estrutura modular, convenções de erro, schemas e PK em UUID.

## Notas de implementação

- A publicação deverá receber o `Prisma.TransactionClient` já aberto por `changeStatus`; abrir uma segunda transação, ou usar o `PrismaClient` singleton de `src/config/database.ts` dentro do callback, viola o critério 1.
- A publicação deverá ocorrer após a gravação do histórico e antes do encerramento do callback, de modo que qualquer exceção anterior impeça a criação do evento.
- Erros de inserção não deverão ser capturados e silenciados no callback: é a propagação que produz o rollback conjunto exigido.
- Nenhuma tentativa de entrega, verificação de disponibilidade do cliente ou I/O de rede deverá ser adicionada ao caminho transacional.

## Histórico

| Data | Alteração | Autor |
|---|---|---|
| Não informada | Decisão tomada na reunião técnica de webhooks (`[09:06]`–`[09:08]`; ponto de integração confirmado em `[09:40]`–`[09:41]`) | Diego, Larissa, Bruno |
| Não informada | Redação inicial do ADR a partir de `TRANSCRICAO.md`, do código e de `MATRIZ-FATOS.md` | Responsável pela entrega |
</content>
