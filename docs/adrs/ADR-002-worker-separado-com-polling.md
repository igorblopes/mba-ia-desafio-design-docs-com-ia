# ADR-002 — Worker separado com polling para processamento da outbox

## Status

`Aceito`

- **Data da decisão:** não informada. `TRANSCRICAO.md` registra apenas "quinta-feira, 09:00", sem data de calendário.
- **Participantes:** Diego (Engenheiro Sênior, Plataforma), Larissa (Tech Lead, conduzindo), Bruno (Engenheiro Pleno, Pedidos) e Marcos (Product Manager), que validou o intervalo em `[09:10]` ("2 segundos serve, perfeito"). Fechada por Larissa em `[09:10]` ("Vamos registrar isso como uma decisão. Worker em polling, 2s") e complementada em `[09:11]`–`[09:13]` e `[09:29]`–`[09:30]`.
- **Escopo:** como os eventos já persistidos na outbox são descobertos e processados fora do ciclo de vida das requisições da API. Não cobre a produção do evento (ADR-001), retry/backoff/DLQ/timeout de envio (ADR-003), segurança do envio (ADR-004), garantia de entrega (ADR-005), nem o esquema da outbox.

## Contexto

O ADR-001 decidiu persistir o evento de webhook em uma outbox no MySQL existente, dentro da mesma transação da mudança de status (CAN-DEC-001, CAN-DEC-002). Essa decisão assegura que o evento exista quando o commit ocorre, mas não o entrega: o evento precisa ser lido e processado **após** o commit. O ADR-001 encerra sua fronteira exatamente aí, remetendo o desenho desse componente a este documento. Executar a chamada HTTP dentro da requisição já foi descartado e registrado lá (`[09:04]` Bruno; `[09:06]` Diego — CAN-ALT-001). Resta a questão complementar: se não é a requisição que entrega, **quem** lê a outbox e **quando**.

Diego colocou a restrição de forma direta em `[09:11]`: "o worker tem que rodar como processo separado, não dentro da mesma instância da API. Senão se a API reinicia, perde o worker". O argumento é de ciclo de vida — reinícios, deploys e escalonamento da API são governados por critérios de tráfego HTTP, que não deveriam determinar se o consumo da outbox está acontecendo. Somar ao processo da API a responsabilidade permanente de consumir a outbox misturaria dois regimes de execução: um dirigido por requisições, outro contínuo.

O repositório não oferece hoje ponto de partida para esse consumo: não existe fila externa, worker, agendador nem mecanismo reativo de banco, e `prisma/schema.prisma` modela apenas os domínios atuais (CAN-COD-002). Por outro lado, o requisito de negócio abre espaço para uma solução simples — "tempo real", para os clientes, é latência abaixo de 10 segundos (`[09:02]` Marcos — CAN-MET-001). Uma consulta periódica de curta duração cabe nessa janela, o que permitiu à primeira versão privilegiar simplicidade operacional em vez de um mecanismo reativo.

## Drivers da decisão

- **Separação de responsabilidades** entre atender requisições e consumir a outbox (`[09:11]` Diego — CAN-RNF-011).
- **Independência do ciclo de vida das requisições e da API**: o processamento não deve ser perdido porque a API reiniciou (`[09:11]` Diego).
- **Isolamento do processamento assíncrono**, que é contínuo, do processamento HTTP, que é dirigido por demanda.
- **Compatibilidade com a outbox no MySQL** decidida no ADR-001: a fonte dos eventos pendentes já é o banco.
- **Reutilização da infraestrutura existente**: mesmo banco, mesma stack — "Sim, mesmo banco, mesma stack. Só não pode ser o mesmo processo" (`[09:11]` Diego).
- **Latência compatível com o requisito de negócio**: "Polling de 2 segundos atende o requisito de 'abaixo de 10 segundos' tranquilo" (`[09:09]` Diego).
- **Simplicidade operacional inicial**, coerente com o tamanho do time (`[09:07]` Diego).
- **Possibilidade de desligar, reiniciar ou implantar o worker separadamente** da API.
- **Evolução futura possível** para estratégias mais sofisticadas, sem introduzi-las agora (`[09:13]` Diego — CAN-FORA-005).

