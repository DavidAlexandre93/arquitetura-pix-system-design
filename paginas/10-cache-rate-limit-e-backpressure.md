← [9. CQRS, Saga e Event Sourcing](09-cqrs-saga-e-event-sourcing.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [11. Escalabilidade e alta disponibilidade](11-escalabilidade-e-alta-disponibilidade.md) →

# 10. Cache, rate limit e backpressure

Este capítulo mostra como proteger latência e capacidade sem comprometer a fonte autoritativa do dinheiro.

## 10.1 TTL

TTL (**Time To Live**) é o período durante o qual um item permanece válido no cache.

```text
pix:123 = PROCESSING
TTL = 15 segundos
```

Após a expiração, a próxima leitura recarrega o valor da fonte.

TTL não produz invalidação imediata. O dado pode ficar desatualizado até expirar.

Estratégia comum:

```text
invalidação explícita ou por evento
+
TTL como rede de segurança
```

Dados críticos, como saldo usado para autorizar um Pix, não devem depender exclusivamente de cache eventualmente desatualizado.

Para evitar cache stampede:

- TTL com jitter;
- request coalescing;
- lock de recomputação;
- stale-while-revalidate;
- atualização antecipada.

Dados que costumam caber em cache:

- configuração não crítica;
- metadados de baixa volatilidade;
- resultado de consulta pública com TTL curto;
- projeções de leitura não autoritativas;
- tokens e chaves públicas com política de rotação.

Dados que exigem cuidado extremo:

- saldo disponível para autorização;
- limites transacionais ativos;
- estado final de pagamento;
- decisões de antifraude em tempo real;
- dados pessoais sensíveis.

## 10.2 Rate limiting

Rate limiting controla a quantidade de requisições por cliente, token, conta, IP ou operação.

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 2
```

Protege contra:

- abuso;
- tráfego acidental;
- consumo injusto de capacidade;
- saturação de dependências.

Dimensões comuns:

- por credencial;
- por cliente;
- por conta;
- por chave Pix;
- por IP;
- por endpoint;
- por dependência externa.

Algoritmos comuns incluem fixed window, sliding window, token bucket e leaky bucket. A escolha depende de previsibilidade, rajadas permitidas e custo de implementação.

## 10.3 Backpressure

Backpressure acontece quando o consumidor ou sistema downstream não acompanha o ritmo do produtor.

```text
Producer rápido
      ↓
Fila crescendo
      ↓
Consumer lento
```

Mecanismos possíveis:

- filas limitadas;
- controle de concorrência;
- pause/resume;
- redução de consumo;
- load shedding;
- `429` ou `503`;
- autoscaling;
- limites por dependência.

> Rate limiting pode fazer parte de uma estratégia de backpressure, mas os dois conceitos não são sinônimos.

Backpressure precisa ser visível para produto e operação. Quando o sistema começa a degradar, ele deve preferir respostas controladas, filas limitadas e estados claros a aceitar trabalho infinito que nunca será processado.

## 10.4 Degradação segura

Em pagamentos, degradação segura significa preservar integridade antes de preservar conveniência.

Exemplos:

- bloquear temporariamente uma funcionalidade não essencial;
- responder `202 Accepted` e acompanhar estado em vez de manter conexão aberta indefinidamente;
- rejeitar novas solicitações quando a dependência crítica está saturada;
- manter consulta e conciliação operando mesmo que notificações estejam atrasadas;
- pausar redrive de DLQ durante incidente.

---

← [9. CQRS, Saga e Event Sourcing](09-cqrs-saga-e-event-sourcing.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [11. Escalabilidade e alta disponibilidade](11-escalabilidade-e-alta-disponibilidade.md) →
