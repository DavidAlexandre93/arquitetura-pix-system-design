[← Voltar ao início](README.md)

# Arquitetura de Sistemas de Pagamento com Pix

> Material de estudo para entrevistas técnicas e discussões de arquitetura.  

## Escopo e premissas

Este documento descreve uma **arquitetura de referência** para uma plataforma que oferece pagamentos via Pix, como um banco, instituição de pagamento, fintech ou plataforma integradora.

Nem todos os componentes descritos são obrigatórios pelo Banco Central. Elementos como Kafka, CQRS, Saga, Event Sourcing, Redis, gRPC e service mesh são **decisões internas de arquitetura**. Já DICT, SPI, MED e requisitos de segurança fazem parte do ecossistema e da regulamentação do Pix.

Também é importante distinguir:

- **Pix**: arranjo de pagamentos instantâneos brasileiro;
- **SPI**: infraestrutura de liquidação entre participantes;
- **DICT**: diretório de chaves Pix;
- **API da plataforma**: interface criada pela instituição para aplicativos, clientes ou parceiros;
- **arquitetura interna**: serviços, bancos, filas, streams e padrões usados pela instituição.

---

## Sumário

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

---

# 1. Princípios arquiteturais

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

## 1.3 Fonte de verdade

Em uma arquitetura baseada em ledger, o ledger é a fonte de verdade financeira. Saldo, extrato e relatórios podem ser projeções dos lançamentos.

Isso é uma recomendação arquitetural, não uma regra que define toda implementação possível do Pix.

---

# 2. Componentes do ecossistema Pix

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

## 2.3 MED

**MED** significa **Mecanismo Especial de Devolução**.

É um mecanismo específico do Pix para apoiar a tentativa de recuperação de valores em situações previstas na regulamentação, como fraude, golpe e coerção, além de determinados casos de falha operacional.

Pontos importantes:

- não é equivalente ao chargeback de cartão;
- não garante recuperação integral;
- depende da análise do caso e da disponibilidade de recursos;
- não é o mecanismo adequado para qualquer desacordo comercial ou erro voluntário de envio;
- as regras são atualizadas pelo Banco Central e devem ser consultadas no regulamento vigente.

Desde **1º de outubro de 2025**, os participantes passaram a disponibilizar o autoatendimento para contestação de transações elegíveis ao MED em seus aplicativos.

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

---

# 3. Arquitetura de referência

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

---

# 4. Consistência financeira, ledger e concorrência

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

---

# 5. Idempotência

## 5.1 Onde validar

A idempotência deve existir em várias camadas:

1. **API**: recebe uma `Idempotency-Key`;
2. **código**: valida formato, payload e estado;
3. **transação**: agrupa as alterações locais;
4. **banco**: garante unicidade atomicamente;
5. **consumer**: deduplica mensagens e eventos;
6. **integrações externas**: usa identificadores estáveis e consulta de status quando necessário.

> `@Transactional` sozinho não garante idempotência. Duas instâncias podem executar a mesma lógica concorrentemente. A garantia final de exclusão deve estar em uma operação atômica, normalmente uma `UNIQUE CONSTRAINT` ou atualização condicional no banco.

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

Evite o padrão vulnerável a corrida:

```java
if (!repository.existsById(key)) {
    repository.save(record);
}
```

Prefira tentar inserir e tratar a violação de unicidade.

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

## 5.4 Idempotência não significa exatamente uma entrega

Idempotência significa que repetir a mesma intenção produz o mesmo efeito observável. Ela não impede necessariamente mensagens duplicadas; ela impede que duplicidades gerem efeitos financeiros duplicados.

---

# 6. Sincronismo, assincronismo e protocolos

## 6.1 O Pix é síncrono ou assíncrono?

A resposta correta é: **depende da fronteira analisada**.

- Uma chamada HTTP do aplicativo para a API é síncrona no transporte: o cliente envia e espera uma resposta.
- A resposta pode já conter o resultado final ou um estado intermediário, conforme o fluxo e os limites de tempo.
- A arquitetura interna pode usar filas, eventos e processamento assíncrono.
- A confirmação posterior pode chegar por consulta, evento ou webhook.
- O SPI possui protocolos e limites de tempo próprios; não deve ser descrito simplesmente como “uma fila assíncrona”.

