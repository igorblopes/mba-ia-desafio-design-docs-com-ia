# ADR-006 — Reuso dos padrões arquiteturais existentes no módulo de webhooks

## Status

`Aceito`

- **Data da decisão:** não informada. `TRANSCRICAO.md` registra apenas "quinta-feira, 09:00", sem data de calendário.
- **Participantes:** Bruno (Engenheiro Pleno, Pedidos), autor da proposta de módulo em `[09:27]`, da convenção de erros em `[09:28]` e do reuso do Pino e do middleware de erro em `[09:29]`; Diego (Engenheiro Sênior, Plataforma), que concordou em `[09:28]` e levantou a infraestrutura compartilhada em `[09:29]`; Larissa (Tech Lead), que fixou o prefixo `WEBHOOK_` em `[09:29]`, fechou a decisão em `[09:30]` ("reuso máximo do que já existe"), confirmou-a em `[09:48]` e fixou o padrão UUID em `[09:51]`; Sofia (Engenheira de Segurança), quanto ao papel de ADMIN no replay em `[09:36]`.
- **Escopo:** como a feature de webhooks é incorporada à aplicação existente — organização modular, composição, roteamento, autenticação, validação, erros, logging e testes. Não cobre outbox (ADR-001), worker e polling (ADR-002), retry/DLQ (ADR-003), HMAC e secrets (ADR-004) nem semântica de entrega (ADR-005).

## Contexto

A aplicação já possui organização modular estabelecida: cada domínio é um diretório em `src/modules` com routes, controller, service, repository e schemas (CAN-COD-009 a CAN-COD-012). Existem mecanismos compartilhados para autenticação e autorização, validação, erros, logging e composição manual de dependências (CAN-COD-004 a CAN-COD-008, CAN-COD-013, CAN-COD-017), todos referenciados na seção *Evidências e rastreabilidade*.

A feature de webhooks introduzirá superfície de API, persistência, processamento assíncrono e mecanismos de segurança. Seria tecnicamente possível implementá-la de forma isolada, com estrutura, contratos de erro e autenticação próprios. Uma arquitetura paralela aumentaria a inconsistência entre módulos e o custo de manutenção, agravado pelo tamanho do time — "a gente é um time pequeno" (`[09:07]` Diego).

Bruno partiu do padrão observado: "Cada domínio é um módulo em src/modules com controller, service, repository, routes e schemas. Webhook vai seguir igual" (`[09:27]`). Larissa fechou: "reuso máximo do que já existe" (`[09:30]` — CAN-DEC-016).

O objetivo **não** é redesenhar a aplicação. O worker terá lifecycle separado (ADR-002), mas deverá reutilizar os componentes compatíveis. Nenhum artefato de webhook existe no repositório: não há `src/modules/webhooks`, entry-point de worker, script `worker`, models de webhook (CAN-COD-002) nem erros `WEBHOOK_*` (CAN-INT-005).

## Drivers da decisão

- **Consistência entre módulos** e previsibilidade de leitura (`[09:27]` Bruno — CAN-RNF-018).
- **Redução de duplicação** de autenticação, validação, erros e logging (`[09:30]` Larissa).
- **Adequação ao tamanho do time** e menor curva de aprendizado (`[09:07]` Diego).
- **Reuso de mecanismos já testados**: o middleware central "vai pegar nossos erros sem precisar mudar nada" (`[09:29]` Bruno — CAN-RNF-017).
- **Uniformidade dos contratos de erro**, via `AppError` e códigos com prefixo (`[09:28]` Bruno; `[09:29]` Larissa — CAN-RNF-015, CAN-RNF-020).
- **Uniformidade de autenticação e validação**, incluindo `requireRole` já existente (`[09:36]` Larissa — CAN-DEC-010).
- **Observabilidade consistente**: Pino "já tá no projeto inteiro. Não vamos botar nada novo" (`[09:29]` Bruno — CAN-RNF-016).
- **Velocidade de entrega** sob prazo de três sprints (`[09:46]` Larissa — CAN-MET-011).
- **Facilidade de testes**, aproveitando setup e factories existentes (CAN-COD-021).
- **Menor risco de criar uma segunda arquitetura** dentro da mesma aplicação.

## Decisão

Foi decidido que:

