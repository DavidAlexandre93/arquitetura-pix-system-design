# Arquitetura de Sistemas de Pagamento com Pix

> Documentação navegável em Markdown para estudo, entrevistas técnicas e discussões de arquitetura.

## Escopo e premissas

Este documento descreve uma **arquitetura de referência** para uma plataforma que oferece pagamentos via Pix, como um banco, instituição de pagamento, fintech ou plataforma integradora.

Nem todos os componentes descritos são obrigatórios pelo Banco Central. Elementos como Kafka, CQRS, Saga, Event Sourcing, Redis, gRPC e service mesh são **decisões internas de arquitetura**. Já DICT, SPI, MED e requisitos de segurança fazem parte do ecossistema e da regulamentação do Pix.

Também é importante distinguir:

- **Pix**: arranjo de pagamentos instantâneos brasileiro;
- **SPI**: infraestrutura de liquidação entre participantes;
- **DICT**: diretório de chaves Pix;
- **API da plataforma**: interface criada pela instituição para aplicativos, clientes ou parceiros;
- **arquitetura interna**: serviços, bancos, filas, streams e padrões usados pela instituição.

---

## Como navegar

Esta documentação foi dividida em páginas independentes. Use o sumário abaixo ou os links **Anterior**, **Início** e **Próxima** existentes no topo e no final de cada capítulo.

## Sumário

1. [Princípios arquiteturais](paginas/01-principios-arquiteturais.md)
2. [Componentes do ecossistema Pix](paginas/02-componentes-do-ecossistema-pix.md)
3. [Arquitetura de referência](paginas/03-arquitetura-de-referencia.md)
4. [Consistência financeira, ledger e concorrência](paginas/04-consistencia-financeira-ledger-e-concorrencia.md)
5. [Idempotência](paginas/05-idempotencia.md)
6. [Sincronismo, assincronismo e protocolos](paginas/06-sincronismo-assincronismo-e-protocolos.md)
7. [Mensageria, filas, PubSub e streaming](paginas/07-mensageria-filas-pubsub-e-streaming.md)
8. [Outbox, CDC, retry e DLQ](paginas/08-outbox-cdc-retry-e-dlq.md)
9. [CQRS, Saga e Event Sourcing](paginas/09-cqrs-saga-e-event-sourcing.md)
10. [Cache, rate limit e backpressure](paginas/10-cache-rate-limit-e-backpressure.md)
11. [Escalabilidade e alta disponibilidade](paginas/11-escalabilidade-e-alta-disponibilidade.md)
12. [Sidecar, service mesh, Ambassador e Adapter](paginas/12-sidecar-service-mesh-ambassador-e-adapter.md)
13. [Webhooks e WebSockets](paginas/13-webhooks-e-websockets.md)
14. [Segurança](paginas/14-seguranca.md)
15. [Observabilidade, operação e documentação](paginas/15-observabilidade-operacao-e-documentacao.md)
16. [Conciliação](paginas/16-conciliacao.md)
17. [Resumo para entrevista](paginas/17-resumo-para-entrevista.md)
18. [Checklist técnico](paginas/18-checklist-tecnico.md)
19. [Referências oficiais e técnicas](paginas/19-referencias-oficiais-e-tecnicas.md)

---

## Estrutura dos arquivos

```text
documentacao-pix-markdown/
├── README.md
├── SUMMARY.md
├── documentacao-completa.md
└── paginas/
    ├── 01-principios-arquiteturais.md
    ├── 02-componentes-do-ecossistema-pix.md
    └── ...
```

> Todos os links são relativos. A navegação funciona em GitHub, GitLab, editores Markdown e ferramentas compatíveis com estruturas como mdBook.
