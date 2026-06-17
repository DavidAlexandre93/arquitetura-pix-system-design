← [3. Arquitetura de referência](03-arquitetura-de-referencia.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [5. Idempotência](05-idempotencia.md) →

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

---

← [3. Arquitetura de referência](03-arquitetura-de-referencia.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [5. Idempotência](05-idempotencia.md) →