1. A feature será incorporada seguindo a **organização modular já utilizada** (`[09:27]` Bruno; `[09:30]` Larissa — CAN-RNF-018).
2. O módulo **deverá ser criado em `src/modules/webhooks`** (CAN-DEC-027).
3. Routes, controller, service, repository e schemas **deverão seguir as responsabilidades observadas** nos módulos existentes.
4. As dependências **deverão ser compostas em `src/app.ts`**, no ponto de composição manual atual (CAN-COD-017).
5. As rotas **deverão ser registradas em `src/routes/index.ts`**, sob o prefixo global `/api/v1` já existente (CAN-COD-018, CAN-INT-003).
6. O CRUD de configuração **deverá reutilizar `authenticate`**; por ora qualquer role autenticada basta (`[09:37]` Sofia — CAN-FORA-004).
7. O replay administrativo **deverá reutilizar `requireRole('ADMIN')`** — "a gente reaproveita o requireRole que já existe" (`[09:36]` Larissa — CAN-DEC-010).
8. Os contratos **deverão ser validados por Zod** através do middleware `validate` existente (CAN-RNF-019).
9. Os erros **deverão usar `AppError`** e o middleware centralizado, sem alteração deste (CAN-RNF-015, CAN-RNF-017).
10. Códigos específicos **deverão usar o prefixo `WEBHOOK_`** — "Prefixo WEBHOOK_ pra tudo do módulo" (`[09:29]` Larissa — CAN-RNF-020).
11. O **logger compartilhado deverá ser reutilizado**, na API e no worker (CAN-RNF-016).
12. A **redaction deverá ser revisada** para proteger secrets e assinaturas (CAN-INT-007).
13. O processo do worker **deverá reutilizar** logger, configuração e padrões de lifecycle compatíveis (ADR-002).
14. O módulo **não deverá duplicar regras do domínio de pedidos**; a máquina de estados permanece em `src/modules/orders/order.status.ts` (CAN-COD-003).
15. As novas tabelas **deverão usar PK em UUID** — "Tudo é uuid" (`[09:51]` Larissa — CAN-DEC-022).
16. Novos padrões **só deverão ser introduzidos** quando os existentes forem inadequados, com o desvio documentado.

Este ADR decide a **integração arquitetural, não a implementação**. Os arquivos citados como futuros **ainda não existem**. Seguir o padrão **não significa copiar cegamente**: diferenças justificadas — como o processo separado do worker — são permitidas.

## Princípios de integração

- **Consistência antes de novidade** — atendendo a necessidade, o mecanismo existente deverá ser reutilizado.
- **Responsabilidades preservadas** — cada camada mantém responsabilidades equivalentes às observadas.
- **Regras de domínio permanecem no domínio** — o módulo não reimplementará transições de status, estoque ou regras de pedidos.
- **Mecanismos transversais permanecem centralizados** — autenticação, autorização, validação, erros e logging não serão duplicados.
- **Extensão localizada** — a feature estende os pontos de composição existentes, sem refatoração global.
- **Exceções documentadas** — quando o padrão não atender, o desvio deverá ser registrado e justificado.

Estes princípios descrevem convenções observadas neste repositório; **não** afirmam adesão a um estilo arquitetural formal.

## Arquitetura existente observada

O fluxo predominante é:

`Router → Middleware → Controller → Service → Repository/Prisma`

- **Routers** definem endpoints e middlewares (`buildOrderRouter` aplica `router.use(authenticate)` e `validate(...)` por rota).
- **Controllers** adaptam HTTP: handlers `RequestHandler` com `try/catch` que encaminham `next(err)` (CAN-COD-010).
- **Services** concentram fluxos e regras de aplicação.
- **Repositories** encapsulam **parte** do acesso aos dados.
- **Prisma** é o mecanismo de persistência; `src/app.ts` compõe manualmente; `src/routes/index.ts` agrega os módulos; middlewares compartilhados tratam aspectos transversais.

**Ressalvas — o padrão não é absoluto:**

- `OrderService.changeStatus` acessa `this.prisma.$transaction` **diretamente**, operando `tx.order.update` e `tx.orderStatusHistory.create` sem passar pelo repository (CAN-COD-001). Nem todo acesso a Prisma ocorre em repository.
- `buildOrderRouter` aplica `authenticate` via `router.use`, enquanto `buildUserRouter` o aplica **por rota**, junto de `requireRole('ADMIN')` (CAN-COD-022). Há mais de uma forma vigente.
- O módulo `auth` não possui repository próprio.

O repositório **não declara** Clean Architecture, Onion, Hexagonal, Ports and Adapters, DDD, CQRS ou Event Sourcing, e nenhuma evidência sustenta esses rótulos. Este ADR não os atribui nem os introduz.

## Estrutura futura do módulo

Em nível conceitual:

```text
src/modules/webhooks/
├── webhook.routes.ts
├── webhook.controller.ts
├── webhook.service.ts
├── webhook.repository.ts
└── webhook.schemas.ts
```

