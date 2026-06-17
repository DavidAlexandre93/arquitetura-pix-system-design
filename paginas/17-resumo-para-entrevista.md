← [16. Conciliação](16-conciliacao.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [18. Checklist técnico](18-checklist-tecnico.md) →

# 17. Resumo para entrevista

> Eu desenharia uma plataforma Pix com API Gateway, autenticação, rate limiting, idempotência e um serviço de pagamentos responsável pela máquina de estados. No caminho financeiro, usaria transações locais, controle de concorrência, reserva de saldo e ledger de dupla entrada. A integração com o participante Pix ficaria isolada por um Adapter.
>
> Para publicar eventos sem dual write, usaria Transactional Outbox, com CDC ou polling. Como a entrega pode ser at least once, producers, consumers e webhooks seriam idempotentes, usando identificadores únicos e constraints no banco. Retries teriam timeout, exponential backoff e jitter, sem repetir cegamente uma operação em estado desconhecido.
>
> REST seria apropriado para APIs externas e webhooks; gRPC poderia ser usado em chamadas internas de baixa latência; GraphQL ficaria principalmente em BFFs e consultas agregadas. Filas seriam usadas para distribuir trabalho, Pub/Sub para fan-out e streaming para manter um log persistente com retenção, offsets e replay.
>
> Saldo, limites e ledger usariam o modelo transacional autoritativo. Projeções, notificações e analytics poderiam ser eventualmente consistentes. A aplicação seria stateless para escalar horizontalmente, com failover seguro, probes, observabilidade, reconciliação, runbooks e postmortems. Segurança incluiria mTLS, gestão de certificados, HSM quando aplicável e proteção contra replay.

---

---

← [16. Conciliação](16-conciliacao.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [18. Checklist técnico](18-checklist-tecnico.md) →
