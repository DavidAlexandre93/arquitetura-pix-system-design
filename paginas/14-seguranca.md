← [13. Webhooks e WebSockets](13-webhooks-e-websockets.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [15. Observabilidade, operação e documentação](15-observabilidade-operacao-e-documentacao.md) →

# 14. Segurança

Segurança em pagamentos precisa cobrir identidade, autorização, criptografia, fraude, privacidade, trilha de auditoria e resposta a incidente.

## 14.1 Controles principais

- autenticação forte;
- autorização por escopo e função;
- mTLS em integrações sensíveis;
- criptografia em trânsito e em repouso;
- gestão e rotação de segredos;
- proteção contra replay;
- assinatura de mensagens/webhooks;
- rate limiting e WAF;
- trilha de auditoria;
- segregação de funções;
- princípio do menor privilégio.

Também considere:

- validação de dispositivo e sessão quando aplicável;
- análise de risco transacional;
- detecção de comportamento anômalo;
- proteção de dados pessoais em logs, eventos e analytics;
- revisão periódica de permissões;
- segregação entre ambientes;
- resposta a incidentes de segurança.

## 14.2 Certificados, TLS e mTLS

No TLS tradicional, o cliente normalmente valida o certificado do servidor.

No mTLS, os dois lados apresentam certificados:

```text
Cliente valida servidor
Servidor valida cliente
```

A operação deve prever:

- inventário;
- renovação;
- alertas de expiração;
- revogação;
- certificados de contingência;
- testes de rotação.

Certificado expirado é incidente previsível. A documentação operacional deve incluir dono, data de expiração, janela de renovação, processo de troca, rollback e alerta antecipado.

## 14.3 HSM

HSM (**Hardware Security Module**) protege material criptográfico e executa operações sem expor a chave privada.

```text
Aplicação solicita assinatura
           ↓
          HSM
           ↓
HSM usa a chave internamente
           ↓
Retorna a assinatura
```

Usos comuns:

- geração e proteção de chaves;
- assinatura digital;
- criptografia;
- atendimento a requisitos de segurança e auditoria.

## 14.4 Proteção contra replay

Um atacante não deve conseguir capturar uma requisição válida e reenviá-la depois.

Controles comuns:

- nonce;
- timestamp com janela curta;
- assinatura incluindo corpo, caminho e timestamp;
- chave de idempotência;
- validação de certificado ou credencial;
- armazenamento temporário de identificadores já vistos.

## 14.5 Dados sensíveis

Não registre em log mais dados do que o necessário. Para observabilidade, prefira identificadores técnicos e mascaramento.

Exemplos de cuidado:

- não vazar documento, chave Pix, token, segredo ou payload completo sem necessidade;
- controlar acesso a trilhas de auditoria;
- definir retenção;
- proteger dados em ambientes de teste;
- evitar dados pessoais em labels de métricas de alta cardinalidade.

---

← [13. Webhooks e WebSockets](13-webhooks-e-websockets.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [15. Observabilidade, operação e documentação](15-observabilidade-operacao-e-documentacao.md) →