- **`webhook.routes.ts`** — deverá definir rotas, aplicar autenticação, autorização quando necessária e validação, e encaminhar ao controller.
- **`webhook.controller.ts`** — deverá tratar aspectos HTTP, extrair dados validados, chamar o service e delegar erros ao middleware via `next`.
- **`webhook.service.ts`** — deverá coordenar casos de uso e regras da feature, sem assumir transporte HTTP e sem duplicar regras de pedidos.
- **`webhook.repository.ts`** — deverá encapsular operações de persistência da configuração e demais dados quando adequado, seguindo as convenções reais e sem abstrações artificiais.
- **`webhook.schemas.ts`** — deverá declarar schemas Zod para body, params e query, incluir a exigência de HTTPS (`[09:23]` Sofia — CAN-RNF-004) e reutilizar enums existentes quando apropriado, como `OrderStatus` no filtro de eventos.

A estrutura foi decidida **em alto nível**. Nomes internos adicionais pertencem ao FDD. O arquivo de processamento do worker permanece **em aberto** entre `webhook.worker.ts` e `webhook.processor.ts` (`[09:28]` Bruno — CAN-OPEN-002); a divisão exata entre repositories (configuração, outbox, DLQ, deliveries) **não está definida**.

## Composição de dependências

`src/app.ts` já instancia repositories, services e controllers em `buildControllers` e devolve o objeto consumido por `buildApiRouter` (CAN-COD-017). O futuro repository, service e controller de webhooks **deverão** ser instanciados nesse mesmo ponto; o tipo `Controllers` de `src/routes/index.ts` **deverá** ser estendido; e o router **deverá** receber o controller necessário.

**Não será introduzido framework ou container de injeção de dependências** para esta feature: seria desproporcional ao escopo e criaria dois modelos de composição coexistindo. A composição do worker **poderá** ocorrer em seu próprio entry point, reutilizando os componentes do módulo. Construtores e interfaces concretos **não são definidos aqui**.

## Roteamento

As novas rotas **deverão** ser agregadas em `src/routes/index.ts`, preservando o prefixo global `/api/v1` aplicado em `src/app.ts`. As rotas de configuração e de replay **deverão** usar os middlewares atuais. O endpoint administrativo **não deverá** criar mecanismo de autorização paralelo. Os caminhos completos — incluindo `POST /admin/webhooks/dead-letter/:id/replay` (`[09:35]` Diego — CAN-RF-009) e a forma de montá-lo sob o prefixo — pertencem ao FDD.

## Autenticação e autorização

`authenticate` continuará sendo o mecanismo padrão: valida o bearer JWT e popula `req.user` com `id`, `email` e `role` (`ADMIN` | `OPERATOR`). `requireRole('ADMIN')` **deverá** proteger o replay (`[09:36]` Larissa e Sofia — CAN-DEC-010), com precedente direto em `buildUserRouter` (CAN-COD-022). O CRUD de configuração seguirá a regra decidida: autenticado, sem restrição de role por ora (`[09:37]` Sofia).

O `customer_id` **não deve ser derivado do JWT atual**, pois o token representa usuário operador, não o cliente — "o customer_id é passado no body ou no path. Não vem do JWT" (`[09:32]` Larissa — CAN-DEC-019). Se será body ou path permanece **em aberto** (CAN-OPEN-001). **Não será criado** outro sistema de roles para webhooks.

## Validação

Zod e o middleware `validate` **deverão** ser reutilizados, mantendo o formato atual de validação de body, params e query e a conversão de `ZodError` em `ValidationError` (CAN-COD-008). A exigência de URL HTTPS será uma validação futura de schema — Sofia a classificou como validação, não decisão arquitetural (`[09:23]`). Os schemas de webhook **ainda não existem** e não são criados aqui. A validação **não deverá** ser duplicada manualmente no controller.

## Logging e proteção de dados sensíveis

O Pino existente **deverá** ser reutilizado, na API e no worker, mantendo o estilo e a estrutura atuais de logs. `secret` e assinatura **não devem** ser registradas.

A `redact` atual cobre `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token` e `*.accessToken` — **`secret` e assinatura não estão na lista** (CAN-COD-013). Ampliá-la é **necessidade futura, não comportamento existente** (CAN-INT-007). Payload de evento e responses de terceiros podem conter informação sensível e exigem cuidado equivalente. A política completa de logging pertence ao FDD e ao ADR-004.

## Configuração

`src/config/env.ts` centraliza a configuração validada com Zod, com falha explícita na inicialização (CAN-COD-014). Novas variáveis, **quando necessárias**, deverão seguir o mesmo padrão. **Não foi decidido** que todos os valores da feature serão variáveis de ambiente: se intervalo de polling e timeout serão constantes ou env permanece pendente (CAN-INT-008). Secrets por endpoint são dados persistidos por cadastro e **não devem** ser tratadas como secret global de ambiente (ADR-004). Nenhuma variável nova é inventada aqui.

