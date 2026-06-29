← [10. Cache, rate limit e backpressure](10-cache-rate-limit-e-backpressure.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [12. Sidecar, service mesh, Ambassador e Adapter](12-sidecar-service-mesh-ambassador-e-adapter.md) →

# 11. Escalabilidade e alta disponibilidade

Escalar é aumentar capacidade. Ter alta disponibilidade é continuar operando apesar de falhas. Um sistema pode escalar bastante e ainda cair mal se não tiver isolamento, failover e operação testada.

## 11.1 Escalabilidade horizontal

Consiste em adicionar instâncias:

```text
Cliente
   ↓
Load Balancer
   ├── App 1
   ├── App 2
   └── App 3
```

Funciona melhor quando as aplicações são stateless.

Escalabilidade horizontal exige que estado crítico esteja fora da instância ou seja reconstruível. Caso contrário, adicionar instâncias aumenta capacidade, mas também aumenta inconsistência e dificuldade operacional.

## 11.2 Stateless e stateful

### Stateless

A instância não mantém estado crítico da sessão ou transação apenas na memória local.

O estado fica em:

- banco de dados;
- Redis quando apropriado;
- broker;
- object storage;
- serviços especializados.

Qualquer instância pode atender a próxima requisição.

### Stateful

A aplicação mantém estado necessário na própria instância. Isso cria afinidade e dificulta:

- autoscaling;
- deploy;
- recuperação após falha;
- redistribuição de tráfego.

## 11.3 Sticky session

Sticky session no ALB tenta enviar o mesmo cliente à mesma instância.

Pode ser útil para sistemas legados, mas:

- não protege o estado se a instância cair;
- pode desequilibrar carga;
- dificulta autoscaling;
- aumenta acoplamento.

Para pagamentos, o estado crítico deve ser persistido fora da instância.

## 11.4 Load balancing e failover

```text
Load balancing
→ distribui tráfego entre instâncias saudáveis.

Failover
→ substitui um componente principal indisponível por outro saudável.
```

No banco, failover normalmente promove uma **réplica/standby**, não um arquivo de backup.

```text
Réplica
→ banco ativo que recebe replicação e pode ser promovido.

Backup
→ cópia usada para restauração, recuperação histórica ou desastre.
```

Defina também:

- **RTO**: tempo máximo aceitável para recuperar o serviço;
- **RPO**: perda máxima aceitável de dados;
- plano de failover por dependência crítica;
- teste periódico de restauração de backup;
- procedimento de retorno ao estado normal.

## 11.5 Split-brain

Split-brain ocorre quando duas instâncias acreditam ser líderes e aceitam escritas.

Mitigações:

- quórum;
- eleição de líder;
- fencing;
- leases;
- epochs/terms;
- proteção contra escrita no antigo primário.

Em sistemas financeiros, split-brain é especialmente grave porque pode permitir escritas conflitantes. A arquitetura deve preferir indisponibilidade controlada a aceitar duas fontes primárias independentes para o mesmo saldo.

## 11.6 Probes no Kubernetes

### Startup probe

Verifica se a aplicação terminou de iniciar. Enquanto não passa, evita que liveness e readiness prejudiquem aplicações com inicialização lenta.

### Liveness probe

Responde:

> O processo está vivo e consegue continuar operando?

Falhas repetidas podem causar reinício do container.

Não coloque indisponibilidades transitórias de banco ou API externa na liveness, pois isso pode criar loops de reinício e piorar o incidente.

### Readiness probe

Responde:

> Esta instância está pronta para receber tráfego agora?

Quando falha, o pod é removido dos endpoints prontos, sem necessariamente ser reiniciado.

Dependências críticas podem influenciar readiness, mas o desenho deve evitar retirar todas as réplicas simultaneamente por causa da mesma dependência externa.

Uma boa prática é separar endpoints:

```text
/live
→ processo vivo, sem testar dependências externas pesadas.

/ready
→ instância pronta para receber tráfego.

/health/deep
→ diagnóstico operacional mais completo, fora do caminho de balanceamento.
```

## 11.7 Circuit breaker, timeout e retry

- **timeout/deadline** limita quanto tempo a chamada pode ocupar recursos;
- **retry** tenta novamente falhas transitórias e seguras;
- **circuit breaker** reduz chamadas para uma dependência degradada;
- **bulkhead** isola pools e recursos;
- **fallback** fornece uma resposta alternativa quando possível.

Retry sem timeout, sem backoff ou sem idempotência pode amplificar uma falha.

## 11.8 Alta disponibilidade por camada

| Camada | Cuidados |
|---|---|
| Aplicação | múltiplas instâncias, deploy gradual, readiness correta |
| Banco | réplica, backup, failover testado, migração segura |
| Broker | partições/filas monitoradas, DLQ, retenção e redrive |
| Cache | tolerância à perda, proteção contra stampede |
| Integração externa | timeout, circuit breaker, contingência e reconciliação |
| Operação | alertas, runbooks, plantão e exercícios de falha |

---

← [10. Cache, rate limit e backpressure](10-cache-rate-limit-e-backpressure.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [12. Sidecar, service mesh, Ambassador e Adapter](12-sidecar-service-mesh-ambassador-e-adapter.md) →
