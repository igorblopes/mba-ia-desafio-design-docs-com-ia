# ADR-005 — Entrega at-least-once com identificação estável de evento

## Status

`Aceito`

- **Data da decisão:** não informada. `TRANSCRICAO.md` registra apenas "quinta-feira, 09:00".
- **Participantes:** Diego (Engenheiro Sênior), autor da proposta em `[09:24]`, do `X-Event-Id` e do descarte de exactly-once em `[09:25]`; Bruno (Engenheiro Pleno), que perguntou como o consumidor diferenciaria as entregas em `[09:25]`; Sofia (Engenheira de Segurança), que registrou a transferência de responsabilidade em `[09:25]`; Marcos (Product Manager), que assumiu a documentação pública em `[09:26]`; Larissa (Tech Lead), que fechou a decisão em `[09:26]`, confirmou-a em `[09:48]` e fixou o padrão UUID em `[09:51]`.
- **Escopo:** qual semântica de entrega é oferecida e como uma entrega repetida do mesmo evento pode ser reconhecida. Não cobre a produção do evento (ADR-001), o worker (ADR-002), retry/DLQ/replay (ADR-003), a segurança do envio (ADR-004) nem o reuso de padrões (ADR-006).

## Contexto

A plataforma passará a entregar eventos de mudança de status a sistemas de terceiros. Rede, processo emissor e receptor falham de forma independente, e o resultado de uma tentativa nem sempre é conclusivo: a mensagem pode ter sido recebida e processada sem que a plataforma obtenha confirmação.

O ADR-003 decidiu retentativa com backoff, DLQ ao esgotar as tentativas e replay administrativo. Retentativa e replay reintroduzem o mesmo evento no fluxo; quando a tentativa anterior foi de fato processada, o reenvio produz entrega repetida.

Eliminar essa repetição exigiria coordenação entre as duas pontas — "Garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo" (`[09:25]` Diego). A plataforma não participa da transação interna do cliente e não confirma atomicamente o efeito de negócio do outro lado. Resta declarar uma semântica honesta e dar ao consumidor o instrumento para reconhecer repetições — "E como ele diferencia?" (`[09:25]` Bruno).

O repositório **não implementa** webhooks: não há outbox, worker, tabela de tentativas, identificador de evento nem headers de webhook (CAN-COD-002).

## Drivers da decisão

- **Evitar perda silenciosa** após falha inconclusiva (`[09:15]` Diego — CAN-RISK-001).
- **Permitir retentativa** sem exigir certeza sobre a tentativa anterior (CAN-RNF-010).
- **Permitir replay administrativo** (CAN-RF-009).
- **Viabilidade entre sistemas independentes**: nenhuma ponta controla a transação da outra (`[09:25]` Diego).
- **Mecanismo explícito de deduplicação** para o consumidor (CAN-DEC-015).
- **Compatibilidade com outbox, worker, retry e DLQ** (ADR-001 a ADR-003).
- **Transparência contratual** aos clientes (`[09:26]` Marcos).
- **Reuso do padrão UUID** — "Tudo é uuid" (`[09:51]` Larissa — CAN-DEC-022, CAN-COD-020).
- **Simplicidade relativa na primeira versão**, coerente com o time pequeno (`[09:07]` Diego).

## Decisão

Foi decidido que:

1. A plataforma oferecerá semântica **at-least-once** (`[09:24]` Diego; `[09:26]` e `[09:48]` Larissa — CAN-DEC-014, CAN-RNF-009).
2. Um mesmo evento lógico **poderá ser entregue mais de uma vez** (`[09:24]` Diego — CAN-RISK-002).
3. Cada evento possuirá **identificador estável, único por evento**, no formato **UUID** (CAN-RF-014, CAN-RF-015).
4. O identificador será **criado quando o evento for persistido na outbox** — "um UUID gerado quando o evento entra na outbox" (`[09:25]` Diego).
5. Será enviado no header **`X-Event-Id`** em cada tentativa (`[09:25]`, `[09:44]` Diego).
6. Retentativas do mesmo evento **deverão preservar sua identidade**; sem isso o header não cumpre a função decidida.
7. O **consumidor deduplicará** — "ele dedupica pelo event_id do lado dele" (`[09:25]` Diego — CAN-DEC-015).
8. A plataforma **não oferecerá exactly-once** (CAN-ALT-011).
9. A **documentação pública deverá explicitar** a semântica (`[09:26]` Marcos).
10. A **geração concreta do UUID deverá ser definida no FDD**; nenhuma biblioteca ou API foi escolhida.