## Worker e lifecycle

O worker será um **processo separado** (ADR-002) e **não será inicializado** como parte do servidor HTTP — "não pode ser o mesmo processo" (`[09:11]` Diego). **Poderá** reutilizar logger, `env` e a fábrica de `PrismaClient` de `src/config/database.ts`, abrindo instância própria por ser outro processo Node (`[09:30]` Bruno — CAN-DEC-017).

Deverá seguir lifecycle coerente com `src/server.ts` — bootstrap, log de início, tratamento de `SIGINT`/`SIGTERM` e `prisma.$disconnect` no shutdown (CAN-COD-016) — em entry point próprio, referido na reunião como `src/worker.ts` com script `npm run worker` (`[09:11]` Larissa — CAN-DEC-028). Esse script **não existe** em `package.json` hoje. O loop de polling não é detalhado aqui. O lifecycle distinto **não invalida** a decisão de reuso: é a exceção justificada.

## Testes

O projeto já usa Vitest e Supertest, com `getTestApp`/`bootstrapAuthenticatedUser` e limpeza de tabelas em `beforeEach` (CAN-COD-021). Os futuros testes HTTP **deverão** seguir esse padrão; as factories **poderão** ser ampliadas; `tests/setup.ts` **deverá** considerar as novas tabelas, respeitando as relações na ordem de limpeza (CAN-INT-009). Testes do worker **poderão** exigir estratégias adicionais, por ser processo separado — a forma **não está definida**. **Não será criado** framework de testes paralelo.

## Alternativas consideradas

### Alternativa — Criar uma arquitetura isolada para webhooks

**Descrição** — Implementar a feature com estrutura, autenticação, validação, erros e logging próprios, independentes do restante da aplicação.

**Vantagens** — Liberdade para desenhar estrutura específica ao processamento assíncrono; possibilidade de experimentar novos padrões sem impactar módulos existentes.

**Desvantagens e trade-offs** — Duplicaria autenticação, contratos de erro, validação e logging, exigindo correções em dois lugares; elevaria a curva de aprendizado num time pequeno; dificultaria a integração com `changeStatus` e com a composição existente; e cada mecanismo duplicado seria menos exercitado que o original.

**Motivo do descarte** — Contraria a decisão de `[09:30]` (Larissa, "reuso máximo do que já existe" — CAN-DEC-016) e a proposta de `[09:27]` (Bruno, "Webhook vai seguir igual").

### Alternativa — Acessar Prisma diretamente nos controllers

**Descrição** — Dispensar service e repository, consultando Prisma a partir dos handlers HTTP.

**Vantagens** — Menos arquivos; implementação inicial aparentemente mais rápida.

**Desvantagens e trade-offs** — Misturaria transporte HTTP e persistência; dificultaria testar regras sem subir requisições; divergiria dos módulos existentes, nos quais controllers apenas adaptam HTTP (CAN-COD-010); e a lógica da feature — filtro de eventos, rotação de secret, replay — não caberia em handlers.

**Motivo do descarte** — Incompatível com CAN-RNF-018 e com o padrão observado em `src/modules/*`.

### Alternativa — Criar um novo formato de erros para webhooks

**Descrição** — Definir envelope e middleware de erro exclusivos da feature.

**Vantagens** — Contratos específicos, moldados ao vocabulário de webhooks.

**Desvantagens e trade-offs** — Produziria duas formas de erro na mesma API sob o mesmo prefixo; exigiria middleware adicional; aumentaria a complexidade para clientes e manutenção; e perderia o tratamento já existente de `AppError`, `ZodError` e erros conhecidos do Prisma.

**Motivo do descarte** — `[09:29]` Bruno: o middleware central "vai pegar nossos erros sem precisar mudar nada". A distinção necessária é obtida pelo prefixo `WEBHOOK_` nos códigos (CAN-RNF-020), sem novo envelope.

### Alternativa — Introduzir um framework de injeção de dependências apenas para a feature

> **Não foi discutida na reunião.** Registrada como alternativa arquitetural analisada durante a redação; **não é atribuída aos participantes**.

**Descrição** — Adotar container de DI para compor o módulo e o worker.

**Vantagens** — Composição automática; recursos de escopo e ciclo de vida.

**Desvantagens e trade-offs** — Faria coexistir dois modelos de composição, já que `buildControllers` permanece manual; aumentaria a complexidade e a curva de aprendizado; e representaria mudança desproporcional ao escopo de uma feature.

**Motivo do descarte** — A composição manual atual atende. Adotar DI é decisão que caberia ao projeto inteiro, em ADR próprio, não como efeito colateral desta feature.

