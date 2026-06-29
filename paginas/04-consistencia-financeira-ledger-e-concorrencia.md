← [3. Arquitetura de referência](03-arquitetura-de-referencia.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [5. Idempotência](05-idempotencia.md) →

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

← [3. Arquitetura de referência](03-arquitetura-de-referencia.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [5. Idempotência](05-idempotencia.md) →
