# ADR-004 — HMAC-SHA256 e gestão de secrets por endpoint

## Status

`Aceito`

- **Data da decisão:** não informada. `TRANSCRICAO.md` registra apenas "quinta-feira, 09:00", sem data de calendário.
- **Participantes:** Sofia (Engenheira de Segurança), autora da proposta e responsável pelo fechamento em `[09:22]`; Bruno (Engenheiro Pleno), que questionou o algoritmo em `[09:20]` e levantou o armazenamento em `[09:21]`; Diego (Engenheiro Sênior), que relatou o vazamento real em `[09:22]` e fixou o header em `[09:44]`; Marcos (Product Manager), que definiu em `[09:31]` a geração da secret pela plataforma; Larissa (Tech Lead), que confirmou a decisão no resumo em `[09:48]`.
- **Escopo:** como o consumidor verifica autenticidade e integridade de uma entrega, e como a credencial dessa verificação é isolada e rotacionada. Não cobre a produção do evento (ADR-001), o worker (ADR-002), retry e DLQ (ADR-003), a garantia de entrega (ADR-005) nem o reuso de padrões (ADR-006).

## Contexto

A plataforma passará a enviar requisições HTTP com dados de pedidos para endpoints fora da sua infraestrutura. Sofia abriu o tema em `[09:19]`: "o cliente tem que conseguir validar que a requisição veio realmente da gente, e que ninguém adulterou o payload no meio". São duas necessidades distintas — **origem** e **integridade** — e nenhuma delas decorre do simples fato de a requisição chegar.

HTTPS protege o transporte, mas não permite ao consumidor verificar, no nível da aplicação, que a mensagem foi produzida por quem detém uma credencial acordada: qualquer terceiro pode abrir uma conexão TLS válida contra um endpoint público e enviar um corpo bem formado. O conteúdo também não deve poder ser alterado sem invalidar a assinatura.

Uma credencial global amplificaria o impacto de um vazamento — "se vaza uma, vaza tudo" (`[09:21]` Sofia). O risco não é hipotético: a plataforma "já teve cliente que vazou secret em log de aplicação dele uma vez" (`[09:22]` Diego — CAN-RISK-003). Secrets podem ser expostas em logs e configurações inadequadas nas duas pontas, e o cliente precisa poder trocar a credencial sem interromper imediatamente a integração.

O módulo de webhooks **não existe** no repositório: não há assinatura, secret, rotação ou configuração de endpoint. `prisma/schema.prisma` modela apenas `User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory` e `OrderNumberSequence` (CAN-COD-002), e não há utilitário de HMAC (CAN-COD-019).

## Drivers da decisão

- **Autenticidade da origem**, verificável pelo consumidor (`[09:19]` Sofia — CAN-PUB-002).
- **Integridade do payload**: detectar adulteração (`[09:19]` Sofia).
- **Isolamento de credenciais por endpoint** (`[09:21]` Sofia — CAN-RNF-002).
- **Redução do blast radius de vazamento**, motivada por incidente real (`[09:22]` Diego — CAN-RISK-003).
- **Suporte operacional à rotação** pelo cliente, via API (`[09:21]` Sofia — CAN-RF-010).
- **Continuidade da integração durante a rotação**: "pra ele ter tempo de migrar os sistemas dele" (`[09:21]` Sofia).
- **Compatibilidade B2B e algoritmo amplamente suportado**: "todo cliente sério tem biblioteca pra isso" (`[09:20]` Sofia — CAN-MET-014).
- **Baixa dependência de infraestrutura adicional**, coerente com o time pequeno (`[09:07]` Diego).
- **Validação independente pelo consumidor**, sem chamada de volta à plataforma (`[09:20]` Sofia).

## Decisão

Foi decidido que:

