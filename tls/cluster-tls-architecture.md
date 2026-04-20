# TLS Architecture no Kubernetes

Todo componente do cluster tem dois papéis simultâneos — **server** (recebe conexões) e **client** (faz conexões). Por isso cada um tem pelo menos um par de certificados.

```
                        ┌─────────────────┐
                        │  kube-apiserver │
                        │  server cert    │ ← kubectl, kubelet, controller, scheduler
                        │  client cert    │ → etcd
                        └────────┬────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                   │
     ┌────────▼───────┐  ┌───────▼──────┐  ┌────────▼───────┐
     │     etcd       │  │   kubelet    │  │  controller +  │
     │  server cert   │  │ server cert  │  │  scheduler     │
     │  client cert   │  │ client cert  │  │  client cert   │
     └────────────────┘  └──────────────┘  └────────────────┘
```

---

## Componentes e seus certificados

### kube-apiserver

O hub central — todo mundo fala com ele.

| Certificado | Path típico | Função |
|---|---|---|
| server cert | `apiserver.crt` | Prova identidade para kubectl, kubelet, controller, scheduler |
| client cert para etcd | `apiserver-etcd-client.crt` | Autentica no etcd |
| client cert para kubelet | `apiserver-kubelet-client.crt` | Autentica no kubelet para logs/exec |

**Use case:** `kubectl exec` — kubectl fala com o apiserver via TLS, o apiserver abre conexão de volta para o kubelet do node usando `apiserver-kubelet-client.crt`.

---

### etcd

Aceita apenas conexões TLS — do apiserver e entre os membros do próprio cluster etcd.

| Certificado | Função |
|---|---|
| server cert | Aceitar conexões do apiserver |
| peer cert | Comunicação entre membros do cluster etcd |

**Use case de risco:** etcd exposto sem TLS na porta 2379 — qualquer processo com acesso lê todos os Secrets do cluster em texto puro.

---

### kubelet

Roda em cada node. Tem papel duplo.

| Certificado | Função |
|---|---|
| server cert | Apiserver conecta para `logs`, `exec`, `port-forward` |
| client cert | Kubelet autentica no apiserver para registrar o node e reportar status |

**Fluxo do `kubectl logs meu-pod`:**
```
kubectl → apiserver        (TLS: apiserver.crt)
apiserver → kubelet        (TLS: apiserver-kubelet-client.crt)
kubelet → responde com os logs
```

---

### controller-manager e scheduler

Apenas clientes — falam com o apiserver, não servem conexões externas.

| Certificado | Função |
|---|---|
| client cert | Autenticar no apiserver para suas operações |

---

## Hierarquia de CAs

```
CA do cluster (ca.crt / ca.key)
├── apiserver.crt
├── apiserver-etcd-client.crt
├── apiserver-kubelet-client.crt
├── kubelet.crt (um por node)
├── controller-manager.crt
└── scheduler.crt

CA do etcd (etcd/ca.crt) — domínio separado
├── etcd-server.crt
└── etcd-peer.crt
```

O etcd usa CA própria por isolamento de domínio de confiança. Se a CA do cluster for comprometida, o etcd continua protegido.

---

## Cenários de falha / ataque

| Cenário | Impacto |
|---|---|
| etcd exposto sem TLS | Leitura de todos os Secrets em texto puro |
| kubelet porta 10250 sem autenticação | Exec em qualquer pod do node sem passar pelo apiserver |
| CA do cluster comprometida | Atacante gera certificados válidos para qualquer componente |
| Certificado expirado no apiserver | Cluster inoperável — kubectl para de funcionar |
| Certificado do kubelet expirado | Node fica NotReady — pods não são agendados no node |

---

## Onde ficam os certificados (kubeadm)

```
/etc/kubernetes/pki/
├── ca.crt / ca.key                        # CA do cluster
├── apiserver.crt / apiserver.key
├── apiserver-etcd-client.crt / .key
├── apiserver-kubelet-client.crt / .key
├── front-proxy-ca.crt / ca.key
├── front-proxy-client.crt / .key
└── etcd/
    ├── ca.crt / ca.key                    # CA do etcd
    ├── server.crt / server.key
    ├── peer.crt / peer.key
    └── healthcheck-client.crt / .key
```

---

## Comandos úteis CKS

```bash
# Ver detalhes de um certificado
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep -E 'Subject|Issuer|Not After|DNS|IP'

# Verificar expiração de todos os certificados
kubeadm certs check-expiration

# Renovar todos os certificados
kubeadm certs renew all

# Ver qual CA assinou um certificado
openssl verify -CAfile /etc/kubernetes/pki/ca.crt /etc/kubernetes/pki/apiserver.crt
```