## Decisão

Foi decidido que:

1. Os eventos da outbox serão processados por um **worker**, e não pela requisição que muda o status.
2. O worker será executado em **processo separado da API** (`[09:11]` Diego — CAN-RNF-011).
3. O worker **consultará periodicamente a outbox no MySQL** já utilizado pela aplicação, sem infraestrutura de mensageria adicional (CAN-DEC-003).
4. O **intervalo de polling decidido é de 2 segundos** (`[09:09]` Diego; `[09:10]` Larissa — CAN-MET-002).
5. A primeira versão utilizará um **único worker** (`[09:12]` Diego — CAN-DEC-005).
6. O worker utilizará uma **instância própria de `PrismaClient`**, por ser outro processo Node: "Separado. PrismaClient é por processo" (`[09:30]` Bruno — CAN-DEC-017).
7. Essa instância utilizará a **mesma `DATABASE_URL`** e a mesma base de dados da API (`[09:11]` Bruno; `[09:30]` Bruno — CAN-DEC-018).
8. O processamento será **orientado aos eventos pendentes mais antigos** (`[09:09]` Diego — CAN-DEC-004) e a implementação **deverá** adotar uma ordenação determinística.
9. Há a **intenção de preservar a ordem relativa dos eventos de um mesmo pedido**; **não há garantia de ordenação global** (`[09:13]` Larissa — CAN-RNF-013, CAN-RNF-014).
10. **Múltiplos workers não fazem parte da primeira versão** (`[09:13]` Diego — CAN-FORA-005).

A estratégia completa de claim, lock, concorrência, tamanho de lote e recuperação **não é definida** por este ADR e **deverá** ser detalhada no FDD (CAN-LAC-006, CAN-LAC-014).

## Semântica do polling

O intervalo de 2 segundos é a **frequência planejada de consulta** à outbox, não uma medida de latência de entrega. Concretamente:

- Um evento persistido pouco antes de um ciclo poderá ser encontrado quase imediatamente.
- Um evento persistido logo após um ciclo aguardará **até aproximadamente 2 segundos** pelo ciclo seguinte apenas para ser **descoberto**.
- Esse tempo **não inclui** a leitura do banco, o processamento interno do worker nem a chamada HTTP ao cliente.
- Portanto, o polling **não garante entrega em 2 segundos**. O requisito continua sendo latência inferior a 10 segundos em condições normais (`[09:02]` Marcos — CAN-MET-001, CAN-RNF-007), dentro da qual a espera do polling é apenas uma parcela.
- A medição real da latência fim a fim **deverá** ser tratada por observabilidade e validação posteriores, não presumida a partir do intervalo.

Registro de auditoria: `ANALISE-EVIDENCIAS.md` (RNF-07, MET-03) reproduz a formulação de Larissa em `[09:10]`, que trata os 2 segundos como piso de latência. A auditoria do Passo 5 reformulou o fato em CAN-MET-003: trata-se de espera adicional de até ~2s na **descoberta** do evento — não de um piso da latência total —, aceita explicitamente por Larissa naquele mesmo momento. É a formulação canônica que vale aqui.

## Fluxo conceitual da decisão

1. O ADR-001 garante que o evento seja persistido na outbox junto com a mudança de status.
2. O processo do worker inicia e estabelece sua própria conexão Prisma.
3. Em ciclos periódicos, o worker consulta eventos elegíveis para processamento.
4. Os eventos encontrados são processados segundo uma ordenação determinística.
5. Cada evento é encaminhado ao fluxo de entrega.
6. O resultado do processamento é persistido.
7. O worker aguarda o próximo ciclo de polling.
8. Em encerramento, o processo interrompe novos ciclos e desconecta o Prisma.

Retry, DLQ e assinatura do envio não pertencem a este fluxo: são tratados em ADR-003 e ADR-004.

## Separação entre API e worker

### Responsabilidades da API

