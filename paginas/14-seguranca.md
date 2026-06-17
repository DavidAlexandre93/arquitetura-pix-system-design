← [13. Webhooks e WebSockets](13-webhooks-e-websockets.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [15. Observabilidade, operação e documentação](15-observabilidade-operacao-e-documentacao.md) →

# 14. Segurança

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

---

---

← [13. Webhooks e WebSockets](13-webhooks-e-websockets.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [15. Observabilidade, operação e documentação](15-observabilidade-operacao-e-documentacao.md) →