### Alternativa — Refatorar toda a aplicação antes de implementar webhooks

> **Não foi discutida na reunião.** Registrada como alternativa de governança analisada durante a redação.

**Descrição** — Padronizar as divergências existentes — acesso direto a Prisma em `changeStatus`, formas distintas de aplicar `authenticate` — antes de iniciar a feature.

**Vantagens** — Padronização global; base uniforme para o novo módulo.

**Desvantagens e trade-offs** — Atrasaria a entrega frente ao prazo de três sprints (CAN-MET-011) e ao risco comercial (CAN-RISK-008); aumentaria o risco sobre código em produção; e ampliaria o escopo muito além do necessário.

**Motivo do descarte** — Nenhuma fonte a exige, e o padrão atual já sustenta a feature. Refatoração geral **não é condição** para implementar webhooks.

## Consequências

### Consequências positivas

- Consistência entre módulos: quem conhece `orders` reconhece `webhooks`.
- Menor duplicação de mecanismos transversais e manutenção centralizada.
- Reuso de mecanismos já exercitados em produção e em testes.
- Respostas de erro uniformes, sem alterar o middleware central.
- Autenticação e autorização uniformes, com precedente real de rota ADMIN.
- Menor curva de aprendizado e revisão de código mais simples.
- Integração localizada nos pontos de composição existentes.
- Rastreabilidade entre documentação, decisões e código.
- Testes e factories reaproveitáveis.

### Consequências negativas

- O novo módulo **herdará as limitações da arquitetura atual**, inclusive as divergências registradas em *Arquitetura existente observada*.
- A composição manual poderá crescer: `src/app.ts` e `src/routes/index.ts` ficarão maiores a cada componente adicionado.
- O repository poderá não cobrir todos os acessos, sobretudo os transacionais — `changeStatus` já demonstra a exceção (CAN-COD-001), e `publishWebhookEvent(tx, ...)` recebe o `tx` diretamente (CAN-DEC-021).
- O worker exigirá adaptações: padrões desenhados para o ciclo de request não cobrem processamento contínuo fora dele.
- A redaction atual é **insuficiente** para secrets e assinaturas e precisará ser tratada antes de registrar objetos de webhook.
- Mudanças futuras em componentes compartilhados poderão afetar o módulo.
- O ADR reduz a liberdade de projetar uma solução independente e obriga decisões locais a respeitar convenções que não foram desenhadas para webhooks.

### Consequências neutras ou operacionais

- Criação futura do diretório `src/modules/webhooks` e alterações em `src/app.ts` e `src/routes/index.ts` (CAN-INT-003).
- Novos schemas Zod e novas classes de erro `WEBHOOK_*` (CAN-INT-005).
- Atualização da lista de `redact` (CAN-INT-007).
- Novo entry point e novo script no `package.json` (CAN-INT-006).
- Ampliação de `tests/setup.ts` e das factories (CAN-INT-009).
- Documentação das novas convenções para o time.

## Riscos e cuidados

| Risco ou cuidado | Impacto | Tratamento ou estado |
| ---------------- | ------- | -------------------- |
| Copiar padrões sem avaliar adequação ao assíncrono | Estrutura inadequada ao worker | Princípio *Exceções documentadas*; desvio justificado é permitido |
| Duplicar regras de pedidos no módulo | Duas fontes da verdade para transições | Proibido pela Decisão nº 14; máquina de estados em `order.status.ts` (CAN-COD-003) |
| Lógica de negócio nos controllers | Divergência do padrão; dificuldade de teste | Critério de conformidade nº 3 |
| Acesso ao Prisma espalhado pelo módulo | Persistência difusa, difícil de evoluir | Convenção de repository, com exceção transacional reconhecida. Divisão exata `Em aberto para detalhamento no FDD` |
| Crescimento excessivo de `src/app.ts` | Composição extensa e ruidosa | Consequência aceita; sem DI. Reorganização eventual exigiria decisão própria |
| Divergência de padrões entre API e worker | Duas formas de logar, configurar e tratar erro | Decisão nº 13; critério nº 12 |
| Novo formato de erro introduzido acidentalmente | Contrato inconsistente para clientes | Middleware central permanece único; alternativa descartada |
| Logs expondo `secret` ou assinatura | Vazamento de credencial, com precedente citado (`[09:22]` Diego) | Redaction insuficiente hoje (CAN-COD-013); ampliação é CAN-INT-007. `Em aberto para detalhamento no FDD` |
| Validações duplicadas no controller | Regras divergentes entre schema e handler | Validação centralizada em `webhook.schemas.ts` |
| Rota administrativa sem `requireRole('ADMIN')` | Replay de DLQ acessível a operador | Decisão nº 7; critério nº 6 (CAN-DEC-010) |
| Módulo acoplado ao Express em todas as camadas | Service inutilizável pelo worker, que não tem request | Service sem responsabilidade de transporte; limite service/processor `Em aberto para detalhamento no FDD` |
| Abstração excessiva (interfaces sem uso real) | Complexidade sem benefício | Princípio *Consistência antes de novidade* |
| Abstração insuficiente (tudo no service) | Camadas indistintas | Responsabilidades equivalentes às observadas |
| Testes frágeis por cleanup incorreto | Falhas intermitentes por FK ou dados residuais | Ordem de limpeza deverá respeitar relações (CAN-INT-009). `Em aberto para detalhamento no FDD` |
| Ciclo de dependências entre `orders` e `webhooks` | Import circular; fronteiras erodidas | Função recebendo `tx` evita injetar repository (CAN-DEC-021). Localização `Em aberto para detalhamento no FDD` |
| Worker importar o bootstrap HTTP | Servidor iniciado por engano no processo do worker | Critério nº 12; nota de implementação correspondente |
| Comportamento futuro apresentado como existente | Documentação enganosa; decisões sobre base falsa | Artefatos futuros escritos no futuro neste ADR |
| Introdução gradual de uma segunda arquitetura | Erosão silenciosa da consistência | Decisão nº 16; critério nº 16 |

