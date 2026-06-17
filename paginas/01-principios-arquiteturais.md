[Início](../README.md) · [Sumário](../SUMMARY.md) · [2. Componentes do ecossistema Pix](02-componentes-do-ecossistema-pix.md) →

# 1. Princípios arquiteturais

## 1.1 Performance não pode comprometer a correção financeira

Uma API de pagamentos deve buscar:

- baixa latência;
- alto throughput;
- estabilidade sob picos;
- escalabilidade horizontal;
- disponibilidade;
- uso eficiente de CPU, memória, rede e banco de dados.

Entretanto, em pagamentos, a prioridade deve ser:

```text
Correção financeira
→ integridade e consistência
→ idempotência
→ segurança
→ resiliência
→ disponibilidade
→ performance
```

Uma resposta rápida que duplica um débito é pior do que uma resposta um pouco mais lenta e correta.

## 1.2 Consistência forte e consistência eventual

Use garantias fortes no caminho que autoriza ou movimenta dinheiro:

- saldo disponível;
- reserva de saldo;
- limites transacionais;
- autorização do pagamento;
- lançamentos no ledger;
- controle de concorrência;
- idempotência da operação financeira.

A consistência eventual costuma ser aceitável em dados derivados:

- dashboards;
- analytics;
- notificações;
- indexação para busca;
- projeções de leitura não autoritativas;
- envio de webhooks;
- relatórios não usados para autorizar transações.

> O extrato pode ser uma projeção, mas a fonte usada para autorizar um Pix deve ser o modelo transacional autoritativo.

## 1.3 Fonte de verdade

Em uma arquitetura baseada em ledger, o ledger é a fonte de verdade financeira. Saldo, extrato e relatórios podem ser projeções dos lançamentos.

Isso é uma recomendação arquitetural, não uma regra que define toda implementação possível do Pix.

---

---

[Início](../README.md) · [Sumário](../SUMMARY.md) · [2. Componentes do ecossistema Pix](02-componentes-do-ecossistema-pix.md) →
