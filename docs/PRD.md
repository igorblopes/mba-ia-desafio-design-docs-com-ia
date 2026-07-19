# PRD — Webhooks de Notificação de Mudança de Status de Pedidos

## Metadados

| Campo             | Valor                                                         |
| ----------------- | ------------------------------------------------------------- |
| Documento         | PRD — Webhooks de Notificação de Mudança de Status de Pedidos |
| Status            | Em revisão                                                    |
| Produto           | Order Management System                                       |
| Público principal | Clientes B2B integrados à API de pedidos                      |
| Responsável       | Responsável pela entrega                                      |
| Data              | Não informada. `TRANSCRICAO.md` registra apenas "quinta-feira, 09:00", sem data de calendário |
| RFC relacionado   | [RFC — Webhooks](./RFC.md)                                    |
| FDD relacionado   | [FDD — Webhooks](./FDD.md)                                    |
| ADRs relacionados | ADR-001 a ADR-006                                             |

---

## Resumo executivo

Os clientes B2B integrados à plataforma precisam saber, o mais rápido possível, quando o status de um pedido muda. Hoje eles obtêm essa informação consultando repetidamente a API de pedidos (`GET /orders`) e comparando as respostas para descobrir se algo mudou. Esse modelo de consulta em intervalos (polling) torna a integração deles mais lenta e mais cara, além de manter cada cliente responsável por uma lógica de verificação contínua.

Esta feature introduz **webhooks outbound**: a plataforma passa a notificar o cliente quando o status de um pedido muda, em vez de esperar que o cliente pergunte. O cliente cadastra um ou mais endpoints HTTPS de destino e escolhe sobre quais status deseja ser notificado. Quando um pedido muda para um status de interesse, a plataforma produz um evento e o entrega ao endpoint configurado.

O envio inclui mecanismos de segurança para que o consumidor possa verificar que a requisição veio realmente da plataforma e que o conteúdo não foi adulterado no caminho. Falhas temporárias de entrega geram novas tentativas automáticas; falhas que esgotam as tentativas são separadas para investigação e podem ser reprocessadas por um administrador. Como a entrega é do tipo *at-least-once* (pelo menos uma vez), o mesmo evento pode chegar mais de uma vez ao cliente — cada evento carrega um identificador estável para que o consumidor reconheça e ignore duplicatas.

A primeira versão é deliberadamente restrita ao evento de **mudança de status de pedido**. Ela cobre o cadastro e a gestão dos webhooks, a produção e a entrega dos eventos, a segurança do envio, a recuperação de falhas e a consulta ao histórico recente de entregas. Itens como notificação por e-mail de falhas, painel visual e novos tipos de evento ficam explicitamente fora desta versão.

---

## Status

`Em revisão`

Este PRD consolida o que foi discutido e decidido na reunião técnica registrada em `TRANSCRICAO.md` e normalizado em `MATRIZ-FATOS.md`. Decisões técnicas de suporte estão nos ADR-001 a ADR-006 (todos com status `Aceito`). Questões que permanecem sem decisão estão listadas em **Questões em aberto** e não devem ser tratadas como resolvidas.

---

## Contexto

A plataforma opera um sistema de gestão de pedidos (Order Management System) com uma API autenticada. Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — pediram formalmente para serem notificados quando o status de seus pedidos muda (`[09:00]` Marcos — CAN-PROB-003).

Hoje, para perceber essas mudanças, os clientes consultam `GET /orders` de tempos em tempos e comparam o resultado para detectar alterações (`[09:00]` Marcos — CAN-PROB-001). Esse padrão de consulta periódica deixa a integração deles "lenta e cara" (`[09:00]` Marcos — CAN-PROB-002).

Para esses clientes, "tempo real" foi definido de forma objetiva: qualquer latência **abaixo de 10 segundos** é aceitável; o essencial é que a informação não fique pendurada, exigindo atualização manual (`[09:02]` Marcos — CAN-MET-001).

A comunicação será **outbound**: a plataforma envia para o cliente; o cliente apenas recebe, não envia webhooks de volta (`[09:02]` Marcos; `[09:03]` Sofia). Existe um componente comercial associado à demanda: a Atlas sinalizou possível migração para um concorrente caso a entrega não ocorra no prazo (`[09:00]`, `[09:45]` Marcos — CAN-PROB-004), com prazo de negócio indicado para o fim de novembro (`[09:45]` Marcos — CAN-MET-013).

Este documento usa somente fatos sustentados pelas fontes. Não há, nas fontes, dados sobre o número total de clientes, o volume de requisições atual, o custo financeiro do polling, pesquisas de satisfação ou estudos de mercado; nenhum desses números é assumido aqui.

---

## Problema

### Problema do cliente

Os clientes precisam reagir rapidamente a mudanças de status dos seus pedidos, mas hoje dependem de perguntar à plataforma repetidamente. Entre uma consulta e a próxima existe uma janela em que a mudança já ocorreu e o cliente ainda não sabe. Diminuir essa frequência aumenta o atraso; aumentá-la eleva o consumo. O cliente não tem como ser avisado no momento em que o status muda — ele só descobre na próxima consulta.

### Problema operacional

Para sustentar o modelo atual, cada cliente precisa, por conta própria: implementar a rotina de polling; definir a frequência das consultas; armazenar o estado anterior de cada pedido; comparar cada resposta com o estado guardado; tratar o atraso inerente ao intervalo; lidar com o consumo de requisições que isso gera; e manter essa lógica funcionando continuamente. Essa é uma carga recorrente, replicada em cada cliente, para um problema que é o mesmo para todos.

### Problema de negócio

A ausência de notificação ativa pode reduzir a qualidade percebida da integração, elevar o custo de operação do lado do cliente e prolongar o tempo de reação a mudanças de pedido. Como a demanda parte de clientes estratégicos, o não atendimento pode dificultar contratos, expansões ou renovações — a própria Atlas indicou que consideraria migrar para um concorrente se a entrega não ocorrer no prazo (`[09:00]`, `[09:45]` Marcos — CAN-PROB-004, CAN-RISK-008). Este PRD não afirma que os clientes abandonarão o produto nem que um contrato será perdido; registra o sinal comercial tal como foi declarado na reunião.

---

## Oportunidade

Ao substituir o polling por notificações ativas, a feature pode:

- reduzir a dependência de consultas periódicas para detectar mudanças;
- reduzir o tempo entre a mudança de status e a percepção dela pelo cliente;
- simplificar as integrações, retirando de cada cliente a lógica de comparação de estado;
- reduzir a lógica duplicada que hoje cada cliente mantém;
- melhorar a experiência de integração B2B;
- aumentar a confiança dos clientes na plataforma;
- atender uma demanda formal de clientes estratégicos;
- estabelecer uma base sobre a qual **outros eventos** poderão ser oferecidos no futuro.

O suporte a novos tipos de evento é apresentado apenas como possibilidade posterior; **não** faz parte do escopo desta primeira versão, que se limita ao evento de mudança de status.

---

## Públicos e stakeholders

| Público ou stakeholder | Necessidade | Participação na feature |
| ---------------------- | ----------- | ----------------------- |
| Clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) | Receber notificação de mudança de status sem polling | Consumidores dos eventos; solicitantes da feature (CAN-PROB-003, CAN-PUB-001) |
| Desenvolvedores dos clientes | Integrar e validar o recebimento dos eventos | Cadastram/configuram endpoints, validam assinatura, processam e deduplicam eventos (CAN-PUB-002) |
| Operadores da plataforma | Acompanhar configurações e entregas | Administram configurações no escopo permitido e consultam histórico (CAN-PUB-004) |
| Administradores (role ADMIN) | Recuperar falhas permanentes | Executam replay da DLQ, com auditoria (CAN-PUB-003, CAN-RF-009, CAN-RNF-006) |
| Engenharia e operações | Operar o processo de entrega | Implantam e monitoram o worker, investigam falhas, dão suporte (CAN-RNF-011) |
| Segurança | Garantir segurança do envio | Revisam assinatura, secrets e logs antes do lançamento (`[09:46]` Sofia — CAN-MET-012) |
| Produto / Liderança | Atender a demanda no prazo | Priorização, comunicação com clientes e acompanhamento comercial (CAN-MET-011, CAN-MET-013) |

### Desenvolvedores dos clientes

Precisam cadastrar ou configurar endpoints; validar a autenticidade das requisições recebidas; processar os eventos; deduplicar entregas repetidas; e investigar falhas do lado deles.

### Operadores da plataforma

Precisam administrar configurações dentro do escopo permitido, consultar o histórico de entregas e identificar problemas de integração.

### Administradores

Precisam executar replay, investigar itens que esgotaram as tentativas e manter rastreabilidade das ações realizadas.

### Engenharia e operações

Precisam acompanhar o backlog de entregas, investigar falhas, operar o processo de entrega (o worker) e prestar suporte.

### Segurança

Precisa revisar a assinatura, a geração e o armazenamento de secrets, e os logs, além de validar o desenho antes do lançamento. A reunião reservou ao menos dois dias úteis para essa revisão (`[09:46]` Sofia — CAN-MET-012).

### Clientes solicitantes

