# Arquitetura de Sistemas de Pagamento com Pix

> Guia navegável em Markdown para estudar, projetar e discutir uma plataforma de pagamentos com Pix com foco em consistência financeira, resiliência, segurança e operação.

## Escopo e premissas

Esta documentação descreve uma **arquitetura de referência** para uma plataforma que oferece pagamentos via Pix, como um banco, instituição de pagamento, fintech ou plataforma integradora.

Nem todos os componentes descritos são obrigatórios pelo Banco Central. Elementos como Kafka, CQRS, Saga, Event Sourcing, Redis, gRPC e service mesh são **decisões internas de arquitetura**. Já DICT, SPI, MED e requisitos de segurança fazem parte do ecossistema e da regulamentação do Pix.

Também é importante distinguir:

- **Pix**: arranjo de pagamentos instantâneos brasileiro;
- **SPI**: infraestrutura de liquidação entre participantes;
- **DICT**: diretório de chaves Pix;
- **API da plataforma**: interface criada pela instituição para aplicativos, clientes ou parceiros;
- **arquitetura interna**: serviços, bancos, filas, streams e padrões usados pela instituição.

O objetivo não é apresentar uma implementação única. O objetivo é ensinar como pensar: onde exigir consistência forte, onde aceitar consistência eventual, como evitar duplicidade financeira, como lidar com timeouts, como publicar eventos sem dual write e como operar o sistema depois que ele está em produção.

## Público-alvo

Este material foi organizado para:

- pessoas estudando arquitetura de sistemas financeiros;
- engenheiros preparando entrevistas técnicas ou design reviews;
- times que precisam discutir uma plataforma Pix de forma estruturada;
- leitores que querem entender a diferença entre regras do ecossistema Pix e escolhas internas de engenharia.

---

## Como navegar

Esta documentação foi dividida em páginas independentes. Use o sumário abaixo ou os links **Anterior**, **Início** e **Próxima** existentes no topo e no final de cada capítulo.

- [Abrir o sumário para leitores compatíveis com mdBook](SUMMARY.md)
- [Ler o documento completo em uma única página](documentacao-completa.md)
- [Abrir o mapa de navegação compacto](NAVEGACAO.md)

Leitura recomendada:

1. Comece pelo roteiro de estudo.
2. Leia os capítulos 1 a 5 para fixar os fundamentos financeiros.
3. Leia os capítulos 6 a 13 para entender integração, mensageria e comunicação.
4. Leia os capítulos 14 a 18 para fechar segurança, operação, conciliação e revisão.
5. Use o glossário e as perguntas finais para revisar.

## Sumário

0. [Como estudar esta documentação](paginas/00-como-estudar-esta-documentacao.md)
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
20. [Glossário e perguntas de revisão](paginas/20-glossario-e-perguntas-de-revisao.md)

---

## Estrutura dos arquivos

```text
documentacao-pix-markdown/
├── README.md
├── SUMMARY.md
├── documentacao-completa.md
└── paginas/
    ├── 00-como-estudar-esta-documentacao.md
    ├── 01-principios-arquiteturais.md
    ├── 02-componentes-do-ecossistema-pix.md
    └── ...
```

> Todos os links são relativos. A navegação funciona em GitHub, GitLab, editores Markdown e ferramentas compatíveis com estruturas como mdBook.
