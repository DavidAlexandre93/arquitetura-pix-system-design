← [7. Mensageria, filas, PubSub e streaming](07-mensageria-filas-pubsub-e-streaming.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [9. CQRS, Saga e Event Sourcing](09-cqrs-saga-e-event-sourcing.md) →

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

---

← [7. Mensageria, filas, PubSub e streaming](07-mensageria-filas-pubsub-e-streaming.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [9. CQRS, Saga e Event Sourcing](09-cqrs-saga-e-event-sourcing.md) →