- Receber requisições HTTP.
- Validar a mudança de status.
- Executar a transação.
- Persistir o evento na outbox (ADR-001).
- Responder sem aguardar a entrega externa.

### Responsabilidades do worker

- Consultar a outbox.
- Selecionar eventos elegíveis.
- Iniciar o processamento assíncrono.
- Persistir o resultado do processamento.
- Respeitar as decisões de retry e DLQ registradas em ADR-003.
- Operar independentemente das requisições da API.

Decorre disso que API e worker **compartilham** o banco de dados e o código reutilizável do projeto, mas **não compartilham** memória nem a mesma instância de `PrismaClient` — são processos Node distintos (`[09:30]` Bruno). E, na primeira versão, o worker **não deverá** ser iniciado implicitamente dentro de cada instância da API.

## Modelo inicial de execução

Em nível arquitetural, a primeira versão prevê: um único processo de worker; execução contínua; polling periódico; processamento orientado aos eventos pendentes; lifecycle independente do da API; e graceful shutdown inspirado no padrão hoje existente em `src/server.ts`.

A supervisão do processo em produção **não está definida** (CAN-LAC-010). Este ADR não escolhe ferramenta de supervisão, nem define quantidade de threads, réplicas ou containers além do single worker decidido.

## Ordenação e limitações

- A primeira versão **deverá** buscar preservar a ordem dos eventos de um mesmo pedido, processando-os a partir dos pendentes mais antigos (`[09:12]` Diego).
- A implementação **deverá** possuir ordenação determinística.
- **Não existe garantia de ordem entre pedidos diferentes**: "Não é garantia de ordering global" (`[09:13]` Larissa — CAN-RNF-013). Marcos registrou em `[09:14]` que os clientes nunca pediram ordering global.
- A existência de apenas um worker **reduz** algumas condições de concorrência, mas **não é suficiente para provar** a preservação da ordem. A intenção declarada em `[09:12]`–`[09:13]` é uma limitação conhecida, não uma garantia formal (CAN-RNF-014).
- Retries podem fazer com que um evento posterior alcance um estado diferente antes de um evento anterior do mesmo pedido (ADR-003).
- Reinícios do processo podem afetar eventos em processamento.
- O FDD **deverá** definir como tratar esses cenários.

Permanecem **não definidos** (CAN-LAC-006): a chave principal de ordenação; o critério de desempate; o comportamento quando um evento anterior do mesmo pedido falha ou entra em retry; se o processamento é sequencial ou concorrente; a recuperação de eventos interrompidos; e a estratégia de claim. Este ADR não os resolve.

## Ponto de integração com o sistema existente

**`src/server.ts` — `bootstrap`** (CAN-COD-016). O arquivo já demonstra o padrão de lifecycle do projeto: `bootstrap` cria a aplicação com `buildApp({ prisma })`, sobe o servidor com `app.listen(env.PORT, ...)` e registra `process.on('SIGINT')`/`process.on('SIGTERM')` apontando para um `shutdown` que fecha o servidor (`server.close`), executa `await prisma.$disconnect()` e encerra o processo; falhas são capturadas em `bootstrap().catch(...)` com `logger.fatal({ err }, 'bootstrap_failed')` e `process.exit(1)`. O futuro `src/worker.ts` **deverá** adotar lifecycle equivalente — captura de sinais, desconexão do Prisma e log estruturado — **sem inicializar o servidor HTTP**. `src/worker.ts` **não existe** hoje; criá-lo é integração futura (CAN-DEC-028, CAN-INT-006).

**`src/config/database.ts` — `createPrismaClient`, `prisma`** (CAN-COD-015). O módulo expõe a fábrica `createPrismaClient()` e uma instância singleton exportada (`export const prisma`), consumida hoje por `src/server.ts`. Esse singleton é um objeto em memória: processos Node.js separados **não compartilham** a mesma instância, ainda que importem o mesmo módulo. O worker **deverá** obter a sua no próprio processo, o que a fábrica já viabiliza. Não há hoje conexão própria de worker, porque não há worker.

