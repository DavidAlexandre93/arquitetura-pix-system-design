← [0. Como estudar esta documentação](00-como-estudar-esta-documentacao.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [2. Componentes do ecossistema Pix](02-componentes-do-ecossistema-pix.md) →

# 1. Princípios arquiteturais

Este capítulo define as prioridades que devem orientar todo o desenho. Em sistemas de pagamento, tecnologia vem depois de invariantes financeiras, rastreabilidade e operação segura.

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

Em termos práticos, otimize a latência sem remover proteções como:

- constraint de unicidade;
- transação local;
- controle de concorrência;
- validação de estado;
- registro auditável;
- reconciliação posterior.

Se uma otimização torna impossível provar o que aconteceu com o dinheiro, ela deve ser redesenhada.

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

Regra de bolso:

| Dado ou decisão | Garantia esperada |
|---|---|
| saldo disponível para autorizar pagamento | consistência forte |
| reserva de saldo | consistência forte |
| ledger | consistência forte e auditável |
| notificação push | consistência eventual |
| dashboard operacional | consistência eventual com atraso conhecido |
| analytics | consistência eventual |

## 1.3 Fonte de verdade

Em uma arquitetura baseada em ledger, o ledger é a fonte de verdade financeira. Saldo, extrato e relatórios podem ser projeções dos lançamentos.

Isso é uma recomendação arquitetural, não uma regra que define toda implementação possível do Pix.

O ponto essencial é existir uma fonte autoritativa clara. Se cada serviço "calcula" o saldo de uma forma, a plataforma perde capacidade de auditoria e correção.

## 1.4 Invariantes que o sistema deve proteger

Invariantes são regras que continuam verdadeiras mesmo sob concorrência, retry, falha parcial ou pico de tráfego.

Exemplos:

- uma mesma intenção de pagamento não gera dois débitos;
- saldo disponível não fica negativo quando isso não é permitido;
- cada lançamento financeiro tem vínculo com uma operação de negócio;
- todo débito esperado possui crédito correspondente;
- estados terminais não voltam para estados intermediários sem uma nova operação formal;
- divergências entram em fluxo de conciliação e não são corrigidas por alteração manual invisível.

## 1.5 Perguntas de revisão

- Qual componente decide se há saldo suficiente?
- Qual fonte prova que a operação foi concluída?
- O que impede duas instâncias de debitarem a mesma conta ao mesmo tempo?
- O que acontece se um evento for publicado duas vezes?
- O que acontece se o cliente repetir a mesma requisição?

---

← [0. Como estudar esta documentação](00-como-estudar-esta-documentacao.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [2. Componentes do ecossistema Pix](02-componentes-do-ecossistema-pix.md) →