Formulação recomendada:

> Uma plataforma Pix pode usar uma interface síncrona na entrada e processamento assíncrono em etapas internas ou derivadas. Isso é uma escolha arquitetural da plataforma, não uma definição de que todo o Pix é assíncrono.

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

---

# 7. Mensageria, filas, PubSub e streaming

## 7.1 Conceitos

**Messaging** é o conceito geral de comunicação por mensagens. Fila, Pub/Sub e streaming são modelos ou capacidades que podem existir em uma plataforma de messaging.

| Conceito | Objetivo | Distribuição | Retenção e replay |
|---|---|---|---|
| Messaging | comunicação desacoplada por mensagens | depende do modelo | depende da tecnologia |
| Fila | distribuir trabalho | uma mensagem para um consumer concorrente | normalmente até processamento/expiração |
| Pub/Sub | notificar interessados | uma publicação para vários subscribers | pode ou não existir |
| Streaming | manter e processar um log contínuo de eventos | vários consumer groups independentes | normalmente central ao modelo |

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

## 7.5 Tópico

“Tópico” é um canal lógico. O comportamento depende da tecnologia:

- em SNS, um tópico distribui mensagens para subscribers;
- em Kafka, um tópico é um log particionado e retido;
- em outros brokers, tópico pode representar roteamento Pub/Sub.

Portanto, tópico não é sinônimo universal de streaming.

## 7.6 Kafka

Kafka é uma plataforma de event streaming.

- produtores escrevem em tópicos;
- tópicos são divididos em partições;
- registros ficam retidos conforme configuração;
- cada consumer group lê independentemente;
- consumers do mesmo grupo dividem as partições;
- replay é feito reposicionando offsets.

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

---

# 8. Outbox, CDC, retry e DLQ

## 8.1 Problema de dual write

Sem Outbox:

```text
1. atualizar o banco
2. publicar no broker
```

Falhas possíveis:

- banco confirma, broker falha: dado existe, evento não foi publicado;
- broker recebe, banco faz rollback: evento representa algo que não foi confirmado.

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

---

# 9. CQRS, Saga e Event Sourcing

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

---

# 10. Cache, rate limit e backpressure

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

---

# 11. Escalabilidade e alta disponibilidade

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

## 11.5 Split-brain

Split-brain ocorre quando duas instâncias acreditam ser líderes e aceitam escritas.

Mitigações:

- quórum;
- eleição de líder;
- fencing;
- leases;
- epochs/terms;
- proteção contra escrita no antigo primário.

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

## 11.7 Circuit breaker, timeout e retry

- **timeout/deadline** limita quanto tempo a chamada pode ocupar recursos;
- **retry** tenta novamente falhas transitórias e seguras;
- **circuit breaker** reduz chamadas para uma dependência degradada;
- **bulkhead** isola pools e recursos;
- **fallback** fornece uma resposta alternativa quando possível.

Retry sem timeout, sem backoff ou sem idempotência pode amplificar uma falha.

---

# 12. Sidecar, service mesh, Ambassador e Adapter

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

## 12.5 Entrada e saída da aplicação

```text
Entrada externa:
PSP → webhook → WAF/Gateway/Ingress → Webhook Adapter → domínio

Saída externa:
domínio → Provider Adapter → Ambassador/Egress opcional → PSP
```

---

# 13. Webhooks e WebSockets

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

---

# 14. Segurança

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

---

# 15. Observabilidade, operação e documentação

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

## 15.3 Runbook

Documento operacional usado durante incidentes ou tarefas recorrentes:

- sintomas;
- diagnóstico;
- mitigação;
- comandos seguros;
- critérios de escalonamento;
- validação da recuperação;
- rollback ou contingência.

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

## 15.6 C4 Model

Níveis:

1. **Context**: sistema, usuários e sistemas externos;
2. **Container**: aplicações, bancos, brokers e frontends;
3. **Component**: componentes internos de uma aplicação;
4. **Code**: classes ou detalhes de implementação, quando necessário.

No C4, “container” não significa apenas Docker; é uma unidade executável ou armazenadora relevante para a arquitetura.

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

