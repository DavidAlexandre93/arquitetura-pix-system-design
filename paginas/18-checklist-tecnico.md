← [17. Resumo para entrevista](17-resumo-para-entrevista.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [19. Referências oficiais e técnicas](19-referencias-oficiais-e-tecnicas.md) →

# 18. Checklist técnico

Use este checklist antes de uma implementação, design review, entrevista ou revisão de produção. Ele não substitui testes, auditoria ou validação regulatória.

## Escopo e responsabilidades

- [ ] papel da instituição definido: participante direto, indireto, provedor, parceiro ou integrador;
- [ ] fronteiras entre Pix, SPI, DICT, MED, API da plataforma e arquitetura interna documentadas;
- [ ] fonte autoritativa de cada estado definida;
- [ ] estados e transições do pagamento documentados;
- [ ] pontos de irreversibilidade identificados.

## Consistência e dinheiro

- [ ] operação financeira atômica;
- [ ] invariantes de saldo protegidas contra concorrência;
- [ ] ledger balanceado e auditável;
- [ ] lançamentos corrigidos por contralançamento ou nova operação formal;
- [ ] reserva de saldo com ciclo de vida definido;
- [ ] estado `UNKNOWN` tratado sem retry cego;
- [ ] reconciliação implementada.

## Idempotência

- [ ] `Idempotency-Key` com escopo definido;
- [ ] hash canônico do payload;
- [ ] `UNIQUE CONSTRAINT` no banco;
- [ ] resposta ou operação original recuperável;
- [ ] consumer idempotente por `messageId/eventId`;
- [ ] processamento e deduplicação na mesma transação local.
- [ ] integrações externas usam identificador idempotente ou consulta de status.

## Mensageria

- [ ] Outbox para evitar dual write;
- [ ] CDC ou polling com retry;
- [ ] DLQ com alerta e redrive controlado;
- [ ] ordenação definida por agregado quando necessária;
- [ ] schema e compatibilidade de eventos;
- [ ] backpressure e limites de concorrência.
- [ ] eventos não carregam dados sensíveis desnecessários.
- [ ] consumer lag e idade de mensagens monitorados.

## Resiliência

- [ ] timeouts/deadlines;
- [ ] retry com backoff e jitter;
- [ ] circuit breaker;
- [ ] bulkheads;
- [ ] health checks corretos;
- [ ] failover sem split-brain;
- [ ] backups testados por restauração.
- [ ] RTO e RPO definidos.
- [ ] degradação segura documentada.
- [ ] dependências externas possuem contingência ou conciliação.

## Segurança

- [ ] autenticação e autorização;
- [ ] TLS/mTLS;
- [ ] rotação de certificados e segredos;
- [ ] assinatura de webhooks;
- [ ] proteção contra replay;
- [ ] dados sensíveis protegidos em logs e eventos;
- [ ] trilha de auditoria.
- [ ] certificados com inventário, dono e alerta de expiração.
- [ ] segredos com rotação e acesso mínimo.
- [ ] permissões revisadas periodicamente.

## Operação

- [ ] logs estruturados;
- [ ] métricas e traces;
- [ ] SLOs e alertas acionáveis;
- [ ] runbooks;
- [ ] postmortems blameless;
- [ ] ADRs e diagramas C4 atualizados.
- [ ] dashboards cobrem estados críticos, Outbox, DLQ, webhooks e conciliação.
- [ ] alertas têm ação e dono.
- [ ] procedimentos de redrive e correção são idempotentes.

## Testes essenciais

- [ ] repetição da mesma requisição com mesma chave;
- [ ] mesma chave com payload diferente;
- [ ] concorrência em saldo;
- [ ] falha após commit local e antes da publicação;
- [ ] duplicidade de mensagem;
- [ ] webhook duplicado, atrasado e fora de ordem;
- [ ] timeout de integração externa;
- [ ] queda de instância durante processamento;
- [ ] failover de banco ou dependência crítica;
- [ ] reconciliação fechando divergência simulada.

---

← [17. Resumo para entrevista](17-resumo-para-entrevista.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [19. Referências oficiais e técnicas](19-referencias-oficiais-e-tecnicas.md) →
