← [8. Outbox, CDC, retry e DLQ](08-outbox-cdc-retry-e-dlq.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [10. Cache, rate limit e backpressure](10-cache-rate-limit-e-backpressure.md) →

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

---

← [8. Outbox, CDC, retry e DLQ](08-outbox-cdc-retry-e-dlq.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [10. Cache, rate limit e backpressure](10-cache-rate-limit-e-backpressure.md) →
