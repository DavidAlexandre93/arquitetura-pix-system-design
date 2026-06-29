← [14. Segurança](14-seguranca.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [16. Conciliação](16-conciliacao.md) →

# 15. Observabilidade, operação e documentação

Um sistema financeiro não termina quando o endpoint responde. Ele precisa ser observado, operado, auditado e melhorado continuamente.

## 15.1 Logs, métricas e traces

### Logs

Devem ser estruturados e correlacionáveis:

```text
paymentId=pix-123
endToEndId=E...
status=UNKNOWN
error=UPSTREAM_TIMEOUT
correlationId=abc-789
traceId=def-456
```

Não registre dados pessoais ou segredos sem necessidade e proteção adequada.

### Métricas

Exemplos:

- taxa de sucesso e rejeição;
- latência p50, p95 e p99;
- pagamentos em estado `UNKNOWN`;
- timeouts por dependência;
- taxa de duplicidade bloqueada;
- consumer lag;
- tamanho e idade da DLQ;
- tempo de processamento de webhook;
- divergências de conciliação;
- saturação de pools e conexões.

### Traces

```text
Gateway
  ↓
Orquestrador
  ↓
Antifraude
  ↓
Ledger
  ↓
Adapter externo
```

Propague `traceId`, `correlationId`, `paymentId` e identificadores de negócio sem transformar dados sensíveis em tags de alta cardinalidade indiscriminadamente.

Uma boa investigação deve conseguir responder:

- qual requisição iniciou a operação;
- qual usuário, cliente ou credencial estava envolvido;
- qual estado foi persistido;
- qual chamada externa foi feita;
- se houve retry;
- se evento foi publicado;
- se a conciliação confirmou ou apontou divergência.

## 15.2 SLI, SLO e SLA

- **SLI**: indicador medido, como taxa de sucesso ou latência p99;
- **SLO**: objetivo interno para o indicador;
- **SLA**: compromisso contratual com consequências definidas.

Exemplo:

```text
SLI: percentual de solicitações válidas processadas com sucesso
SLO: 99,95% por mês
SLA: compromisso contratual de 99,9%
```

SLIs úteis para Pix:

- taxa de sucesso por tipo de operação;
- latência p95/p99 por endpoint;
- tempo em `PROCESSING`;
- quantidade e idade de pagamentos `UNKNOWN`;
- atraso de publicação da Outbox;
- consumer lag;
- idade máxima de mensagem em DLQ;
- divergências abertas na conciliação.

Alertas devem ter ação clara. Alerta sem runbook vira ruído.

## 15.3 Runbook

Documento operacional usado durante incidentes ou tarefas recorrentes:

- sintomas;
- diagnóstico;
- mitigação;
- comandos seguros;
- critérios de escalonamento;
- validação da recuperação;
- rollback ou contingência.

Modelo simples:

```text
Sintoma:
Impacto:
Métricas e dashboards:
Hipóteses prováveis:
Passos de diagnóstico:
Mitigação segura:
Critério de escalonamento:
Critério de recuperação:
Comunicação:
```

## 15.4 Postmortem

Documento produzido após um incidente:

- resumo e impacto;
- linha do tempo;
- causa técnica;
- fatores contribuintes;
- detecção e resposta;
- o que funcionou e o que falhou;
- ações corretivas com responsáveis e prazos.

Deve ser **blameless**: analisa condições do sistema e do processo, não procura um culpado individual.

Boas ações corretivas são específicas:

- o que será feito;
- por quem;
- até quando;
- como será validado;
- qual risco será reduzido.

## 15.5 ADR

**Architecture Decision Record** registra:

- contexto;
- decisão;
- alternativas;
- consequências;
- status;
- data e responsáveis.

Exemplo:

```text
ADR-007 — Adotar Transactional Outbox para eventos de pagamento
```

Modelo enxuto:

```text
Título:
Status:
Contexto:
Decisão:
Alternativas consideradas:
Consequências:
Data:
```

## 15.6 C4 Model

Níveis:

1. **Context**: sistema, usuários e sistemas externos;
2. **Container**: aplicações, bancos, brokers e frontends;
3. **Component**: componentes internos de uma aplicação;
4. **Code**: classes ou detalhes de implementação, quando necessário.

No C4, “container” não significa apenas Docker; é uma unidade executável ou armazenadora relevante para a arquitetura.

Para esta documentação, os níveis mais úteis são Context e Container. Component pode ser usado para detalhar o orquestrador de pagamentos, ledger, adapter, outbox relay e conciliação.

## 15.7 OKR

- **Objective**: resultado qualitativo desejado;
- **Key Results**: resultados mensuráveis que comprovam o avanço.

Exemplo:

```text
Objective:
Aumentar a confiabilidade da plataforma de pagamentos.

Key Results:
- reduzir pagamentos em estado UNKNOWN em 50%;
- reduzir MTTR de 40 para 20 minutos;
- manter sucesso acima de 99,95%;
- reduzir p99 do endpoint de consulta em 30%.
```

Implementar uma tecnologia é uma iniciativa, não necessariamente um Key Result.

## 15.8 Relação entre os artefatos

```text
OKR
→ define os resultados desejados.

ADR
→ registra decisões técnicas.

C4 Model
→ representa a arquitetura.

Runbook
→ orienta a operação e resposta.

Postmortem
→ gera aprendizado após incidentes.
```

## 15.9 Documentação mínima de produção

- visão C4 de contexto e containers;
- contratos de API e webhooks;
- estados e transições;
- políticas de idempotência;
- decisão de consistência por dado;
- runbooks dos fluxos críticos;
- ADRs das decisões relevantes;
- checklist de segurança e operação;
- procedimento de conciliação;
- plano de testes de recuperação.

---

← [14. Segurança](14-seguranca.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [16. Conciliação](16-conciliacao.md) →
