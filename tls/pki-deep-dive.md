# PKI do Kubernetes — Deep Dive

## Por que existem SANs (aliases)?

O TLS valida não só "esse certificado é legítimo?" mas "esse certificado é válido para **esse endereço**?". O apiserver é acessado de vários lugares diferentes simultaneamente:

```
kubectl externo       → api.meucluster.com
kubelet no node       → 10.0.0.10 (IP privado do control plane)
pod dentro do cluster → kubernetes.default.svc.cluster.local
                      → kubernetes.default.svc
                      → kubernetes.default
                      → kubernetes
                      → 10.96.0.1 (ClusterIP do service kubernetes)
```

Por isso o cert do apiserver tem todos como SANs:

```bash
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep -A20 "Subject Alternative"
# DNS:kubernetes
# DNS:kubernetes.default
# DNS:kubernetes.default.svc
# DNS:kubernetes.default.svc.cluster.local
# DNS:meu-control-plane
# IP:10.96.0.1
# IP:10.0.0.10
```

Adicionar IP/DNS externo:
```bash
kubeadm init --apiserver-cert-extra-sans=api.meucluster.com,1.2.3.4
```

---

## Grupos — campo O (Organization) do certificado

O apiserver extrai identidade diretamente do certificado X.509:
- **CN (Common Name)** → username
- **O (Organization)** → grupos RBAC

| Kubeconfig | CN | O (grupo) | Efeito |
|---|---|---|---|
| `admin.conf` | `kubernetes-admin` | `kubeadm:cluster-admins` | Bound ao `cluster-admin` ClusterRole |
| `super-admin.conf` | `kubernetes-super-admin` | `system:masters` | Bypassa RBAC completamente |
| `kubelet.conf` (cada node) | `system:node:<nome>` | `system:nodes` | Acesso restrito ao próprio node |
| `controller-manager.conf` | `system:kube-controller-manager` | — | Role própria |
| `scheduler.conf` | `system:kube-scheduler` | — | Role própria |

`system:masters` é especial — não é verificado pelo RBAC, bypassa tudo. Use só em emergências.

---

## Como o kubeadm automatiza o PKI

```
kubeadm init
└── phase certs
    ├── ca                        → ca.crt + ca.key (CA do cluster)
    ├── apiserver                 → apiserver.crt (com todos os SANs)
    ├── apiserver-kubelet-client  → cert para apiserver autenticar no kubelet
    ├── apiserver-etcd-client     → cert para apiserver autenticar no etcd
    ├── etcd/ca                   → CA separada do etcd
    ├── etcd/server               → cert server do etcd
    └── etcd/peer                 → cert entre membros do cluster etcd

└── phase kubeconfig
    ├── admin                     → admin.conf
    ├── super-admin               → super-admin.conf
    ├── kubelet                   → kubelet.conf do control plane node
    ├── controller-manager        → controller-manager.conf
    └── scheduler                 → scheduler.conf
```

---

## TLS Bootstrap — como workers recebem seus certs

O node worker não tem a CA key. O fluxo de auto-registro:

```
1. kubeadm join --token <token> --discovery-token-ca-cert-hash sha256:<hash>

2. Node usa o token para autenticação inicial (acesso mínimo)

3. Kubelet gera par de chaves localmente

4. Kubelet envia CSR ao apiserver
   CN = system:node:<nome-do-node>
   O  = system:nodes

5. Controller-manager aprova o CSR automaticamente

6. Apiserver assina com a CA do cluster e devolve o cert

7. Kubelet salva e usa para todas as comunicações
```

O `kubelet.conf` gerado no node aponta para o cert:

```yaml
users:
- name: default-auth
  user:
    client-certificate: /var/lib/kubelet/pki/kubelet-client-current.pem
    client-key: /var/lib/kubelet/pki/kubelet-client-current.pem
```

O symlink `kubelet-client-current.pem` é atualizado automaticamente na renovação — sem restart.

---

## Server cert do kubelet — um por node

Cada node tem seu próprio server cert com o SAN do seu IP. O apiserver conecta diretamente no kubelet do node para `logs`, `exec`, `port-forward`.

```
node-1: kubelet.crt  CN=system:node:node-1  SAN=IP do node-1
node-2: kubelet.crt  CN=system:node:node-2  SAN=IP do node-2
```

**Gap de segurança:** por padrão o kubeadm gera um self-signed cert para o server do kubelet — o apiserver não verifica.

**Hardening (CKS):** habilitar `serverTLSBootstrap: true` no kubelet para que o server cert também seja assinado pela CA do cluster:

```yaml
# /var/lib/kubelet/config.yaml
serverTLSBootstrap: true
```

O CSR gerado precisa ser aprovado manualmente ou via controller:
```bash
kubectl get csr
kubectl certificate approve <csr-name>
```

---

## Referências

- [PKI certificates and requirements — kubernetes.io](https://kubernetes.io/docs/setup/best-practices/certificates/)
- [kubeadm implementation details — kubernetes.io](https://kubernetes.io/docs/reference/setup-tools/kubeadm/implementation-details/)
- [Authenticating — kubernetes.io](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [TLS bootstrapping — kubernetes.io](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/)