**`src/config/env.ts` — `envSchema`, `env`** (CAN-COD-014). A `DATABASE_URL` já existe e é validada por Zod, com `process.exit(1)` em configuração inválida; é essa variável que o worker **deverá** reutilizar (CAN-DEC-018). Não existem variáveis específicas do worker: intervalo de polling, tamanho de lote, concorrência e timeout **não estão configurados no código**, e a reunião não decidiu que serão variáveis de ambiente — constantes ou env vars é questão pendente (CAN-INT-008).

**`package.json` — `scripts`** (CAN-COD-019). Existem `dev`, `build`, `start`, `db:migrate`, `db:reset`, `db:seed`, `test`, `test:watch`, `lint` e `format`. **Não há script `worker`**; sua criação (`[09:11]` Larissa — CAN-DEC-028) é implementação futura. As ferramentas de execução TypeScript já estão disponíveis e em uso por `dev` e `start` (`tsx`; `tsc` + `node` sobre `dist/`).

**`src/shared/logger/index.ts` — `createLogger`, `logger`** (CAN-COD-013). O logger Pino já é padrão do projeto. O worker **poderá** reutilizá-lo diretamente: "o logger, que é Pino, já tá no projeto inteiro. Não vamos botar nada novo" (`[09:29]` Bruno — CAN-RNF-016).

**`prisma/schema.prisma` — `datasource db`** (CAN-COD-002). O provider é `mysql`, com `url = env("DATABASE_URL")`. **Não existem models de outbox nem de worker**; sua modelagem é integração futura (CAN-INT-002).

## Alternativas consideradas

### Alternativa — Processamento dentro do processo da API

**Descrição** — Manter o loop de consumo da outbox dentro da própria instância da API, sem processo adicional.

**Vantagens** — Menor quantidade inicial de processos a executar e observar; simplicidade aparente de deploy; reuso direto do processo e das conexões existentes.

**Desvantagens e trade-offs**

- Acoplamento ao lifecycle da API: "se a API reinicia, perde o worker" (`[09:11]` Diego).
- Com mais de uma instância da API em execução, cada uma iniciaria seu próprio processamento, criando concorrência não prevista sobre as mesmas linhas.
- Mistura de responsabilidades: um processo dirigido por requisições passaria a hospedar um regime contínuo.
- Impacto potencial em recursos compartilhados com o atendimento HTTP.
- Deploy, restart e escala da API e do consumo deixariam de ser controláveis separadamente.

**Motivo do descarte** — Rejeitada explicitamente em `[09:11]` por Diego: "não dentro da mesma instância da API".

### Alternativa — Trigger de banco ou mecanismo reativo do MySQL

**Descrição** — Usar trigger no MySQL para notificar o processamento de forma reativa, em vez de consultar periodicamente. Levantada por Bruno em `[09:09]`: "Não dá pra usar trigger do banco pra ser mais reativo?" (CAN-ALT-003).

**Vantagens** — Expectativa de processamento imediato, sem espera de ciclo; eliminação de consultas em vazio.

**Desvantagens e trade-offs**

- O MySQL não possui listener nativo equivalente ao `NOTIFY`/`LISTEN` do PostgreSQL (`[09:09]` Diego).
- Trigger executa SQL; não notifica processo externo. Avisar o worker exigiria improvisos — Diego citou escrever em arquivo ou bater em um endpoint —, descritos como "esquisito" (`[09:09]`).
- Aumento do acoplamento da lógica de integração ao banco de dados.
- Dificuldade operacional de manter e depurar esse caminho.

**Motivo do descarte** — O ganho de reatividade não se justifica dentro do requisito de latência: "Polling de 2 segundos atende o requisito de 'abaixo de 10 segundos' tranquilo" (`[09:09]` Diego).

### Alternativa — Fila externa ou plataforma de mensageria

**Descrição** — Redis Streams ou fila dedicada como substrato dos eventos, consumida de forma reativa por um worker (`[09:07]` Larissa — CAN-ALT-002). A alternativa foi discutida como substituta da outbox, mas seu modelo de consumo também substituiria o polling, o que a torna parcialmente aplicável a esta decisão.

