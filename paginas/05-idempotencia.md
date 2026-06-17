← [4. Consistência financeira, ledger e concorrência](04-consistencia-financeira-ledger-e-concorrencia.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [6. Sincronismo, assincronismo e protocolos](06-sincronismo-assincronismo-e-protocolos.md) →

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

---

← [4. Consistência financeira, ledger e concorrência](04-consistencia-financeira-ledger-e-concorrencia.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [6. Sincronismo, assincronismo e protocolos](06-sincronismo-assincronismo-e-protocolos.md) →