**Ponto não fechado — identidade no replay.** As fontes decidem que o replay "recoloca na outbox como pendente" (`[09:18]` Diego — CAN-RF-009), mas **não definem** se o item conserva o `event_id` original, recebe um novo ou constitui novo evento lógico correlacionado. A escolha altera o que o consumidor observa e **não é decidida aqui**: o **FDD deverá** documentar a regra, distinguindo replay do mesmo evento lógico da criação de evento novo.

## Definições conceituais

**Evento lógico** — o fato de domínio ocorrido (ex.: o pedido mudou de `PAID` para `PROCESSING`), com identidade própria expressa pelo `event_id`.

**Tentativa de entrega** — execução concreta de envio a um endpoint. Um evento **pode ter várias tentativas**; tentativa e evento **não são a mesma entidade**.

**Duplicidade** — o consumidor recebe mais de uma entrega do **mesmo evento lógico**.

**Deduplicação** — processo pelo qual o consumidor reconhece que aquele `event_id` já foi processado e evita reaplicar os efeitos de negócio. O algoritmo interno é dele e **não é definido aqui**.

## Semântica at-least-once

Enquanto o evento permanecer no fluxo de entrega, o mecanismo de processamento **poderá** realizar múltiplas tentativas e produzir entregas duplicadas. A plataforma prioriza não abandonar silenciosamente uma entrega inconclusiva; a duplicidade é consequência aceita, e o `X-Event-Id` é o meio de reconhecê-la.

A expressão **não significa** que toda entrega sempre ocorrerá. A garantia se lê dentro dos limites do ADR-003: esgotadas as tentativas, o evento vai para a DLQ (CAN-DEC-007, CAN-RF-013); indisponibilidade permanente do endpoint impede a entrega; e o retorno de um item da DLQ ao fluxo depende de ação operacional (CAN-DEC-010).

## Por que duplicidades podem ocorrer

Cenários típicos de sistemas distribuídos — exemplos conceituais, **não** incidentes observados neste sistema: o consumidor processa a mensagem e a resposta se perde; a conexão é encerrada após o processamento e antes da confirmação; o timeout expira enquanto o consumidor conclui a operação; o worker reinicia após enviar e antes de persistir o resultado; um evento em DLQ é reprocessado por replay; uma falha operacional deixa a tentativa inconclusiva.

A estratégia de claim/lock das linhas pelo worker, inclusive em restart, **não está definida** (CAN-LAC-006) e influencia a frequência desses casos.

## Identidade do evento

- O `event_id` identifica o **evento lógico**, não a tentativa, e **não deve mudar** a cada retentativa.
- **Não substitui a assinatura HMAC** e **não estabelece autenticidade**: qualquer terceiro pode enviar um header arbitrário. Autenticidade e integridade são do ADR-004.
- **Não garante ordenação** — não há ordering global (CAN-RNF-013); a ordem por `order_id` é intenção sob single-worker, não garantia formal (CAN-RNF-014).
- **Não informa, sozinho, a versão do payload**, e **não deve ser reutilizado** para eventos lógicos diferentes.
- No **replay**, a preservação da identidade permanece **em aberto** (ver *Decisão*).

## UUID como formato

