← [15. Observabilidade, operação e documentação](15-observabilidade-operacao-e-documentacao.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [17. Resumo para entrevista](17-resumo-para-entrevista.md) →

# 16. Conciliação

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

---

---

← [15. Observabilidade, operação e documentação](15-observabilidade-operacao-e-documentacao.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [17. Resumo para entrevista](17-resumo-para-entrevista.md) →