## Lacunas de implementação

**Não definidos** por esta decisão:

- Nomes definitivos de arquivos internos adicionais; escolha entre `webhook.worker.ts` e `webhook.processor.ts` (CAN-OPEN-002).
- Divisão exata entre repositories de configuração, outbox, DLQ e deliveries.
- Localização de `publishWebhookEvent` e assinatura da função transacional (CAN-DEC-021, CAN-INT-001).
- Tipos compartilhados entre API e worker, e forma de compartilhar código entre os dois processos.
- Composição específica do worker e limite entre service e processor.
- Lista definitiva de erros `WEBHOOK_*` (CAN-INT-005).
- Extensão exata da redaction (CAN-INT-007).
- Novas variáveis de ambiente, se houver (CAN-INT-008).
- Forma de testar o worker; factories de webhook; atualização exata do cleanup (CAN-INT-009).
- Organização dos utilitários de HMAC (CAN-INT-010) e localização dos tipos do payload — não definida em nenhuma fonte; o conteúdo do payload é decidido em CAN-DEC-024, não sua tipagem no código.
- Uso de interfaces ou sua ausência; nível aceitável de acesso direto ao Prisma.
- Health checks e métricas do worker; supervisão em produção (CAN-LAC-010).
- Auditoria do replay (`[09:36]` Sofia — CAN-RNF-006) e contratos completos dos endpoints.

## Critérios de conformidade arquitetural

Uma implementação futura estará em conformidade se:

1. O módulo estiver **integrado à estrutura existente**, em `src/modules/webhooks`.
2. As novas rotas usarem o router agregador e os middlewares atuais, sob o prefixo global.
3. Controllers **não** acessarem persistência nem transporte externo sem justificativa registrada.
4. As validações usarem **Zod** e o middleware compartilhado.
5. Os erros derivarem de **`AppError`** e saírem no formato do middleware central.
6. O replay usar **`requireRole('ADMIN')`**.
7. Os logs usarem o **logger compartilhado**.
8. **Secrets e assinaturas não aparecerem em logs**.
9. As dependências forem compostas nos **pontos previstos**.
10. As **regras de transição permanecerem** no módulo de pedidos.
11. O worker **não iniciar o servidor HTTP**.
12. O worker reutilizar componentes compatíveis **sem importar o bootstrap da API**.
13. **Não** existir segundo formato de autenticação, erro ou validação.
14. Os **desvios estiverem documentados**.
15. Uma **refatoração geral não for condição** para a entrega.
16. Qualquer adoção de arquitetura paralela exigir **revisão deste ADR**.

## Evidências e rastreabilidade