O formato escolhido é **UUID** (`[09:25]` Diego — CAN-RF-015), coerente com a convenção vigente: todos os models de domínio de `prisma/schema.prisma` — `User`, `Customer`, `Product`, `Order`, `OrderItem` e `OrderStatusHistory` — usam `String @id @default(uuid()) @db.Char(36)` (CAN-COD-002, CAN-COD-020). A única exceção é `OrderNumberSequence`, tabela de sequência de linha única com PK `Int @id @default(1)`, cuja natureza não é a de uma entidade de domínio. Larissa fixou o padrão para as novas tabelas em `[09:51]` (CAN-DEC-022). O formato dispensa gerador central sequencial e é adequado à propagação entre sistemas independentes.

Este ADR **não escolhe biblioteca**. A dependência `uuid` 11.0.3 constar de `package.json` **não a torna obrigatória**; `engines.node` é `>=20`, o que torna `crypto.randomUUID()` tecnicamente disponível, mas **isso não é decisão deste ADR** (CAN-COD-019). Versão do UUID e método de geração **deverão** ser detalhados no FDD. O UUID **não elimina colisões de forma absoluta**; torna-as improváveis.

## Responsabilidade do consumidor

A transferência foi explicitada na reunião — "Isso joga responsabilidade pro cliente" (`[09:25]` Sofia), com concordância de Diego. O consumidor **deverá**: ler o `X-Event-Id`; registrar os IDs processados ou usar mecanismo equivalente; verificar o ID antes de aplicar efeitos não idempotentes; tratar o recebimento repetido como comportamento possível, não anomalia; definir sua própria política de retenção; combinar deduplicação com validação da assinatura (ADR-004); e não assumir que cada request representa um evento novo.

**Não são determinados aqui**: banco, cache, tabela, algoritmo, duração da retenção ou comportamento após detectar duplicidade — responsabilidade do consumidor e da documentação de integração.

## Responsabilidade da plataforma

A plataforma **deverá**: gerar identidade estável no nascimento do evento; persisti-la junto ao evento na outbox; enviar o ID em todas as tentativas; não reutilizá-lo para eventos distintos; documentar a semântica at-least-once (`[09:26]` Marcos); registrar as tentativas, sustentando o histórico de `GET /webhooks/:id/deliveries` (CAN-RF-008); manter correlação entre evento, tentativas e DLQ; e preservar rastreabilidade em retentativa e replay conforme o FDD definir. O schema correspondente **não é desenhado aqui** (CAN-INT-002, CAN-LAC-001).

## Fluxo conceitual

1. Uma transição válida de status gera um evento lógico.
2. O evento recebe um UUID.
3. Evento e ID são persistidos na outbox, na transação da mudança de status (ADR-001).
4. O worker inicia uma tentativa de entrega (ADR-002).
5. O ID segue no header `X-Event-Id`.
6. Se a entrega for inconclusiva, o mesmo evento poderá ser tentado novamente (ADR-003).
7. O consumidor verifica se aquele ID já foi processado.
8. Se já foi, evita repetir os efeitos.
9. Se não foi, valida e aplica o evento.
10. O fluxo segue a política de retry, DLQ e replay do ADR-003.

## Limites da garantia

A decisão **não oferece**: exactly-once; ausência de duplicidade; transação distribuída entre plataforma e consumidor; idempotência automática; ordenação global ou derivada do ID; entrega indefinida no tempo; recuperação após expiração ou remoção dos dados; confirmação do efeito de negócio interno do consumidor; proteção criptográfica; confidencialidade; proteção contra falsificação.

Fronteiras: autenticidade e integridade no **ADR-004**; retentativa, DLQ, replay e timeout no **ADR-003**; processamento fora do ciclo de request no **ADR-002**; atomicidade da criação do evento no **ADR-001**.

## Ponto de integração com o sistema existente

**`src/modules/orders/order.service.ts` — `OrderService.changeStatus`** (CAN-COD-001). Hoje, em `this.prisma.$transaction`, valida a transição, ajusta estoque, executa `tx.order.update` e cria `tx.orderStatusHistory`.

> O método **ainda não cria eventos** e **não produz `X-Event-Id`**. A futura integração transacional do ADR-001, via `publishWebhookEvent(tx, order, fromStatus, toStatus)` (CAN-DEC-021, CAN-INT-001), **deverá** gerar e persistir a identidade do evento junto à outbox.

