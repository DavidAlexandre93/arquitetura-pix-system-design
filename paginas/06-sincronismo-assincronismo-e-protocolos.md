← [5. Idempotência](05-idempotencia.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [7. Mensageria, filas, PubSub e streaming](07-mensageria-filas-pubsub-e-streaming.md) →

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

← [5. Idempotência](05-idempotencia.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [7. Mensageria, filas, PubSub e streaming](07-mensageria-filas-pubsub-e-streaming.md) →
