← [12. Sidecar, service mesh, Ambassador e Adapter](12-sidecar-service-mesh-ambassador-e-adapter.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [14. Segurança](14-seguranca.md) →

# 13. Webhooks e WebSockets

## 13.1 Webhook

Webhook é uma chamada HTTP disparada quando um evento ocorre.

Ele pode ser:

- **de entrada**: um PSP chama sua aplicação;
- **de saída**: sua plataforma notifica um cliente.

```text
Sistema emissor
     ↓ HTTP POST
Endpoint do receptor
```

Webhooks podem:

- chegar duplicados;
- atrasar;
- chegar fora de ordem;
- sofrer retry;
- falhar após o receptor processar, mas antes de responder.

O receptor deve:

- validar assinatura, certificado ou credencial;
- validar timestamp e proteção contra replay;
- deduplicar por `eventId`;
- processar idempotentemente;
- responder rapidamente com `2xx` após aceitar de forma segura;
- mover processamento pesado para fila quando adequado.

## 13.2 WebSocket

WebSocket mantém uma conexão persistente e bidirecional:

```text
Cliente ⇄ Servidor
```

É útil para:

- atualização de status em tempo real;
- notificações na interface;
- dashboards;
- chats e colaboração.

Exemplo combinado:

```text
PSP confirma pagamento
      ↓ webhook
Backend atualiza estado
      ↓ evento
Gateway WebSocket
      ↓
Tela recebe atualização
```

---

---

← [12. Sidecar, service mesh, Ambassador e Adapter](12-sidecar-service-mesh-ambassador-e-adapter.md) · [Início](../README.md) · [Sumário](../SUMMARY.md) · [14. Segurança](14-seguranca.md) →