---

# 16. Conciliação

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

---

# 17. Resumo para entrevista

> Eu desenharia uma plataforma Pix com API Gateway, autenticação, rate limiting, idempotência e um serviço de pagamentos responsável pela máquina de estados. No caminho financeiro, usaria transações locais, controle de concorrência, reserva de saldo e ledger de dupla entrada. A integração com o participante Pix ficaria isolada por um Adapter.
>
> Para publicar eventos sem dual write, usaria Transactional Outbox, com CDC ou polling. Como a entrega pode ser at least once, producers, consumers e webhooks seriam idempotentes, usando identificadores únicos e constraints no banco. Retries teriam timeout, exponential backoff e jitter, sem repetir cegamente uma operação em estado desconhecido.
>
> REST seria apropriado para APIs externas e webhooks; gRPC poderia ser usado em chamadas internas de baixa latência; GraphQL ficaria principalmente em BFFs e consultas agregadas. Filas seriam usadas para distribuir trabalho, Pub/Sub para fan-out e streaming para manter um log persistente com retenção, offsets e replay.
>
> Saldo, limites e ledger usariam o modelo transacional autoritativo. Projeções, notificações e analytics poderiam ser eventualmente consistentes. A aplicação seria stateless para escalar horizontalmente, com failover seguro, probes, observabilidade, reconciliação, runbooks e postmortems. Segurança incluiria mTLS, gestão de certificados, HSM quando aplicável e proteção contra replay.

---

# 18. Checklist técnico

## Consistência e dinheiro

- [ ] operação financeira atômica;
- [ ] invariantes de saldo protegidas contra concorrência;
- [ ] ledger balanceado e auditável;
- [ ] estado `UNKNOWN` tratado sem retry cego;
- [ ] reconciliação implementada.

## Idempotência

- [ ] `Idempotency-Key` com escopo definido;
- [ ] hash canônico do payload;
- [ ] `UNIQUE CONSTRAINT` no banco;
- [ ] resposta ou operação original recuperável;
- [ ] consumer idempotente por `messageId/eventId`;
- [ ] processamento e deduplicação na mesma transação local.

## Mensageria

- [ ] Outbox para evitar dual write;
- [ ] CDC ou polling com retry;
- [ ] DLQ com alerta e redrive controlado;
- [ ] ordenação definida por agregado quando necessária;
- [ ] schema e compatibilidade de eventos;
- [ ] backpressure e limites de concorrência.

## Resiliência

- [ ] timeouts/deadlines;
- [ ] retry com backoff e jitter;
- [ ] circuit breaker;
- [ ] bulkheads;
- [ ] health checks corretos;
- [ ] failover sem split-brain;
- [ ] backups testados por restauração.

## Segurança

- [ ] autenticação e autorização;
- [ ] TLS/mTLS;
- [ ] rotação de certificados e segredos;
- [ ] assinatura de webhooks;
- [ ] proteção contra replay;
- [ ] dados sensíveis protegidos em logs e eventos;
- [ ] trilha de auditoria.

## Operação

- [ ] logs estruturados;
- [ ] métricas e traces;
- [ ] SLOs e alertas acionáveis;
- [ ] runbooks;
- [ ] postmortems blameless;
- [ ] ADRs e diagramas C4 atualizados.

---

# 19. Referências oficiais e técnicas

## Banco Central do Brasil

- [Diretório de Identificadores de Contas Transacionais — DICT](https://www.bcb.gov.br/estabilidadefinanceira/dict)
- [Segurança no Pix e Mecanismo Especial de Devolução](https://www.bcb.gov.br/estabilidadefinanceira/pix-seguranca)
- [Guia de implementação dos procedimentos de devolução no Pix](https://www.bcb.gov.br/content/estabilidadefinanceira/pix/Guia_MED.pdf)
- [Manual de Segurança do Pix](https://www.bcb.gov.br/content/estabilidadefinanceira/cedsfn/Manual_de_Seguranca_PIX.pdf)
- [Manual de Tempos do Pix](https://www.bcb.gov.br/content/estabilidadefinanceira/pix/Regulamento_Pix/IX_ManualdeTemposdoPix.pdf)

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
