# Tokens e Usuários no Kubeconfig

Como configurar usuários no kubeconfig usando tokens de ServiceAccount, e como criar tokens manualmente.

---

## Gerar token de uma ServiceAccount manualmente

### Modo antigo — Secret estático (ainda funciona, não recomendado)

```bash
# Criar SA
kubectl create serviceaccount minha-sa

# Criar Secret de token vinculado à SA
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: minha-sa-token
  annotations:
    kubernetes.io/service-account.name: minha-sa
type: kubernetes.io/service-account-token
EOF

# Pegar o token
kubectl get secret minha-sa-token -o jsonpath='{.data.token}' | base64 -d
```

Token sem expiração — ruim para segurança.

### Modo atual — TokenRequest API (recomendado)

```bash
# Token com TTL de 1 hora
kubectl create token minha-sa

# Token com TTL customizado
kubectl create token minha-sa --duration=8h

# Token com audience específico
kubectl create token minha-sa --audience=meu-servico
```

---

## Configurar usuário no kubeconfig com token

```bash
# 1. Gerar o token
TOKEN=$(kubectl create token minha-sa --duration=8h)

# 2. Adicionar usuário no kubeconfig
kubectl config set-credentials sa-user --token=$TOKEN

# 3. Criar contexto
kubectl config set-context sa-context \
  --cluster=meu-cluster \
  --user=sa-user \
  --namespace=dev

# 4. Usar
kubectl config use-context sa-context
kubectl get pods   # age com as permissões da SA
```

---

## Verificar quem você é com um token

```bash
kubectl auth whoami
# NAME         UID   GROUPS
# system:serviceaccount:default:minha-sa   ...   [system:serviceaccounts system:serviceaccounts:default system:authenticated]
```

Ou decodificando o JWT manualmente:
```bash
TOKEN=$(kubectl create token minha-sa)
echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool
# {
#   "iss": "kubernetes/serviceaccount",
#   "kubernetes.io/serviceaccount/namespace": "default",
#   "kubernetes.io/serviceaccount/service-account.name": "minha-sa",
#   "exp": 1234567890,
#   ...
# }
```

---

## Configurar usuário com certificado (alternativa ao token)

```bash
# Fluxo completo: gera cert via Certificate API → adiciona no kubeconfig

# 1. Gerar chave + CSR
openssl genrsa -out joao.key 2048
openssl req -new -key joao.key -out joao.csr -subj "/CN=joao/O=dev"

# 2. Criar e aprovar CSR no K8s
kubectl apply -f - <<EOF
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: joao
spec:
  request: $(cat joao.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages: [client auth]
EOF

kubectl certificate approve joao
kubectl get csr joao -o jsonpath='{.status.certificate}' | base64 -d > joao.crt

# 3. Adicionar no kubeconfig
kubectl config set-credentials joao \
  --client-certificate=joao.crt \
  --client-key=joao.key

kubectl config set-context joao-ctx \
  --cluster=meu-cluster \
  --user=joao \
  --namespace=dev
```

---

## Comparação: token vs certificado no kubeconfig

| | Token (SA) | Certificado |
|---|---|---|
| Expiração | Tem TTL (horas) | Tem validade (dias/anos) |
| Revogação | Deletar o Secret ou deixar expirar | Não tem revogação nativa — precisa revogar via CA |
| Identidade | system:serviceaccount:ns:name | CN do cert = username, O = group |
| Uso típico | CI/CD, automação | Usuários humanos, componentes |
| RBAC | Sujeito = ServiceAccount | Sujeito = User/Group |
