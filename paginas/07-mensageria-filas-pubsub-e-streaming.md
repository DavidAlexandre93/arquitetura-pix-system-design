← [6. Sincronismo, assincronismo e protocolos](06-sincronismo-assincronismo-e-protocolos.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [8. Outbox, CDC, retry e DLQ](08-outbox-cdc-retry-e-dlq.md) →

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

← [6. Sincronismo, assincronismo e protocolos](06-sincronismo-assincronismo-e-protocolos.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [8. Outbox, CDC, retry e DLQ](08-outbox-cdc-retry-e-dlq.md) →