1. Cada entrega **será assinada com HMAC-SHA256** (CAN-DEC-011, CAN-RNF-001, CAN-MET-014).
2. O HMAC **será calculado sobre o corpo do request** (`[09:22]` Sofia); ver *Representação assinada*.
3. Cada endpoint **possuirá sua própria secret** (CAN-DEC-012, CAN-RNF-002).
4. **Não será usada secret global** compartilhada entre clientes (`[09:21]` Sofia).
5. A secret **será gerada pela plataforma** no cadastro (`[09:31]` Marcos — CAN-RF-003). Geração, persistência e apresentação **não foram definidas**.
6. A assinatura **será enviada no header `X-Signature`** (`[09:20]` Sofia; `[09:44]` Diego — CAN-RF-016). Formato, encoding e eventual prefixo do valor **não foram definidos**.
7. A URL **deverá usar HTTPS**; `http` será recusada em validação (`[09:23]` Sofia — CAN-RNF-004).
8. **Haverá rotação** de secret solicitada pelo cliente via API (CAN-RF-010).
9. **Por 24 horas após a rotação**, a secret anterior e a nova permanecerão válidas em paralelo (CAN-DEC-013, CAN-MET-006).
10. **Após o prazo, a anterior será invalidada** — "Depois disso, a antiga morre" (`[09:21]` Sofia — CAN-RF-011).
11. O **consumidor será responsável por validar a assinatura** antes de confiar no conteúdo (CAN-PUB-002).
12. Secrets e assinaturas **não deverão ser expostas** em logs, traces ou erros.
13. Geração, armazenamento e apresentação **deverão ser detalhados no FDD** e passar pela revisão de segurança reservada em `[09:46]` (CAN-MET-012, CAN-LAC-003).

## Propriedades de segurança fornecidas

### Autenticidade

O consumidor que conhece a secret verifica que a assinatura foi produzida por **uma parte com acesso à mesma credencial**. A propriedade é sobre posse da secret: o HMAC **não** identifica pessoa física, usuário ou processo.

### Integridade

Qualquer modificação nos bytes assinados **deverá** produzir assinatura diferente. A propriedade só se sustenta se o consumidor validar **exatamente a mesma representação** que foi assinada.

### Isolamento

Com credencial própria por endpoint, o comprometimento de uma secret **não compromete automaticamente os demais**, e a resposta ao incidente fica restrita ao cliente afetado.

### Continuidade durante rotação

As 24 horas (CAN-MET-006) reduzem o risco de indisponibilidade enquanto o consumidor atualiza a credencial, evitando coordenação simultânea entre as pontas.

## Propriedades não fornecidas

Isoladamente, a decisão **não oferece**:

- **Confidencialidade do payload** — o HMAC assina, não cifra; confidencialidade em trânsito depende do HTTPS.
- **Garantia de entrega** e **exactly-once** — a entrega é at-least-once (ADR-005 — CAN-ALT-011).
- **Proteção completa contra replay** — mensagem íntegra pode ser reenviada. A reunião registrou o `X-Timestamp` para o cliente "detectar replay attack se quiser" (`[09:44]` Diego — CAN-RISK-009); **nenhuma política anti-replay foi decidida**.
- **Autorização de usuário final**.
- **Proteção se o consumidor armazenar mal a secret**, ou se **a plataforma registrá-la em logs**.
- **Não repúdio** — a secret é simétrica; ambas as partes podem produzir assinatura válida.
- **Identidade baseada em certificado**.
- **Segurança se a secret tiver baixa entropia** (CAN-LAC-003).
- **Qualquer proteção se o consumidor não validar a assinatura**.

## Fluxo conceitual de assinatura

1. O worker prepara o corpo final da entrega.
2. A configuração do endpoint fornece a secret ativa.
3. A plataforma calcula o HMAC-SHA256 sobre os bytes desse corpo.
4. A assinatura é incluída no header `X-Signature`.
5. A requisição é enviada por HTTPS.
6. O consumidor calcula o HMAC sobre o corpo recebido, com a secret que possui.
7. O consumidor compara as assinaturas.
8. Quando a validação falha, rejeita ou ignora a mensagem.

## Fluxo conceitual de rotação

