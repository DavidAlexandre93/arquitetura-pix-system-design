← [1. Princípios arquiteturais](01-principios-arquiteturais.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [3. Arquitetura de referência](03-arquitetura-de-referencia.md) →

# 2. Componentes do ecossistema Pix

Este capítulo separa os componentes externos do Pix das escolhas internas da plataforma. Essa separação evita explicar o Pix como se ele fosse apenas uma fila, uma API REST ou uma arquitetura de microsserviços.

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

Cuidados de modelagem:

- consultas ao DICT podem envolver regras de segurança, privacidade, limitação e auditoria;
- dados obtidos do DICT não devem ser tratados como cadastro permanente sem regra clara de atualização;
- uma plataforma integradora pode acessar o DICT indiretamente, por meio de outro participante ou provedor.

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

Ao documentar a arquitetura, deixe explícito qual é o papel da instituição:

- participante direto;
- participante indireto;
- provedor de serviço tecnológico;
- plataforma integradora que consome API de um PSP;
- aplicação cliente que apenas inicia ou acompanha pagamentos.

Essa informação muda responsabilidades, certificados, integrações, observabilidade, conciliação e plano de contingência.

## 2.3 MED

**MED** significa **Mecanismo Especial de Devolução**.

É um mecanismo específico do Pix para apoiar a tentativa de recuperação de valores em situações previstas na regulamentação, como fraude, golpe e coerção, além de determinados casos de falha operacional.

Pontos importantes:

- não é equivalente ao chargeback de cartão;
- não garante recuperação integral;
- depende da análise do caso e da disponibilidade de recursos;
- não é o mecanismo adequado para qualquer desacordo comercial ou erro voluntário de envio;
- as regras são atualizadas pelo Banco Central e devem ser consultadas no regulamento vigente.

Desde **1º de outubro de 2025**, o autoatendimento do MED passou a ser obrigatório no ambiente Pix dos aplicativos dos participantes, permitindo que usuários pessoa física contestem transações elegíveis sem depender inicialmente de atendimento humano.

O MED também deve ser entendido como processo operacional e regulatório, não como simples endpoint. Ele envolve análise, notificações, possível bloqueio, prazos, evidências, marcações de fraude e, quando aplicável, devolução total ou parcial.

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

## 2.4 Como explicar sem misturar conceitos

Formulação segura:

> DICT localiza dados associados a uma chave Pix. SPI liquida transferências entre participantes. MED define procedimentos de devolução em hipóteses específicas. A arquitetura interna da instituição pode usar APIs, bancos, filas, Outbox, cache, streams e conciliação para cumprir essas responsabilidades com segurança.

Evite dizer:

- "Pix é Kafka";
- "SPI é uma fila";
- "MED é chargeback";
- "todo Pix precisa usar microsserviços";
- "idempotência é garantida só por `@Transactional`".

---

← [1. Princípios arquiteturais](01-principios-arquiteturais.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [3. Arquitetura de referência](03-arquitetura-de-referencia.md) →
