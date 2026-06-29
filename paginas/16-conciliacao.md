← [15. Observabilidade, operação e documentação](15-observabilidade-operacao-e-documentacao.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [17. Resumo para entrevista](17-resumo-para-entrevista.md) →

# 16. Conciliação

Conciliação é o mecanismo que procura divergências entre fontes independentes e transforma incerteza em investigação, correção ou confirmação.

Conciliação compara fontes independentes para detectar divergências.

```text
Ledger interno
Banco operacional
Relatório ou fonte externa
          ↓
Motor de conciliação
```

Chaves e atributos possíveis:

- `endToEndId`;
- identificador interno;
- valor;
- status;
- data e hora;
- participante pagador;
- participante recebedor.

Divergências:

- transação ausente;
- duplicidade;
- valor divergente;
- status divergente;
- registro apenas no interno;
- registro apenas no externo.

Correções devem ser:

- idempotentes;
- auditáveis;
- aprovadas conforme governança;
- feitas por contralançamentos ou novas operações quando houver impacto financeiro;
- monitoradas até o fechamento.

Conciliação é uma defesa adicional e não substitui integridade transacional.

## 16.1 Tipos de conciliação

- **online ou quase em tempo real**: acompanha estados recentes e reduz tempo em `UNKNOWN`;
- **batch diária**: compara movimentos do dia contra fontes internas e externas;
- **contábil**: valida se lançamentos financeiros fecham;
- **operacional**: procura filas paradas, webhooks não enviados, eventos não publicados e estados pendentes.

## 16.2 Fluxo recomendado

```text
coletar fontes
      ↓
normalizar campos
      ↓
comparar por chaves
      ↓
classificar divergência
      ↓
abrir pendência
      ↓
corrigir com trilha auditável
      ↓
validar fechamento
```

## 16.3 Critérios de qualidade

- toda divergência tem dono e prazo;
- correção financeira usa contralançamento ou nova operação formal;
- correção técnica deixa trilha;
- divergências recorrentes geram ação preventiva;
- reconciliação possui métricas e alertas;
- reprocessamento é idempotente.

---

← [15. Observabilidade, operação e documentação](15-observabilidade-operacao-e-documentacao.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [17. Resumo para entrevista](17-resumo-para-entrevista.md) →
