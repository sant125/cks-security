# Kubelet — Portas e Segurança

O kubelet expõe duas portas HTTP. Entender a diferença é crítico para o CKS.

---

## As duas portas

```
┌─────────────────────────────────────────────────────────────┐
│                         KUBELET                             │
│                                                             │
│  :10250  — porta principal (HTTPS)                          │
│    ├── autenticação habilitada                              │
│    ├── autorização habilitada                               │
│    ├── exec, logs, port-forward, metrics                   │
│    └── usada pelo apiserver para se comunicar              │
│                                                             │
│  :10255  — porta read-only (HTTP) ← DEPRECATED/DESABILITADA│
│    ├── sem autenticação                                     │
│    ├── sem autorização                                      │
│    ├── só leitura: /metrics, /pods, /healthz               │
│    └── qualquer processo na rede acessa                     │
└─────────────────────────────────────────────────────────────┘
```

---

## Porta 10250 — Configuração segura

```yaml
# /var/lib/kubelet/config.yaml
authentication:
  anonymous:
    enabled: false          # bloqueia requests sem credencial
  webhook:
    enabled: true           # delega autorização ao apiserver
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt  # aceita certs assinados pela CA

authorization:
  mode: Webhook             # apiserver decide o que cada client pode fazer
```

**Com essa config:**
- Só o apiserver (com `apiserver-kubelet-client.crt`) consegue fazer exec/logs
- Qualquer outro request sem cert válido → 401

---

## Porta 10255 — O problema

Read-only port sem autenticação. Se aberta:

```bash
# Qualquer processo na rede consegue
curl http://NODE_IP:10255/pods          # lista todos os pods do node
curl http://NODE_IP:10255/metrics       # métricas do kubelet
curl http://NODE_IP:10255/healthz       # health check
```

**Não é possível fazer exec ou escrever via 10255 — mas vaza informações sobre pods, namespaces, imagens em uso, etc.**

Para desabilitar:
```yaml
# /var/lib/kubelet/config.yaml
readOnlyPort: 0   # 0 = desabilitado
```

---

## Anonymous auth habilitado — o pior cenário

Se `authentication.anonymous.enabled: true` na porta 10250:

```bash
# Exec em qualquer pod sem autenticação nenhuma
curl -sk https://NODE_IP:10250/run/default/meu-pod/meu-container \
  -d "cmd=id"
# root
```

Isso dá execução de comandos arbitrários em qualquer container do node, sem precisar passar pelo apiserver ou RBAC.

**Esse é o tipo de misconfiguration que o CKS cobra explicitamente.**

---

## Fluxo de uma chamada legítima (kubectl exec)

```
kubectl exec meu-pod -- /bin/bash
        │
        ▼
apiserver recebe o request
        │  valida RBAC do usuário (pode fazer exec nesse pod?)
        ▼
apiserver abre conexão para kubelet :10250
        │  usando apiserver-kubelet-client.crt
        ▼
kubelet verifica: esse cert é da CA do cluster?
        │  chama webhook de autorização no apiserver
        │  "esse client pode fazer exec no pod X?"
        ▼
kubelet executa no container via CRI (containerd)
        │
        ▼
stream bidirecional kubectl ↔ apiserver ↔ kubelet ↔ container
```

---

## Verificar configuração atual

```bash
# Ver config do kubelet
cat /var/lib/kubelet/config.yaml | grep -A5 -E "authentication|authorization|readOnly"

# Ver se porta 10255 está aberta
ss -tlnp | grep 10255

# Testar se anonymous está habilitado (deve retornar 401, não 200)
curl -sk https://NODE_IP:10250/pods
```

---

## Resumo CKS

| Item | Seguro | Inseguro |
|---|---|---|
| `authentication.anonymous.enabled` | `false` | `true` |
| `authorization.mode` | `Webhook` | `AlwaysAllow` |
| `readOnlyPort` | `0` | `10255` |
| Acesso à porta 10250 | Só apiserver (via firewall/NetworkPolicy) | Aberto para qualquer IP |