Atlas Comercial, MaxDistribuição e Nova Cargo estão confirmados na transcrição como solicitantes (`[09:00]` Marcos — CAN-PROB-003). Nenhum acesso direto do cliente a recursos administrativos internos é previsto: a gestão de webhooks é feita pela API autenticada da plataforma (`[09:32]` Marcos/Larissa — CAN-PUB-004).

---

## Personas ou perfis de usuário

Perfis funcionais, sem nomes fictícios ou características pessoais inventadas.

### Desenvolvedor de integração do cliente

**Objetivo:** receber mudanças de pedido sem realizar polling frequente.

**Necessidades:** contrato de payload e headers claro; autenticação verificável da origem; identificação estável de cada evento; possibilidade de diagnóstico via histórico de entregas; comportamento previsível diante de falhas e de entregas repetidas.

### Operador interno

**Objetivo:** acompanhar as configurações de webhook e o histórico de entregas dentro do escopo permitido.

### Administrador interno

**Objetivo:** investigar e reprocessar falhas permanentes (itens da DLQ), com controle de acesso (role ADMIN) e registro de auditoria de quem executou a ação.

---

## Cenários de uso

### Cenário 1 — Cadastro de webhook

1. Um usuário autenticado informa o cliente ao qual o webhook pertence.
2. Informa uma URL HTTPS de destino.
3. Seleciona os status sobre os quais quer ser notificado.
4. A plataforma valida os dados informados.
5. Uma configuração de webhook é criada.
6. Uma secret é disponibilizada conforme o contrato final.

*Em aberto:* se o `customer_id` é informado no body ou no path (CAN-OPEN-001); se a secret é exibida uma única vez; se a secret pode ser recuperada posteriormente (CAN-LAC-003/CAN-LAC-004).

### Cenário 2 — Entrega bem-sucedida

1. Um pedido muda para um status selecionado.
2. A plataforma cria o evento.
3. O evento é entregue ao endpoint configurado.
4. O consumidor valida a assinatura.
5. O consumidor processa o evento.
6. A tentativa é registrada no histórico.

### Cenário 3 — Falha temporária

1. O endpoint do cliente está indisponível.
2. Uma tentativa de entrega falha.
3. A plataforma programa novas tentativas segundo a progressão de backoff.
4. O cliente volta a responder.
5. Uma tentativa posterior é concluída com sucesso.

### Cenário 4 — Falha permanente

1. As tentativas de entrega se esgotam.
2. O evento é separado para investigação (movido para a DLQ).
3. Um administrador identifica a causa.
4. O administrador executa o replay após a correção.
5. A ação de replay é auditada.

### Cenário 5 — Evento duplicado

1. O consumidor processa um evento.
2. A confirmação é perdida ou considerada inconclusiva.
3. A plataforma envia novamente o mesmo evento.
4. O consumidor identifica o `X-Event-Id`.
5. O consumidor evita aplicar os efeitos novamente.

### Cenário 6 — Rotação de secret

1. O cliente solicita a rotação da secret.
2. Uma nova secret é gerada.
3. O cliente atualiza a configuração do seu lado.
4. A secret anterior permanece válida por 24 horas em paralelo.
5. Após o prazo, a secret anterior deixa de ser válida.

---

## Objetivos do produto

Objetivos atômicos e verificáveis:

- **O-1.** Permitir a configuração de webhooks por cliente (CAN-RF-002).
- **O-2.** Notificar o cliente quando um pedido muda para um status de interesse (CAN-RF-001).
- **O-3.** Reduzir a dependência de polling para detectar mudanças de status (CAN-PROB-001/CAN-PROB-002).
- **O-4.** Entregar eventos em menos de 10 segundos em condições normais (CAN-RNF-007, CAN-MET-001).
- **O-5.** Permitir que o consumidor valide a autenticidade e a integridade do evento recebido (CAN-RF-016).
- **O-6.** Recuperar automaticamente falhas temporárias de entrega (CAN-RF-012).
- **O-7.** Permitir a investigação e o reprocessamento de falhas permanentes (CAN-RF-009, CAN-RF-013).
- **O-8.** Fornecer histórico das entregas recentes (CAN-RF-008).
- **O-9.** Permitir que o consumidor identifique entregas duplicadas (CAN-RF-014/CAN-RF-015).
- **O-10.** Permitir a rotação de secrets, com convivência temporária da secret anterior (CAN-RF-010/CAN-RF-011).
- **O-11.** Manter comportamento consistente com a API e os padrões existentes da plataforma (CAN-DEC-016).
- **O-12.** Atender os clientes solicitantes dentro do prazo acordado, quando sustentado pelas fontes (CAN-MET-011, CAN-MET-013).

---

## Não objetivos

Esta versão **não** se propõe a:

- garantir entrega *exactly-once* (exatamente uma vez) — descartado em favor de *at-least-once* (CAN-ALT-011);
- garantir ausência de entregas duplicadas (CAN-RNF-009);
- garantir ordenação global de eventos (CAN-RNF-013);
- oferecer múltiplos workers em paralelo nesta versão (CAN-DEC-005, CAN-FORA-005);
- fornecer dashboard/painel visual (CAN-FORA-002);
- enviar alertas por e-mail sobre falhas (CAN-FORA-001);
- oferecer rate limiting de saída (outbound) nesta fase (CAN-OPEN-003);
- suportar tipos de evento além de mudança de status (CAN-LAC-011);
- fornecer SDK de consumo;
- implementar o sistema consumidor do cliente;
- garantir a idempotência no sistema do cliente (responsabilidade do consumidor — CAN-DEC-015);
- substituir completamente a API de consulta de pedidos (o `GET /orders/:id` continua sendo a fonte de detalhes — CAN-DEC-025);
- introduzir mensageria externa (CAN-ALT-002);
- redesenhar a arquitetura geral da aplicação (CAN-DEC-016);
- oferecer arquivamento automático de linhas entregues nesta fase (CAN-FORA-003).

---

## Escopo

### Escopo da primeira versão

Itens sustentados pelas fontes:

- criação de configuração de webhook (CAN-RF-002);
- listagem de configurações (CAN-RF-006);
- atualização de configuração (CAN-RF-004);
- remoção de configuração (CAN-RF-005);
- associação da configuração a um cliente (CAN-DEC-019);
- seleção dos status de interesse por endpoint (CAN-RF-007);
- exigência de URL HTTPS (CAN-RNF-004);
- geração de secret por endpoint (CAN-RF-003, CAN-RNF-002);
- rotação de secret (CAN-RF-010);
- grace period de 24 horas da secret anterior (CAN-RF-011, CAN-MET-006);
- evento `order.status_changed` (CAN-DEC-024);
- payload compacto (CAN-DEC-025);
- payload como snapshot do momento da mudança (CAN-RF-018, CAN-DEC-023);
- headers de entrega definidos (CAN-RF-017);
- assinatura HMAC-SHA256 (CAN-RF-016, CAN-MET-014);
- identificador de evento `X-Event-Id` (CAN-RF-014/CAN-RF-015);
- novas tentativas automáticas em caso de falha (CAN-RF-012);
- cinco tentativas, conforme a terminologia da reunião (CAN-MET-004);
- intervalos de backoff definidos (CAN-MET-005);
- histórico das últimas 100 entregas (CAN-RF-008, CAN-MET-009);
- DLQ para falhas permanentes (CAN-RF-013);
- replay administrativo (CAN-RF-009);
- auditoria do replay (CAN-RNF-006).

### Fora de escopo

- webhooks inbound (a plataforma não recebe webhooks — `[09:02]` Marcos);
- dashboard/painel visual (CAN-FORA-002);
- alertas por e-mail (CAN-FORA-001);
- implementação do sistema consumidor do cliente;
- garantia *exactly-once* (CAN-ALT-011);
- ordenação global de eventos (CAN-RNF-013);
- mensageria externa (CAN-ALT-002);
- outros tipos de evento além de mudança de status (CAN-LAC-011);
- replay pelo cliente externo (não decidido — o replay é administrativo — CAN-RF-009);
- portal de desenvolvedores como produto separado;
- SDKs de consumo;
- mecanismos de autenticação adicionais para o envio (por exemplo, mTLS ou OAuth específico) — não discutidos.

### Itens adiados

Registrados como adiados (não descartados permanentemente):

- múltiplos workers e escala horizontal (CAN-FORA-005);
- rate limiting de saída (CAN-OPEN-003);
- arquivamento automático de linhas entregues (~30 dias) (CAN-FORA-003, CAN-MET-010);
- retenção avançada de histórico e DLQ (CAN-LAC-009);
- eventos adicionais além de mudança de status (CAN-LAC-011);
- melhorias de escalabilidade (CAN-FORA-005);
- notificação por e-mail e/ou dashboards de acompanhamento (CAN-FORA-001);
- endurecimento da autorização do CRUD de configuração (CAN-FORA-004);
- mecanismos avançados de supervisão do worker em produção (CAN-LAC-010).

---

## Jornada do usuário

```text
Autenticação
    ↓
Identificação do cliente
    ↓
Cadastro da URL HTTPS
    ↓
Seleção dos status
    ↓
Recebimento da secret
    ↓
Configuração da validação no consumidor
    ↓
Ativação da integração
    ↓
Recebimento de eventos
    ↓
Processamento e deduplicação
    ↓
Consulta ao histórico quando necessário
```