1. O cliente solicita a rotação pela API (CAN-RF-010).
2. Uma nova secret é criada.
3. A nova secret torna-se utilizável.
4. A anterior permanece válida por 24 horas (CAN-MET-006).
5. Durante a janela, o consumidor atualiza seus sistemas.
6. Após o prazo, a anterior é invalidada (CAN-RF-011).

**Não definidos** e **não inferidos aqui**: se a nova secret passa a assinar imediatamente; se duas assinaturas são enviadas; se o servidor tenta validar ambas; se há ativação futura; se o contador de 24h inicia na solicitação, na criação ou na resposta; se a secret é exibida uma única vez; e se existe revogação manual.

## Representação assinada

A decisão diz "sobre o corpo do request" (`[09:22]` Sofia); disso decorre uma restrição arquitetural:

- A assinatura **deverá** ser calculada sobre a **representação exata enviada**.
- O corpo **não deverá** ser serializado novamente depois de assinado.
- Produtor e consumidor **precisam** operar sobre os mesmos bytes.
- Transformações intermediárias — reserialização, reordenação, reindentação, recodificação — **podem invalidar** a assinatura de mensagem legítima.
- O formato definitivo **deverá** ser especificado no FDD (CAN-INT-010).

**Não são escolhidos aqui** — nenhum foi discutido: canonicalização de JSON, ordenação de propriedades, mecanismo de serialização, encoding, prefixo, ou combinação com timestamp.

## Gestão das secrets

- Cada configuração de endpoint **possui credencial isolada** (CAN-DEC-012).
- A plataforma **precisará armazenar material suficiente para assinar**. Bruno levantou em `[09:21]` que a configuração armazena "url + secret + customer_id + estado ativo", com concordância de Sofia; o conjunto de campos **não foi fechado** (CAN-LAC-002).
- O acesso **deverá** ser restrito aos componentes que assinam ou gerenciam a rotação.
- Logs, traces e erros **não deverão** expor o valor.
- Recuperação e exibição **deverão** ser tratadas como operações sensíveis.
- A implementação **deverá** permitir a **coexistência temporária** das duas secrets (CAN-RF-011).

**Não determinados**: criptografia em repouso (CAN-LAC-004), hashing, KMS, secret manager, chave mestra, envelope encryption, tamanho, formato, encoding, reversibilidade e backup. Uma secret de endpoint é **credencial por cliente** e não se confunde com a configuração global `JWT_SECRET` de `src/config/env.ts`; nada nas fontes sustenta armazená-la como variável de ambiente.

## HTTPS obrigatório

Endpoints `http://` **deverão ser rejeitados** na validação futura: "Se o cliente cadastrar http, recusamos com erro de validação" (`[09:23]` Sofia — CAN-RNF-004). Sofia classificou o item como validação de schema, não como decisão arquitetural; ele é registrado aqui por integrar a postura de segurança da entrega.

HTTPS protege confidencialidade e integridade **em trânsito**; o HMAC complementa o transporte com verificação **no nível da aplicação**. Um não elimina a necessidade do outro. A validação **ainda não existe**, porque o módulo de webhooks não foi implementado.

**Não definidos**: versão mínima de TLS, cipher suites, mTLS, validação privada de certificados, certificados autoassinados e política de redirecionamentos.

## Ponto de integração com o sistema existente

**`src/shared/logger/index.ts` — `createLogger`, `logger`, `redactPaths`** (CAN-COD-013). Pino com `redact: { paths: redactPaths, censor: '[REDACTED]' }`. A lista atual é exatamente `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token`, `*.accessToken`.

> O logger atual já mascara tokens, senhas e cabeçalhos de autorização, mas **não há mascaramento para `secret`, `webhookSecret`, `signature`, `X-Signature` ou headers futuros do webhook**. O logger existe; a proteção específica de secrets de webhook **não está confirmada**. A futura implementação **deverá** revisar a redaction (CAN-INT-007; ANALISE §15 INT-07).

**`src/middlewares/validate.middleware.ts` — `validate`** (CAN-COD-008). Valida `body`/`query`/`params` com Zod e converte `ZodError` em `ValidationError`.