**Vantagens** — Consumo reativo, sem espera de ciclo; recursos nativos de distribuição, escala e coordenação de consumidores.

**Desvantagens e trade-offs**

- Nova infraestrutura a provisionar, versionar, operar e monitorar.
- Complexidade desproporcional para a primeira versão e para um time descrito como pequeno (`[09:07]` Diego).
- Sobreposição com a decisão do ADR-001, onde a análise completa está registrada e não é repetida aqui.

**Motivo do descarte** — Rejeitada em `[09:07]` por Diego como overengineering: "Outbox no MySQL existente resolve". Descartada a fila, o consumo reativo que ela ofereceria deixa de estar disponível, e a leitura periódica passa a ser o mecanismo coerente com a fonte de dados escolhida.

### Alternativa — Múltiplos workers desde a primeira versão

**Descrição** — Executar vários workers em paralelo desde a v1 (CAN-ALT-007).

**Vantagens** — Maior throughput; maior disponibilidade do processamento, sem ponto único de parada.

**Desvantagens e trade-offs**

- Exigiria locking, particionamento ou coordenação entre consumidores — Diego apontou particionar por `order_id` ou lock pessimista como caminhos possíveis (`[09:13]`).
- Impacto na ordenação: "Se a gente escala pra múltiplos workers em paralelo no futuro, perde a garantia" (`[09:12]` Diego).
- Complexidade não necessária para o volume e o prazo da primeira versão.

**Motivo do adiamento** — **Adiado**, não invalidado tecnicamente: "isso é problema do futuro, não agora" (`[09:13]` Diego — CAN-FORA-005). A adoção futura exige nova decisão.

## Consequências

### Consequências positivas

- Separação efetiva entre a API e o processamento assíncrono, com responsabilidades distintas.
- Lifecycle independente: reiniciar a API não interrompe o consumo, e vice-versa; cada processo pode ser implantado, parado e reiniciado à parte.
- Redução do impacto do processamento de webhooks sobre o atendimento das requisições.
- Compatibilidade direta com o padrão Outbox do ADR-001: a fonte dos eventos já é o MySQL, reutilizado sem infraestrutura nova.
- Implementação inicial mais simples do que introduzir e operar mensageria.
- Evolução futura possível para consumo mais sofisticado, sem bloquear a primeira versão.

### Consequências negativas

- Passa a ser necessário operar e supervisionar um segundo processo, com deploy, logs e observabilidade próprios.
- O polling gera consultas periódicas ao banco mesmo quando não há eventos pendentes.
- Cada evento pode aguardar até aproximadamente um intervalo de polling apenas para ser descoberto.
- O single worker limita o throughput ao que um processo consegue realizar.
- Se o worker parar, nenhum evento é entregue, ainda que os status continuem mudando e os eventos continuem sendo persistidos.
- Ausência de redundância na primeira versão: o worker é ponto único de processamento.
- Será necessário lidar com eventos interrompidos por reinício, cujo tratamento não está definido (CAN-LAC-006).
- Será necessário garantir operacionalmente que apenas um worker esteja ativo, sem mecanismo definido para isso.
- Aprofunda a dependência do MySQL, que passa a servir armazenamento e consumo dos eventos.
- Ordenação, claim e recuperação passam a exigir definição explícita no FDD.

### Consequências neutras ou operacionais

- Criação futura de `src/worker.ts` como entry-point separado (CAN-DEC-028, CAN-INT-006).
- Criação futura do script `worker` no `package.json`.
- Configuração de execução em produção a definir (CAN-LAC-010).
- Logs e métricas próprios do worker, ainda não especificados.
- Health check e readiness do worker: não definidos.
- Testes específicos do worker deverão ser previstos (CAN-INT-009).
- Procedimentos de startup e shutdown do worker passam a integrar a operação.
- Documentação operacional do segundo processo será necessária.

## Riscos e cuidados