**Pontos de incerteza da jornada:** localização do `customer_id` (body ou path — CAN-OPEN-001); como a secret será apresentada e se pode ser recuperada depois (CAN-LAC-003/CAN-LAC-004); como será feita a rotação na prática; como o histórico será paginado além das últimas 100 entregas (CAN-LAC-008).

---

## Experiência esperada do consumidor

O consumidor (o sistema do cliente que recebe os eventos) deve compreender que:

- a URL de destino precisa utilizar **HTTPS** (CAN-RNF-004);
- a **assinatura** enviada deve ser validada antes de processar o evento (CAN-RF-016);
- cada evento tem uma **identidade estável** (`X-Event-Id`) que não muda entre tentativas do mesmo evento (CAN-RF-015);
- o **mesmo evento pode ser recebido mais de uma vez** (CAN-RNF-009);
- a deduplicação é responsabilidade do consumidor e deve ser feita pelo `X-Event-Id` (CAN-DEC-015);
- o endpoint deve responder dentro do **timeout** de envio (CAN-RNF-012, CAN-MET-008);
- falhas de entrega geram **novas tentativas** posteriores (CAN-RF-012);
- as tentativas **não são infinitas** (CAN-MET-004);
- eventos que esgotam as tentativas podem **terminar na DLQ** (CAN-RF-013);
- um **replay** administrativo pode causar nova entrega de um evento (CAN-RF-009);
- a plataforma **não garante** *exactly-once* (CAN-ALT-011);
- os **detalhes completos do pedido** permanecem disponíveis na API (`GET /orders/:id`), pois o payload é compacto (CAN-DEC-025).

A assinatura HMAC-SHA256 serve para autenticidade e integridade — permite verificar **quem enviou** e se o conteúdo **não foi adulterado**. Ela **não** protege a confidencialidade do payload. O `X-Event-Id` identifica o evento; **não** é uma assinatura. A implementação da validação e da deduplicação no lado do consumidor é responsabilidade do cliente e não faz parte do escopo da plataforma.

---

## Requisitos funcionais

### RF-001 — Criar configuração de webhook

**Descrição:** O usuário autenticado deverá poder cadastrar uma configuração de webhook associada a um cliente, informando URL HTTPS e status de interesse.
**Ator:** Usuário autenticado.
**Resultado esperado:** A configuração é registrada e uma secret é gerada e disponibilizada.
**Fonte:** CAN-RF-002, CAN-RF-003 — `[09:31]` Marcos; `[09:33]` Bruno.

### RF-002 — Listar configurações

**Descrição:** O usuário autenticado deverá poder listar as configurações de webhook de um cliente.
**Ator:** Usuário autenticado.
**Resultado esperado:** As configurações do cliente são retornadas.
**Fonte:** CAN-RF-006 — `[09:33]` Bruno.

### RF-003 — Atualizar configuração

**Descrição:** O usuário autenticado deverá poder editar uma configuração de webhook existente.
**Ator:** Usuário autenticado.
**Resultado esperado:** A configuração é atualizada.
**Fonte:** CAN-RF-004 — `[09:33]` Bruno.

### RF-004 — Remover configuração

**Descrição:** O usuário autenticado deverá poder remover uma configuração de webhook.
**Ator:** Usuário autenticado.
**Resultado esperado:** A configuração é removida.
**Fonte:** CAN-RF-005 — `[09:33]` Bruno.

### RF-005 — Selecionar status de interesse

**Descrição:** Cada configuração de webhook deverá permitir escolher um subconjunto dos status do pedido sobre os quais deseja ser notificada.
**Ator:** Usuário autenticado.
**Resultado esperado:** Apenas os status selecionados geram entrega para aquele endpoint.
**Fonte:** CAN-RF-007, CAN-DEC-020 — `[09:33]` Marcos; `[09:34]` Bruno/Diego.

### RF-006 — Criar evento após mudança de status

**Descrição:** Quando um pedido muda para um status de interesse de algum webhook do cliente, a plataforma deverá produzir um evento de notificação.
**Ator:** Plataforma (sistema).
**Resultado esperado:** Um evento é registrado, atrelado à mudança de status que o originou. Se a mudança de status não for efetivada, o evento não existe (CAN-RNF-008).
**Fonte:** CAN-RF-001, CAN-DEC-002, CAN-DEC-020 — `[09:00]` Marcos; `[09:34]` Bruno; `[09:40]` Bruno; `[09:43]` Diego.

### RF-007 — Entregar evento ao endpoint

**Descrição:** A plataforma deverá entregar cada evento produzido ao endpoint HTTPS configurado.
**Ator:** Plataforma (processo de entrega).
**Resultado esperado:** Uma requisição HTTP é enviada ao endpoint do cliente com o evento.
**Fonte:** CAN-RF-001, CAN-DEC-003 — `[09:09]` Diego; `[09:43]` Diego.

### RF-008 — Enviar payload definido

**Descrição:** Cada entrega deverá conter um payload JSON com os campos definidos: `event_id`, `event_type` (`order.status_changed`), `timestamp` (ISO 8601), `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents`, representando um snapshot do momento da mudança, sem os itens do pedido.
**Ator:** Plataforma (sistema).
**Resultado esperado:** O consumidor recebe o payload no formato acordado.
**Fonte:** CAN-DEC-024, CAN-DEC-025, CAN-RF-018 — `[09:43]` Diego; `[09:44]` Bruno; `[09:52]` Larissa/Diego.

### RF-009 — Enviar headers definidos

