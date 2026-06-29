← [16. Conciliação](16-conciliacao.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [18. Checklist técnico](18-checklist-tecnico.md) →

# 17. Resumo para entrevista

Use este capítulo como resposta compacta. Ele não substitui o restante do material; serve para organizar uma explicação oral.

## 17.1 Resposta principal

> Eu desenharia uma plataforma Pix com API Gateway, autenticação, rate limiting, idempotência e um serviço de pagamentos responsável pela máquina de estados. No caminho financeiro, usaria transações locais, controle de concorrência, reserva de saldo e ledger de dupla entrada. A integração com o participante Pix ficaria isolada por um Adapter.
>
> Para publicar eventos sem dual write, usaria Transactional Outbox, com CDC ou polling. Como a entrega pode ser at least once, producers, consumers e webhooks seriam idempotentes, usando identificadores únicos e constraints no banco. Retries teriam timeout, exponential backoff e jitter, sem repetir cegamente uma operação em estado desconhecido.
>
> REST seria apropriado para APIs externas e webhooks; gRPC poderia ser usado em chamadas internas de baixa latência; GraphQL ficaria principalmente em BFFs e consultas agregadas. Filas seriam usadas para distribuir trabalho, Pub/Sub para fan-out e streaming para manter um log persistente com retenção, offsets e replay.
>
> Saldo, limites e ledger usariam o modelo transacional autoritativo. Projeções, notificações e analytics poderiam ser eventualmente consistentes. A aplicação seria stateless para escalar horizontalmente, com failover seguro, probes, observabilidade, reconciliação, runbooks e postmortems. Segurança incluiria mTLS, gestão de certificados, HSM quando aplicável e proteção contra replay.

## 17.2 Versão curta

> Eu priorizaria consistência financeira, idempotência e rastreabilidade. A API validaria a intenção e usaria chave idempotente; o domínio protegeria saldo, limites e ledger em transações locais; eventos seriam publicados por Outbox; integrações externas teriam adapter, timeout, retry seguro e tratamento de estado desconhecido. O sistema seria observável, reconciliável, seguro e operável em falha parcial.

## 17.3 Pontos que impressionam em design review

- explicar por que timeout não é falha definitiva;
- separar fonte autoritativa de projeções;
- mostrar como evita dual write;
- falar de idempotência em API, consumer e integração externa;
- definir estados e transições;
- prever conciliação e operação;
- saber dizer quando uma tecnologia não é necessária.

## 17.4 Perguntas prováveis

| Pergunta | Resposta esperada |
|---|---|
| Por que não usar cache para saldo? | Porque saldo usado para autorizar dinheiro precisa de fonte autoritativa consistente. |
| Como evitar pagamento duplicado? | Idempotency key, constraint atômica, estado persistido, consumers idempotentes e identificadores externos estáveis. |
| O que fazer em timeout? | Marcar como `UNKNOWN`, consultar fonte autoritativa ou reconciliar. |
| Por que Outbox? | Para confirmar estado local e intenção de publicar evento na mesma transação. |
| Saga resolve rollback financeiro? | Não depois de ponto irreversível; compensação vira nova operação financeira. |

---

← [16. Conciliação](16-conciliacao.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [18. Checklist técnico](18-checklist-tecnico.md) →