**`prisma/schema.prisma`** (CAN-COD-002, CAN-COD-020).

> O schema Prisma atual utiliza IDs UUID armazenados como `Char(36)`. Essa convenção sustenta o formato escolhido para o identificador do evento, mas **não define a biblioteca responsável pela geração**. **Não existem** model de outbox, campo `eventId` nem tabela de tentativas.

**`src/modules/orders/order.status.ts` — `canTransition`, `shouldDebitStock`, `shouldReplenishStock`** (CAN-COD-003). Delimita quando existe evento lógico: `changeStatus` rejeita transições inválidas com `InvalidStatusTransitionError` antes de qualquer efeito.

**`package.json`** (CAN-COD-019). `uuid` 11.0.3 em `dependencies` e `engines.node` `>=20` são **fatos de disponibilidade, não decisões**.

**Headers.** Nenhum header de webhook existe no repositório. `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id` (`[09:44]` Diego e Sofia — CAN-RF-017) são **futuros**; só o primeiro pertence a este ADR.

## Alternativas consideradas

### Alternativa — Entrega exactly-once

**Descrição** — Assegurar que cada evento seja entregue e processado exatamente uma vez.

**Vantagens** — Nenhuma duplicidade; experiência aparentemente mais simples para o consumidor, dispensado de deduplicar.

**Desvantagens e trade-offs** — Exigiria coordenação entre sistemas independentes e, potencialmente, transação distribuída ou protocolo de confirmação bilateral; a plataforma não confirma atomicamente o efeito de negócio dentro do sistema do cliente, de modo que a garantia dependeria de comportamento fora do seu controle; complexidade desproporcional; e o rótulo criaria falsa sensação de garantia, desestimulando proteções que o consumidor ainda precisaria manter.

**Motivo do descarte** — `[09:25]` Diego: "Garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com event_id resolve 99% dos casos" (CAN-ALT-011), apoiado na prática de mercado citada na reunião — "Stripe faz assim, GitHub faz assim".

### Alternativa — Entrega at-most-once

**Descrição** — Tentar a entrega uma única vez e desistir em caso de falha.

> **Não foi discutida na reunião.** Registrada como contraponto arquitetural analisado durante a redação; **não é atribuída aos participantes**.

**Vantagens** — Nenhuma duplicidade por retentativa; consumidor sem obrigação de deduplicar; implementação trivial.

**Desvantagens e trade-offs** — Perda definitiva após uma única falha, inclusive transitória de rede; nenhuma recuperação automática; incompatível com a preocupação declarada em `[09:14]`–`[09:16]` com clientes indisponíveis por horas.

**Motivo do descarte** — Inadequada ao requisito de confiabilidade que motivou retry e DLQ (ADR-003); contraria a decisão de `[09:24]`.

### Alternativa — Retry sem identidade estável

**Descrição** — Manter as retentativas sem enviar identificador de evento.

**Vantagens** — Aparentemente mais simples: nada a gerar, persistir ou propagar; um header a menos no contrato.

**Desvantagens e trade-offs** — O consumidor não distinguiria com segurança retentativa de evento novo, sobretudo em mudanças de status próximas no tempo; a deduplicação exigiria inferência pelo conteúdo, frágil e específica de cada cliente; maior risco de efeitos repetidos; suporte e auditoria ficariam sem chave de correlação entre as pontas.

**Motivo do descarte** — Responde à pergunta de Bruno em `[09:25]`: sem identidade estável, at-least-once transfere ao cliente um problema sem instrumento. Daí o `X-Event-Id` (CAN-RF-014, CAN-DEC-015).

## Consequências

### Consequências positivas

- Retentativa possível após falhas inconclusivas, reduzindo a chance de perda silenciosa.
- Semântica clara e declarada; duplicidades identificáveis por chave estável.
- Rastreabilidade entre tentativas do mesmo evento e auditoria do seu ciclo de vida.
- Compatibilidade com DLQ e replay (ADR-003).
- Baixo acoplamento: nenhuma ponta participa da transação da outra, sem protocolo distribuído adicional.

