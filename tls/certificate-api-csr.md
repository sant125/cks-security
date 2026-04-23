# Certificate API e CSR no Kubernetes

O Kubernetes tem uma API nativa para emitir certificados — útil para novos usuários, nodes e componentes sem precisar acessar a CA diretamente.

---

## Fluxo completo

```
1. Gerar chave privada + CSR localmente
        │
        ▼
2. Criar objeto CertificateSigningRequest no K8s
        │
        ▼
3. Approver aprova (manual ou automático)
        │
        ▼
4. Signer assina e publica o certificado no objeto CSR
        │
        ▼
5. Baixar o certificado assinado
```

---

## Passo a passo prático

### 1. Gerar chave e CSR

```bash
# Gerar chave privada
openssl genrsa -out joao.key 2048

# Gerar CSR — CN vira username, O vira group no K8s
openssl req -new -key joao.key -out joao.csr \
  -subj "/CN=joao/O=dev-team"
```

### 2. Criar objeto CSR no K8s

```bash
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: joao
spec:
  request: $(cat joao.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400   # 24h
  usages:
  - client auth
EOF
```

### 3. Aprovar

```bash
# Ver CSRs pendentes
kubectl get csr

# Aprovar
kubectl certificate approve joao

# Ou rejeitar
kubectl certificate deny joao
```

### 4. Baixar o certificado assinado

```bash
kubectl get csr joao -o jsonpath='{.status.certificate}' | base64 -d > joao.crt
```

### 5. Usar no kubeconfig

```bash
kubectl config set-credentials joao \
  --client-certificate=joao.crt \
  --client-key=joao.key

kubectl config set-context joao-context \
  --cluster=meu-cluster \
  --user=joao
```

---

## Signers — quem assina o quê

O `signerName` no CSR determina quem pode assinar e o que o cert pode fazer.

| Signer | Uso | Quem aprova |
|---|---|---|
| `kubernetes.io/kube-apiserver-client` | Autenticar no apiserver (usuários, ferramentas) | Manual ou controller |
| `kubernetes.io/kube-apiserver-client-kubelet` | Certificado de client do kubelet | Controller automático |
| `kubernetes.io/kubelet-serving` | Certificado de server do kubelet | Manual (por segurança) |
| `kubernetes.io/legacy-unknown` | Legado — deprecado | Manual |

---

## Controller Manager e os Signers

O `controller-manager` tem controllers específicos para cada signer:

```
controller-manager
├── csrapproving controller
│     └── aprova automaticamente CSRs do tipo kubelet-bootstrap
│           (durante o kubeadm join — bootstrap token → CSR aprovado automaticamente)
│
└── csrsigning controller
      ├── signer: kube-apiserver-client       → assina com ca.crt/ca.key
      ├── signer: kube-apiserver-client-kubelet → assina com ca.crt/ca.key
      └── signer: kubelet-serving             → assina com ca.crt/ca.key
```

**Flags do controller-manager que controlam isso:**
```
--cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
--cluster-signing-key-file=/etc/kubernetes/pki/ca.key
```

Sem essas flags, o controller-manager não consegue assinar nada — CSRs ficam em `Pending` para sempre.

---

## Estados de um CSR

```
kubectl get csr

NAME   AGE   SIGNERNAME                            REQUESTOR   CONDITION
joao   10s   kubernetes.io/kube-apiserver-client   admin       Pending
joao   20s   kubernetes.io/kube-apiserver-client   admin       Approved,Issued
```

| Condition | Significado |
|---|---|
| `Pending` | Aguardando aprovação |
| `Approved` | Aprovado, aguardando assinatura |
| `Approved,Issued` | Aprovado e certificado emitido |
| `Denied` | Rejeitado |

---

## Por que usar a Certificate API em vez de assinar manualmente

**Manual (direto na CA):**
```bash
openssl x509 -req -in joao.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out joao.crt -days 365
```
- Requer acesso ao `ca.key` — arquivo mais sensível do cluster
- Sem auditoria no K8s
- Sem controle de TTL via API

**Via Certificate API:**
- Não precisa de acesso ao `ca.key`
- Aprovação fica registrada no audit log
- TTL controlado via `expirationSeconds`
- Pode ser automatizado com RBAC (só admins aprovam CSRs)

---

## RBAC para controlar quem aprova CSRs

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: csr-approver
rules:
- apiGroups: ["certificates.k8s.io"]
  resources: ["certificatesigningrequests/approval"]
  verbs: ["update"]
- apiGroups: ["certificates.k8s.io"]
  resources: ["signers"]
  resourceNames: ["kubernetes.io/kube-apiserver-client"]
  verbs: ["approve"]
```