| Tipo | Localização | Evidência sustentada |
| ---- | ----------- | -------------------- |
| TRANSCRICAO | `[09:27]` Bruno | "Cada domínio é um módulo em src/modules com controller, service, repository, routes e schemas. Webhook vai seguir igual"; proposta de `src/modules/webhooks` (CAN-DEC-027, CAN-RNF-018) |
| TRANSCRICAO | `[09:28]` Bruno; `[09:28]` Diego | Entry separada e processamento dentro do módulo; nome em aberto entre `webhook.worker.ts` e `webhook.processor.ts` (CAN-DEC-029, CAN-OPEN-002) |
| TRANSCRICAO | `[09:28]` Bruno; `[09:29]` Larissa | `AppError` e subclasses com códigos; "Prefixo WEBHOOK_ pra tudo do módulo" (CAN-RNF-015, CAN-RNF-020) |
| TRANSCRICAO | `[09:29]` Bruno | Pino "já tá no projeto inteiro"; middleware central "vai pegar nossos erros sem precisar mudar nada" (CAN-RNF-016, CAN-RNF-017) |
| TRANSCRICAO | `[09:30]` Larissa; `[09:48]` Larissa | "reuso máximo do que já existe. AppError, Pino, error middleware, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro"; confirmado no resumo (CAN-DEC-016) |
| TRANSCRICAO | `[09:07]` Diego | "a gente é um time pequeno" — driver de consistência e simplicidade |
| TRANSCRICAO | `[09:36]` Larissa; `[09:36]` Sofia | "role ADMIN obrigatório no replay e a gente reaproveita o requireRole que já existe"; auditoria do replay (CAN-DEC-010, CAN-RNF-006) |
| TRANSCRICAO | `[09:32]` Bruno; `[09:32]` Larissa; `[09:37]` Sofia | `customer_id` no body ou path, não do JWT; CRUD por ora com qualquer role autenticada (CAN-DEC-019, CAN-OPEN-001, CAN-FORA-004) |
| TRANSCRICAO | `[09:23]` Sofia | URL `https` obrigatória — "é só uma validação no schema Zod" (CAN-RNF-004) |
| TRANSCRICAO | `[09:40]`/`[09:41]` Bruno; `[09:41]` Diego | Integração em `changeStatus` via `publishWebhookEvent(tx, ...)`; "função pura recebendo o tx" (CAN-DEC-021, CAN-INT-001) |
| TRANSCRICAO | `[09:11]` Larissa; `[09:11]`/`[09:30]` Bruno | `src/worker.ts` "tipo o que a gente já tem em src/server.ts", script `npm run worker`, `PrismaClient` próprio (CAN-DEC-028, CAN-DEC-017) |
| TRANSCRICAO | `[09:51]` Larissa | "UUID, segue o padrão do resto do projeto. Tudo é uuid" (CAN-DEC-022) |
| CODIGO | `src/app.ts` — `buildControllers`, `buildApp` | Composição manual de repositories/services/controllers; `express.json({ limit: '1mb' })`; `/api/v1`; `NotFoundError` de rota; `errorMiddleware` ao fim. **Nenhum componente de webhook registrado** (CAN-COD-017) |
| CODIGO | `src/routes/index.ts` — `Controllers`, `buildApiRouter` | Tipo de controllers e agregação por prefixo de domínio; padrão de inclusão de novos módulos (CAN-COD-018) |
| CODIGO | `src/modules/orders/order.routes.ts` — `buildOrderRouter` | `router.use(authenticate)`, `validate({ params, body, query })` e ligação ao controller (CAN-COD-009) |
| CODIGO | `src/modules/users/user.routes.ts` — `buildUserRouter` | `authenticate` + `requireRole('ADMIN')` por rota — precedente do replay e evidência de convenção não uniforme (CAN-COD-022) |
| CODIGO | `src/modules/orders/order.controller.ts` — `OrderController` | Handlers `RequestHandler` com `try/catch` e `next(err)` (CAN-COD-010) |
| CODIGO | `src/modules/orders/order.service.ts` — `changeStatus` | Regras em service; `this.prisma.$transaction` **direto**, sem repository; ponto futuro de publicação; **nenhuma publicação de webhook hoje** (CAN-COD-001) |
| CODIGO | `src/modules/orders/order.repository.ts` — `OrderRepository`; `src/modules/orders/order.schemas.ts` — `createOrderSchema`, `orderIdParamSchema` | Encapsulamento de consultas Prisma e limite real da abstração; Zod com `z.string().uuid()`, `z.nativeEnum(OrderStatus)` e `z.infer` (CAN-COD-011, CAN-COD-012) |
| CODIGO | `src/middlewares/auth.middleware.ts` — `authenticate`, `requireRole` | JWT bearer; `req.user` com `id`/`email`/`role`; roles `ADMIN` e `OPERATOR` (CAN-COD-007) |
| CODIGO | `src/middlewares/validate.middleware.ts` — `validate` | Validação genérica de `body`/`query`/`params` com Zod; `ZodError` → `ValidationError` (CAN-COD-008) |
| CODIGO | `src/shared/errors/app-error.ts` — `AppError`; `src/shared/errors/http-errors.ts`; `src/shared/errors/index.ts` | `statusCode`, `errorCode`, `details`; subclasses e convenção de códigos (`INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`); exportação centralizada (CAN-COD-004, CAN-COD-005) |
| CODIGO | `src/middlewares/error.middleware.ts` — `errorMiddleware` | `AppError`, `ZodError`, Prisma `P2002`/`P2025`, fallback 500 com `logger.error`; envelope `{ error: { code, message, details? } }` (CAN-COD-006) |
| CODIGO | `src/shared/logger/index.ts` — `redactPaths`, `createLogger` | Pino com nível por `env.LOG_LEVEL`; redaction **sem** `secret` e **sem** assinatura (CAN-COD-013) |
| CODIGO | `src/config/env.ts`; `src/config/database.ts` | Configuração validada por Zod, **sem variáveis de webhook**; `createPrismaClient`/`prisma` disponíveis ao worker (CAN-COD-014, CAN-COD-015) |
| CODIGO | `src/server.ts` — `bootstrap` | `listen`, log de início, `SIGINT`/`SIGTERM`, `prisma.$disconnect` — referência de lifecycle (CAN-COD-016) |
| CODIGO | `prisma/schema.prisma`; `package.json` — `scripts`, `dependencies` | MySQL, PKs `@default(uuid()) @db.Char(36)` nos models de domínio (`OrderNumberSequence` usa PK `Int`), `@@map` snake_case, **nenhum model de webhook**; Vitest, Supertest, Zod, Pino e `uuid` disponíveis, **script `worker` inexistente** (CAN-COD-002, CAN-COD-020, CAN-COD-019) |
| CODIGO | `tests/setup.ts`; `tests/orders.test.ts`; `tests/helpers/factories.ts` | `deleteMany` em `beforeEach` respeitando relações; Supertest com `Authorization: Bearer`; `getTestApp`, `bootstrapAuthenticatedUser` (CAN-COD-021) |
| MATRIZ | `MATRIZ-FATOS.md` §3 — índice ADR-006 | CAN-DEC-016, CAN-DEC-022, CAN-DEC-027, CAN-RNF-015 a CAN-RNF-020, CAN-ALT-010, CAN-COD-004 a CAN-COD-012, CAN-COD-022, CAN-INT-004, CAN-INT-005 |
| ANALISE | `ANALISE-EVIDENCIAS.md` §15 INT-03/INT-04/INT-05/INT-07 | Registro das integrações futuras de rotas, autorização, erros e redaction |
| ADR | [ADR-002](./ADR-002-worker-separado-com-polling.md) | Worker como processo separado — exceção justificada ao lifecycle da API |

