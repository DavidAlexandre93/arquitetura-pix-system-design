← [2. Componentes do ecossistema Pix](02-componentes-do-ecossistema-pix.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [4. Consistência financeira, ledger e concorrência](04-consistencia-financeira-ledger-e-concorrencia.md) →

# 3. Arquitetura de referência

Esta arquitetura é uma referência conceitual. Ela mostra fronteiras e responsabilidades, não uma obrigação de produto, cloud ou framework.

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

Leia o diagrama em dois caminhos:

- **caminho financeiro síncrono ou coordenado**: validação, idempotência, limites, antifraude, saldo, ledger e integração Pix;
- **caminho derivado assíncrono**: notificações, webhooks, conciliação, extrato, auditoria e analytics.

O primeiro caminho protege dinheiro. O segundo distribui consequências de algo que já foi aceito, confirmado ou precisa ser acompanhado.

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

O gateway pode rejeitar requisições obviamente inválidas, mas não deve ser a fonte autoritativa de saldo, ledger, limites financeiros ou estado final do pagamento.

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

Em sistemas menores, o orquestrador pode ser um módulo dentro de um serviço monolítico bem estruturado. Em sistemas maiores, pode ser um serviço dedicado. O ponto importante é manter a coordenação explícita e auditável.

Decisões que precisam aparecer no desenho:

- quem cria o identificador interno do pagamento;
- quem valida idempotência;
- quem reserva e libera saldo;
- quem registra lançamentos no ledger;
- quem traduz estados do provedor para estados internos;
- quem publica eventos;
- quem inicia conciliação quando o estado fica incerto.

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

Estados devem ter transições permitidas. Isso evita correções improvisadas e torna incidentes rastreáveis.

Exemplo de regra:

```text
COMPLETED não volta para PROCESSING.
REJECTED não vira COMPLETED sem uma nova evidência externa e trilha de auditoria.
UNKNOWN exige consulta, webhook confiável, conciliação ou intervenção operacional controlada.
```

## 3.4 Anti-patterns comuns

- colocar regra financeira apenas no gateway;
- tratar timeout como falha definitiva;
- repetir envio externo sem idempotência e sem consulta de status;
- atualizar banco e publicar evento em operações separadas sem Outbox ou mecanismo equivalente;
- usar cache como fonte autoritativa de saldo;
- não registrar o motivo de transições manuais;
- não separar estado interno de estado informado por provedor externo.

---

← [2. Componentes do ecossistema Pix](02-componentes-do-ecossistema-pix.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [4. Consistência financeira, ledger e concorrência](04-consistencia-financeira-ledger-e-concorrencia.md) →