| Risco ou cuidado | Impacto | Tratamento ou estado |
|---|---|---|
| Worker parado sem detecção | Eventos acumulam na outbox; nenhum cliente é notificado, sem sinal visível | Sem supervisão nem health check definidos (CAN-LAC-010). `Em aberto para detalhamento no FDD` |
| Dois workers iniciados acidentalmente | Concorrência sobre as mesmas linhas; envio duplicado e ordem afetada | Single worker é decisão (CAN-DEC-005); a garantia operacional e o claim não estão definidos (CAN-LAC-006). `Em aberto para detalhamento no FDD` |
| Processamento duplicado do mesmo evento | Cliente recebe o evento mais de uma vez | Consequência aceita do at-least-once; dedup delegada ao cliente via `X-Event-Id` (ADR-005 — CAN-DEC-014, CAN-DEC-015) |
| Consultas excessivas ao banco em períodos sem eventos | Carga constante no MySQL, ainda que baixa | Consequência aceita do polling de 2s (`[09:09]` Diego); política de idle não definida. `Em aberto para detalhamento no FDD` |
| Evento preso em estado intermediário | Notificação nunca entregue apesar do commit | Depende de claim, lease e recuperação, não definidos (CAN-LAC-006). `Em aberto para detalhamento no FDD` |
| Perda de ordem entre eventos de um mesmo pedido | Cliente observa transições fora de sequência | Intenção de ordenação sob single worker, sem garantia formal (CAN-RNF-014). `Em aberto para detalhamento no FDD` |
| Starvation de eventos | Evento nunca selecionado por um ciclo | Depende da query de seleção e do critério de ordenação, não definidos (CAN-LAC-006, CAN-LAC-014). `Em aberto para detalhamento no FDD` |
| Crescimento do backlog acima da capacidade do worker | Latência acima dos 10s exigidos (CAN-MET-001) | Escala por múltiplos workers adiada (CAN-FORA-005); métricas de lag não definidas. `Em aberto para detalhamento no FDD` |
| Reinício no meio do processamento | Linhas em processamento sem dono após restart | Não definido (CAN-LAC-006). `Em aberto para detalhamento no FDD` |
| Falha na conexão do worker com o banco | Ciclos falham; nenhum evento avança | Comportamento do worker diante de falha de ciclo não definido. `Em aberto para detalhamento no FDD` |
| Graceful shutdown incompleto | Evento interrompido entre envio e persistência do resultado | `src/server.ts` oferece o padrão de sinais e `$disconnect` (CAN-COD-016); a aplicação ao worker é futura. `Em aberto para detalhamento no FDD` |
| Falta de supervisão do processo em produção | Worker não reinicia sozinho após queda | Nenhuma ferramenta escolhida (CAN-LAC-010). `Em aberto para detalhamento no FDD` |
| Capacidade insuficiente do single worker | Throughput limitado a um processo | Trade-off aceito na v1 (CAN-DEC-005); múltiplos workers exigem nova decisão (CAN-FORA-005) |

## Lacunas de implementação

Os pontos abaixo **não são definidos** por esta decisão e não devem ser inferidos a partir dela:

- **Seleção e consumo** — query de seleção dos eventos elegíveis; estratégia de claim; locking; lease ou timeout de processamento; critério de recuperação após restart (CAN-LAC-001, CAN-LAC-006).
- **Lote e concorrência** — tamanho do batch ("batch pequeno" foi citado em `[09:08]` por Diego, sem número) e se o processamento é sequencial ou concorrente dentro do worker (CAN-LAC-014).
- **Estados do evento** — status intermediário: "pendente, processando, falhou, entregue" foram citados em `[09:08]`, sem conjunto nem transições fechados (CAN-LAC-001).
- **Ordenação** — chave de ordenação, critério de desempate e comportamento quando um evento anterior do mesmo pedido entra em retry (CAN-LAC-006, CAN-RNF-014).
- **Configuração** — se o intervalo de polling será constante no código ou variável de ambiente (CAN-INT-008); política de idle entre ciclos sem eventos; limites de conexão do `PrismaClient` do worker.
- **Operação** — supervisão do processo em produção e mecanismo de deploy (CAN-LAC-010); health check e readiness; detalhes do shutdown gracioso.
- **Observabilidade e carga** — métricas de lag e de profundidade da fila; tratamento de backlog acumulado.
- **Escala** — estratégia futura para múltiplos workers (CAN-FORA-005).

