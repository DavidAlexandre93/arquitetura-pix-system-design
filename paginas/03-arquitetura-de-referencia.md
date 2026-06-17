← [2. Componentes do ecossistema Pix](02-componentes-do-ecossistema-pix.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [4. Consistência financeira, ledger e concorrência](04-consistencia-financeira-ledger-e-concorrencia.md) →

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

---

← [2. Componentes do ecossistema Pix](02-componentes-do-ecossistema-pix.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [4. Consistência financeira, ledger e concorrência](04-consistencia-financeira-ledger-e-concorrencia.md) →
