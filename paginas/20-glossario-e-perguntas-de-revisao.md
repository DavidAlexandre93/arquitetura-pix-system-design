← [19. Referências oficiais e técnicas](19-referencias-oficiais-e-tecnicas.md) · [Início](../README.md) · [Sumário](../SUMMARY.md)

# 20. Glossário e perguntas de revisão

Este capítulo fecha a trilha com definições curtas e perguntas para testar entendimento.

## 20.1 Glossário

| Termo | Definição curta |
|---|---|
| Pix | Arranjo brasileiro de pagamentos instantâneos. |
| DICT | Diretório de chaves Pix e dados associados para iniciação do pagamento. |
| SPI | Infraestrutura de liquidação de pagamentos instantâneos entre participantes. |
| MED | Mecanismo Especial de Devolução para hipóteses específicas previstas nas regras do Pix. |
| PSP | Prestador de Serviços de Pagamento. |
| Ledger | Registro autoritativo das movimentações financeiras internas. |
| Dupla entrada | Modelo em que lançamentos financeiros permanecem balanceados entre débitos e créditos. |
| Reserva de saldo | Bloqueio lógico de saldo para impedir uso concorrente antes da conclusão. |
| Idempotência | Capacidade de repetir a mesma intenção sem repetir o efeito. |
| `UNKNOWN` | Estado em que o resultado da operação não é conhecido com segurança. |
| Outbox | Padrão que grava estado local e intenção de publicar evento na mesma transação. |
| CDC | Captura de mudanças no banco, normalmente pelo log transacional. |
| DLQ | Fila para mensagens que falharam após tentativas configuradas. |
| Saga | Coordenação de transações locais com compensações possíveis. |
| CQRS | Separação entre modelo de comandos e modelo de consultas. |
| Event Sourcing | Modelo em que eventos são a fonte de verdade do agregado. |
| Backpressure | Controle aplicado quando consumidores ou dependências não acompanham o volume. |
| Circuit breaker | Proteção que reduz chamadas a uma dependência degradada. |
| mTLS | TLS com autenticação mútua entre cliente e servidor. |
| HSM | Módulo de segurança para proteger e usar chaves criptográficas. |
| SLI | Indicador medido de nível de serviço. |
| SLO | Objetivo interno para um SLI. |
| SLA | Compromisso contratual de nível de serviço. |
| ADR | Registro de decisão arquitetural. |
| C4 Model | Modelo para representar arquitetura em níveis de contexto, container, componente e código. |

## 20.2 Perguntas de revisão

1. Por que performance não pode vir antes de correção financeira?
2. Qual é a diferença entre DICT e SPI?
3. Por que o MED não deve ser explicado como chargeback?
4. O que torna um ledger auditável?
5. Como a reserva de saldo evita concorrência?
6. Por que `@Transactional` não basta para idempotência?
7. O que fazer quando uma operação externa termina em timeout?
8. Qual é a diferença entre fila, Pub/Sub e streaming?
9. Qual problema o Outbox resolve?
10. Por que a DLQ exige processo operacional?
11. Quando CQRS ajuda e quando atrapalha?
12. Por que Saga não é strong consistency global?
13. Quando cache é perigoso em pagamentos?
14. Qual é a diferença entre liveness e readiness?
15. O que um Adapter protege no domínio?
16. Por que webhook não substitui conciliação?
17. Quais dados não deveriam aparecer em logs?
18. Quais métricas indicam risco em pagamentos `UNKNOWN`?
19. Como uma divergência financeira deve ser corrigida?
20. Que decisões precisam virar ADR?

## 20.3 Resposta mental final

Uma arquitetura Pix confiável protege dinheiro primeiro. Ela diferencia regra externa de decisão interna, mantém fonte autoritativa clara, torna operações idempotentes, trata estado desconhecido com cuidado, publica eventos sem dual write, opera falhas com observabilidade e fecha divergências por conciliação auditável.

---

← [19. Referências oficiais e técnicas](19-referencias-oficiais-e-tecnicas.md) · [Início](../README.md) · [Sumário](../SUMMARY.md)