### Consequências negativas

- Consumidores precisam implementar deduplicação e armazenar ou reconhecer IDs processados; sem isso, a decisão não os protege.
- Mensagens duplicadas passam a ser comportamento esperado, não incidente.
- Operações não idempotentes exigem cuidado explícito; o replay pode repetir efeitos (CAN-RISK-002).
- Retenção insuficiente dos IDs permite reprocessar evento antigo.
- Suporte e observabilidade precisam distinguir evento de tentativa.
- A documentação precisa ser inequívoca: "at-least-once" é frequentemente lido como garantia absoluta.
- A plataforma não controla nem verifica a implementação do consumidor: erros dele produzem efeitos duplicados mesmo com o `event_id` correto.
- O UUID não fornece autenticação nem qualquer propriedade de segurança.

### Consequências neutras ou operacionais

- Criação futura do campo de identificador e inclusão do header no envio (CAN-INT-002, CAN-RF-014).
- Correlação a modelar entre outbox, tentativas e DLQ (CAN-LAC-001, CAN-LAC-005).
- Testes de retentativa verificando preservação do ID, e de replay após a regra ser fechada.
- Documentação de integração e exemplos para consumidores (`[09:26]` Marcos).
- Definição de retenção nos dois lados — na plataforma, ainda em aberto (CAN-LAC-009).

## Riscos e cuidados

| Risco ou cuidado | Impacto | Tratamento ou estado |
| ---------------- | ------- | -------------------- |
| Novo ID a cada retentativa | Retentativa lida como evento novo; dedup inócua | Contraria a decisão: o ID nasce na inserção (`[09:25]` Diego). Critério nº 4 |
| Mesmo ID reutilizado para eventos diferentes | Evento legítimo descartado como duplicado | Unicidade por evento é decisão (CAN-RF-014); unicidade no banco `Em aberto para detalhamento no FDD` |
| Consumidor ignora o header | Efeitos aplicados em duplicidade | Fora do controle da plataforma; documentação pública (`[09:26]` Marcos) |
| Consumidor retém IDs por tempo insuficiente | Reprocessamento de evento antigo, ex.: após replay | `Em aberto para detalhamento na documentação do consumidor` |
| Consumidor marca o ID antes de concluir, ou conclui e falha ao registrá-lo | Evento perdido, ou reprocessado na entrega seguinte | Ordem interna é do consumidor. `Em aberto para detalhamento na documentação do consumidor` |
| Replay produz efeitos duplicados | Efeito de negócio repetido no cliente | Consequência aceita do at-least-once (ADR-003); identidade no replay `Em aberto para detalhamento no FDD` |
| Divergência entre `event_id` do payload e `X-Event-Id` do header | Dedup por chaves distintas; suporte sem correlação | O payload inclui `event_id` (CAN-DEC-024); a correspondência **não foi definida**. `Em aberto para detalhamento no FDD` |
| Perda de correlação ao mover para a DLQ | Item da DLQ não ligável ao evento e às tentativas | Esquema da DLQ não detalhado (CAN-DEC-009, CAN-LAC-001). `Em aberto para detalhamento no FDD` |
| At-least-once lido como garantia absoluta | Expectativa incorreta; cliente sem plano para DLQ | Seção *Semântica at-least-once*; documentação pública (`[09:26]` Marcos) |
| UUID malformado, ou colisão | Chave rejeitada; evento distinto tratado como duplicado | Colisão é improvável, não impossível; geração não definida (CAN-COD-019). `Em aberto para detalhamento no FDD` |
| Logs e traces sem o identificador | Diagnóstico sem correlação entre tentativas | `Em aberto para detalhamento no FDD` |
| `X-Event-Id` usado como prova de autenticidade | Falsa confiança; header forjável | O ID **não** autentica; autenticidade é do ADR-004 |
| Confusão entre evento lógico e tentativa | Métricas e suporte incorretos; dedup pela chave errada | Seção *Definições conceituais*; modelagem `Em aberto para detalhamento no FDD` |
| Worker reinicia após enviar e antes de registrar o resultado | Reenvio do mesmo evento | Claim/lock não definido (CAN-LAC-006). `Em aberto para detalhamento no FDD` |