## Critérios de conformidade arquitetural

Uma implementação futura estará em conformidade com este ADR se:

1. O worker for executado em **processo separado** da API.
2. O servidor HTTP **não iniciar** o loop de consumo da outbox.
3. O worker utilizar **instância própria de `PrismaClient`**, no seu próprio processo.
4. API e worker utilizarem a **mesma origem de dados** (mesma `DATABASE_URL`).
5. O polling ocorrer no **intervalo arquitetural decidido** de 2 segundos.
6. O processamento da outbox **não ocorrer dentro da requisição** de mudança de status.
7. A implementação **não pressupor ordering global** nem apresentar a ordem por pedido como garantia formal.
8. **Múltiplos workers não forem introduzidos** sem nova decisão ou revisão deste ADR.
9. O lifecycle do worker contemplar **inicialização, encerramento e desconexão** do Prisma.
10. Os detalhes de claim, lock e recuperação forem **documentados no FDD**, e não decididos silenciosamente no código.
11. O worker **reutilizar** o logger e as configurações compatíveis com o projeto (CAN-RNF-016).
12. Qualquer desvio relevante dos critérios acima exigir novo ADR ou atualização deste documento.

## Evidências e rastreabilidade

| Tipo | Localização | Evidência sustentada |
|---|---|---|
| TRANSCRICAO | `[09:02]` Marcos | "Tempo real" = abaixo de 10 segundos; janela que viabiliza o polling (CAN-MET-001, CAN-RNF-007) |
| TRANSCRICAO | `[09:08]` Diego | Worker lê os pendentes em "batch pequeno", processa e marca; tamanho não informado (CAN-DEC-004, CAN-LAC-014) |
| TRANSCRICAO | `[09:09]` Diego | "Polling em loop. A cada 2 segundos, busca os eventos pendentes mais antigos"; 2s atende o requisito de <10s (CAN-DEC-003, CAN-MET-002) |
| TRANSCRICAO | `[09:09]` Bruno; `[09:09]` Diego | Trigger de banco levantada e descartada: MySQL não tem listener nativo; trigger não notifica processo externo (CAN-ALT-003) |
| TRANSCRICAO | `[09:10]` Marcos; `[09:10]` Larissa | "2 segundos serve"; fechamento da decisão e aceite da espera adicional do polling (CAN-MET-002, CAN-MET-003) |
| TRANSCRICAO | `[09:11]` Diego | "o worker tem que rodar como processo separado, não dentro da mesma instância da API"; "mesmo banco, mesma stack. Só não pode ser o mesmo processo" (CAN-RNF-011) |
| TRANSCRICAO | `[09:11]` Larissa; `[09:28]` Bruno | Entry-point futuro `src/worker.ts` e script `npm run worker`, espelhando `src/server.ts` (CAN-DEC-028) |
| TRANSCRICAO | `[09:11]` Bruno; `[09:30]` Bruno | Mesmo banco; "PrismaClient é por processo. Mesmo banco, mesma DATABASE_URL, mas instância nova porque é outro processo Node" (CAN-DEC-017, CAN-DEC-018) |
| TRANSCRICAO | `[09:12]` Diego; `[09:13]` Larissa; `[09:14]` Marcos | Single-worker na v1; intenção de ordem por pedido; sem ordering global, registrado como limitação conhecida (CAN-DEC-005, CAN-RNF-013, CAN-RNF-014) |
| TRANSCRICAO | `[09:13]` Diego | Múltiplos workers, particionamento por `order_id` e lock pessimista adiados: "problema do futuro, não agora" (CAN-ALT-007, CAN-FORA-005) |
| TRANSCRICAO | `[09:29]` Bruno | Logger Pino já é padrão do projeto e não será substituído (CAN-RNF-016) |
| CODIGO | `src/server.ts` — `bootstrap`, `shutdown` | Padrão de lifecycle: `buildApp`, `listen`, `SIGINT`/`SIGTERM`, `server.close`, `prisma.$disconnect`, `logger.fatal` em `bootstrap_failed` (CAN-COD-016) |
| CODIGO | `src/config/database.ts` — `createPrismaClient`, `prisma` | Fábrica e singleton em memória; processos distintos não compartilham a instância (CAN-COD-015, CAN-DEC-017) |
| CODIGO | `src/config/env.ts` — `envSchema`, `env` | `DATABASE_URL` validada por Zod; nenhuma variável de polling, batch, concorrência ou timeout existe hoje (CAN-COD-014, CAN-INT-008) |
| CODIGO | `package.json` — `scripts` | Scripts `dev`/`build`/`start`/`db:*`/`test`/`lint`/`format`; **não há** script `worker`; `tsx` e `tsc` disponíveis para execução TypeScript (CAN-COD-019) |
| CODIGO | `src/shared/logger/index.ts` — `createLogger`, `logger` | Logger Pino existente, reutilizável pelo worker (CAN-COD-013) |
| CODIGO | `prisma/schema.prisma` — `datasource db` | Provider `mysql` com `url = env("DATABASE_URL")`; ausência de models de outbox ou worker (CAN-COD-002) |
| ADR | [ADR-001](./ADR-001-outbox-no-mysql.md) — Decisão, item 6 | O evento é persistido na outbox na transação de status; o processamento assíncrono posterior é remetido a este ADR |
| MATRIZ | `MATRIZ-FATOS.md` §3 — índice ADR-002 | Conjunto canônico deste ADR: CAN-DEC-003, CAN-DEC-004, CAN-DEC-005, CAN-DEC-017, CAN-DEC-018, CAN-DEC-028, CAN-RNF-011, CAN-RNF-013, CAN-RNF-014, CAN-ALT-003, CAN-ALT-007, CAN-MET-002, CAN-MET-003, CAN-COD-016, CAN-INT-006, CAN-FORA-005 |

