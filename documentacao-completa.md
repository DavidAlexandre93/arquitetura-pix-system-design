[← Voltar ao início](README.md)

# Arquitetura de Sistemas de Pagamento com Pix

> Guia completo em página única para estudo, entrevistas técnicas e discussões de arquitetura.

## Escopo e premissas

Esta documentação descreve uma arquitetura de referência para uma plataforma que oferece pagamentos via Pix, como um banco, instituição de pagamento, fintech ou plataforma integradora.

Nem todos os componentes descritos são obrigatórios pelo Banco Central. Elementos como Kafka, CQRS, Saga, Event Sourcing, Redis, gRPC e service mesh são decisões internas de arquitetura. Já DICT, SPI, MED e requisitos de segurança fazem parte do ecossistema e da regulamentação do Pix.

O objetivo é ensinar como pensar uma plataforma segura: onde exigir consistência forte, onde aceitar consistência eventual, como evitar duplicidade financeira, como lidar com timeouts, como publicar eventos sem dual write e como operar o sistema em produção.

## Sumário

0. [Como estudar esta documentação](#0-como-estudar-esta-documentação)
1. [Princípios arquiteturais](#1-princípios-arquiteturais)
2. [Componentes do ecossistema Pix](#2-componentes-do-ecossistema-pix)
3. [Arquitetura de referência](#3-arquitetura-de-referência)
4. [Consistência financeira, ledger e concorrência](#4-consistência-financeira-ledger-e-concorrência)
5. [Idempotência](#5-idempotência)
6. [Sincronismo, assincronismo e protocolos](#6-sincronismo-assincronismo-e-protocolos)
7. [Mensageria, filas, PubSub e streaming](#7-mensageria-filas-pubsub-e-streaming)
8. [Outbox, CDC, retry e DLQ](#8-outbox-cdc-retry-e-dlq)
9. [CQRS, Saga e Event Sourcing](#9-cqrs-saga-e-event-sourcing)
10. [Cache, rate limit e backpressure](#10-cache-rate-limit-e-backpressure)
11. [Escalabilidade e alta disponibilidade](#11-escalabilidade-e-alta-disponibilidade)
12. [Sidecar, service mesh, Ambassador e Adapter](#12-sidecar-service-mesh-ambassador-e-adapter)
13. [Webhooks e WebSockets](#13-webhooks-e-websockets)
14. [Segurança](#14-segurança)
15. [Observabilidade, operação e documentação](#15-observabilidade-operação-e-documentação)
16. [Conciliação](#16-conciliação)
17. [Resumo para entrevista](#17-resumo-para-entrevista)
18. [Checklist técnico](#18-checklist-técnico)
19. [Referências oficiais e técnicas](#19-referências-oficiais-e-técnicas)
20. [Glossário e perguntas de revisão](#20-glossário-e-perguntas-de-revisão)

---

# 0. Como estudar esta documentação

Este guia foi escrito para ser lido de duas formas: como trilha de estudo, do começo ao fim, e como material de consulta durante uma discussão de arquitetura.

## 0.1 Modelo mental

Ao projetar uma plataforma Pix, separe sempre quatro planos:

```text
Regra do ecossistema Pix
→ o que vem de regulamento, DICT, SPI, MED, segurança e manuais oficiais.

Contrato externo da plataforma
→ APIs, webhooks, autenticação, idempotência e experiência dos clientes.

Domínio financeiro interno
→ saldo, reserva, ledger, máquina de estados, limites e conciliação.

Infraestrutura de execução
→ banco, cache, filas, streaming, Kubernetes, observabilidade e operação.
```

Muitas confusões aparecem quando esses planos são misturados. Por exemplo, Kafka pode ser excelente internamente, mas não é uma exigência do Pix. Da mesma forma, o SPI é parte do ecossistema de liquidação, não "a fila" da sua aplicação.

## 0.2 Ordem recomendada

1. Leia os princípios arquiteturais antes de qualquer tecnologia.
2. Entenda os componentes do ecossistema Pix.
3. Estude a arquitetura de referência como um mapa, não como receita fechada.
4. Aprofunde consistência, ledger e idempotência.
5. Só então compare REST, gRPC, filas, Pub/Sub, streaming, Outbox, Saga e CQRS.
6. Feche com segurança, observabilidade, conciliação e checklist.

## 0.3 Perguntas que guiam o desenho

Antes de escolher uma tecnologia, responda:

- Quem é a fonte autoritativa deste estado?
- O efeito é financeiro, operacional ou apenas derivado?
- A operação pode ser repetida sem duplicar dinheiro?
- O que acontece se o timeout ocorrer depois que o pagamento foi aceito?
- Como detectar e corrigir divergência?
- Como auditar a decisão meses depois?
- Como operar o sistema em incidente, pico de carga ou indisponibilidade parcial?

## 0.4 O que é decisão de produto, regra externa e decisão interna

| Tipo | Exemplos | Como tratar |
|---|---|---|
| Regra externa | DICT, SPI, MED, requisitos de segurança, manuais oficiais | validar sempre no material vigente do Banco Central |
| Decisão de produto | experiência do cliente, limites comerciais, canais, notificações | alinhar com negócio, jurídico, risco e atendimento |
| Decisão interna de arquitetura | banco, cache, broker, service mesh, CQRS, Saga | justificar por requisitos de consistência, escala, operação e custo |

## 0.5 Como usar em entrevista técnica

Em entrevista, evite começar por ferramentas. Comece por invariantes:

```text
Não duplicar débito.
Não autorizar sem saldo ou limite.
Não repetir cegamente operação desconhecida.
Não perder evento confirmado.
Não depender de cache para saldo autoritativo.
Não operar sem trilha de auditoria e conciliação.
```

Depois explique as escolhas: ledger, idempotência, transações locais, Outbox, consumidores idempotentes, estado `UNKNOWN`, reconciliação, segurança e observabilidade.

---

# 1. Princípios arquiteturais

Este capítulo define as prioridades que devem orientar todo o desenho. Em sistemas de pagamento, tecnologia vem depois de invariantes financeiras, rastreabilidade e operação segura.

## 1.1 Performance não pode comprometer a correção financeira

Uma API de pagamentos deve buscar:

- baixa latência;
- alto throughput;
- estabilidade sob picos;
- escalabilidade horizontal;
- disponibilidade;
- uso eficiente de CPU, memória, rede e banco de dados.

Entretanto, em pagamentos, a prioridade deve ser:

```text
Correção financeira
→ integridade e consistência
→ idempotência
→ segurança
→ resiliência
→ disponibilidade
→ performance
```

Uma resposta rápida que duplica um débito é pior do que uma resposta um pouco mais lenta e correta.

Em termos práticos, otimize a latência sem remover proteções como:

- constraint de unicidade;
- transação local;
- controle de concorrência;
- validação de estado;
- registro auditável;
- reconciliação posterior.

Se uma otimização torna impossível provar o que aconteceu com o dinheiro, ela deve ser redesenhada.

## 1.2 Consistência forte e consistência eventual

Use garantias fortes no caminho que autoriza ou movimenta dinheiro:

- saldo disponível;
- reserva de saldo;
- limites transacionais;
- autorização do pagamento;
- lançamentos no ledger;
- controle de concorrência;
- idempotência da operação financeira.

A consistência eventual costuma ser aceitável em dados derivados:

- dashboards;
- analytics;
- notificações;
- indexação para busca;
- projeções de leitura não autoritativas;
- envio de webhooks;
- relatórios não usados para autorizar transações.

> O extrato pode ser uma projeção, mas a fonte usada para autorizar um Pix deve ser o modelo transacional autoritativo.

Regra de bolso:

| Dado ou decisão | Garantia esperada |
|---|---|
| saldo disponível para autorizar pagamento | consistência forte |
| reserva de saldo | consistência forte |
| ledger | consistência forte e auditável |
| notificação push | consistência eventual |
| dashboard operacional | consistência eventual com atraso conhecido |
| analytics | consistência eventual |

## 1.3 Fonte de verdade

Em uma arquitetura baseada em ledger, o ledger é a fonte de verdade financeira. Saldo, extrato e relatórios podem ser projeções dos lançamentos.

Isso é uma recomendação arquitetural, não uma regra que define toda implementação possível do Pix.

O ponto essencial é existir uma fonte autoritativa clara. Se cada serviço "calcula" o saldo de uma forma, a plataforma perde capacidade de auditoria e correção.

## 1.4 Invariantes que o sistema deve proteger

Invariantes são regras que continuam verdadeiras mesmo sob concorrência, retry, falha parcial ou pico de tráfego.

Exemplos:

- uma mesma intenção de pagamento não gera dois débitos;
- saldo disponível não fica negativo quando isso não é permitido;
- cada lançamento financeiro tem vínculo com uma operação de negócio;
- todo débito esperado possui crédito correspondente;
- estados terminais não voltam para estados intermediários sem uma nova operação formal;
- divergências entram em fluxo de conciliação e não são corrigidas por alteração manual invisível.

## 1.5 Perguntas de revisão

- Qual componente decide se há saldo suficiente?
- Qual fonte prova que a operação foi concluída?
- O que impede duas instâncias de debitarem a mesma conta ao mesmo tempo?
- O que acontece se um evento for publicado duas vezes?
- O que acontece se o cliente repetir a mesma requisição?

---

# 2. Componentes do ecossistema Pix

Este capítulo separa os componentes externos do Pix das escolhas internas da plataforma. Essa separação evita explicar o Pix como se ele fosse apenas uma fila, uma API REST ou uma arquitetura de microsserviços.

## 2.1 DICT

**DICT** significa **Diretório de Identificadores de Contas Transacionais**.

Ele é gerido e operado pelo Banco Central e permite associar uma chave Pix aos dados necessários para iniciar o pagamento, conforme as regras de acesso e exposição de dados do arranjo.

Fluxo conceitual:

```text
Chave Pix
   ↓
DICT
   ↓
Dados da conta e do participante recebedor
```

Definição para entrevista:

> O DICT é o diretório autoritativo de chaves Pix, utilizado para localizar os dados da conta transacional vinculada à chave e mitigar riscos na iniciação do pagamento.

Cuidados de modelagem:

- consultas ao DICT podem envolver regras de segurança, privacidade, limitação e auditoria;
- dados obtidos do DICT não devem ser tratados como cadastro permanente sem regra clara de atualização;
- uma plataforma integradora pode acessar o DICT indiretamente, por meio de outro participante ou provedor.

## 2.2 SPI

**SPI** significa **Sistema de Pagamentos Instantâneos**.

É a infraestrutura operada pelo Banco Central que processa e liquida transferências Pix entre participantes.

```text
DICT
→ resolve a chave e identifica o destino.

SPI
→ processa a liquidação entre os participantes.
```

Uma fintech ou plataforma pode integrar-se ao ecossistema por meio de um participante direto, participante indireto, provedor ou parceiro. Portanto, nem toda aplicação empresarial chama DICT e SPI diretamente.

Ao documentar a arquitetura, deixe explícito qual é o papel da instituição:

- participante direto;
- participante indireto;
- provedor de serviço tecnológico;
- plataforma integradora que consome API de um PSP;
- aplicação cliente que apenas inicia ou acompanha pagamentos.

Essa informação muda responsabilidades, certificados, integrações, observabilidade, conciliação e plano de contingência.

## 2.3 MED

**MED** significa **Mecanismo Especial de Devolução**.

É um mecanismo específico do Pix para apoiar a tentativa de recuperação de valores em situações previstas na regulamentação, como fraude, golpe e coerção, além de determinados casos de falha operacional.

Pontos importantes:

- não é equivalente ao chargeback de cartão;
- não garante recuperação integral;
- depende da análise do caso e da disponibilidade de recursos;
- não é o mecanismo adequado para qualquer desacordo comercial ou erro voluntário de envio;
- as regras são atualizadas pelo Banco Central e devem ser consultadas no regulamento vigente.

Desde **1º de outubro de 2025**, o autoatendimento do MED passou a ser obrigatório no ambiente Pix dos aplicativos dos participantes, permitindo que usuários pessoa física contestem transações elegíveis sem depender inicialmente de atendimento humano.

O MED também deve ser entendido como processo operacional e regulatório, não como simples endpoint. Ele envolve análise, notificações, possível bloqueio, prazos, evidências, marcações de fraude e, quando aplicável, devolução total ou parcial.

Fluxo conceitual:

```text
Usuário contesta a transação
          ↓
Instituição inicia o procedimento
          ↓
Participantes envolvidos são notificados
          ↓
Recursos elegíveis podem ser bloqueados
          ↓
Análise do caso
      ┌───┴────┐
      │        │
Procedente   Improcedente
      │        │
Devolução   Desbloqueio
```

## 2.4 Como explicar sem misturar conceitos

Formulação segura:

> DICT localiza dados associados a uma chave Pix. SPI liquida transferências entre participantes. MED define procedimentos de devolução em hipóteses específicas. A arquitetura interna da instituição pode usar APIs, bancos, filas, Outbox, cache, streams e conciliação para cumprir essas responsabilidades com segurança.

Evite dizer:

- "Pix é Kafka";
- "SPI é uma fila";
- "MED é chargeback";
- "todo Pix precisa usar microsserviços";
- "idempotência é garantida só por `@Transactional`".

---

# 3. Arquitetura de referência

Esta arquitetura é uma referência conceitual. Ela mostra fronteiras e responsabilidades, não uma obrigação de produto, cloud ou framework.

```text
┌──────────────────────────┐
│ Aplicativo / Integrador  │
└────────────┬─────────────┘
             ↓
┌──────────────────────────┐
│ WAF / API Gateway / ALB  │
│ Auth, rate limit, routing│
└────────────┬─────────────┘
             ↓
┌──────────────────────────────────────────┐
│ API / Orquestrador de Pagamentos         │
│ validação, idempotência, máquina de estado│
└───────┬────────────┬────────────┬────────┘
        │            │            │
        ↓            ↓            ↓
┌────────────┐ ┌─────────────┐ ┌───────────────┐
│ Antifraude │ │ Limites     │ │ Conta / Ledger│
└────────────┘ └─────────────┘ └───────┬───────┘
                                      │
                                      ↓
                         ┌────────────────────────┐
                         │ Adapter de integração  │
                         │ PSP / participante Pix │
                         └───────────┬────────────┘
                                     ↓
                             DICT / SPI / parceiro

Transação local:
┌──────────────────────────────────────────┐
│ Alteração de negócio + registro Outbox   │
└───────────────────┬──────────────────────┘
                    ↓
              CDC ou Polling
                    ↓
          Kafka / SQS / outro broker
            ├── notificações
            ├── webhooks
            ├── conciliação
            ├── extrato/projeções
            └── auditoria/analytics
```

Leia o diagrama em dois caminhos:

- **caminho financeiro síncrono ou coordenado**: validação, idempotência, limites, antifraude, saldo, ledger e integração Pix;
- **caminho derivado assíncrono**: notificações, webhooks, conciliação, extrato, auditoria e analytics.

O primeiro caminho protege dinheiro. O segundo distribui consequências de algo que já foi aceito, confirmado ou precisa ser acompanhado.

## 3.1 API Gateway

Responsabilidades comuns:

- roteamento;
- autenticação e autorização;
- rate limiting e quotas;
- WAF e proteção contra abuso;
- validações superficiais de protocolo;
- geração ou propagação de `correlationId`;
- observabilidade de entrada.

Regras de negócio financeiras não devem ficar integralmente no gateway.

O gateway pode rejeitar requisições obviamente inválidas, mas não deve ser a fonte autoritativa de saldo, ledger, limites financeiros ou estado final do pagamento.

## 3.2 Orquestrador de pagamentos

Pode coordenar:

1. validação da requisição;
2. idempotência;
3. limites;
4. análise antifraude;
5. reserva de saldo;
6. envio para a integração Pix;
7. acompanhamento do estado;
8. confirmação ou liberação da reserva;
9. atualização do ledger;
10. publicação de eventos.

O desenho exato depende de quem é a fonte autoritativa de cada estado e de como a instituição acessa o ecossistema Pix.

Em sistemas menores, o orquestrador pode ser um módulo dentro de um serviço monolítico bem estruturado. Em sistemas maiores, pode ser um serviço dedicado. O ponto importante é manter a coordenação explícita e auditável.

Decisões que precisam aparecer no desenho:

- quem cria o identificador interno do pagamento;
- quem valida idempotência;
- quem reserva e libera saldo;
- quem registra lançamentos no ledger;
- quem traduz estados do provedor para estados internos;
- quem publica eventos;
- quem inicia conciliação quando o estado fica incerto.

## 3.3 Máquina de estados

Exemplo simplificado:

```text
CREATED
   ↓
VALIDATING
   ↓
PROCESSING
   ├──→ COMPLETED
   ├──→ REJECTED
   └──→ UNKNOWN / PENDING_RECONCILIATION
```

`UNKNOWN` não deve ser tratado automaticamente como falha. Em pagamentos, um timeout pode significar que a operação foi concluída, mas a resposta não chegou. O sistema deve consultar a fonte autoritativa ou executar reconciliação antes de repetir uma operação potencialmente irreversível.

Estados devem ter transições permitidas. Isso evita correções improvisadas e torna incidentes rastreáveis.

Exemplo de regra:

```text
COMPLETED não volta para PROCESSING.
REJECTED não vira COMPLETED sem uma nova evidência externa e trilha de auditoria.
UNKNOWN exige consulta, webhook confiável, conciliação ou intervenção operacional controlada.
```

## 3.4 Anti-patterns comuns

- colocar regra financeira apenas no gateway;
- tratar timeout como falha definitiva;
- repetir envio externo sem idempotência e sem consulta de status;
- atualizar banco e publicar evento em operações separadas sem Outbox ou mecanismo equivalente;
- usar cache como fonte autoritativa de saldo;
- não registrar o motivo de transições manuais;
- não separar estado interno de estado informado por provedor externo.

---

# 4. Consistência financeira, ledger e concorrência

Este capítulo trata do núcleo financeiro. Se esta parte estiver errada, o restante da arquitetura pode estar elegante e ainda assim o sistema será inseguro.

## 4.1 Ledger

Ledger é o registro autoritativo das movimentações financeiras da plataforma.

Características desejáveis:

- lançamentos imutáveis ou corrigidos por contralançamentos;
- rastreabilidade;
- auditoria;
- idempotência;
- vínculo com a transação de negócio;
- integridade transacional;
- reconciliação.

Campos comuns em um lançamento:

- `entryId`;
- `transactionId` ou `paymentId`;
- conta contábil ou bucket;
- valor;
- moeda;
- natureza do lançamento;
- data de criação;
- referência externa, como `endToEndId` quando aplicável;
- metadados auditáveis.

## 4.2 Dupla entrada

Em um ledger de dupla entrada, todo evento contábil gera lançamentos balanceados:

```text
Total de débitos = total de créditos
```

Exemplo conceitual de transferência entre buckets internos:

```text
Reserva:
- reduz disponível
- aumenta reservado

Liquidação:
- reduz reservado
- aumenta conta de liquidação/destino

Rejeição antes da liquidação:
- reduz reservado
- restaura disponível
```

> A nomenclatura exata de débito e crédito depende do plano de contas e da natureza contábil de cada conta. Os exemplos acima representam movimentação lógica de saldo e não substituem a modelagem contábil da instituição.

Validações esperadas:

- soma de débitos e créditos balanceada por transação;
- nenhuma alteração destrutiva sem contralançamento;
- vínculo entre lançamento e evento de negócio;
- unicidade para impedir o mesmo efeito financeiro duas vezes;
- trilha de quem, quando e por qual motivo realizou ajustes operacionais.

## 4.3 Reserva de saldo

A reserva impede que o mesmo recurso seja usado simultaneamente por duas operações.

```text
Saldo disponível: R$ 1.000
Pix A reserva:    R$   700
Disponível:       R$   300
Pix B solicita:   R$   600
Resultado: saldo insuficiente
```

A reserva deve ser atômica, por exemplo com:

- atualização condicional;
- `SELECT ... FOR UPDATE` quando apropriado;
- versionamento otimista;
- serialização por conta ou chave de particionamento;
- constraints e invariantes no banco.

Reserva não é apenas uma flag. Ela é um compromisso temporário do saldo para uma finalidade específica. Por isso deve ter:

- identificador da operação;
- valor reservado;
- estado da reserva;
- prazo ou política de expiração;
- relação com liquidação, rejeição ou cancelamento;
- reconciliação quando ficar pendente além do esperado.

## 4.4 Controle de concorrência

### Pessimista

Bloqueia o registro durante a operação:

```sql
SELECT balance
FROM account
WHERE account_id = :accountId
FOR UPDATE;
```

É simples para invariantes críticas, mas pode aumentar contenção e latência.

### Otimista

Usa uma versão:

```sql
UPDATE account
SET balance = :newBalance,
    version = version + 1
WHERE account_id = :accountId
  AND version = :expectedVersion;
```

Se nenhuma linha for atualizada, ocorreu concorrência e a operação deve ser reavaliada.

## 4.5 Fronteira transacional

Uma fronteira transacional define o conjunto de alterações que confirmam ou falham juntas.

Bom candidato para a mesma transação local:

```text
registrar pagamento
reservar saldo
criar lançamentos necessários
gravar registro de idempotência
gravar evento na outbox
```

Mau candidato para a mesma transação local:

```text
atualizar banco local
chamar PSP externo
esperar webhook
publicar em broker externo
```

Chamadas externas e brokers não devem ser assumidos como parte da transação do banco local. Por isso padrões como Outbox, Saga, consulta de status e conciliação são importantes.

---

# 5. Idempotência

Idempotência é uma das proteções mais importantes em pagamentos. Ela permite repetir uma intenção sem repetir o efeito financeiro.

## 5.1 Onde validar

A idempotência deve existir em várias camadas:

1. **API**: recebe uma `Idempotency-Key`;
2. **código**: valida formato, payload e estado;
3. **transação**: agrupa as alterações locais;
4. **banco**: garante unicidade atomicamente;
5. **consumer**: deduplica mensagens e eventos;
6. **integrações externas**: usa identificadores estáveis e consulta de status quando necessário.

> `@Transactional` sozinho não garante idempotência. Duas instâncias podem executar a mesma lógica concorrentemente. A garantia final de exclusão deve estar em uma operação atômica, normalmente uma `UNIQUE CONSTRAINT` ou atualização condicional no banco.

Idempotência deve ser pensada por escopo. Uma mesma chave pode ser única por cliente, por conta, por endpoint, por operação ou por parceiro. O escopo precisa ser documentado, porque uma chave global demais causa colisões desnecessárias e uma chave ampla de menos permite duplicidade.

## 5.2 API HTTP

```http
POST /v1/pix/payments
Idempotency-Key: 0d8e9f26-6cab-4f61-bca3-0e26889e11fb
```

Armazene pelo menos:

- chave de idempotência;
- escopo da chave, como cliente e operação;
- hash canônico do payload;
- identificador da operação criada;
- estado (`PROCESSING`, `COMPLETED`, `FAILED`);
- resposta recuperável;
- data de criação e expiração.

Inclua também, quando aplicável:

- versão do contrato;
- endpoint ou tipo de operação;
- código de erro recuperável;
- usuário ou credencial que iniciou a chamada;
- data da última leitura;
- lock lógico ou estado `IN_PROGRESS`.

Exemplo de constraint:

```sql
CREATE TABLE idempotency_record (
    client_id        UUID        NOT NULL,
    idempotency_key  VARCHAR(128) NOT NULL,
    request_hash     VARCHAR(128) NOT NULL,
    operation_id     UUID,
    status           VARCHAR(32) NOT NULL,
    response_payload JSONB,
    created_at       TIMESTAMPTZ NOT NULL,
    expires_at       TIMESTAMPTZ,
    CONSTRAINT uk_idempotency UNIQUE (client_id, idempotency_key)
);
```

Comportamento recomendado:

| Situação | Resultado |
|---|---|
| chave nova | cria e processa a operação |
| mesma chave e mesmo payload | retorna a operação/resposta original |
| mesma chave e payload diferente | retorna conflito |
| operação ainda em andamento | retorna estado atual, sem reexecutar o efeito |

Respostas HTTP típicas:

| Situação | Status possível |
|---|---|
| operação criada | `201 Created` ou `202 Accepted` |
| repetição com mesmo payload concluído | mesmo status e corpo recuperável da resposta original |
| repetição enquanto processa | `202 Accepted` ou `409 Conflict`, conforme contrato |
| mesma chave com payload diferente | `409 Conflict` |

Evite o padrão vulnerável a corrida:

```java
if (!repository.existsById(key)) {
    repository.save(record);
}
```

Prefira tentar inserir e tratar a violação de unicidade.

O hash do payload deve ser canônico. Campos em ordem diferente, espaços ou metadados irrelevantes não deveriam transformar a mesma intenção em outra operação, a menos que essa seja uma decisão explícita do contrato.

## 5.3 Consumer idempotente

Brokers normalmente podem entregar novamente uma mensagem. O consumer deve assumir entrega **at least once**.

```sql
CREATE TABLE processed_message (
    consumer_name VARCHAR(100) NOT NULL,
    message_id    UUID         NOT NULL,
    processed_at  TIMESTAMPTZ  NOT NULL,
    CONSTRAINT uk_processed_message
        UNIQUE (consumer_name, message_id)
);
```

O registro da mensagem e o efeito local devem ocorrer na mesma transação:

```text
BEGIN
  inserir message_id processado
  executar regra de negócio
  atualizar estado local
COMMIT
```

Se a regra falhar, o registro de deduplicação também deve sofrer rollback.

Se o consumer chama outro serviço, a chamada externa também precisa de chave idempotente ou identificador de negócio estável. Caso contrário, o consumer local pode ser idempotente e ainda duplicar o efeito no sistema chamado.

## 5.4 Idempotência não significa exatamente uma entrega

Idempotência significa que repetir a mesma intenção produz o mesmo efeito observável. Ela não impede necessariamente mensagens duplicadas; ela impede que duplicidades gerem efeitos financeiros duplicados.

## 5.5 Erros comuns

- usar `exists` seguido de `insert` sem proteção atômica;
- criar idempotência apenas na API e esquecer consumers;
- aceitar mesma chave com payload diferente;
- expirar o registro cedo demais para o risco operacional;
- não salvar resposta recuperável;
- não distinguir falha definitiva de estado desconhecido;
- repetir chamada externa sem identificador idempotente.

---

# 6. Sincronismo, assincronismo e protocolos

Este capítulo ajuda a separar tempo de resposta, protocolo de comunicação e modelo de processamento. Eles se relacionam, mas não são a mesma coisa.

## 6.1 O Pix é síncrono ou assíncrono?

A resposta correta é: **depende da fronteira analisada**.

- Uma chamada HTTP do aplicativo para a API é síncrona no transporte: o cliente envia e espera uma resposta.
- A resposta pode já conter o resultado final ou um estado intermediário, conforme o fluxo e os limites de tempo.
- A arquitetura interna pode usar filas, eventos e processamento assíncrono.
- A confirmação posterior pode chegar por consulta, evento ou webhook.
- O SPI possui protocolos e limites de tempo próprios; não deve ser descrito simplesmente como “uma fila assíncrona”.

Formulação recomendada:

> Uma plataforma Pix pode usar uma interface síncrona na entrada e processamento assíncrono em etapas internas ou derivadas. Isso é uma escolha arquitetural da plataforma, não uma definição de que todo o Pix é assíncrono.

Pergunte sempre: síncrono em qual fronteira?

| Fronteira | Pode ser |
|---|---|
| app chamando sua API | síncrona via HTTP |
| serviço chamando outro serviço interno | síncrona via REST/gRPC ou assíncrona via evento |
| integração com participante/provedor | depende do contrato |
| notificação para cliente | assíncrona via webhook, push ou WebSocket |
| conciliação | normalmente assíncrona e recorrente |

## 6.2 REST, gRPC e GraphQL

| Tecnologia | Uso mais comum na arquitetura |
|---|---|
| REST | APIs públicas, parceiros, webhooks e integrações amplamente compatíveis |
| gRPC | comunicação interna síncrona, contratos tipados e baixa sobrecarga |
| GraphQL | BFF, consultas agregadas e interfaces com necessidades flexíveis de leitura |
| Kafka, SQS ou RabbitMQ | processamento assíncrono, eventos e distribuição de trabalho |

### REST

É uma escolha comum para interfaces externas por causa de:

- compatibilidade;
- semântica HTTP;
- observabilidade;
- autenticação e rate limiting em gateways;
- suporte simples a idempotency key;
- facilidade para parceiros.

### gRPC

É útil entre serviços controlados pela organização:

- contrato definido em `.proto`;
- geração de clientes e servidores;
- serialização eficiente;
- suporte a diferentes linguagens;
- chamadas unary e streaming.

Requer boa configuração de:

- deadlines e timeouts;
- retries seguros;
- circuit breaker;
- observabilidade;
- compatibilidade de schema.

### GraphQL

GraphQL suporta `query`, `mutation` e `subscription`. Portanto, tecnicamente pode executar comandos de pagamento.

Entretanto, para o núcleo transacional, REST ou gRPC costumam facilitar:

- comandos explícitos;
- idempotência por operação;
- governança de payload;
- controles de quota;
- semântica operacional e auditoria.

GraphQL é especialmente útil em consultas agregadas, BFFs e dashboards. Não é proibido no core, mas deve ser escolhido conscientemente.

## 6.3 Critérios de escolha

| Pergunta | Tendência |
|---|---|
| É API pública ou para parceiros variados? | REST costuma ser mais simples de operar |
| É comunicação interna com contrato forte e baixa latência? | gRPC pode ser adequado |
| É leitura agregada para frontend? | GraphQL pode reduzir chamadas e adaptar consultas |
| É trabalho que pode ser processado depois? | fila ou evento |
| É fan-out para vários interessados? | Pub/Sub |
| É histórico reprocessável de eventos? | streaming |

## 6.4 Erros comuns

- escolher GraphQL para comando financeiro sem estratégia clara de idempotência e auditoria;
- usar fila para esconder latência sem tratar estado intermediário;
- chamar tudo de "assíncrono" sem definir confirmação, retry e reconciliação;
- fazer retry automático em operação externa sem idempotência;
- confundir resposta rápida com processamento concluído.

---

# 7. Mensageria, filas, PubSub e streaming

Mensageria reduz acoplamento, mas não elimina complexidade. Ela troca chamada direta por problemas de entrega, ordenação, duplicidade, observabilidade e reprocessamento.

## 7.1 Conceitos

**Messaging** é o conceito geral de comunicação por mensagens. Fila, Pub/Sub e streaming são modelos ou capacidades que podem existir em uma plataforma de messaging.

| Conceito | Objetivo | Distribuição | Retenção e replay |
|---|---|---|---|
| Messaging | comunicação desacoplada por mensagens | depende do modelo | depende da tecnologia |
| Fila | distribuir trabalho | uma mensagem para um consumer concorrente | normalmente até processamento/expiração |
| Pub/Sub | notificar interessados | uma publicação para vários subscribers | pode ou não existir |
| Streaming | manter e processar um log contínuo de eventos | vários consumer groups independentes | normalmente central ao modelo |

Antes de escolher a tecnologia, defina:

- se a mensagem representa comando ou evento;
- se precisa de ordenação por conta, pagamento ou cliente;
- quanto tempo precisa reter;
- se precisa de replay;
- como tratar duplicidade;
- como versionar o schema.

## 7.2 Fila

A fila segue o modelo **point-to-point por mensagem**.

```text
Producers
   ↓
Queue
   ├── Consumer 1
   ├── Consumer 2
   └── Consumer 3
```

Pode haver vários consumers, mas cada mensagem é entregue para um deles por tentativa de processamento.

Resumo preciso:

```text
N produtores → uma fila → N consumers concorrentes
Cada mensagem → um consumer por tentativa
```

Exemplo:

```text
PROCESSAR_PIX
```

A intenção é que um worker assuma o trabalho. Ainda assim, redelivery pode acontecer; o consumer precisa ser idempotente.

Use fila quando o objetivo principal for distribuir trabalho entre workers concorrentes, como envio de webhook, processamento de relatório ou tentativa controlada de integração.

## 7.3 Pub/Sub

No modelo publish/subscribe, uma publicação pode ser distribuída para vários assinantes:

```text
PIX_COMPLETED
   ├── notificações
   ├── conciliação
   ├── auditoria
   └── analytics
```

Resumo simplificado:

```text
1 ou N publishers → tópico → N subscribers
```

Pub/Sub define principalmente **como distribuir**. Ele não garante, por si só, retenção longa, ordenação ou replay.

Use Pub/Sub quando o mesmo fato precisa ser conhecido por vários destinos independentes. Exemplo: `PIX_COMPLETED` pode alimentar notificação, conciliação, antifraude pós-evento e analytics.

## 7.4 Streaming

Streaming mantém um fluxo persistente de eventos:

```text
PIX_CREATED → PIX_PROCESSING → PIX_COMPLETED
```

Características comuns:

- retenção;
- partições;
- offsets;
- replay;
- consumer groups;
- processamento contínuo;
- ordenação dentro de uma partição.

A simplificação `N para N` é aceitável, mas a principal diferença não é a quantidade de participantes; é o modelo de **log persistente e reprocessável**.

Streaming é especialmente útil quando consumidores diferentes precisam evoluir no próprio ritmo ou reprocessar histórico. O custo é maior disciplina de schema, particionamento, retenção e governança de eventos.

## 7.5 Tópico

“Tópico” é um canal lógico. O comportamento depende da tecnologia:

- em SNS, um tópico distribui mensagens para subscribers;
- em Kafka, um tópico é um log particionado e retido;
- em outros brokers, tópico pode representar roteamento Pub/Sub.

Portanto, tópico não é sinônimo universal de streaming.

Ao escrever documentação técnica, explique "tópico de quê": tópico SNS, tópico Kafka, exchange, routing key ou outro conceito da ferramenta adotada.

## 7.6 Kafka

Kafka é uma plataforma de event streaming.

- produtores escrevem em tópicos;
- tópicos são divididos em partições;
- registros ficam retidos conforme configuração;
- cada consumer group lê independentemente;
- consumers do mesmo grupo dividem as partições;
- replay é feito reposicionando offsets.

Para pagamentos, escolha a chave de partição com cuidado. Se eventos de um mesmo pagamento ou conta precisam ser processados em ordem, a chave deve preservar essa ordenação dentro de uma partição.

## 7.7 Amazon SQS

SQS é um serviço de filas.

Características:

- polling;
- visibility timeout;
- retry/redelivery;
- DLQ;
- filas Standard e FIFO;
- desacoplamento entre produtores e consumers.

Em filas Standard, duplicidades e variação de ordem devem ser consideradas. Mesmo em recursos com deduplicação, a regra de negócio deve permanecer idempotente.

Em filas FIFO, ordenação e deduplicação existem dentro das regras e limites do serviço. Ainda assim, a aplicação deve manter idempotência, porque falhas podem ocorrer antes ou depois do efeito de negócio.

## 7.8 Amazon SNS

SNS implementa publicação por tópicos e fan-out para destinos como:

- SQS;
- Lambda;
- HTTP/HTTPS;
- outros endpoints suportados.

Padrão comum:

```text
SNS topic
 ├── SQS notificações
 ├── SQS conciliação
 └── SQS webhooks
```

Isso combina distribuição Pub/Sub com durabilidade e consumo desacoplado nas filas.

## 7.9 Garantias de entrega

Termos úteis:

- **at most once**: pode perder mensagem, mas não entrega mais de uma vez;
- **at least once**: pode duplicar, mas tende a não perder;
- **effectively once**: efeito final único obtido por idempotência, transações e deduplicação;
- **exactly once**: depende de fronteira e tecnologia; raramente significa efeito financeiro exatamente uma vez em todos os sistemas envolvidos.

Em pagamentos, documente a garantia do efeito financeiro, não apenas a garantia prometida pelo broker.

## 7.10 Schema e compatibilidade

Eventos publicados viram contrato. Boas práticas:

- incluir `eventId`, `eventType`, `eventVersion`, `occurredAt` e identificador de negócio;
- preferir evolução compatível;
- não remover campos usados por consumidores sem plano de migração;
- evitar dados sensíveis desnecessários;
- documentar semântica de status e transições.

---

# 8. Outbox, CDC, retry e DLQ

Este capítulo trata de falhas entre banco, broker e dependências externas. O objetivo é não perder fatos confirmados e não transformar retry em duplicidade financeira.

## 8.1 Problema de dual write

Sem Outbox:

```text
1. atualizar o banco
2. publicar no broker
```

Falhas possíveis:

- banco confirma, broker falha: dado existe, evento não foi publicado;
- broker recebe, banco faz rollback: evento representa algo que não foi confirmado.

Esse problema é perigoso porque normalmente aparece só em falha parcial: queda de rede, timeout, deploy, indisponibilidade do broker ou erro depois do commit.

## 8.2 Transactional Outbox

A alteração de negócio e a intenção de publicação são persistidas na mesma transação local:

```text
BEGIN
  atualizar pagamento
  inserir evento na outbox
COMMIT
```

Depois, um relay publica o evento:

```text
Outbox
  ↓
CDC ou Polling
  ↓
Broker
```

Garantia precisa:

> O Outbox garante atomicidade entre o estado local e o registro da intenção de publicar. Ele não garante entrega exatamente uma vez até todos os consumidores.

Duplicidades ainda podem ocorrer se o publisher publicar e falhar antes de marcar o registro como concluído.

Exemplo de campos para uma tabela de outbox:

```text
outbox_id
aggregate_type
aggregate_id
event_type
event_version
payload
status
attempt_count
next_attempt_at
created_at
published_at
```

O consumidor continua precisando ser idempotente, porque Outbox reduz perda de evento, mas não elimina duplicidade ponta a ponta.

## 8.3 CDC

**Change Data Capture** captura mudanças no banco, normalmente a partir do log transacional.

Exemplo:

```text
PostgreSQL WAL
      ↓
Debezium
      ↓
Kafka
```

Com Outbox, o CDC observa inserções na tabela de outbox e as transforma em mensagens.

```text
Outbox Pattern
→ estratégia de consistência local.

CDC
→ mecanismo de captura e propagação da mudança.
```

CDC costuma reduzir polling e preservar ordenação do log do banco, mas exige operação cuidadosa: conectores, offsets, slots de replicação, retenção de log, monitoramento de lag e plano de recuperação.

## 8.4 Polling publisher

Alternativa ao CDC:

```text
buscar eventos PENDING
        ↓
publicar
        ↓
marcar como publicado
```

Deve tratar:

- concorrência entre publishers;
- lock ou claim de registros;
- ordenação por agregado quando necessária;
- retry;
- duplicidade;
- retenção e limpeza da tabela.

Um polling publisher robusto normalmente usa claim atômico:

```text
buscar PENDING vencidos
marcar como PROCESSING com owner e lease
publicar
marcar como PUBLISHED
```

Se o processo cair no meio, o lease expira e outro publisher pode retomar.

## 8.5 Retry

Retry é adequado para falhas transitórias:

- timeout de rede;
- indisponibilidade momentânea;
- throttling;
- falha temporária de infraestrutura.

Use:

- limite de tentativas;
- exponential backoff;
- jitter;
- timeout/deadline;
- circuit breaker;
- idempotência.

> Nunca repita cegamente uma operação financeira cujo resultado ficou desconhecido. Primeiro consulte o estado autoritativo ou execute reconciliação.

Classifique falhas antes de repetir:

| Falha | Ação comum |
|---|---|
| timeout de rede antes de resposta | consultar status ou reconciliar |
| `429` ou throttling | retry com backoff e respeito a `Retry-After` |
| `5xx` transitório | retry limitado |
| erro de validação | não repetir sem corrigir payload |
| estado desconhecido | não executar novo efeito financeiro |

## 8.6 DLQ

A Dead-Letter Queue isola mensagens que falharam após as tentativas configuradas.

```text
Fila principal
    ↓
Consumer
    ↓ falha
Retry/backoff
    ↓ excedeu limite
DLQ
```

A DLQ não resolve a falha automaticamente. É necessário:

- alerta;
- diagnóstico;
- correção da causa;
- política de retenção;
- redrive controlado;
- processamento idempotente.

Fluxo recomendado de DLQ:

```text
alertar
classificar causa
corrigir dado, contrato ou código
testar reprocessamento em pequena amostra
redrive controlado
validar métricas e conciliação
registrar aprendizado
```

Nunca trate DLQ como lixeira silenciosa. Mensagem financeira parada é trabalho operacional pendente.

---

# 9. CQRS, Saga e Event Sourcing

Estes padrões são úteis, mas frequentemente usados cedo demais. Em pagamentos, adote quando o problema justificar a complexidade.

## 9.1 CQRS

CQRS separa o modelo de comandos do modelo de consultas.

```text
Commands → Write Model
Queries  → Read Model
```

Não exige bancos separados e não exige Event Sourcing.

### Strong consistency com CQRS

Para uma projeção ser atualizada com consistência forte, ela precisa participar da mesma fronteira transacional, normalmente no mesmo banco:

```text
Transação única
├── atualizar Write Model
└── atualizar projeção crítica
```

Se Write Model e Read Model estiverem em bancos diferentes e forem sincronizados por eventos, a consistência será normalmente eventual.

Estratégia recomendada:

- autorização, saldo e limites: consultar o modelo transacional;
- dashboards e históricos derivados: usar projeções assíncronas;
- read-your-writes: retornar o estado no comando, consultar o write model ou aguardar uma versão mínima da projeção.

Transações distribuídas, como 2PC/XA, podem fornecer coordenação mais forte entre recursos, mas aumentam acoplamento, latência e riscos operacionais. Em microsserviços, são usadas com cautela.

Use CQRS quando:

- o modelo de escrita é diferente do modelo de leitura;
- consultas precisam de projeções otimizadas;
- dashboards e extratos derivados têm volume alto;
- você aceita ou controla atraso de projeção.

Evite CQRS quando:

- o domínio ainda é simples;
- a equipe não consegue operar projeções atrasadas;
- a leitura derivada seria usada incorretamente para autorizar dinheiro.

## 9.2 Saga

Saga coordena uma operação distribuída como uma sequência de transações locais.

Pode ser:

- **orquestrada**: um coordenador decide os próximos passos;
- **coreografada**: participantes reagem a eventos.

Exemplo:

```text
PIX_CREATED
   ↓
FRAUD_APPROVED
   ↓
BALANCE_RESERVED
   ↓
PAYMENT_SENT
   ├──→ SETTLED
   └──→ REJECTED → RELEASE_RESERVATION
```

Após o ponto irreversível, como a liquidação, não existe rollback técnico da mesma transferência. Uma devolução é uma nova operação financeira, com identidade, autorização e lançamentos próprios.

Saga oferece consistência eventual por compensação; não é strong consistency global.

Ao desenhar uma Saga, documente:

- ponto de início;
- passos locais;
- eventos esperados;
- timeouts;
- compensações possíveis;
- ponto de irreversibilidade;
- como lidar com estados pendentes;
- quem opera casos que exigem intervenção.

## 9.3 Event Sourcing

Event Sourcing usa a sequência de eventos como fonte de verdade do estado do agregado:

```text
PIX_CREATED
→ PIX_VALIDATED
→ PIX_SENT
→ PIX_SETTLED
```

O estado atual é reconstruído aplicando os eventos em ordem.

Benefícios:

- histórico completo;
- auditoria;
- reconstrução de projeções;
- replay;
- análise temporal.

Cuidados:

- versionamento de eventos;
- evolução de schema;
- snapshots;
- custo de replay;
- consistência eventual das projeções;
- proteção de dados sensíveis;
- complexidade de operação.

Durante o replay, não execute novamente efeitos externos:

```text
apply(event)
→ reconstrói estado interno.

handler de integração
→ envia Pix, webhook ou notificação.
```

Event Sourcing não é apenas uma tabela de auditoria. Nele, o stream de eventos é a fonte autoritativa do agregado.

Use Event Sourcing quando o histórico de mudanças é parte central do domínio e a equipe está preparada para versionamento, replay e operação das projeções.

Evite quando a motivação é apenas "ter auditoria". Auditoria pode ser atendida com ledger, trilhas de evento e logs imutáveis sem transformar todo o modelo de domínio em Event Sourcing.

## 9.4 Relação entre os padrões

```text
CQRS
→ separa comandos e consultas.

Saga
→ coordena transações locais em fluxo distribuído.

Event Sourcing
→ usa eventos como fonte de verdade do agregado.
```

Eles podem coexistir, mas nenhum exige automaticamente o outro.

---

# 10. Cache, rate limit e backpressure

Este capítulo mostra como proteger latência e capacidade sem comprometer a fonte autoritativa do dinheiro.

## 10.1 TTL

TTL (**Time To Live**) é o período durante o qual um item permanece válido no cache.

```text
pix:123 = PROCESSING
TTL = 15 segundos
```

Após a expiração, a próxima leitura recarrega o valor da fonte.

TTL não produz invalidação imediata. O dado pode ficar desatualizado até expirar.

Estratégia comum:

```text
invalidação explícita ou por evento
+
TTL como rede de segurança
```

Dados críticos, como saldo usado para autorizar um Pix, não devem depender exclusivamente de cache eventualmente desatualizado.

Para evitar cache stampede:

- TTL com jitter;
- request coalescing;
- lock de recomputação;
- stale-while-revalidate;
- atualização antecipada.

Dados que costumam caber em cache:

- configuração não crítica;
- metadados de baixa volatilidade;
- resultado de consulta pública com TTL curto;
- projeções de leitura não autoritativas;
- tokens e chaves públicas com política de rotação.

Dados que exigem cuidado extremo:

- saldo disponível para autorização;
- limites transacionais ativos;
- estado final de pagamento;
- decisões de antifraude em tempo real;
- dados pessoais sensíveis.

## 10.2 Rate limiting

Rate limiting controla a quantidade de requisições por cliente, token, conta, IP ou operação.

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 2
```

Protege contra:

- abuso;
- tráfego acidental;
- consumo injusto de capacidade;
- saturação de dependências.

Dimensões comuns:

- por credencial;
- por cliente;
- por conta;
- por chave Pix;
- por IP;
- por endpoint;
- por dependência externa.

Algoritmos comuns incluem fixed window, sliding window, token bucket e leaky bucket. A escolha depende de previsibilidade, rajadas permitidas e custo de implementação.

## 10.3 Backpressure

Backpressure acontece quando o consumidor ou sistema downstream não acompanha o ritmo do produtor.

```text
Producer rápido
      ↓
Fila crescendo
      ↓
Consumer lento
```

Mecanismos possíveis:

- filas limitadas;
- controle de concorrência;
- pause/resume;
- redução de consumo;
- load shedding;
- `429` ou `503`;
- autoscaling;
- limites por dependência.

> Rate limiting pode fazer parte de uma estratégia de backpressure, mas os dois conceitos não são sinônimos.

Backpressure precisa ser visível para produto e operação. Quando o sistema começa a degradar, ele deve preferir respostas controladas, filas limitadas e estados claros a aceitar trabalho infinito que nunca será processado.

## 10.4 Degradação segura

Em pagamentos, degradação segura significa preservar integridade antes de preservar conveniência.

Exemplos:

- bloquear temporariamente uma funcionalidade não essencial;
- responder `202 Accepted` e acompanhar estado em vez de manter conexão aberta indefinidamente;
- rejeitar novas solicitações quando a dependência crítica está saturada;
- manter consulta e conciliação operando mesmo que notificações estejam atrasadas;
- pausar redrive de DLQ durante incidente.

---

# 11. Escalabilidade e alta disponibilidade

Escalar é aumentar capacidade. Ter alta disponibilidade é continuar operando apesar de falhas. Um sistema pode escalar bastante e ainda cair mal se não tiver isolamento, failover e operação testada.

## 11.1 Escalabilidade horizontal

Consiste em adicionar instâncias:

```text
Cliente
   ↓
Load Balancer
   ├── App 1
   ├── App 2
   └── App 3
```

Funciona melhor quando as aplicações são stateless.

Escalabilidade horizontal exige que estado crítico esteja fora da instância ou seja reconstruível. Caso contrário, adicionar instâncias aumenta capacidade, mas também aumenta inconsistência e dificuldade operacional.

## 11.2 Stateless e stateful

### Stateless

A instância não mantém estado crítico da sessão ou transação apenas na memória local.

O estado fica em:

- banco de dados;
- Redis quando apropriado;
- broker;
- object storage;
- serviços especializados.

Qualquer instância pode atender a próxima requisição.

### Stateful

A aplicação mantém estado necessário na própria instância. Isso cria afinidade e dificulta:

- autoscaling;
- deploy;
- recuperação após falha;
- redistribuição de tráfego.

## 11.3 Sticky session

Sticky session no ALB tenta enviar o mesmo cliente à mesma instância.

Pode ser útil para sistemas legados, mas:

- não protege o estado se a instância cair;
- pode desequilibrar carga;
- dificulta autoscaling;
- aumenta acoplamento.

Para pagamentos, o estado crítico deve ser persistido fora da instância.

## 11.4 Load balancing e failover

```text
Load balancing
→ distribui tráfego entre instâncias saudáveis.

Failover
→ substitui um componente principal indisponível por outro saudável.
```

No banco, failover normalmente promove uma **réplica/standby**, não um arquivo de backup.

```text
Réplica
→ banco ativo que recebe replicação e pode ser promovido.

Backup
→ cópia usada para restauração, recuperação histórica ou desastre.
```

Defina também:

- **RTO**: tempo máximo aceitável para recuperar o serviço;
- **RPO**: perda máxima aceitável de dados;
- plano de failover por dependência crítica;
- teste periódico de restauração de backup;
- procedimento de retorno ao estado normal.

## 11.5 Split-brain

Split-brain ocorre quando duas instâncias acreditam ser líderes e aceitam escritas.

Mitigações:

- quórum;
- eleição de líder;
- fencing;
- leases;
- epochs/terms;
- proteção contra escrita no antigo primário.

Em sistemas financeiros, split-brain é especialmente grave porque pode permitir escritas conflitantes. A arquitetura deve preferir indisponibilidade controlada a aceitar duas fontes primárias independentes para o mesmo saldo.

## 11.6 Probes no Kubernetes

### Startup probe

Verifica se a aplicação terminou de iniciar. Enquanto não passa, evita que liveness e readiness prejudiquem aplicações com inicialização lenta.

### Liveness probe

Responde:

> O processo está vivo e consegue continuar operando?

Falhas repetidas podem causar reinício do container.

Não coloque indisponibilidades transitórias de banco ou API externa na liveness, pois isso pode criar loops de reinício e piorar o incidente.

### Readiness probe

Responde:

> Esta instância está pronta para receber tráfego agora?

Quando falha, o pod é removido dos endpoints prontos, sem necessariamente ser reiniciado.

Dependências críticas podem influenciar readiness, mas o desenho deve evitar retirar todas as réplicas simultaneamente por causa da mesma dependência externa.

Uma boa prática é separar endpoints:

```text
/live
→ processo vivo, sem testar dependências externas pesadas.

/ready
→ instância pronta para receber tráfego.

/health/deep
→ diagnóstico operacional mais completo, fora do caminho de balanceamento.
```

## 11.7 Circuit breaker, timeout e retry

- **timeout/deadline** limita quanto tempo a chamada pode ocupar recursos;
- **retry** tenta novamente falhas transitórias e seguras;
- **circuit breaker** reduz chamadas para uma dependência degradada;
- **bulkhead** isola pools e recursos;
- **fallback** fornece uma resposta alternativa quando possível.

Retry sem timeout, sem backoff ou sem idempotência pode amplificar uma falha.

## 11.8 Alta disponibilidade por camada

| Camada | Cuidados |
|---|---|
| Aplicação | múltiplas instâncias, deploy gradual, readiness correta |
| Banco | réplica, backup, failover testado, migração segura |
| Broker | partições/filas monitoradas, DLQ, retenção e redrive |
| Cache | tolerância à perda, proteção contra stampede |
| Integração externa | timeout, circuit breaker, contingência e reconciliação |
| Operação | alertas, runbooks, plantão e exercícios de falha |

---

# 12. Sidecar, service mesh, Ambassador e Adapter

Este capítulo diferencia padrões que costumam ser confundidos. Sidecar e service mesh são mais ligados à implantação e comunicação. Ambassador é um proxy auxiliar. Adapter é um padrão de integração entre contratos.

## 12.1 Sidecar

Sidecar é um componente executado ao lado da aplicação e com ciclo de vida relacionado.

```text
Pod
├── aplicação
└── sidecar
```

Usos:

- proxy;
- coleta de logs;
- agente de segurança;
- configuração;
- adaptação de protocolo;
- telemetria.

É um padrão de implantação, não uma solução específica.

Use com parcimônia. Cada sidecar adiciona consumo de CPU, memória, configuração, logs, métricas e superfície de falha.

## 12.2 Service mesh

Service mesh é a infraestrutura lógica que gerencia comunicação entre workloads.

No modo tradicional com sidecars:

```text
App A → Proxy A → Proxy B → App B
```

Componentes:

```text
Data plane
→ proxies que processam o tráfego.

Control plane
→ configura proxies e políticas.
```

Pode oferecer:

- mTLS;
- load balancing;
- roteamento;
- retries e timeouts;
- circuit breaker;
- métricas e tracing;
- políticas de segurança.

Formulação correta:

> Cada aplicação pode ter um proxy sidecar; os proxies e o control plane formam um único service mesh lógico.

Nem todo service mesh atual exige sidecar por pod. Modelos ambient podem usar proxies por nó e waypoints.

Service mesh faz sentido quando a organização precisa de políticas consistentes de tráfego, mTLS, observabilidade e segurança entre muitos serviços. Pode ser excesso para poucos serviços ou para um time que ainda não consegue operar bem a complexidade básica.

## 12.3 Ambassador Pattern

Ambassador é um proxy auxiliar colocado próximo ao cliente para realizar chamadas de rede em nome da aplicação.

```text
Aplicação
   ↓
Ambassador
   ↓
Serviço externo
```

Pode encapsular:

- TLS/mTLS;
- autenticação;
- roteamento;
- retries;
- circuit breaker;
- logging e métricas.

É comum em comunicação **de saída**, mas não é obrigatório para todo egress. Um egress gateway do service mesh também pode centralizar políticas de saída.

Exemplo em uma plataforma Pix: encapsular conexão mTLS, retry seguro, deadline e métricas para um provedor externo, sem espalhar detalhes de rede pelo domínio financeiro.

## 12.4 Adapter Pattern

Adapter é um padrão de código ou componente que converte uma interface para o contrato esperado pelo domínio.

```text
Domínio Pix
   ↓ interface interna
ProviderAdapter
   ↓
REST, SOAP, SDK ou protocolo do parceiro
```

Pode converter:

- modelos de dados;
- nomes e semânticas de status;
- REST para SOAP;
- SDK proprietário para interface interna;
- formatos de erros.

Webhook e WebSocket não são, por si só, Adapter Pattern. Um adapter pode ser usado para receber ou enviar esses protocolos e convertê-los para eventos/comandos internos.

O adapter deve proteger o domínio contra mudanças do provedor:

- status externo vira enum interno controlado;
- erro externo vira erro de domínio;
- payload externo vira comando ou evento interno;
- diferenças de contrato ficam isoladas em uma fronteira testável.

## 12.5 Entrada e saída da aplicação

```text
Entrada externa:
PSP → webhook → WAF/Gateway/Ingress → Webhook Adapter → domínio

Saída externa:
domínio → Provider Adapter → Ambassador/Egress opcional → PSP
```

## 12.6 Pergunta prática

Se a mudança é de contrato de negócio, provavelmente é Adapter. Se a mudança é de transporte, segurança de conexão, telemetria ou política de tráfego, provavelmente é Ambassador, gateway, ingress, egress ou service mesh.

---

# 13. Webhooks e WebSockets

Webhooks e WebSockets ajudam a comunicar mudanças de estado, mas não devem substituir a fonte autoritativa nem a conciliação.

## 13.1 Webhook

Webhook é uma chamada HTTP disparada quando um evento ocorre.

Ele pode ser:

- **de entrada**: um PSP chama sua aplicação;
- **de saída**: sua plataforma notifica um cliente.

```text
Sistema emissor
     ↓ HTTP POST
Endpoint do receptor
```

Webhooks podem:

- chegar duplicados;
- atrasar;
- chegar fora de ordem;
- sofrer retry;
- falhar após o receptor processar, mas antes de responder.

O receptor deve:

- validar assinatura, certificado ou credencial;
- validar timestamp e proteção contra replay;
- deduplicar por `eventId`;
- processar idempotentemente;
- responder rapidamente com `2xx` após aceitar de forma segura;
- mover processamento pesado para fila quando adequado.

Contrato mínimo recomendado:

- `eventId` único;
- `eventType`;
- versão do evento;
- timestamp;
- identificador de negócio;
- assinatura ou mecanismo equivalente;
- política de retry;
- documentação de ordenação, duplicidade e retenção.

Ao receber webhook, salve o evento bruto ou um registro auditável antes de executar processamento pesado. Isso ajuda em replay, suporte e investigação.

## 13.2 WebSocket

WebSocket mantém uma conexão persistente e bidirecional:

```text
Cliente ⇄ Servidor
```

É útil para:

- atualização de status em tempo real;
- notificações na interface;
- dashboards;
- chats e colaboração.

Exemplo combinado:

```text
PSP confirma pagamento
      ↓ webhook
Backend atualiza estado
      ↓ evento
Gateway WebSocket
      ↓
Tela recebe atualização
```

WebSocket é ótimo para experiência em tempo real, mas a tela deve conseguir consultar o estado atual por API. A conexão pode cair, reconectar, perder mensagens ou receber eventos fora de ordem.

## 13.3 Webhook não é conciliação

Webhook informa mudança, mas pode falhar. Conciliação compara fontes independentes e fecha lacunas.

Arquitetura robusta usa ambos:

```text
webhook
→ atualiza rápido.

consulta de status e conciliação
→ confirmam e corrigem divergências.
```

---

# 14. Segurança

Segurança em pagamentos precisa cobrir identidade, autorização, criptografia, fraude, privacidade, trilha de auditoria e resposta a incidente.

## 14.1 Controles principais

- autenticação forte;
- autorização por escopo e função;
- mTLS em integrações sensíveis;
- criptografia em trânsito e em repouso;
- gestão e rotação de segredos;
- proteção contra replay;
- assinatura de mensagens/webhooks;
- rate limiting e WAF;
- trilha de auditoria;
- segregação de funções;
- princípio do menor privilégio.

Também considere:

- validação de dispositivo e sessão quando aplicável;
- análise de risco transacional;
- detecção de comportamento anômalo;
- proteção de dados pessoais em logs, eventos e analytics;
- revisão periódica de permissões;
- segregação entre ambientes;
- resposta a incidentes de segurança.

## 14.2 Certificados, TLS e mTLS

No TLS tradicional, o cliente normalmente valida o certificado do servidor.

No mTLS, os dois lados apresentam certificados:

```text
Cliente valida servidor
Servidor valida cliente
```

A operação deve prever:

- inventário;
- renovação;
- alertas de expiração;
- revogação;
- certificados de contingência;
- testes de rotação.

Certificado expirado é incidente previsível. A documentação operacional deve incluir dono, data de expiração, janela de renovação, processo de troca, rollback e alerta antecipado.

## 14.3 HSM

HSM (**Hardware Security Module**) protege material criptográfico e executa operações sem expor a chave privada.

```text
Aplicação solicita assinatura
           ↓
          HSM
           ↓
HSM usa a chave internamente
           ↓
Retorna a assinatura
```

Usos comuns:

- geração e proteção de chaves;
- assinatura digital;
- criptografia;
- atendimento a requisitos de segurança e auditoria.

## 14.4 Proteção contra replay

Um atacante não deve conseguir capturar uma requisição válida e reenviá-la depois.

Controles comuns:

- nonce;
- timestamp com janela curta;
- assinatura incluindo corpo, caminho e timestamp;
- chave de idempotência;
- validação de certificado ou credencial;
- armazenamento temporário de identificadores já vistos.

## 14.5 Dados sensíveis

Não registre em log mais dados do que o necessário. Para observabilidade, prefira identificadores técnicos e mascaramento.

Exemplos de cuidado:

- não vazar documento, chave Pix, token, segredo ou payload completo sem necessidade;
- controlar acesso a trilhas de auditoria;
- definir retenção;
- proteger dados em ambientes de teste;
- evitar dados pessoais em labels de métricas de alta cardinalidade.

---

# 15. Observabilidade, operação e documentação

Um sistema financeiro não termina quando o endpoint responde. Ele precisa ser observado, operado, auditado e melhorado continuamente.

## 15.1 Logs, métricas e traces

### Logs

Devem ser estruturados e correlacionáveis:

```text
paymentId=pix-123
endToEndId=E...
status=UNKNOWN
error=UPSTREAM_TIMEOUT
correlationId=abc-789
traceId=def-456
```

Não registre dados pessoais ou segredos sem necessidade e proteção adequada.

### Métricas

Exemplos:

- taxa de sucesso e rejeição;
- latência p50, p95 e p99;
- pagamentos em estado `UNKNOWN`;
- timeouts por dependência;
- taxa de duplicidade bloqueada;
- consumer lag;
- tamanho e idade da DLQ;
- tempo de processamento de webhook;
- divergências de conciliação;
- saturação de pools e conexões.

### Traces

```text
Gateway
  ↓
Orquestrador
  ↓
Antifraude
  ↓
Ledger
  ↓
Adapter externo
```

Propague `traceId`, `correlationId`, `paymentId` e identificadores de negócio sem transformar dados sensíveis em tags de alta cardinalidade indiscriminadamente.

Uma boa investigação deve conseguir responder:

- qual requisição iniciou a operação;
- qual usuário, cliente ou credencial estava envolvido;
- qual estado foi persistido;
- qual chamada externa foi feita;
- se houve retry;
- se evento foi publicado;
- se a conciliação confirmou ou apontou divergência.

## 15.2 SLI, SLO e SLA

- **SLI**: indicador medido, como taxa de sucesso ou latência p99;
- **SLO**: objetivo interno para o indicador;
- **SLA**: compromisso contratual com consequências definidas.

Exemplo:

```text
SLI: percentual de solicitações válidas processadas com sucesso
SLO: 99,95% por mês
SLA: compromisso contratual de 99,9%
```

SLIs úteis para Pix:

- taxa de sucesso por tipo de operação;
- latência p95/p99 por endpoint;
- tempo em `PROCESSING`;
- quantidade e idade de pagamentos `UNKNOWN`;
- atraso de publicação da Outbox;
- consumer lag;
- idade máxima de mensagem em DLQ;
- divergências abertas na conciliação.

Alertas devem ter ação clara. Alerta sem runbook vira ruído.

## 15.3 Runbook

Documento operacional usado durante incidentes ou tarefas recorrentes:

- sintomas;
- diagnóstico;
- mitigação;
- comandos seguros;
- critérios de escalonamento;
- validação da recuperação;
- rollback ou contingência.

Modelo simples:

```text
Sintoma:
Impacto:
Métricas e dashboards:
Hipóteses prováveis:
Passos de diagnóstico:
Mitigação segura:
Critério de escalonamento:
Critério de recuperação:
Comunicação:
```

## 15.4 Postmortem

Documento produzido após um incidente:

- resumo e impacto;
- linha do tempo;
- causa técnica;
- fatores contribuintes;
- detecção e resposta;
- o que funcionou e o que falhou;
- ações corretivas com responsáveis e prazos.

Deve ser **blameless**: analisa condições do sistema e do processo, não procura um culpado individual.

Boas ações corretivas são específicas:

- o que será feito;
- por quem;
- até quando;
- como será validado;
- qual risco será reduzido.

## 15.5 ADR

**Architecture Decision Record** registra:

- contexto;
- decisão;
- alternativas;
- consequências;
- status;
- data e responsáveis.

Exemplo:

```text
ADR-007 — Adotar Transactional Outbox para eventos de pagamento
```

Modelo enxuto:

```text
Título:
Status:
Contexto:
Decisão:
Alternativas consideradas:
Consequências:
Data:
```

## 15.6 C4 Model

Níveis:

1. **Context**: sistema, usuários e sistemas externos;
2. **Container**: aplicações, bancos, brokers e frontends;
3. **Component**: componentes internos de uma aplicação;
4. **Code**: classes ou detalhes de implementação, quando necessário.

No C4, “container” não significa apenas Docker; é uma unidade executável ou armazenadora relevante para a arquitetura.

Para esta documentação, os níveis mais úteis são Context e Container. Component pode ser usado para detalhar o orquestrador de pagamentos, ledger, adapter, outbox relay e conciliação.

## 15.7 OKR

- **Objective**: resultado qualitativo desejado;
- **Key Results**: resultados mensuráveis que comprovam o avanço.

Exemplo:

```text
Objective:
Aumentar a confiabilidade da plataforma de pagamentos.

Key Results:
- reduzir pagamentos em estado UNKNOWN em 50%;
- reduzir MTTR de 40 para 20 minutos;
- manter sucesso acima de 99,95%;
- reduzir p99 do endpoint de consulta em 30%.
```

Implementar uma tecnologia é uma iniciativa, não necessariamente um Key Result.

## 15.8 Relação entre os artefatos

```text
OKR
→ define os resultados desejados.

ADR
→ registra decisões técnicas.

C4 Model
→ representa a arquitetura.

Runbook
→ orienta a operação e resposta.

Postmortem
→ gera aprendizado após incidentes.
```

## 15.9 Documentação mínima de produção

- visão C4 de contexto e containers;
- contratos de API e webhooks;
- estados e transições;
- políticas de idempotência;
- decisão de consistência por dado;
- runbooks dos fluxos críticos;
- ADRs das decisões relevantes;
- checklist de segurança e operação;
- procedimento de conciliação;
- plano de testes de recuperação.

---

# 16. Conciliação

Conciliação é o mecanismo que procura divergências entre fontes independentes e transforma incerteza em investigação, correção ou confirmação.

Conciliação compara fontes independentes para detectar divergências.

```text
Ledger interno
Banco operacional
Relatório ou fonte externa
          ↓
Motor de conciliação
```

Chaves e atributos possíveis:

- `endToEndId`;
- identificador interno;
- valor;
- status;
- data e hora;
- participante pagador;
- participante recebedor.

Divergências:

- transação ausente;
- duplicidade;
- valor divergente;
- status divergente;
- registro apenas no interno;
- registro apenas no externo.

Correções devem ser:

- idempotentes;
- auditáveis;
- aprovadas conforme governança;
- feitas por contralançamentos ou novas operações quando houver impacto financeiro;
- monitoradas até o fechamento.

Conciliação é uma defesa adicional e não substitui integridade transacional.

## 16.1 Tipos de conciliação

- **online ou quase em tempo real**: acompanha estados recentes e reduz tempo em `UNKNOWN`;
- **batch diária**: compara movimentos do dia contra fontes internas e externas;
- **contábil**: valida se lançamentos financeiros fecham;
- **operacional**: procura filas paradas, webhooks não enviados, eventos não publicados e estados pendentes.

## 16.2 Fluxo recomendado

```text
coletar fontes
      ↓
normalizar campos
      ↓
comparar por chaves
      ↓
classificar divergência
      ↓
abrir pendência
      ↓
corrigir com trilha auditável
      ↓
validar fechamento
```

## 16.3 Critérios de qualidade

- toda divergência tem dono e prazo;
- correção financeira usa contralançamento ou nova operação formal;
- correção técnica deixa trilha;
- divergências recorrentes geram ação preventiva;
- reconciliação possui métricas e alertas;
- reprocessamento é idempotente.

---

# 17. Resumo para entrevista

Use este capítulo como resposta compacta. Ele não substitui o restante do material; serve para organizar uma explicação oral.

## 17.1 Resposta principal

> Eu desenharia uma plataforma Pix com API Gateway, autenticação, rate limiting, idempotência e um serviço de pagamentos responsável pela máquina de estados. No caminho financeiro, usaria transações locais, controle de concorrência, reserva de saldo e ledger de dupla entrada. A integração com o participante Pix ficaria isolada por um Adapter.
>
> Para publicar eventos sem dual write, usaria Transactional Outbox, com CDC ou polling. Como a entrega pode ser at least once, producers, consumers e webhooks seriam idempotentes, usando identificadores únicos e constraints no banco. Retries teriam timeout, exponential backoff e jitter, sem repetir cegamente uma operação em estado desconhecido.
>
> REST seria apropriado para APIs externas e webhooks; gRPC poderia ser usado em chamadas internas de baixa latência; GraphQL ficaria principalmente em BFFs e consultas agregadas. Filas seriam usadas para distribuir trabalho, Pub/Sub para fan-out e streaming para manter um log persistente com retenção, offsets e replay.
>
> Saldo, limites e ledger usariam o modelo transacional autoritativo. Projeções, notificações e analytics poderiam ser eventualmente consistentes. A aplicação seria stateless para escalar horizontalmente, com failover seguro, probes, observabilidade, reconciliação, runbooks e postmortems. Segurança incluiria mTLS, gestão de certificados, HSM quando aplicável e proteção contra replay.

## 17.2 Versão curta

> Eu priorizaria consistência financeira, idempotência e rastreabilidade. A API validaria a intenção e usaria chave idempotente; o domínio protegeria saldo, limites e ledger em transações locais; eventos seriam publicados por Outbox; integrações externas teriam adapter, timeout, retry seguro e tratamento de estado desconhecido. O sistema seria observável, reconciliável, seguro e operável em falha parcial.

## 17.3 Pontos que impressionam em design review

- explicar por que timeout não é falha definitiva;
- separar fonte autoritativa de projeções;
- mostrar como evita dual write;
- falar de idempotência em API, consumer e integração externa;
- definir estados e transições;
- prever conciliação e operação;
- saber dizer quando uma tecnologia não é necessária.

## 17.4 Perguntas prováveis

| Pergunta | Resposta esperada |
|---|---|
| Por que não usar cache para saldo? | Porque saldo usado para autorizar dinheiro precisa de fonte autoritativa consistente. |
| Como evitar pagamento duplicado? | Idempotency key, constraint atômica, estado persistido, consumers idempotentes e identificadores externos estáveis. |
| O que fazer em timeout? | Marcar como `UNKNOWN`, consultar fonte autoritativa ou reconciliar. |
| Por que Outbox? | Para confirmar estado local e intenção de publicar evento na mesma transação. |
| Saga resolve rollback financeiro? | Não depois de ponto irreversível; compensação vira nova operação financeira. |

---

# 18. Checklist técnico

Use este checklist antes de uma implementação, design review, entrevista ou revisão de produção. Ele não substitui testes, auditoria ou validação regulatória.

## Escopo e responsabilidades

- [ ] papel da instituição definido: participante direto, indireto, provedor, parceiro ou integrador;
- [ ] fronteiras entre Pix, SPI, DICT, MED, API da plataforma e arquitetura interna documentadas;
- [ ] fonte autoritativa de cada estado definida;
- [ ] estados e transições do pagamento documentados;
- [ ] pontos de irreversibilidade identificados.

## Consistência e dinheiro

- [ ] operação financeira atômica;
- [ ] invariantes de saldo protegidas contra concorrência;
- [ ] ledger balanceado e auditável;
- [ ] lançamentos corrigidos por contralançamento ou nova operação formal;
- [ ] reserva de saldo com ciclo de vida definido;
- [ ] estado `UNKNOWN` tratado sem retry cego;
- [ ] reconciliação implementada.

## Idempotência

- [ ] `Idempotency-Key` com escopo definido;
- [ ] hash canônico do payload;
- [ ] `UNIQUE CONSTRAINT` no banco;
- [ ] resposta ou operação original recuperável;
- [ ] consumer idempotente por `messageId/eventId`;
- [ ] processamento e deduplicação na mesma transação local.
- [ ] integrações externas usam identificador idempotente ou consulta de status.

## Mensageria

- [ ] Outbox para evitar dual write;
- [ ] CDC ou polling com retry;
- [ ] DLQ com alerta e redrive controlado;
- [ ] ordenação definida por agregado quando necessária;
- [ ] schema e compatibilidade de eventos;
- [ ] backpressure e limites de concorrência.
- [ ] eventos não carregam dados sensíveis desnecessários.
- [ ] consumer lag e idade de mensagens monitorados.

## Resiliência

- [ ] timeouts/deadlines;
- [ ] retry com backoff e jitter;
- [ ] circuit breaker;
- [ ] bulkheads;
- [ ] health checks corretos;
- [ ] failover sem split-brain;
- [ ] backups testados por restauração.
- [ ] RTO e RPO definidos.
- [ ] degradação segura documentada.
- [ ] dependências externas possuem contingência ou conciliação.

## Segurança

- [ ] autenticação e autorização;
- [ ] TLS/mTLS;
- [ ] rotação de certificados e segredos;
- [ ] assinatura de webhooks;
- [ ] proteção contra replay;
- [ ] dados sensíveis protegidos em logs e eventos;
- [ ] trilha de auditoria.
- [ ] certificados com inventário, dono e alerta de expiração.
- [ ] segredos com rotação e acesso mínimo.
- [ ] permissões revisadas periodicamente.

## Operação

- [ ] logs estruturados;
- [ ] métricas e traces;
- [ ] SLOs e alertas acionáveis;
- [ ] runbooks;
- [ ] postmortems blameless;
- [ ] ADRs e diagramas C4 atualizados.
- [ ] dashboards cobrem estados críticos, Outbox, DLQ, webhooks e conciliação.
- [ ] alertas têm ação e dono.
- [ ] procedimentos de redrive e correção são idempotentes.

## Testes essenciais

- [ ] repetição da mesma requisição com mesma chave;
- [ ] mesma chave com payload diferente;
- [ ] concorrência em saldo;
- [ ] falha após commit local e antes da publicação;
- [ ] duplicidade de mensagem;
- [ ] webhook duplicado, atrasado e fora de ordem;
- [ ] timeout de integração externa;
- [ ] queda de instância durante processamento;
- [ ] failover de banco ou dependência crítica;
- [ ] reconciliação fechando divergência simulada.

---

# 19. Referências oficiais e técnicas

Use fontes oficiais para validar regras do Pix. Esta documentação explica arquitetura de referência, mas normas, manuais, prazos e requisitos devem ser conferidos no material vigente do Banco Central e nos contratos dos participantes ou provedores envolvidos.

## Banco Central do Brasil

- [Diretório de Identificadores de Contas Transacionais — DICT](https://www.bcb.gov.br/estabilidadefinanceira/dict)
- [Segurança no Pix e Mecanismo Especial de Devolução](https://www.bcb.gov.br/estabilidadefinanceira/pix-seguranca)
- [Guia de implementação dos procedimentos de devolução no Pix](https://www.bcb.gov.br/content/estabilidadefinanceira/pix/Guia_MED.pdf)
- [Manual de Segurança do Pix](https://www.bcb.gov.br/content/estabilidadefinanceira/cedsfn/Manual_de_Seguranca_PIX.pdf)
- [Manual de Tempos do Pix](https://www.bcb.gov.br/content/estabilidadefinanceira/pix/Regulamento_Pix/IX_ManualdeTemposdoPix.pdf)
- [Regulamento do Pix](https://www.bcb.gov.br/estabilidadefinanceira/pix)
- [FAQ Pix para cidadãos](https://www.bcb.gov.br/meubc/faqs/s/pix)

## Kubernetes e service mesh

- [Kubernetes — Liveness, Readiness e Startup Probes](https://kubernetes.io/docs/concepts/workloads/pods/probes/)
- [Istio — Architecture](https://istio.io/latest/docs/ops/deployment/architecture/)
- [Istio — Sidecar e Ambient Data Plane](https://istio.io/latest/docs/overview/dataplane-modes/)

## Mensageria e integração

- [Apache Kafka — Documentation](https://kafka.apache.org/documentation/)
- [Amazon SQS — Developer Guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)
- [Amazon SNS — Developer Guide](https://docs.aws.amazon.com/sns/latest/dg/welcome.html)
- [Debezium — Outbox Event Router](https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html)

## APIs e padrões

- [gRPC — Introduction](https://grpc.io/docs/what-is-grpc/introduction/)
- [GraphQL — Learn](https://graphql.org/learn/)
- [Azure Architecture Center — Transactional Outbox](https://learn.microsoft.com/en-us/azure/architecture/databases/guide/transactional-out-box-cosmos)
- [Azure Architecture Center — CQRS](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [Azure Architecture Center — Saga](https://learn.microsoft.com/en-us/azure/architecture/patterns/saga)
- [Azure Architecture Center — Event Sourcing](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)
- [Azure Architecture Center — Ambassador](https://learn.microsoft.com/en-us/azure/architecture/patterns/ambassador)
- [Azure Architecture Center — Sidecar](https://learn.microsoft.com/en-us/azure/architecture/patterns/sidecar)

## Como manter as referências

- revisar links oficiais sempre que houver mudança regulatória;
- registrar data de consulta em materiais de treinamento formal;
- preferir fonte primária quando houver divergência entre blog, notícia e regulamento;
- separar requisito regulatório de recomendação arquitetural interna;
- manter ADRs para decisões locais que não vêm de norma.

---

# 20. Glossário e perguntas de revisão

Este capítulo fecha a trilha com definições curtas e perguntas para testar entendimento.

## 20.1 Glossário

| Termo | Definição curta |
|---|---|
| Pix | Arranjo brasileiro de pagamentos instantâneos. |
| DICT | Diretório de chaves Pix e dados associados para iniciação do pagamento. |
| SPI | Infraestrutura de liquidação de pagamentos instantâneos entre participantes. |
| MED | Mecanismo Especial de Devolução para hipóteses específicas previstas nas regras do Pix. |
| PSP | Prestador de Serviços de Pagamento. |
| Ledger | Registro autoritativo das movimentações financeiras internas. |
| Dupla entrada | Modelo em que lançamentos financeiros permanecem balanceados entre débitos e créditos. |
| Reserva de saldo | Bloqueio lógico de saldo para impedir uso concorrente antes da conclusão. |
| Idempotência | Capacidade de repetir a mesma intenção sem repetir o efeito. |
| `UNKNOWN` | Estado em que o resultado da operação não é conhecido com segurança. |
| Outbox | Padrão que grava estado local e intenção de publicar evento na mesma transação. |
| CDC | Captura de mudanças no banco, normalmente pelo log transacional. |
| DLQ | Fila para mensagens que falharam após tentativas configuradas. |
| Saga | Coordenação de transações locais com compensações possíveis. |
| CQRS | Separação entre modelo de comandos e modelo de consultas. |
| Event Sourcing | Modelo em que eventos são a fonte de verdade do agregado. |
| Backpressure | Controle aplicado quando consumidores ou dependências não acompanham o volume. |
| Circuit breaker | Proteção que reduz chamadas a uma dependência degradada. |
| mTLS | TLS com autenticação mútua entre cliente e servidor. |
| HSM | Módulo de segurança para proteger e usar chaves criptográficas. |
| SLI | Indicador medido de nível de serviço. |
| SLO | Objetivo interno para um SLI. |
| SLA | Compromisso contratual de nível de serviço. |
| ADR | Registro de decisão arquitetural. |
| C4 Model | Modelo para representar arquitetura em níveis de contexto, container, componente e código. |

## 20.2 Perguntas de revisão

1. Por que performance não pode vir antes de correção financeira?
2. Qual é a diferença entre DICT e SPI?
3. Por que o MED não deve ser explicado como chargeback?
4. O que torna um ledger auditável?
5. Como a reserva de saldo evita concorrência?
6. Por que `@Transactional` não basta para idempotência?
7. O que fazer quando uma operação externa termina em timeout?
8. Qual é a diferença entre fila, Pub/Sub e streaming?
9. Qual problema o Outbox resolve?
10. Por que a DLQ exige processo operacional?
11. Quando CQRS ajuda e quando atrapalha?
12. Por que Saga não é strong consistency global?
13. Quando cache é perigoso em pagamentos?
14. Qual é a diferença entre liveness e readiness?
15. O que um Adapter protege no domínio?
16. Por que webhook não substitui conciliação?
17. Quais dados não deveriam aparecer em logs?
18. Quais métricas indicam risco em pagamentos `UNKNOWN`?
19. Como uma divergência financeira deve ser corrigida?
20. Que decisões precisam virar ADR?

## 20.3 Resposta mental final

Uma arquitetura Pix confiável protege dinheiro primeiro. Ela diferencia regra externa de decisão interna, mantém fonte autoritativa clara, torna operações idempotentes, trata estado desconhecido com cuidado, publica eventos sem dual write, opera falhas com observabilidade e fecha divergências por conciliação auditável.

---