## Decisões relacionadas

- [ADR-001 — Transactional Outbox no MySQL para eventos de webhook](./ADR-001-outbox-no-mysql.md) — atomicidade do evento; contexto do ponto de integração em `changeStatus`.
- [ADR-002 — Worker separado com polling para processamento da outbox](./ADR-002-worker-separado-com-polling.md) — lifecycle próprio do worker.
- [ADR-003 — Retry com backoff e Dead-Letter Queue para entregas de webhook](./ADR-003-retry-backoff-e-dlq.md) — origem do endpoint administrativo de replay.
- [ADR-004 — HMAC-SHA256 e gestão de secrets por endpoint](./ADR-004-hmac-sha256-e-gestao-de-secrets.md) — dados sensíveis que motivam a revisão da redaction.
- [ADR-005 — Entrega at-least-once com Event ID](./ADR-005-entrega-at-least-once-com-event-id.md) — identidade do evento em UUID, coerente com a convenção do projeto.

## Notas de implementação

- **Observar um módulo existente antes de criar cada camada**, tomando `src/modules/orders` como referência viva.
- **Manter os controllers finos**: extrair dados validados, chamar o service, encaminhar erros ao middleware.
- **Centralizar a validação nos schemas**, sem repeti-la nos handlers.
- **Evitar importar o bootstrap HTTP no worker**: reutilizar `env`, logger e a fábrica de Prisma, não `buildApp`.
- **Preservar o limite do domínio de pedidos**: consumir a transição, não reimplementá-la.
- **Ampliar a redaction antes de registrar objetos de webhook** em log.
- **Adicionar componentes por composição explícita** nos pontos existentes, sem container de DI.
- **Registrar qualquer desvio** do padrão junto da justificativa, no FDD ou em ADR próprio.

## Histórico

| Data | Alteração | Autor |
| ---- | --------- | ----- |
| Não informada | Decisão tomada na reunião técnica de webhooks (`[09:27]`–`[09:30]`; role ADMIN em `[09:36]`; resumo em `[09:48]`; padrão UUID em `[09:51]`) | Bruno, Diego, Larissa, Sofia |
| Não informada | Redação inicial do ADR a partir de `TRANSCRICAO.md`, do código e de `MATRIZ-FATOS.md` | Responsável pela entrega |
