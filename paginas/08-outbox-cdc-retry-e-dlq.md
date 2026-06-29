← [7. Mensageria, filas, PubSub e streaming](07-mensageria-filas-pubsub-e-streaming.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [9. CQRS, Saga e Event Sourcing](09-cqrs-saga-e-event-sourcing.md) →

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

← [7. Mensageria, filas, PubSub e streaming](07-mensageria-filas-pubsub-e-streaming.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [9. CQRS, Saga e Event Sourcing](09-cqrs-saga-e-event-sourcing.md) →