> O middleware de validação já utiliza Zod. Os **futuros** schemas de webhook **deverão** rejeitar URLs sem HTTPS (CAN-RNF-004). Essa validação **não existe** hoje.

**`src/shared/errors/app-error.ts` — `AppError`** (CAN-COD-004). Base com `statusCode`, `errorCode` e `details` opcional.

**`src/shared/errors/http-errors.ts`** (CAN-COD-005). Subclasses com códigos em maiúsculas (`ValidationError`/`VALIDATION_ERROR`, `ForbiddenError`/`FORBIDDEN`). Erros de secret e assinatura **deverão** seguir a convenção com prefixo `WEBHOOK_` (ADR-006); **nenhum existe hoje** (CAN-INT-005).

**`src/middlewares/error.middleware.ts` — `errorMiddleware`** (CAN-COD-006). Responde `{ error: { code, message, details? } }` e, no fallback, registra `logger.error({ err, requestId, method, path }, ...)` antes do 500. **Cuidado:** `details` de um erro futuro que carregue secret ou assinatura seria registrado, dada a redaction atual.

**`src/config/env.ts` — `envSchema`, `env`** (CAN-COD-014). Configuração validada com Zod (`DATABASE_URL`, `JWT_SECRET`, `JWT_EXPIRES_IN`, `LOG_LEVEL`, `NODE_ENV`, `PORT`), com `process.exit(1)` se inválida. **Não há configuração de HMAC ou de gestão de secrets.**

**`package.json`** (CAN-COD-019). `engines.node` é `>=20`, o que torna `node:crypto` disponível como builtin. **Não há biblioteca criptográfica externa**. A disponibilidade é **fato**, não decisão: o util de assinatura é proposta a especificar no FDD (CAN-INT-010; ANALISE §15 INT-10).

**`prisma/schema.prisma`** (CAN-COD-002). **Não existe configuração de webhook, campo de secret ou mecanismo de rotação.** A modelagem é integração futura (CAN-INT-002) e não é desenhada aqui.

## Alternativas consideradas

### Alternativa — Secret global compartilhada

**Descrição** — Credencial única da plataforma para assinar todas as entregas, rejeitada por Sofia em `[09:21]`: "Não é uma secret global da nossa plataforma."

**Vantagens** — Configuração simples; uma credencial a gerenciar e distribuir; nenhuma modelagem por endpoint; menor superfície de armazenamento.

**Desvantagens e trade-offs** — Impacto máximo em vazamento: uma exposição compromete todos os clientes; impossibilidade de revogar apenas um cliente; rotação vira evento coordenado com toda a base; sem isolamento, o incidente de um vira incidente de todos.

**Motivo do descarte** — "Senão se vaza uma, vaza tudo" (`[09:21]` Sofia), com o vazamento real de `[09:22]` tornando o cenário concreto.

### Alternativa — Entregas sem assinatura de aplicação

**Descrição** — Enviar apenas sobre HTTPS, sem assinar o corpo, deixando a segurança a cargo do transporte.

**Vantagens** — Implementação mais simples: nenhuma secret a gerar, proteger ou rotacionar; nada a implementar no consumidor; menor superfície de dados sensíveis.

**Desvantagens e trade-offs** — O consumidor não verificaria, no nível da aplicação, uma credencial compartilhada, e não distinguiria requisição legítima de qualquer chamada externa bem formada contra um endpoint público; adulteração por intermediário que termine o TLS ficaria indetectável.

**Motivo do descarte** — Contraria a necessidade declarada em `[09:19]` por Sofia. A alternativa **não foi debatida** na reunião — o HMAC foi apresentado diretamente como padrão em `[09:20]`; é registrada aqui como contraponto explícito da decisão.

### Alternativa — Rotação imediata sem período de convivência

**Descrição** — Invalidar a secret anterior no instante da rotação.

**Vantagens** — Menor duração da credencial anterior; revogação mais rápida; um único valor válido por endpoint, simplificando a configuração e eliminando ambiguidade sobre qual credencial assinou.

