← [1. Princípios arquiteturais](01-principios-arquiteturais.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [3. Arquitetura de referência](03-arquitetura-de-referencia.md) →

# 2. Componentes do ecossistema Pix

## 2.1 DICT

**DICT** significa **Diretório de Identificadores de Contas Transacionais**.

Ele é gerido e operado pelo Banco Central e permite associar uma chave Pix aos dados necessários para iniciar o pagamento, conforme as regras de acesso e exposição de dados do arranjo.

Fluxo conceitual:

```text
Chave Pix
   ↓
DICT
   ↓
Dados da conta e do participante recebedor
```

Definição para entrevista:

> O DICT é o diretório autoritativo de chaves Pix, utilizado para localizar os dados da conta transacional vinculada à chave e mitigar riscos na iniciação do pagamento.

## 2.2 SPI

**SPI** significa **Sistema de Pagamentos Instantâneos**.

É a infraestrutura operada pelo Banco Central que processa e liquida transferências Pix entre participantes.

```text
DICT
→ resolve a chave e identifica o destino.

SPI
→ processa a liquidação entre os participantes.
```

Uma fintech ou plataforma pode integrar-se ao ecossistema por meio de um participante direto, participante indireto, provedor ou parceiro. Portanto, nem toda aplicação empresarial chama DICT e SPI diretamente.

## 2.3 MED

**MED** significa **Mecanismo Especial de Devolução**.

É um mecanismo específico do Pix para apoiar a tentativa de recuperação de valores em situações previstas na regulamentação, como fraude, golpe e coerção, além de determinados casos de falha operacional.

Pontos importantes:

- não é equivalente ao chargeback de cartão;
- não garante recuperação integral;
- depende da análise do caso e da disponibilidade de recursos;
- não é o mecanismo adequado para qualquer desacordo comercial ou erro voluntário de envio;
- as regras são atualizadas pelo Banco Central e devem ser consultadas no regulamento vigente.

Desde **1º de outubro de 2025**, os participantes passaram a disponibilizar o autoatendimento para contestação de transações elegíveis ao MED em seus aplicativos.

Fluxo conceitual:

```text
Usuário contesta a transação
          ↓
Instituição inicia o procedimento
          ↓
Participantes envolvidos são notificados
          ↓
Recursos elegíveis podem ser bloqueados
          ↓
Análise do caso
      ┌───┴────┐
      │        │
Procedente   Improcedente
      │        │
Devolução   Desbloqueio
```

---

---

← [1. Princípios arquiteturais](01-principios-arquiteturais.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [3. Arquitetura de referência](03-arquitetura-de-referencia.md) →