## Lacunas de implementação

**Não definidos** por esta decisão, e não inferíveis a partir dela:

- Versão do UUID, método de geração, biblioteca ou API (CAN-COD-019).
- Localização do identificador no schema, tipo físico, unicidade e indexação (CAN-INT-002, CAN-LAC-001).
- Correspondência entre o `event_id` do payload (CAN-DEC-024) e o header `X-Event-Id`.
- Regra de identidade no replay: preservação, novo ID ou novo evento lógico correlacionado (CAN-RF-009).
- Correlação entre identificador do evento e da tentativa (CAN-LAC-005).
- Retenção de eventos, tentativas e DLQ (CAN-LAC-009); retenção recomendada ao consumidor.
- Comportamento do consumidor ao detectar duplicidade e código HTTP correspondente.
- Exposição do ID no histórico de entregas e estratégia de auditoria (CAN-RF-008).
- Propagação do identificador para logs e traces.
- Comportamento quando o header estiver ausente ou malformado.
- Regra para eventos reconstruídos ou reenviados manualmente fora do replay.
- Tratamento de colisão, comportamento em migrações (CAN-LAC-012) e versionamento do contrato.

## Critérios de conformidade arquitetural

Uma implementação futura estará em conformidade se:

1. A semântica pública declarada for **at-least-once**.
2. A **duplicidade** for reconhecida como possível, não tratada como defeito.
3. Cada evento lógico possuir **identificador UUID estável**.
4. Retentativas **não criarem nova identidade** sem justificativa arquitetural registrada.
5. **`X-Event-Id` for enviado em toda tentativa**.
6. IDs **não forem reutilizados** para eventos lógicos diferentes.
7. O consumidor for **informado de sua responsabilidade** de deduplicar.
8. A plataforma **não prometer exactly-once** em documentação, contrato ou código.
9. A identidade for **persistida de forma compatível** com outbox, tentativas e DLQ.
10. O evento só for criado **após transição válida** de status.
11. O ID **não for tratado como mecanismo de autenticação**.
12. A **regra de replay for documentada no FDD** antes da implementação.
13. Qualquer mudança da semântica exigir **revisão deste ADR**; qualquer promessa de exactly-once exigir **novo ADR**.

## Evidências e rastreabilidade