**Desvantagens e trade-offs** — Toda entrega entre a rotação e a atualização do consumidor falharia; exigiria coordenação exata; na prática desencorajaria rotacionar, enfraquecendo a resposta a vazamentos.

**Motivo da escolha por grace period de 24h** — Sofia condicionou a rotação à convivência já na proposta (`[09:21]`): a antiga "fica válida por 24 horas em paralelo, pra ele ter tempo de migrar os sistemas dele". A janela troca exposição residual limitada por continuidade (CAN-MET-006).

> Comparações com JWT, OAuth, mTLS, assinaturas assimétricas e certificados digitais **não ocorreram** na reunião e não são atribuídas a ela.

## Consequências

### Consequências positivas

- Verificação de autenticidade da origem de cada entrega.
- Detecção de alteração do payload assinado.
- Isolamento por endpoint: incidentes contidos em um cliente.
- Menor impacto de vazamento, motivação direta do incidente de `[09:22]`.
- Revogação e rotação individuais.
- Transição de credencial sem interrupção imediata.
- HMAC-SHA256 é amplamente suportado (`[09:20]` Sofia).
- Nenhuma infraestrutura criptográfica adicional obrigatória na primeira versão.

### Consequências negativas

- A plataforma precisará armazenar material capaz de assinar e mantê-lo utilizável — não basta um hash irreversível, como se faz com senha.
- Aumento da superfície de proteção de dados sensíveis em base, backups e observabilidade.
- O consumidor precisa implementar a validação; sem isso, a decisão não entrega proteção.
- Risco de divergência na serialização do corpo, invalidando mensagens legítimas.
- Convivência temporária de duas secrets válidas por endpoint.
- Complexidade adicional no modelo de configuração (anterior, atual, expiração).
- Necessidade de proteger logs e traces, hoje **não** cobertos para `secret` (CAN-COD-013).
- Risco operacional durante a rotação, se o consumidor não migrar em 24h.
- Necessidade de definir revogação, ainda inexistente.
- O HMAC **não** impede replay sozinho (CAN-RISK-009).
- O comprometimento de uma secret permite forjar mensagens **daquele** endpoint enquanto ela valer.

### Consequências neutras ou operacionais

- Criação futura de estrutura para secrets e rotação (CAN-INT-002).
- Criação futura do endpoint de rotação (CAN-RF-010).
- Atualização dos schemas Zod para exigir HTTPS (CAN-RNF-004).
- Inclusão de redaction adicional no logger (CAN-INT-007).
- Documentação e exemplos de validação para consumidores (CAN-INT-010).
- Testes de assinatura e de rotação.
- Revisão de segurança antes do deploy, com dois dias úteis reservados (`[09:46]` Sofia — CAN-MET-012).

## Riscos e cuidados

