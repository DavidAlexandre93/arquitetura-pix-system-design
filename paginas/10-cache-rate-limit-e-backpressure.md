← [9. CQRS, Saga e Event Sourcing](09-cqrs-saga-e-event-sourcing.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [11. Escalabilidade e alta disponibilidade](11-escalabilidade-e-alta-disponibilidade.md) →

# 10. Cache, rate limit e backpressure

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

---

---

← [9. CQRS, Saga e Event Sourcing](09-cqrs-saga-e-event-sourcing.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [11. Escalabilidade e alta disponibilidade](11-escalabilidade-e-alta-disponibilidade.md) →
