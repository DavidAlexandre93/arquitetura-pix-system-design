← [14. Segurança](14-seguranca.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [16. Conciliação](16-conciliacao.md) →

# 15. Observabilidade, operação e documentação

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

## 15.3 Runbook

Documento operacional usado durante incidentes ou tarefas recorrentes:

- sintomas;
- diagnóstico;
- mitigação;
- comandos seguros;
- critérios de escalonamento;
- validação da recuperação;
- rollback ou contingência.

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

## 15.6 C4 Model

Níveis:

1. **Context**: sistema, usuários e sistemas externos;
2. **Container**: aplicações, bancos, brokers e frontends;
3. **Component**: componentes internos de uma aplicação;
4. **Code**: classes ou detalhes de implementação, quando necessário.

No C4, “container” não significa apenas Docker; é uma unidade executável ou armazenadora relevante para a arquitetura.

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

---

---

← [14. Segurança](14-seguranca.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [16. Conciliação](16-conciliacao.md) →
