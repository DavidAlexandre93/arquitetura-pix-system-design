← [11. Escalabilidade e alta disponibilidade](11-escalabilidade-e-alta-disponibilidade.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [13. Webhooks e WebSockets](13-webhooks-e-websockets.md) →

# 12. Sidecar, service mesh, Ambassador e Adapter

Este capítulo diferencia padrões que costumam ser confundidos. Sidecar e service mesh são mais ligados à implantação e comunicação. Ambassador é um proxy auxiliar. Adapter é um padrão de integração entre contratos.

## 12.1 Sidecar

Sidecar é um componente executado ao lado da aplicação e com ciclo de vida relacionado.

```text
Pod
├── aplicação
└── sidecar
```

Usos:

- proxy;
- coleta de logs;
- agente de segurança;
- configuração;
- adaptação de protocolo;
- telemetria.

É um padrão de implantação, não uma solução específica.

Use com parcimônia. Cada sidecar adiciona consumo de CPU, memória, configuração, logs, métricas e superfície de falha.

## 12.2 Service mesh

Service mesh é a infraestrutura lógica que gerencia comunicação entre workloads.

No modo tradicional com sidecars:

```text
App A → Proxy A → Proxy B → App B
```

Componentes:

```text
Data plane
→ proxies que processam o tráfego.

Control plane
→ configura proxies e políticas.
```

Pode oferecer:

- mTLS;
- load balancing;
- roteamento;
- retries e timeouts;
- circuit breaker;
- métricas e tracing;
- políticas de segurança.

Formulação correta:

> Cada aplicação pode ter um proxy sidecar; os proxies e o control plane formam um único service mesh lógico.

Nem todo service mesh atual exige sidecar por pod. Modelos ambient podem usar proxies por nó e waypoints.

Service mesh faz sentido quando a organização precisa de políticas consistentes de tráfego, mTLS, observabilidade e segurança entre muitos serviços. Pode ser excesso para poucos serviços ou para um time que ainda não consegue operar bem a complexidade básica.

## 12.3 Ambassador Pattern

Ambassador é um proxy auxiliar colocado próximo ao cliente para realizar chamadas de rede em nome da aplicação.

```text
Aplicação
   ↓
Ambassador
   ↓
Serviço externo
```

Pode encapsular:

- TLS/mTLS;
- autenticação;
- roteamento;
- retries;
- circuit breaker;
- logging e métricas.

É comum em comunicação **de saída**, mas não é obrigatório para todo egress. Um egress gateway do service mesh também pode centralizar políticas de saída.

Exemplo em uma plataforma Pix: encapsular conexão mTLS, retry seguro, deadline e métricas para um provedor externo, sem espalhar detalhes de rede pelo domínio financeiro.

## 12.4 Adapter Pattern

Adapter é um padrão de código ou componente que converte uma interface para o contrato esperado pelo domínio.

```text
Domínio Pix
   ↓ interface interna
ProviderAdapter
   ↓
REST, SOAP, SDK ou protocolo do parceiro
```

Pode converter:

- modelos de dados;
- nomes e semânticas de status;
- REST para SOAP;
- SDK proprietário para interface interna;
- formatos de erros.

Webhook e WebSocket não são, por si só, Adapter Pattern. Um adapter pode ser usado para receber ou enviar esses protocolos e convertê-los para eventos/comandos internos.

O adapter deve proteger o domínio contra mudanças do provedor:

- status externo vira enum interno controlado;
- erro externo vira erro de domínio;
- payload externo vira comando ou evento interno;
- diferenças de contrato ficam isoladas em uma fronteira testável.

## 12.5 Entrada e saída da aplicação

```text
Entrada externa:
PSP → webhook → WAF/Gateway/Ingress → Webhook Adapter → domínio

Saída externa:
domínio → Provider Adapter → Ambassador/Egress opcional → PSP
```

## 12.6 Pergunta prática

Se a mudança é de contrato de negócio, provavelmente é Adapter. Se a mudança é de transporte, segurança de conexão, telemetria ou política de tráfego, provavelmente é Ambassador, gateway, ingress, egress ou service mesh.

---

← [11. Escalabilidade e alta disponibilidade](11-escalabilidade-e-alta-disponibilidade.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [13. Webhooks e WebSockets](13-webhooks-e-websockets.md) →