**Descrição:** Cada entrega deverá incluir os headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`.
**Ator:** Plataforma (sistema).
**Resultado esperado:** O consumidor recebe os headers de identificação, assinatura e metadados.
**Fonte:** CAN-RF-014, CAN-RF-016, CAN-RF-017 — `[09:25]` Diego; `[09:44]` Diego/Sofia.

### RF-010 — Gerar secret por endpoint

**Descrição:** A plataforma deverá gerar uma secret única para cada endpoint de webhook no momento da criação.
**Ator:** Plataforma (sistema).
**Resultado esperado:** Cada endpoint possui sua própria secret, disponibilizada na criação.
**Fonte:** CAN-RF-003, CAN-RNF-002 — `[09:21]` Sofia; `[09:31]` Marcos.

### RF-011 — Rotacionar secret

**Descrição:** O cliente deverá poder solicitar a rotação da secret via API; a secret anterior permanece válida por 24 horas em paralelo e depois é invalidada.
**Ator:** Usuário autenticado (representando o cliente).
**Resultado esperado:** Uma nova secret é gerada; a anterior convive por 24 horas.
**Fonte:** CAN-RF-010, CAN-RF-011, CAN-MET-006 — `[09:21]` Sofia.

### RF-012 — Registrar tentativas de entrega

**Descrição:** A plataforma deverá registrar as tentativas de entrega (resultado, payload, resposta e tempo de resposta) para consulta posterior.
**Ator:** Plataforma (sistema).
**Resultado esperado:** As tentativas ficam disponíveis no histórico de entregas.
**Fonte:** CAN-RF-008 — `[09:34]` Marcos.

### RF-013 — Consultar as últimas 100 entregas

**Descrição:** O usuário autenticado deverá poder consultar o histórico das últimas 100 entregas de um webhook.
**Ator:** Usuário autenticado.
**Resultado esperado:** As últimas 100 entregas são retornadas com sucesso/falha, payload, resposta e tempo de resposta.
**Fonte:** CAN-RF-008, CAN-MET-009 — `[09:34]` Marcos.

### RF-014 — Repetir entregas com falha

**Descrição:** A plataforma deverá reagendar automaticamente as entregas que falharem, segundo a progressão de backoff, até o teto de tentativas.
**Ator:** Plataforma (processo de entrega).
**Resultado esperado:** Entregas falhas são retentadas nos intervalos definidos.
**Fonte:** CAN-RF-012, CAN-RNF-010, CAN-MET-005 — `[09:15]` Diego; `[09:17]` Larissa.

### RF-015 — Encaminhar evento à DLQ

**Descrição:** Esgotadas as tentativas de entrega, o evento deverá ser movido para a DLQ (fila de falhas permanentes) para investigação.
**Ator:** Plataforma (sistema).
**Resultado esperado:** O evento sai do fluxo normal de entrega e fica registrado como falha permanente.
**Fonte:** CAN-RF-013, CAN-DEC-007 — `[09:15]` Diego; `[09:17]` Larissa.

### RF-016 — Executar replay administrativo

**Descrição:** Um administrador deverá poder reprocessar manualmente um item da DLQ, recolocando-o no fluxo de entrega.
**Ator:** Administrador (role ADMIN).
**Resultado esperado:** O item da DLQ volta a ser processado para entrega.
**Fonte:** CAN-RF-009 — `[09:18]` Diego; `[09:35]` Diego.

### RF-017 — Restringir replay à role ADMIN

**Descrição:** O endpoint de replay deverá exigir role ADMIN.
**Ator:** Plataforma (autorização).
**Resultado esperado:** Apenas usuários ADMIN conseguem executar replay.
**Fonte:** CAN-DEC-010, CAN-PUB-003 — `[09:36]` Sofia; `[09:36]` Larissa.

### RF-018 — Registrar o executor do replay

**Descrição:** A plataforma deverá registrar quem executou cada replay, para auditoria.
**Ator:** Plataforma (sistema).
**Resultado esperado:** Cada replay fica atribuído ao administrador que o executou.
**Fonte:** CAN-RNF-006 — `[09:36]` Sofia.

---

## Requisitos não funcionais

### RNF-001 — Menos de 10 segundos em condições normais

**Descrição:** Em condições normais, a primeira tentativa de entrega deverá ocorrer em menos de dez segundos após a confirmação da mudança de status.
**Fonte:** CAN-RNF-007, CAN-MET-001 — `[09:02]` Marcos.

### RNF-002 — Polling de 2 segundos

**Descrição:** O processo consultará os eventos a cada dois segundos. O intervalo poderá adicionar até aproximadamente dois segundos antes da descoberta do evento. Esta espera adicional é distinta do tempo total de entrega e não constitui uma "latência mínima" isolada.
**Fonte:** CAN-MET-002, CAN-MET-003 — `[09:09]` Diego; `[09:10]` Larissa.

### RNF-003 — Timeout de 10 segundos

**Descrição:** Uma chamada de entrega que exceder dez segundos deverá ser tratada como falha e marcada para nova tentativa.
**Fonte:** CAN-RNF-012, CAN-MET-008 — `[09:42]` Diego.

### RNF-004 — Payload máximo de 64 KB

**Descrição:** Um payload que exceder 64 KB deverá ser recusado com erro; não é truncado nem enviado.
**Fonte:** CAN-RNF-005, CAN-MET-007 — `[09:23]` Sofia; `[09:24]` Diego/Larissa.

### RNF-005 — HTTPS obrigatório

**Descrição:** A URL de destino deverá utilizar HTTPS; URLs HTTP são recusadas na validação.
**Fonte:** CAN-RNF-004 — `[09:23]` Sofia.

### RNF-006 — HMAC-SHA256

**Descrição:** Cada entrega deverá ser assinada com HMAC-SHA256 sobre o corpo do request, permitindo ao consumidor verificar autenticidade e integridade.
**Fonte:** CAN-RNF-001, CAN-MET-014 — `[09:20]` Sofia.

### RNF-007 — Secret por endpoint

**Descrição:** Cada endpoint de webhook deverá ter uma secret única; não há secret global.
**Fonte:** CAN-RNF-002 — `[09:21]` Sofia.

### RNF-008 — Grace period de 24 horas

**Descrição:** Após a rotação, a secret anterior deverá permanecer válida por 24 horas em paralelo à nova, e então ser invalidada.
**Fonte:** CAN-RNF-003, CAN-MET-006 — `[09:21]` Sofia.

### RNF-009 — Entrega at-least-once

**Descrição:** A entrega deverá ser *at-least-once* (pelo menos uma vez): a plataforma tenta entregar cada evento até obter sucesso ou esgotar as tentativas, podendo resultar em o mesmo evento ser recebido mais de uma vez. Não há garantia de entrega única.
**Fonte:** CAN-RNF-009 — `[09:24]` Diego.

### RNF-010 — Identificador UUID

**Descrição:** Cada evento deverá ter um identificador único no formato UUID (`X-Event-Id`), gerado quando o evento entra no fluxo de entrega e estável entre as tentativas do mesmo evento.
**Fonte:** CAN-RF-015 — `[09:25]` Diego; `[09:51]` Larissa.

### RNF-011 — Cinco tentativas

**Descrição:** A entrega deverá ser retentada até um teto de cinco tentativas antes de ser considerada falha permanente.
**Fonte:** CAN-RNF-010, CAN-MET-004 — `[09:15]` Diego; `[09:17]` Larissa.

### RNF-012 — Intervalos de 1m, 5m, 30m, 2h e 12h

**Descrição:** As novas tentativas deverão seguir a progressão de backoff de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas (cobrindo uma janela de indisponibilidade de aproximadamente 15 horas).
**Fonte:** CAN-MET-005 — `[09:17]` Diego.

### RNF-013 — Worker separado

**Descrição:** O processo de entrega deverá rodar como um processo separado da instância da API, para não ser perdido se a API reiniciar.
**Fonte:** CAN-RNF-011, CAN-DEC-028 — `[09:11]` Diego.

### RNF-014 — Um worker na primeira versão

**Descrição:** A primeira versão deverá operar com um único worker (sem múltiplos workers em paralelo).
**Fonte:** CAN-DEC-005, CAN-RNF-014 — `[09:12]` Diego; `[09:13]` Larissa.

### RNF-015 — Sem ordering global

**Descrição:** A plataforma não garante ordenação global de eventos. Sob single-worker, pretende-se preservar a ordem dos eventos de um mesmo pedido, mas isso é uma limitação/intenção conhecida, não uma garantia formal.
**Fonte:** CAN-RNF-013, CAN-RNF-014 — `[09:12]` Diego; `[09:13]` Larissa; `[09:14]` Marcos.

### RNF-016 — Compatibilidade com padrões existentes

**Descrição:** O módulo de webhooks deverá reutilizar os padrões arquiteturais existentes (estrutura modular, tratamento de erros, validação, logging, autenticação e autorização) sem redesenhar a aplicação.
**Fonte:** CAN-DEC-016, CAN-RNF-015 a CAN-RNF-020 — `[09:30]` Larissa.

### RNF-017 — Proteção de secrets e assinaturas em logs

**Descrição:** Secrets e assinaturas não deverão ser expostas em logs. Observação: a configuração atual de logging redige tokens e senhas, mas **não** inclui `secret` na lista de redação — este ponto deve ser tratado antes do lançamento.
**Fonte:** CAN-INT-007, CAN-COD-013 — `[09:29]` Bruno.

### RNF-018 — Respostas do histórico limitadas às últimas 100 entregas

**Descrição:** O histórico de entregas deverá expor as últimas 100 entregas de um webhook.
**Fonte:** CAN-MET-009, CAN-RF-008 — `[09:34]` Marcos.

---

## Decisões e trade-offs principais

As decisões técnicas desta feature estão registradas em detalhe nos ADR-001 a ADR-006. Esta seção resume as **decisões estruturais** que moldam o produto, sempre acompanhadas da **alternativa descartada** e do **trade-off assumido**, para que a escolha seja compreensível também fora da engenharia. Nenhuma decisão aqui é nova: todas vêm da reunião (`TRANSCRICAO.md`) e estão normalizadas em `MATRIZ-FATOS.md`.

| ID | Decisão adotada | Alternativa descartada | Trade-off assumido | Fonte / ADR |
| -- | --------------- | ---------------------- | ------------------ | ----------- |
| D-1 | Entrega assíncrona via padrão outbox na base MySQL existente | Disparo síncrono na transação de status (CAN-ALT-001); Redis Streams ou fila dedicada (CAN-ALT-002) | Reaproveita a infraestrutura atual e garante atomicidade (o evento só existe se o status commitar); em troca, abre mão de push reativo e aceita a latência do polling | CAN-DEC-001 / ADR-001 |
| D-2 | Worker em processo separado, lendo a outbox por polling de 2 segundos | Trigger de banco para notificar o worker de forma reativa (CAN-ALT-003) | Simplicidade e independência da API (o worker sobrevive a um restart da API); em troca, adiciona até cerca de 2 segundos de espera à captura do evento | CAN-DEC-003 / ADR-002 |
| D-3 | Single-worker na primeira versão | Múltiplos workers em paralelo com ordenação global desde já (CAN-ALT-007) | Preserva a ordem dos eventos de um mesmo pedido e simplifica a operação; em troca, abre mão de escala horizontal e de garantia formal de ordenação | CAN-DEC-005 / ADR-002 |
| D-4 | Retry com teto de 5 tentativas e backoff de 1m, 5m, 30m, 2h e 12h | 3 tentativas (CAN-ALT-004); retry indefinido (CAN-ALT-005) | Cobre uma janela de indisponibilidade de aproximadamente 15 horas sem deixar o evento pendurado para sempre; em troca, falhas mais longas que essa janela exigem replay manual | CAN-DEC-006 / ADR-003 |
| D-5 | DLQ em tabela dedicada (`webhook_dead_letter`) | Marcar o evento como "failed" na própria outbox (CAN-ALT-006) | Mantém a outbox limpa e cria um registro claro de falhas para investigação; em troca, adiciona uma tabela e um fluxo de reprocessamento | CAN-DEC-008 / ADR-003 |
| D-6 | Assinatura HMAC-SHA256 com secret única por endpoint e rotação com grace de 24h | Secret global compartilhada por todos os endpoints | Limita o alcance de um vazamento (o problema "se vaza uma, vaza tudo" deixa de valer) e permite trocar a secret sem interrupção; em troca, há mais secrets para gerir e um período em que duas convivem | CAN-DEC-011, CAN-DEC-012, CAN-DEC-013 / ADR-004 |
| D-7 | Entrega at-least-once com deduplicação delegada ao cliente via `X-Event-Id` | Entrega exactly-once (CAN-ALT-011) | Evita coordenação bilateral complexa e segue o padrão de mercado (Stripe/GitHub); em troca, o mesmo evento pode chegar mais de uma vez e o cliente precisa deduplicar | CAN-DEC-014, CAN-DEC-015 / ADR-005 |
| D-8 | Payload gravado como snapshot no momento da inserção na outbox | Guardar apenas o `order_id` e renderizar o payload no envio (CAN-ALT-009) | Preserva o estado do pedido no instante exato da mudança; em troca, armazena o payload já renderizado na outbox | CAN-DEC-023 / FDD |
| D-9 | Payload compacto, sem os itens do pedido | Payload completo com os itens do pedido | Mantém o envio pequeno (favorece o limite de 64 KB) e desacopla o contrato dos detalhes; em troca, o cliente faz uma chamada extra a `GET /orders/:id` quando precisa de detalhes | CAN-DEC-025 / RFC, FDD |
| D-10 | Payload acima de 64 KB é recusado com erro | Truncar o payload que excede o limite (CAN-ALT-008) | Falha visível e diagnosticável em vez de entregar dado corrompido; em troca, um evento acima do limite não é entregue | CAN-RNF-005 / RFC, FDD |
| D-11 | Filtro de status aplicado na inserção na outbox | Filtrar os status apenas no momento do envio | Não grava linhas para status que nenhum webhook do cliente deseja; em troca, a lógica de inserção passa a depender da configuração vigente | CAN-DEC-020 / ADR-001 |
| D-12 | Replay da DLQ manual, restrito à role ADMIN e auditado | Replay automático ou acessível a qualquer operador ou ao cliente | Reprocessamento controlado e rastreável ("mexer em fila de entrega de notificação não é coisa de operador"); em troca, depende de ação humana com privilégio | CAN-DEC-010, CAN-RNF-006 / ADR-003 |
| D-13 | `customer_id` informado no body ou no path, não derivado do JWT | Derivar o `customer_id` do JWT do requisitante | O JWT atual é do operador, não do cliente, então derivar seria incorreto; em troca, o contrato precisa transportar o `customer_id` (body vs path ainda em aberto — CAN-OPEN-001) | CAN-DEC-019 / RFC |
| D-14 | PK em UUID nas novas tabelas | ID auto-incremental para a outbox (CAN-ALT-010) | Mantém consistência com o padrão de identificadores do projeto; sem contrapartida relevante | CAN-DEC-022 / ADR-006 |
| D-15 | Reuso máximo dos padrões existentes (estrutura modular, `AppError`, Pino, middleware de erro, Zod, `requireRole`) | Redesenhar a arquitetura para o módulo de webhooks | Velocidade e consistência para um time pequeno; em troca, herda eventuais limitações do design atual | CAN-DEC-016 / ADR-006 |

**Trade-off central da feature:** priorizou-se **simplicidade operacional e reuso** (uma única base, um único worker, sem infraestrutura nova) em detrimento de **escala máxima e garantias fortes** (múltiplos workers, exactly-once, ordenação global). Essa escolha permite atender os três clientes solicitantes dentro do prazo, deixando escala e garantias mais fortes como evolução posterior (ver **Itens adiados**). As decisões marcadas com ADR têm o registro completo do contexto, das forças e das consequências nos respectivos documentos ADR-001 a ADR-006.

---

## Regras de negócio

- **RN-001.** Apenas URLs HTTPS podem ser cadastradas (CAN-RNF-004).
- **RN-002.** Cada endpoint possui uma secret própria; não há secret global (CAN-RNF-002, CAN-DEC-012).
- **RN-003.** Apenas mudanças para status selecionados por algum webhook do cliente produzem uma entrega (CAN-RF-007, CAN-DEC-020).
- **RN-004.** Transições de status inválidas não efetivam mudança e, portanto, não produzem evento (CAN-COD-003, CAN-RNF-008).
- **RN-005.** Após a rotação, a secret anterior permanece válida por 24 horas e depois é invalidada (CAN-RF-011, CAN-MET-006).
- **RN-006.** O tipo do evento desta versão é `order.status_changed` (CAN-DEC-024).
- **RN-007.** O payload representa um snapshot do estado do pedido no momento da mudança (CAN-RF-018, CAN-DEC-023).
- **RN-008.** O payload não inclui os itens do pedido; detalhes são obtidos via `GET /orders/:id` (CAN-DEC-025).
- **RN-009.** O payload não ultrapassa 64 KB; acima disso, erro (CAN-RNF-005, CAN-MET-007).
- **RN-010.** O mesmo evento pode ser entregue mais de uma vez (CAN-RNF-009).
- **RN-011.** O consumidor deduplica pelo `X-Event-Id` (CAN-DEC-015).
- **RN-012.** As novas tentativas do mesmo evento preservam o mesmo `X-Event-Id` (CAN-RF-015). *Nota: se o replay a partir da DLQ preserva ou não o mesmo identificador está `Em aberto` (ver Questões em aberto).*
- **RN-013.** O replay exige role ADMIN (CAN-DEC-010).
- **RN-014.** O replay deve ser auditado (executor registrado) (CAN-RNF-006).
- **RN-015.** O esgotamento das tentativas leva o evento à DLQ (CAN-RF-013, CAN-DEC-007).
- **RN-016.** Critério de falha por resposta HTTP (quais respostas contam como falha, além do timeout de 10s): `Em aberto` (CAN-LAC-007).

---

## Critérios de aceite

### CA-001 — Criar webhook com URL HTTPS

**Dado** que o usuário está autenticado
**E** informa um cliente válido
**E** informa uma URL HTTPS válida
**E** informa status permitidos
**Quando** solicita a criação da configuração
**Então** a configuração deve ser registrada
**E** uma secret deve ser gerada.

### CA-002 — Rejeitar URL HTTP

**Dado** que o usuário está autenticado
**E** informa uma URL HTTP (não HTTPS)
**Quando** solicita a criação da configuração
**Então** a criação deve ser recusada com erro de validação.

### CA-003 — Listar configurações

**Dado** que existem webhooks cadastrados para um cliente
**Quando** o usuário autenticado solicita a listagem daquele cliente
**Então** as configurações do cliente devem ser retornadas.

### CA-004 — Atualizar configuração

**Dado** que existe uma configuração de webhook
**Quando** o usuário autenticado envia uma atualização válida
**Então** a configuração deve refletir os novos dados.

### CA-005 — Remover configuração

**Dado** que existe uma configuração de webhook
**Quando** o usuário autenticado solicita a remoção
**Então** a configuração deve deixar de existir.

### CA-006 — Filtro de status

**Dado** que um webhook seleciona apenas um subconjunto de status
**Quando** um pedido muda para um status **não** selecionado por nenhum webhook do cliente
**Então** nenhuma entrega deve ser produzida para aquele status.

### CA-007 — Transição válida produz evento

**Dado** que um webhook seleciona um determinado status
**E** ocorre uma transição de status válida para esse status
**Quando** a mudança de status é efetivada
**Então** um evento correspondente deve ser produzido.

### CA-008 — Transição inválida não produz evento

**Dado** que uma transição de status é inválida
**Quando** a mudança não é efetivada
**Então** nenhum evento deve ser produzido.

### CA-009 — Payload correto

**Dado** que um evento é produzido
**Quando** ele é entregue
**Então** o payload deve conter os campos definidos (`event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents`)
**E** deve representar o snapshot do momento da mudança
**E** não deve incluir os itens do pedido.

### CA-010 — Headers corretos

**Dado** que um evento é entregue
**Quando** a requisição chega ao consumidor
**Então** ela deve conter `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`.

### CA-011 — HMAC validável

**Dado** que o consumidor conhece a secret do endpoint
**Quando** recebe um evento com `X-Signature`
**Então** deve conseguir recalcular o HMAC-SHA256 sobre o corpo e confirmar a assinatura.

### CA-012 — X-Event-Id presente

**Dado** que um evento é entregue
**Quando** a requisição chega ao consumidor
**Então** ela deve conter um `X-Event-Id` no formato UUID.

### CA-013 — Preservação do ID em retries

**Dado** que uma entrega falhou e será retentada
**Quando** o mesmo evento é reenviado
**Então** o `X-Event-Id` deve ser o mesmo das tentativas anteriores.

### CA-014 — Timeout tratado como falha

**Dado** que o endpoint do cliente não responde em até 10 segundos
**Quando** a tentativa expira
**Então** ela deve ser tratada como falha e marcada para nova tentativa.

### CA-015 — Retry após falha

**Dado** que uma entrega falhou
**Quando** o teto de tentativas ainda não foi atingido
**Então** uma nova tentativa deve ser programada.

### CA-016 — Intervalos de backoff

**Dado** que ocorrem falhas sucessivas
**Quando** as novas tentativas são programadas
**Então** elas devem seguir a progressão 1m, 5m, 30m, 2h e 12h.

### CA-017 — DLQ após esgotamento

**Dado** que as cinco tentativas se esgotaram
**Quando** a última tentativa falha
**Então** o evento deve ser movido para a DLQ.

### CA-018 — Replay exige ADMIN

**Dado** que um item está na DLQ
**Quando** um usuário sem role ADMIN tenta executar o replay
**Então** a ação deve ser recusada
**E** quando um usuário ADMIN executa, o item deve voltar ao fluxo de entrega.

### CA-019 — Auditoria do replay

**Dado** que um administrador executa um replay
**Quando** a ação é concluída
**Então** o executor deve ficar registrado para auditoria.

### CA-020 — Rotação de secret

**Dado** que existe um webhook com secret
**Quando** o cliente solicita a rotação
**Então** uma nova secret deve ser gerada.

### CA-021 — Convivência de secrets por 24 horas

**Dado** que uma rotação de secret ocorreu
**Quando** um evento é entregue dentro das 24 horas seguintes
**Então** a assinatura deve poder ser validada com a secret anterior
**E** após 24 horas, a secret anterior deve deixar de ser válida.

### CA-022 — Proteção de logs

**Dado** que a plataforma registra logs durante a entrega
**Quando** um log é gravado
**Então** ele não deve expor a secret nem a assinatura.

### CA-023 — Últimas 100 entregas

**Dado** que um webhook teve mais de 100 entregas
**Quando** o usuário autenticado consulta o histórico
**Então** as últimas 100 entregas devem ser retornadas.

### CA-024 — Limite de 64 KB

**Dado** que um payload excederia 64 KB
**Quando** a plataforma prepara a entrega
**Então** o envio deve ser recusado com erro.

### CA-025 — Cliente sem configuração não recebe evento

**Dado** que um cliente não tem nenhum webhook configurado para um status
**Quando** um pedido desse cliente muda para esse status
**Então** nenhuma entrega deve ser produzida.

*Não são definidos aqui critérios sobre: status HTTP considerado sucesso; política para 4xx/5xx; preservação do `event_id` no replay; paginação do histórico além das 100; e forma de armazenamento da secret — pois essas questões permanecem em aberto (ver Questões em aberto).*

---

## Estratégia de testes e validação

Esta seção descreve **como a feature será validada** antes e durante o lançamento. Ela reaproveita a infraestrutura de testes já existente no projeto (supertest, `tests/setup.ts` com limpeza por `beforeEach`, e factories que criam usuário/cliente/produto — CAN-COD-021) e cobre os comportamentos definidos nos **Critérios de aceite** (CA-001 a CA-025). A estimativa de três sprints inclui explicitamente testes de integração ponta a ponta e a revisão de segurança (`[09:46]` Larissa/Sofia — CAN-MET-011, CAN-INT-009).

### Níveis de teste

- **Testes unitários** — lógica isolada do módulo de webhooks: geração e verificação da assinatura HMAC-SHA256, montagem do payload e dos headers, cálculo da progressão de backoff, decisão de mover o evento para a DLQ ao esgotar as tentativas e aplicação do filtro de status.
- **Testes de integração** — exercitam o fluxo pela API com supertest, no mesmo padrão de `tests/orders.test.ts` (CAN-COD-021): CRUD de configuração, produção do evento dentro da transação de mudança de status e consulta ao histórico de entregas.
- **Testes de ponta a ponta do worker** — validam o ciclo completo: mudança de status → gravação na outbox → leitura pelo worker via polling → envio HTTP → registro da tentativa → retry ou DLQ. Exigem um endpoint de destino controlado (stub) para simular sucesso, timeout e falha.
- **Ajuste da base de testes** — `tests/setup.ts` precisará limpar as novas tabelas (configuração, outbox, DLQ e histórico de entregas) no `beforeEach`, seguindo o padrão atual (CAN-COD-021, CAN-INT-009).

### Mapa de validação por comportamento

| Comportamento a validar | Critério(s) de aceite | Nível de teste |
| ----------------------- | --------------------- | -------------- |
| Criação de webhook com URL HTTPS e geração de secret | CA-001 | Integração |
| Recusa de URL HTTP | CA-002 | Integração |
| Listar, atualizar e remover configuração | CA-003, CA-004, CA-005 | Integração |
| Filtro de status (status não assinado não gera entrega) | CA-006, CA-025 | Integração / unitário |
| Transição válida produz evento; transição inválida não produz | CA-007, CA-008 | Integração (atomicidade da outbox) |
| Contrato do payload e dos headers | CA-009, CA-010 | Unitário / integração |
| Assinatura HMAC-SHA256 recalculável pelo consumidor | CA-011 | Unitário |
| `X-Event-Id` em UUID e preservado entre retries | CA-012, CA-013 | Unitário / ponta a ponta |
| Timeout de 10 segundos tratado como falha | CA-014 | Ponta a ponta (worker) |
| Retry após falha e progressão de backoff | CA-015, CA-016 | Unitário / ponta a ponta |
| Envio à DLQ após esgotar as 5 tentativas | CA-017 | Ponta a ponta (worker) |
| Replay restrito a ADMIN e auditado | CA-018, CA-019 | Integração |
| Rotação de secret e convivência por 24 horas | CA-020, CA-021 | Integração / unitário |
| Secret e assinatura não expostas em logs | CA-022 | Unitário (redação do Pino) |
| Histórico das últimas 100 entregas | CA-023 | Integração |
| Limite de 64 KB recusado com erro | CA-024 | Unitário / integração |

### Validação de segurança

Antes do deploy, a feature passa por uma **revisão de segurança dedicada**, com pelo menos **dois dias úteis** reservados (`[09:46]` Sofia — CAN-MET-012). O foco declarado é: o algoritmo e a entropia da geração da secret por endpoint (CAN-LAC-003), a assinatura HMAC-SHA256, e a garantia de que `secret` e `X-Signature` não vazem em log — ponto de atenção, porque a redação atual do Pino **não** inclui `secret` (CAN-COD-013, CAN-INT-007). O armazenamento da secret em repouso (CAN-LAC-004) também deve ser confirmado nessa revisão.

### Validação com os clientes solicitantes

A validação de negócio ocorre na **Fase 2** com Atlas Comercial, MaxDistribuição e Nova Cargo (CAN-PROB-003), em ativação controlada, verificando na prática: a latência de entrega em condições normais (referência inferior a 10 segundos — CAN-MET-001), o correto recebimento e a validação da assinatura pelo consumidor, o comportamento de retry e o conteúdo da DLQ. O feedback dessa fase alimenta os ajustes da Fase 3.

### Pontos ainda não testáveis (dependem de questões em aberto)

Alguns comportamentos só terão critérios de teste definitivos após decisão nas **Questões em aberto**:

- qual resposta HTTP conta como sucesso e quais falhas não devem gerar retry (CAN-LAC-007);
- se o replay a partir da DLQ preserva o mesmo `event_id`;
- paginação do histórico além das últimas 100 entregas (CAN-LAC-008);
- comportamento dos eventos pendentes após remoção ou alteração de um webhook.

Enquanto essas questões não forem resolvidas, os testes correspondentes ficam pendentes e não devem ser presumidos como aprovados.

---

## Métricas de sucesso

### Métricas de produto

- quantidade de clientes com webhook configurado — *Métrica a acompanhar; meta ainda não definida.*
- quantidade de endpoints ativos — *Métrica a acompanhar; meta ainda não definida.*
- percentual dos clientes solicitantes (Atlas, MaxDistribuição, Nova Cargo) ativados — *Métrica a acompanhar; meta ainda não definida.*
- redução observada do polling dos clientes participantes — *Métrica a acompanhar; meta ainda não definida.*
- tempo necessário para a integração do cliente — *Métrica a acompanhar; meta ainda não definida.*
- quantidade de eventos consumidos com sucesso — *Métrica a acompanhar; meta ainda não definida.*
- utilização do histórico de entregas — *Métrica a acompanhar; meta ainda não definida.*
- quantidade de clientes que concluem a integração — *Métrica a acompanhar; meta ainda não definida.*

### Métricas operacionais

- latência entre a mudança de status e a primeira tentativa de entrega (referência: < 10s em condições normais) — *Métrica a acompanhar; meta ainda não definida além da referência.*
- quantidade de eventos criados — *Métrica a acompanhar.*
- entregas concluídas na primeira tentativa — *Métrica a acompanhar; meta ainda não definida.*
- entregas concluídas após retry — *Métrica a acompanhar; meta ainda não definida.*
- taxa de falhas de entrega — *Métrica a acompanhar; meta ainda não definida.*
- profundidade da Outbox (eventos pendentes) — *Métrica a acompanhar.*
- idade do evento pendente mais antigo — *Métrica a acompanhar.*
- quantidade de itens na DLQ — *Métrica a acompanhar; meta ainda não definida.*
- quantidade de replays executados — *Métrica a acompanhar.*
- falhas por endpoint — *Métrica a acompanhar.*
- tempo de resposta do endpoint do cliente — *Métrica a acompanhar.*
- quantidade de duplicidades identificadas, quando observável — *Métrica a acompanhar; meta ainda não definida.*

### Métricas ainda não definidas

Nenhuma meta numérica de sucesso (por exemplo, disponibilidade, taxa de sucesso, redução percentual de polling ou adoção) foi definida nas fontes. As únicas quantidades sustentadas são limites operacionais e de prazo: menos de 10 segundos (CAN-MET-001); polling de 2 segundos (CAN-MET-002); timeout de 10 segundos (CAN-MET-008); 64 KB (CAN-MET-007); cinco tentativas (CAN-MET-004); intervalos 1m/5m/30m/2h/12h (CAN-MET-005); grace period de 24 horas (CAN-MET-006); últimas 100 entregas (CAN-MET-009); três sprints (CAN-MET-011); e ao menos dois dias úteis de revisão de segurança (CAN-MET-012). Esses valores são limites de projeto, não metas de resultado.

---

## Riscos

| ID | Risco | Impacto | Evidência ou probabilidade qualitativa | Mitigação ou estado |
| -- | ----- | ------- | -------------------------------------- | ------------------- |
| R-01 | Cliente indisponível por período longo deixa eventos pendentes | Atraso na notificação; possível perda do evento | Ocorrência já vista (indisponibilidade de horas); não quantificada (CAN-RISK-001) | Backoff + 5 tentativas + DLQ |
| R-02 | Cliente recebe o mesmo evento mais de uma vez | Efeitos aplicados em duplicidade no consumidor | Inerente ao *at-least-once* (CAN-RISK-002) | `X-Event-Id` para dedup no cliente |
| R-03 | Consumidor não implementar deduplicação | Processamento duplicado do lado do cliente | Não avaliada; responsabilidade do cliente (CAN-DEC-015) | Documentar contrato com destaque |
| R-04 | Secret exposta (ex.: em log do cliente) | Comprometimento da autenticidade daquele endpoint | Ocorrência já vista (CAN-RISK-003) | Secret por endpoint + rotação com grace de 24h |
| R-05 | Assinatura validada incorretamente pelo consumidor | Rejeição indevida ou aceitação insegura | Não avaliada; responsabilidade do cliente | Documentação clara de validação HMAC-SHA256 |
| R-06 | Worker parado/derrubado | Interrupção de todas as entregas | Não avaliada; supervisão em produção é lacuna (CAN-LAC-010) | Worker como processo separado; supervisão a definir |
| R-07 | Backlog de eventos acumulados | Aumento da latência de entrega | Não avaliada | Monitorar profundidade da Outbox |
| R-08 | Crescimento da Outbox sem arquivamento | Degradação de desempenho ao longo do tempo | Não avaliada; arquivamento adiado (CAN-FORA-003) | Arquivamento adiado; monitorar |
| R-09 | Ordenação incorreta ao escalar para múltiplos workers | Eventos fora de ordem para um pedido | Conhecida; mitigada por single-worker (CAN-RISK-004) | Single-worker na v1; escala adiada |
| R-10 | Retry storm (muitas tentativas simultâneas) | Sobrecarga do worker e/ou do cliente | Não avaliada | Backoff crescente; observar |
| R-11 | Payload exceder o limite de tamanho | Evento não entregue | Baixa (payload é compacto) (CAN-RISK-006) | Limite de 64 KB com erro |
| R-12 | Replay indevido | Reentrega não intencional | Não avaliada | Replay restrito a ADMIN + auditoria |
| R-13 | Baixa adoção pelos clientes | Feature subutilizada | Não avaliada | Suporte próximo na ativação inicial |
| R-14 | Atraso no prazo comercial | Risco de migração da Atlas para concorrente | Declarado na reunião (CAN-RISK-008) | Estimativa de 3 sprints; acompanhamento |
| R-15 | Documentação de integração insuficiente | Dificuldade de integração pelos clientes | Não avaliada | Documentação pública dedicada |
| R-16 | Cliente remover/alterar endpoint com eventos pendentes | Entregas órfãs ou perdidas | Não avaliada; comportamento em aberto | Definir comportamento (ver Questões em aberto) |
| R-17 | Política de retenção não definida (DLQ/histórico) | Crescimento indefinido de dados | Não avaliada; lacuna (CAN-LAC-009) | Definir retenção |
| R-18 | Classificação incorreta de falhas (4xx vs 5xx) | Retentar o que não deveria, ou desistir cedo | Não avaliada; lacuna (CAN-LAC-007) | Definir critério de falha |
| R-19 | Suporte operacional insuficiente na ativação | Falhas não tratadas a tempo | Não avaliada | Reserva de suporte na Fase 2 |

---

## Dependências

- fluxo atual de mudança de status do pedido (ponto de integração — CAN-COD-001);
- banco de dados MySQL existente (CAN-COD-002);
- ORM/acesso a dados existente (Prisma) (CAN-COD-015);
- autenticação existente por JWT (CAN-COD-007);
- roles existentes, incluindo ADMIN (CAN-COD-022);
- validação de entrada existente (Zod) (CAN-COD-008);
- logging existente (Pino) (CAN-COD-013);
- implantação e operação de um processo separado (o worker) (CAN-DEC-028);
- novas estruturas persistentes (configuração, Outbox, DLQ, histórico de entregas) (CAN-INT-002);
- revisão de segurança antes do deploy (CAN-MET-012);
- documentação de integração para os clientes (CAN-DEC-015);
- disponibilidade dos clientes solicitantes para testes;
- suporte de operações;
- processo de deploy;
- testes de integração (CAN-INT-009).

Não há, nas fontes, decisão por infraestrutura adicional. Redis, Kafka, KMS, HSM, Prometheus, Grafana ou Kubernetes **não** são dependências desta versão (mensageria externa foi explicitamente descartada — CAN-ALT-002).

---

## Premissas

- os clientes conseguem disponibilizar um endpoint HTTPS de recebimento;
- os clientes conseguem validar assinaturas HMAC-SHA256 (padrão de mercado — `[09:20]` Sofia);
- os clientes conseguem deduplicar eventos pelo `X-Event-Id`;
- o volume inicial é compatível com um único worker (CAN-DEC-005);
- o MySQL existente suporta a carga inicial da Outbox e demais tabelas;
- os clientes respondem dentro do timeout esperado de 10 segundos;
- o payload compacto é suficiente para o cliente disparar suas ações, buscando detalhes na API quando necessário (CAN-DEC-025);
- os detalhes completos do pedido podem ser obtidos via `GET /orders/:id`;
- os clientes solicitantes participarão da fase de validação;
- a equipe poderá operar um segundo processo (o worker) em produção.

Estas são premissas, não fatos. Distinguem-se de: **fatos confirmados** (ex.: existência do fluxo de mudança de status no código — CAN-COD-001); **decisões** (ex.: single-worker — CAN-DEC-005); e **riscos** (ex.: cliente indisponível — CAN-RISK-001).

---

## Restrições

- prazo estimado de três sprints, incluindo a revisão de segurança (CAN-MET-011);
- ao menos dois dias úteis reservados para revisão de segurança antes do deploy (CAN-MET-012);
- uso do MySQL existente, sem nova infraestrutura de banco (CAN-DEC-001);
- um único worker na primeira versão (CAN-DEC-005);
- sem infraestrutura de mensageria adicional (CAN-ALT-002);
- payload máximo de 64 KB (CAN-MET-007);
- timeout de entrega de 10 segundos (CAN-MET-008);
- entrega em menos de 10 segundos em condições normais (CAN-MET-001);
- polling de 2 segundos (CAN-MET-002);
- compatibilidade com o código e os padrões atuais (CAN-DEC-016);
- entrega *at-least-once* (CAN-RNF-009);
- sem ordenação global (CAN-RNF-013);
- sem garantia *exactly-once* (CAN-ALT-011).

Não há, nas fontes, número de desenvolvedores, orçamento, datas de calendário, quantidade de servidores, capacidade máxima ou quantidade máxima de clientes; nada disso é assumido.

---

## Questões em aberto

Somente questões com impacto em produto, contrato, experiência, segurança, operação ou suporte:

1. O `customer_id` será informado no body ou no path dos endpoints de configuração? (CAN-OPEN-001)
2. Qual será o caminho do endpoint de rotação de secret?
3. A secret será exibida somente na criação e na rotação?
4. A secret poderá ser recuperada posteriormente? (CAN-LAC-003/CAN-LAC-004)
5. Como a secret será apresentada ao usuário?
6. Configurações poderão ser desativadas sem remoção?
7. O que acontecerá com eventos pendentes após remoção ou desativação de um webhook?
8. Qual será a paginação do histórico de entregas? (CAN-LAC-008)
9. O histórico sempre mostrará apenas as últimas 100 entregas ou terá navegação? (CAN-MET-009/CAN-LAC-008)
10. Qual resposta HTTP será considerada sucesso? (CAN-LAC-007)
11. Quais falhas não deverão gerar retry? (CAN-LAC-007)
12. O replay preservará o mesmo `event_id`?
13. O cliente terá visibilidade de itens da DLQ?
14. Qual política de retenção (DLQ e histórico) será comunicada? (CAN-LAC-009)
15. Como será comunicada a expiração da secret anterior?
16. Haverá revogação imediata de secret (fora do grace period)?
17. Haverá limite de webhooks por cliente?
18. Haverá limite de endpoints iguais (mesma URL)?
19. Como será tratado um endpoint alterado com eventos pendentes?
20. Como serão comunicadas mudanças futuras no contrato do evento? (CAN-LAC-011)
21. A configuração começa ativa imediatamente após a criação?
22. Como o usuário saberá que uma entrega entrou na DLQ?
23. Haverá forma de testar o webhook antes da ativação?
24. Qual será a política de suporte aos clientes iniciais?

Questões puramente internas de implementação (por exemplo, estratégia de claim/lock, nome de arquivo do worker, cliente HTTP) não são listadas aqui, exceto quando têm impacto direto no produto.

---

## Estratégia de lançamento

### Fase 1 — Preparação interna

- conclusão do desenvolvimento da feature;
- testes internos, incluindo testes de integração (CAN-INT-009);
- revisão de segurança (HMAC e geração de secret), com ao menos dois dias úteis reservados (CAN-MET-012);
- documentação de integração para os clientes;
- preparação operacional (implantação e operação do worker);
- validação de logs e da proteção de secrets/assinaturas (CAN-INT-007);
- validação do processo do worker separado (CAN-DEC-028).

### Fase 2 — Clientes solicitantes

Clientes confirmados: Atlas Comercial, MaxDistribuição e Nova Cargo (CAN-PROB-003).

- ativação controlada;
- acompanhamento das primeiras entregas;
- verificação da latência de entrega;
- análise de falhas e da DLQ;
- coleta de feedback;
- suporte próximo durante a integração.

A ordem de ativação entre os três clientes não é definida por falta de evidência.

### Fase 3 — Expansão

- correção dos problemas encontrados na Fase 2;
- consolidação da documentação;
- disponibilização para outros clientes;
- avaliação de rate limiting de saída (CAN-OPEN-003);
- avaliação de múltiplos workers (CAN-FORA-005);
- avaliação de novos tipos de evento (CAN-LAC-011).

Permanecem em aberto: uso de beta, feature flag, rollout percentual, datas e quantidade de clientes por fase.

---

## Critérios de aprovação do PRD

O PRD será considerado aprovado quando:

- o problema estiver claramente descrito;
- os públicos estiverem identificados;
- os objetivos forem verificáveis;
- escopo e fora de escopo estiverem separados;
- itens adiados estiverem identificados;
- os requisitos funcionais forem atômicos;
- os requisitos não funcionais estiverem sustentados pelas fontes;
- os critérios de aceite forem testáveis;
- as métricas não tiverem metas inventadas;
- riscos e dependências estiverem explícitos;
- as questões abertas não estiverem resolvidas silenciosamente;
- o documento estiver consistente com RFC, FDD e ADRs;
- nenhum detalhe técnico indevido tiver sido incluído;
- a rastreabilidade estiver completa;
- o documento puder ser entendido por áreas não técnicas.

---

## Documentos relacionados

- [RFC — Webhooks de Notificação de Mudança de Status de Pedidos](./RFC.md)
- [FDD — Webhooks de Notificação de Mudança de Status de Pedidos](./FDD.md)
- [ADR-001 — Transactional Outbox no MySQL](./adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker separado com polling](./adrs/ADR-002-worker-separado-com-polling.md)
- [ADR-003 — Retry, backoff e DLQ](./adrs/ADR-003-retry-backoff-e-dlq.md)
- [ADR-004 — HMAC-SHA256 e gestão de secrets](./adrs/ADR-004-hmac-sha256-e-gestao-de-secrets.md)
- [ADR-005 — Entrega at-least-once e Event ID](./adrs/ADR-005-entrega-at-least-once-com-event-id.md)
- [ADR-006 — Reuso dos padrões do projeto](./adrs/ADR-006-reuso-dos-padroes-do-projeto.md)

---

## Evidências e rastreabilidade

| Item do PRD | Tipo de fonte | Localização | IDs canônicos relacionados |
| ----------- | ------------- | ----------- | -------------------------- |
| Problema de polling | Transcrição | `[09:00]` Marcos | CAN-PROB-001, CAN-PROB-002 |
| Clientes solicitantes | Transcrição | `[09:00]` Marcos | CAN-PROB-003 |
| Meta inferior a 10 segundos | Transcrição | `[09:02]` Marcos | CAN-MET-001, CAN-RNF-007 |
| Risco comercial | Transcrição | `[09:00]`, `[09:45]` Marcos | CAN-PROB-004, CAN-RISK-008, CAN-MET-013 |
| CRUD de configuração | Transcrição | `[09:31]` Marcos; `[09:33]` Bruno | CAN-RF-002, CAN-RF-004, CAN-RF-005, CAN-RF-006 |
| Filtro de status | Transcrição | `[09:33]` Marcos; `[09:34]` Bruno/Diego | CAN-RF-007, CAN-DEC-020 |
| Rotação de secret | Transcrição | `[09:21]` Sofia | CAN-RF-010, CAN-RF-011 |
| Últimas 100 entregas | Transcrição | `[09:34]` Marcos | CAN-RF-008, CAN-MET-009 |
| Cinco tentativas | Transcrição | `[09:15]` Diego; `[09:17]` Larissa | CAN-MET-004, CAN-RNF-010 |
| Intervalos de backoff | Transcrição | `[09:17]` Diego | CAN-MET-005 |
| DLQ | Transcrição | `[09:15]` Diego; `[09:18]` Diego | CAN-RF-013, CAN-DEC-007, CAN-DEC-008 |
| Replay ADMIN | Transcrição | `[09:35]` Diego; `[09:36]` Sofia/Larissa | CAN-RF-009, CAN-DEC-010 |
| HMAC-SHA256 | Transcrição | `[09:20]` Sofia | CAN-RF-016, CAN-RNF-001, CAN-MET-014 |
| Secret por endpoint | Transcrição | `[09:21]` Sofia | CAN-RNF-002, CAN-DEC-012 |
| Grace period de 24 horas | Transcrição | `[09:21]` Sofia | CAN-MET-006, CAN-RNF-003 |
| Entrega at-least-once | Transcrição | `[09:24]` Diego | CAN-RNF-009, CAN-DEC-014 |
| `X-Event-Id` | Transcrição | `[09:25]` Diego | CAN-RF-014, CAN-RF-015, CAN-DEC-015 |
| Payload (contrato) | Transcrição | `[09:43]` Diego; `[09:44]` Bruno | CAN-DEC-024, CAN-DEC-025 |
| Payload como snapshot | Transcrição | `[09:52]` Larissa/Diego | CAN-RF-018, CAN-DEC-023 |
| Limite de 64 KB | Transcrição | `[09:23]` Sofia; `[09:24]` Diego/Larissa | CAN-RNF-005, CAN-MET-007 |
| Timeout de 10 segundos | Transcrição | `[09:42]` Diego | CAN-RNF-012, CAN-MET-008 |
| Fora de escopo (dashboard, e-mail) | Transcrição | `[09:37]`, `[09:40]` Marcos/Larissa | CAN-FORA-001, CAN-FORA-002 |
| Itens adiados (multi-worker, rate limiting, arquivamento) | Transcrição | `[09:08]`, `[09:13]`, `[09:38]` Diego | CAN-FORA-003, CAN-FORA-005, CAN-OPEN-003, CAN-MET-010 |
| Riscos | Transcrição | `[09:12]`–`[09:44]` (vários) | CAN-RISK-001 a CAN-RISK-009 |
| Métricas (limites e prazo) | Transcrição | `[09:02]`, `[09:46]` Marcos/Larissa/Sofia | CAN-MET-001 a CAN-MET-014 |
| Worker separado | Transcrição | `[09:11]` Diego | CAN-RNF-011, CAN-DEC-028 |
| Sem ordering global | Transcrição | `[09:12]` Diego; `[09:14]` Marcos | CAN-RNF-013, CAN-RNF-014 |
| Reuso de padrões | Transcrição | `[09:30]` Larissa | CAN-DEC-016, CAN-RNF-015 a CAN-RNF-020 |
| Ponto de integração (mudança de status) | Código (contexto/restrição) | `src/modules/orders/order.service.ts` — `changeStatus` | CAN-COD-001, CAN-INT-001 |
| Autenticação/roles existentes | Código (contexto/restrição) | `src/middlewares/auth.middleware.ts`; `src/modules/users/user.routes.ts` | CAN-COD-007, CAN-COD-022 |
| Proteção de secrets em logs | Código (contexto/restrição) | `src/shared/logger/index.ts` — `redactPaths` | CAN-COD-013, CAN-INT-007 |
| `customer_id` (body/path) | Transcrição (em aberto) | `[09:32]` Larissa; `[09:33]` Bruno | CAN-OPEN-001, CAN-DEC-019 |
| Rate limiting de saída (em aberto) | Transcrição (em aberto) | `[09:38]` Diego; `[09:39]` Larissa | CAN-OPEN-003, CAN-RISK-007 |
| Prazo (3 sprints / fim de novembro) | Transcrição | `[09:46]` Larissa; `[09:45]` Marcos | CAN-MET-011, CAN-MET-013 |
| Revisão de segurança (2 dias úteis) | Transcrição | `[09:46]` Sofia | CAN-MET-012 |

> Rastreabilidade adicional consolidada em `MATRIZ-FATOS.md` (matriz canônica de fatos, seção 2) e em `TRANSCRICAO.md` (registro da reunião). Os detalhes técnicos de cada decisão estão nos ADR-001 a ADR-006, no RFC e no FDD; este PRD referencia essas decisões sem reproduzir seu conteúdo.