## Decisões relacionadas

As decisões abaixo não são repetidas aqui.

- [ADR-001 — Transactional Outbox no MySQL para eventos de webhook](./ADR-001-outbox-no-mysql.md) — produção e persistência atômica do evento; fronteira anterior a esta decisão.
- [ADR-003 — Retry, backoff e DLQ](./ADR-003-retry-backoff-e-dlq.md) — retentativa, dead-letter, replay administrativo e timeout de envio.
- [ADR-005 — Entrega at-least-once com Event ID](./ADR-005-entrega-at-least-once-com-event-id.md) — garantia de entrega e idempotência delegada ao cliente.
- [ADR-006 — Reuso dos padrões do projeto](./ADR-006-reuso-dos-padroes-do-projeto.md) — estrutura modular, convenções de erro, logger e schemas.

## Notas de implementação

- O loop de polling **não deverá** bloquear o event loop indefinidamente.
- O processo **deverá** responder a `SIGINT` e `SIGTERM`, interrompendo novos ciclos e desconectando o Prisma, à semelhança do `shutdown` de `src/server.ts`.
- A falha de um ciclo **não deverá** encerrar o worker silenciosamente: qualquer interrupção precisa ser registrada pelo logger.
- O processamento **deverá** ser observável — início de ciclo, eventos selecionados e resultado.
- A seleção de eventos **deverá** ser determinística, de modo que a ordem de processamento seja reproduzível e auditável.
- O worker **não deverá** importar e reutilizar o singleton `prisma` esperando compartilhá-lo com a API: a instância é por processo.

## Histórico

| Data | Alteração | Autor |
|---|---|---|
| Não informada | Decisão tomada na reunião técnica de webhooks (`[09:09]`–`[09:13]`; `PrismaClient` e `DATABASE_URL` confirmados em `[09:29]`–`[09:30]`) | Diego, Larissa, Bruno, Marcos |
| Não informada | Redação inicial do ADR a partir de `TRANSCRICAO.md`, do código e de `MATRIZ-FATOS.md` | Responsável pela entrega |
