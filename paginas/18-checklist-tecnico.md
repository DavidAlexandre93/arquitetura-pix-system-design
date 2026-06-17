← [17. Resumo para entrevista](17-resumo-para-entrevista.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [19. Referências oficiais e técnicas](19-referencias-oficiais-e-tecnicas.md) →

# 18. Checklist técnico

## Consistência e dinheiro

- [ ] operação financeira atômica;
- [ ] invariantes de saldo protegidas contra concorrência;
- [ ] ledger balanceado e auditável;
- [ ] estado `UNKNOWN` tratado sem retry cego;
- [ ] reconciliação implementada.

## Idempotência

- [ ] `Idempotency-Key` com escopo definido;
- [ ] hash canônico do payload;
- [ ] `UNIQUE CONSTRAINT` no banco;
- [ ] resposta ou operação original recuperável;
- [ ] consumer idempotente por `messageId/eventId`;
- [ ] processamento e deduplicação na mesma transação local.

## Mensageria

- [ ] Outbox para evitar dual write;
- [ ] CDC ou polling com retry;
- [ ] DLQ com alerta e redrive controlado;
- [ ] ordenação definida por agregado quando necessária;
- [ ] schema e compatibilidade de eventos;
- [ ] backpressure e limites de concorrência.

## Resiliência

- [ ] timeouts/deadlines;
- [ ] retry com backoff e jitter;
- [ ] circuit breaker;
- [ ] bulkheads;
- [ ] health checks corretos;
- [ ] failover sem split-brain;
- [ ] backups testados por restauração.

## Segurança

- [ ] autenticação e autorização;
- [ ] TLS/mTLS;
- [ ] rotação de certificados e segredos;
- [ ] assinatura de webhooks;
- [ ] proteção contra replay;
- [ ] dados sensíveis protegidos em logs e eventos;
- [ ] trilha de auditoria.

## Operação

- [ ] logs estruturados;
- [ ] métricas e traces;
- [ ] SLOs e alertas acionáveis;
- [ ] runbooks;
- [ ] postmortems blameless;
- [ ] ADRs e diagramas C4 atualizados.

---

---

← [17. Resumo para entrevista](17-resumo-para-entrevista.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [19. Referências oficiais e técnicas](19-referencias-oficiais-e-tecnicas.md) →
