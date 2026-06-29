[Início](../README.md) · [Sumário](../SUMMARY.md) · [1. Princípios arquiteturais](01-principios-arquiteturais.md) →

# 0. Como estudar esta documentação

Este guia foi escrito para ser lido de duas formas: como trilha de estudo, do começo ao fim, e como material de consulta durante uma discussão de arquitetura.

## 0.1 Modelo mental

Ao projetar uma plataforma Pix, separe sempre quatro planos:

```text
Regra do ecossistema Pix
→ o que vem de regulamento, DICT, SPI, MED, segurança e manuais oficiais.

Contrato externo da plataforma
→ APIs, webhooks, autenticação, idempotência e experiência dos clientes.

Domínio financeiro interno
→ saldo, reserva, ledger, máquina de estados, limites e conciliação.

Infraestrutura de execução
→ banco, cache, filas, streaming, Kubernetes, observabilidade e operação.
```

Muitas confusões aparecem quando esses planos são misturados. Por exemplo, Kafka pode ser excelente internamente, mas não é uma exigência do Pix. Da mesma forma, o SPI é parte do ecossistema de liquidação, não "a fila" da sua aplicação.

## 0.2 Ordem recomendada

1. Leia os princípios arquiteturais antes de qualquer tecnologia.
2. Entenda os componentes do ecossistema Pix.
3. Estude a arquitetura de referência como um mapa, não como receita fechada.
4. Aprofunde consistência, ledger e idempotência.
5. Só então compare REST, gRPC, filas, Pub/Sub, streaming, Outbox, Saga e CQRS.
6. Feche com segurança, observabilidade, conciliação e checklist.

## 0.3 Perguntas que guiam o desenho

Antes de escolher uma tecnologia, responda:

- Quem é a fonte autoritativa deste estado?
- O efeito é financeiro, operacional ou apenas derivado?
- A operação pode ser repetida sem duplicar dinheiro?
- O que acontece se o timeout ocorrer depois que o pagamento foi aceito?
- Como detectar e corrigir divergência?
- Como auditar a decisão meses depois?
- Como operar o sistema em incidente, pico de carga ou indisponibilidade parcial?

## 0.4 O que é decisão de produto, regra externa e decisão interna

| Tipo | Exemplos | Como tratar |
|---|---|---|
| Regra externa | DICT, SPI, MED, requisitos de segurança, manuais oficiais | validar sempre no material vigente do Banco Central |
| Decisão de produto | experiência do cliente, limites comerciais, canais, notificações | alinhar com negócio, jurídico, risco e atendimento |
| Decisão interna de arquitetura | banco, cache, broker, service mesh, CQRS, Saga | justificar por requisitos de consistência, escala, operação e custo |

## 0.5 Como usar em entrevista técnica

Em entrevista, evite começar por ferramentas. Comece por invariantes:

```text
Não duplicar débito.
Não autorizar sem saldo ou limite.
Não repetir cegamente operação desconhecida.
Não perder evento confirmado.
Não depender de cache para saldo autoritativo.
Não operar sem trilha de auditoria e conciliação.
```

Depois explique as escolhas: ledger, idempotência, transações locais, Outbox, consumidores idempotentes, estado `UNKNOWN`, reconciliação, segurança e observabilidade.

---

[Início](../README.md) · [Sumário](../SUMMARY.md) · [1. Princípios arquiteturais](01-principios-arquiteturais.md) →