| Risco ou cuidado | Impacto | Tratamento ou estado |
| ---------------- | ------- | -------------------- |
| Secret exposta em log da plataforma | Comprometimento do endpoint; forja de mensagens | `secret` **não** consta da redaction atual (CAN-COD-013, CAN-INT-007). `Em aberto para detalhamento no FDD` |
| Secret exposta em trace ou telemetria | Vazamento por canal secundário | `Em aberto para detalhamento no FDD` |
| Secret devolvida indevidamente em listagens | Exposição a qualquer usuário autenticado | Contrato de leitura não definido. `Em aberto para detalhamento no FDD` |
| Baixa entropia na geração | Assinatura forjável | Geração é lacuna (CAN-LAC-003); revisão exigida em `[09:46]`. `Em aberto para detalhamento no FDD` |
| Assinatura calculada sobre corpo diferente do enviado | Falha de validação em mensagens legítimas | Restrição em *Representação assinada*; formato definitivo no FDD |
| Comparação insegura de assinaturas no consumidor | Vazamento por canal de tempo | Fora do controle da plataforma. `Em aberto para detalhamento no FDD` |
| Uso de secret incorreta na assinatura | Entregas rejeitadas pelo cliente | Seleção da secret ativa não definida. `Em aberto para detalhamento no FDD` |
| Rotação incompleta (cliente não migra em 24h) | Interrupção da integração após a janela | Grace de 24h (CAN-MET-006); alerta ao cliente não definido. `Em aberto para detalhamento no FDD` |
| Secret anterior mantida além de 24h | Janela de exposição maior que a decidida | Invalidação é decisão (CAN-RF-011); mecanismo de expiração não definido. `Em aberto para detalhamento no FDD` |
| Invalidação prematura da anterior | Perda de entregas dentro da janela prometida | Instante inicial das 24h não definido. `Em aberto para detalhamento no FDD` |
| Acesso excessivo ao material secreto | Ampliação do raio de comprometimento interno | Menor privilégio como diretriz; controle não definido. `Em aberto para detalhamento no FDD` |
| Backup contendo secrets | Vazamento fora do runtime | Armazenamento em repouso é lacuna (CAN-LAC-004). `Em aberto para detalhamento no FDD` |
| Consumidor não validar a assinatura | Decisão não entrega proteção | Responsabilidade do consumidor (CAN-PUB-002); documentação no README/FDD |
| Tentativa de replay contra o endpoint do cliente | Reprocessamento de mensagem legítima | HMAC não trata replay; `X-Timestamp` permite detecção "se quiser" (`[09:44]` Diego — CAN-RISK-009). `Em aberto para detalhamento no FDD` |
| Timestamp não verificado pelo consumidor | Replay indetectado | Verificação não é exigida por esta decisão. `Em aberto para detalhamento no FDD` |
| Endpoint configurado com HTTP | Tráfego e payload expostos em trânsito | Rejeição em validação é decisão (CAN-RNF-004); schema ainda não existe |
| Assinatura registrada em log | Facilita análise por terceiro com acesso ao log | Não coberta pela redaction atual. `Em aberto para detalhamento no FDD` |
| Resposta de erro expondo secret ou assinatura | Vazamento pelo contrato de erro | `errorMiddleware` devolve `details` quando presente (CAN-COD-006). `Em aberto para detalhamento no FDD` |

## Lacunas de implementação

Os pontos abaixo **não são definidos** por esta decisão e não devem ser inferidos a partir dela:

- Tamanho, entropia mínima, encoding e formato da secret; algoritmo ou API de geração (CAN-LAC-003).
- Armazenamento em repouso: criptografia, chave mestra, KMS ou secret manager (CAN-LAC-004).
- Se a secret é exibida uma única vez ou pode ser recuperada depois.
- Conteúdo da resposta de criação e da resposta de rotação.
- Versionamento de secrets e representação da secret anterior.
- Instante exato de início do grace period; mecanismo de expiração e invalidação.
- Revogação emergencial fora do fluxo de rotação.
- Formato, encoding e eventual prefixo do valor de `X-Signature`.
- Uso do timestamp no cálculo da assinatura.
- Política anti-replay e tolerância de relógio.
- Comparação em tempo constante; tratamento de múltiplas assinaturas.
- Canonicalização ou serialização definitiva do corpo.
- Proteção de secrets e assinaturas nos logs (CAN-INT-007).
- Política de backup e retenção de material secreto.
- Auditoria da rotação e autorização para executá-la — o CRUD permanece aberto a qualquer role autenticada por ora (`[09:37]` Sofia — CAN-FORA-004).
- Comportamento quando a rotação é solicitada novamente durante o grace period.
- Campos definitivos da configuração de webhook (CAN-LAC-002).

## Critérios de conformidade arquitetural

Uma implementação futura estará em conformidade com este ADR se:

1. Cada endpoint possuir **sua própria secret**.
2. **Não existir secret global** compartilhada.
3. Cada entrega for assinada com **HMAC-SHA256**.
4. A assinatura for calculada sobre os **bytes efetivamente enviados**.
5. A entrega ocorrer sobre **HTTPS**, e URLs `http` forem recusadas na validação.
6. Existir **operação de rotação** acionável pelo cliente.
7. A secret anterior permanecer válida por **24 horas**, conforme a decisão.
8. Após o prazo, a anterior **deixar de ser válida**.
9. Secrets e assinaturas **não aparecerem** em logs, traces ou respostas de erro.
10. O consumidor receber informação suficiente para **validar de forma independente**.
11. O formato definitivo for **documentado no FDD**, não decidido silenciosamente no código.
12. A implementação **não prometer** proteção completa contra replay apenas com HMAC.
13. Qualquer alteração de algoritmo, isolamento ou grace period exigir **revisão deste ADR**.
14. A adoção de mecanismo diferente exigir **novo ADR ou atualização formal**.

## Evidências e rastreabilidade

| Tipo | Localização | Evidência sustentada |
| ---- | ----------- | -------------------- |
| TRANSCRICAO | `[09:19]` Sofia | Necessidade de validar origem e detectar adulteração (CAN-PUB-002) |
| TRANSCRICAO | `[09:20]` Sofia; `[09:20]` Bruno | HMAC como padrão; assinatura com secret compartilhada; header `X-Signature`; escolha de SHA-256 (CAN-DEC-011, CAN-RNF-001, CAN-MET-014, CAN-RF-016) |
| TRANSCRICAO | `[09:21]` Sofia | Secret por endpoint; rejeição de secret global — "se vaza uma, vaza tudo"; rotação via API; antiga válida 24h; "a antiga morre" (CAN-DEC-012, CAN-RNF-002, CAN-RF-010, CAN-RF-011, CAN-DEC-013, CAN-MET-006) |
| TRANSCRICAO | `[09:21]` Bruno | Configuração armazena `url + secret + customer_id + estado ativo`, com concordância de Sofia; campos não fechados (CAN-LAC-002) |
| TRANSCRICAO | `[09:22]` Diego; `[09:22]` Sofia | Vazamento real de secret em log de cliente; fechamento — "HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h" (CAN-RISK-003) |
| TRANSCRICAO | `[09:23]` Sofia | TLS obrigatório; `http` recusado; classificado por ela como validação de schema (CAN-RNF-004) |
| TRANSCRICAO | `[09:31]` Marcos | Secret gerada pela plataforma e devolvida na criação (CAN-RF-003) |
| TRANSCRICAO | `[09:44]` Diego | `X-Signature` com o HMAC; `X-Timestamp` para o cliente "detectar replay attack se quiser" (CAN-RF-016, CAN-RF-017, CAN-RISK-009) |
| TRANSCRICAO | `[09:46]` Sofia | Dois dias úteis de revisão: "HMAC e geração de secret eu quero olhar com calma" (CAN-MET-012, CAN-LAC-003) |
| TRANSCRICAO | `[09:48]` Larissa | Resumo confirmado: "HMAC-SHA256 sobre payload, secret por endpoint, rotação com grace period de 24h" |
| CODIGO | `src/shared/logger/index.ts` — `createLogger`, `logger`, `redactPaths` | Pino com `redact` e censor `[REDACTED]`; cobre `authorization`, `cookie`, `*.password`, `*.passwordHash`, `*.token`, `*.accessToken`; **`secret` e `signature` ausentes** (CAN-COD-013) |
| CODIGO | `src/config/env.ts` — `envSchema`, `env` | Configuração Zod; `JWT_SECRET` é credencial global da API, distinta da secret por endpoint; nenhuma configuração de HMAC (CAN-COD-014) |
| CODIGO | `src/middlewares/validate.middleware.ts` — `validate` | Padrão Zod de `body`/`query`/`params` a reutilizar na exigência de HTTPS (CAN-COD-008) |
| CODIGO | `src/shared/errors/app-error.ts` — `AppError` | Base com `statusCode`, `errorCode`, `details` (CAN-COD-004) |
| CODIGO | `src/shared/errors/http-errors.ts` — `ValidationError`, `ConflictError` | Convenção de subclasses e códigos em maiúsculas (CAN-COD-005) |
| CODIGO | `src/middlewares/error.middleware.ts` — `errorMiddleware` | Resposta `{ error: { code, message, details? } }`; fallback registra o erro com `logger.error` (CAN-COD-006) |
| CODIGO | `package.json` — `engines`, `dependencies` | `node >= 20`, `node:crypto` builtin; nenhuma biblioteca criptográfica externa (CAN-COD-019) |
| CODIGO | `prisma/schema.prisma` — models existentes | Ausência de configuração de webhook, secret e rotação (CAN-COD-002, CAN-INT-002) |
| MATRIZ | `MATRIZ-FATOS.md` §3 — índice ADR-004 | Conjunto canônico: CAN-DEC-011, CAN-DEC-012, CAN-DEC-013, CAN-RF-010, CAN-RF-011, CAN-RF-016, CAN-RNF-001, CAN-RNF-002, CAN-RNF-003, CAN-RNF-004, CAN-MET-006, CAN-MET-014, CAN-COD-013, CAN-INT-007, CAN-INT-010 |
| ANALISE | `ANALISE-EVIDENCIAS.md` §15 INT-07; §15 INT-10 | Redaction de `secret` como ponto de atenção; util HMAC-SHA256 a especificar no FDD |
| ANALISE | `ANALISE-EVIDENCIAS.md` §19 LAC-03; §19 LAC-04 | Geração/entropia da secret e criptografia em repouso são lacunas de alto impacto |
| ADR | [ADR-003](./ADR-003-retry-backoff-e-dlq.md) — Classificação das falhas | Uma entrega assinada pode falhar e ser retentada; a política de retry independe da assinatura |