| Tipo | Localização | Evidência sustentada |
| ---- | ----------- | -------------------- |
| TRANSCRICAO | `[09:24]` Diego | "a gente vai garantir at-least-once. Pode acontecer de o cliente receber o mesmo evento duas vezes." (CAN-DEC-014, CAN-RNF-009, CAN-RISK-002) |
| TRANSCRICAO | `[09:25]` Bruno | "E como ele diferencia?" — origem da necessidade de identidade estável |
| TRANSCRICAO | `[09:25]` Diego | `event_id` no header `X-Event-Id`; UUID gerado quando o evento entra na outbox; único por evento; dedup pelo cliente; descarte de exactly-once por exigir coordenação bilateral, com Stripe e GitHub citados (CAN-RF-014, CAN-RF-015, CAN-DEC-015, CAN-ALT-011) |
| TRANSCRICAO | `[09:25]` Sofia | "Isso joga responsabilidade pro cliente" (CAN-DEC-015) |
| TRANSCRICAO | `[09:26]` Marcos; `[09:26]` Larissa | Documentação no portal do desenvolvedor; fechamento — "At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão." (CAN-DEC-014) |
| TRANSCRICAO | `[09:44]` Diego | `X-Event-Id` com o UUID entre os headers de envio (CAN-RF-014, CAN-RF-017) |
| TRANSCRICAO | `[09:48]` Larissa | Resumo: "Idempotência por X-Event-Id, garantia at-least-once" |
| TRANSCRICAO | `[09:51]` Larissa | "UUID, segue o padrão do resto do projeto. Tudo é uuid" (CAN-DEC-022) |
| TRANSCRICAO | `[09:18]` Diego | Replay "recoloca na outbox como pendente"; identidade do item reintroduzido não especificada (CAN-RF-009) |
| CODIGO | `prisma/schema.prisma` — models `Order`, `OrderStatusHistory` | PKs `String @id @default(uuid()) @db.Char(36)`; ausência de outbox, campo `eventId` e tabela de tentativas (CAN-COD-002, CAN-COD-020) |
| CODIGO | `src/modules/orders/order.service.ts` — `OrderService.changeStatus` | Transação com `order.update` e `orderStatusHistory.create`; nenhuma criação de evento nem header (CAN-COD-001, CAN-INT-001) |
| CODIGO | `src/modules/orders/order.status.ts` — `canTransition`, `shouldDebitStock` | Transições válidas que delimitam quando existe evento lógico (CAN-COD-003) |
| CODIGO | `package.json` — `dependencies`, `engines` | `uuid` 11.0.3 instalado; `node >= 20` torna `crypto.randomUUID()` disponível; nenhuma decisão de implementação (CAN-COD-019) |
| MATRIZ | `MATRIZ-FATOS.md` §3 — índice ADR-005 | CAN-DEC-014, CAN-DEC-015, CAN-RF-014, CAN-RF-015, CAN-RNF-009, CAN-ALT-011, CAN-RISK-002 |
| ANALISE | `ANALISE-EVIDENCIAS.md` §19 LAC-06 | Claim/lock do worker não definido; afeta duplicidade (CAN-LAC-006) |
| ADR | [ADR-003](./ADR-003-retry-backoff-e-dlq.md) — Replay administrativo | Retentativas e replays "poderão produzir entregas duplicadas"; consistente com esta decisão |

## Decisões relacionadas

ADR-001 a ADR-004 existem em `docs/adrs/`; ADR-006 está **planejado e ainda não foi escrito**.

- [ADR-001 — Transactional Outbox no MySQL para eventos de webhook](./ADR-001-outbox-no-mysql.md) — momento em que o evento nasce e recebe sua identidade.
- [ADR-002 — Worker separado com polling para processamento da outbox](./ADR-002-worker-separado-com-polling.md) — executa as tentativas; ordenação e single-worker.
- [ADR-003 — Retry com backoff e Dead-Letter Queue para entregas de webhook](./ADR-003-retry-backoff-e-dlq.md) — retentativa, DLQ e replay, origens da duplicidade.
- [ADR-004 — HMAC-SHA256 e gestão de secrets por endpoint](./ADR-004-hmac-sha256-e-gestao-de-secrets.md) — autenticidade e integridade; o `X-Event-Id` não cumpre esse papel.
- [ADR-006 — Reuso dos padrões do projeto](./ADR-006-reuso-dos-padroes-do-projeto.md) — convenções do projeto, incluindo PK em UUID.

## Notas de implementação

- O identificador **deverá** ser gerado **uma única vez**, no nascimento do evento, e **persistido antes** de qualquer tentativa.
- O mesmo identificador **deverá** ser propagado em todas as retentativas.
- Logs, tentativas e itens da DLQ **deverão** ser correlacionáveis pelo identificador.
- Cenários de timeout, reenvio e reinício do worker **deverão** ser testados verificando a estabilidade do ID.
- A documentação do consumidor **deverá** afirmar de forma inequívoca que entregas repetidas ocorrem e que a deduplicação é responsabilidade dele.
- A regra de identidade no replay **deverá** ser fechada no FDD antes da implementação.

## Histórico

| Data | Alteração | Autor |
| ---- | --------- | ----- |
| Não informada | Decisão tomada na reunião técnica de webhooks (`[09:24]`–`[09:26]`; header em `[09:44]`; resumo em `[09:48]`; padrão UUID em `[09:51]`) | Diego, Bruno, Sofia, Marcos, Larissa |
| Não informada | Redação inicial do ADR a partir de `TRANSCRICAO.md`, do código e de `MATRIZ-FATOS.md` | Responsável pela entrega |