## Decisões relacionadas

ADR-001 a ADR-003 já existem em `docs/adrs/`; ADR-005 e ADR-006 estão **planejados e ainda não foram escritos**, com nomes já referenciados pelos ADRs anteriores.

- [ADR-001 — Transactional Outbox no MySQL para eventos de webhook](./ADR-001-outbox-no-mysql.md) — persistência atômica do evento cujo corpo será assinado.
- [ADR-002 — Worker separado com polling para processamento da outbox](./ADR-002-worker-separado-com-polling.md) — o componente que assina e envia.
- [ADR-003 — Retry com backoff e Dead-Letter Queue para entregas de webhook](./ADR-003-retry-backoff-e-dlq.md) — o que ocorre quando a entrega assinada falha.
- [ADR-005 — Entrega at-least-once com Event ID](./ADR-005-entrega-at-least-once-com-event-id.md) — garantia de entrega e idempotência; a assinatura não altera essa semântica.
- [ADR-006 — Reuso dos padrões do projeto](./ADR-006-reuso-dos-padroes-do-projeto.md) — convenções de erro, logger, schemas Zod e estrutura modular.

## Notas de implementação

- O corpo **deverá** ser finalizado antes do cálculo da assinatura, e enviado sem transformação posterior.
- O valor da secret **nunca deverá** ir a logs, traces ou erros — inclusive via `details` de erros que estendam `AppError`.
- A documentação para o consumidor **deverá** explicitar que a validação usa a **mesma representação** assinada.
- O acesso às secrets **deverá** seguir o princípio do menor privilégio.
- A rotação **deverá** ser auditável.
- A implementação **deverá** ter testes com payload modificado, secret incorreta, secret anterior dentro da janela e secret anterior expirada.
- A revisão de segurança reservada em `[09:46]` **deverá** preceder o deploy.

## Histórico

| Data | Alteração | Autor |
| ---- | --------- | ----- |
| Não informada | Decisão tomada na reunião técnica de webhooks (`[09:19]`–`[09:23]`; geração da secret em `[09:31]`; header em `[09:44]`; resumo em `[09:48]`) | Sofia, Diego, Bruno, Marcos, Larissa |
| Não informada | Redação inicial do ADR a partir de `TRANSCRICAO.md`, do código e de `MATRIZ-FATOS.md` | Responsável pela entrega |
