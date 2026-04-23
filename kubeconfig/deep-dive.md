# Kubeconfig — Deep Dive

O kubeconfig é o arquivo que o kubectl usa para saber onde está o cluster e como se autenticar. Fica em `~/.kube/config` por padrão.

---

## Estrutura do arquivo

```yaml
apiVersion: v1
kind: Config

# 1. CLUSTERS — onde estão os clusters
clusters:
- name: meu-cluster
  cluster:
    server: https://34.74.203.254:6443      # endpoint do apiserver
    certificate-authority-data: <base64>    # CA do cluster (ou certificate-authority: /path/ca.crt)

# 2. USERS — como autenticar
users:
- name: admin
  user:
    client-certificate-data: <base64>       # cert do client
    client-key-data: <base64>               # chave privada

- name: joao
  user:
    token: eyJhbGciOiJSUzI1NiIsI...         # bearer token (ServiceAccount ou OIDC)

# 3. CONTEXTS — combinação de cluster + user + namespace
contexts:
- name: admin@meu-cluster
  context:
    cluster: meu-cluster
    user: admin
    namespace: default

- name: joao@meu-cluster
  context:
    cluster: meu-cluster
    user: joao
    namespace: dev

# 4. CONTEXTO ATUAL
current-context: admin@meu-cluster
```

---

## Comandos do dia a dia

```bash
# Ver contexto atual
kubectl config current-context

# Listar todos os contextos
kubectl config get-contexts

# Trocar de contexto
kubectl config use-context joao@meu-cluster

# Ver o kubeconfig completo
kubectl config view

# Ver sem ocultar dados sensíveis
kubectl config view --raw
```

---

## Manipular usuários e contextos via CLI

### Adicionar um usuário com certificado

```bash
kubectl config set-credentials joao \
  --client-certificate=joao.crt \
  --client-key=joao.key
```

### Adicionar um usuário com token

```bash
kubectl config set-credentials joao-sa \
  --token=eyJhbGciOiJSUzI1NiIsI...
```

### Criar contexto

```bash
kubectl config set-context joao-ctx \
  --cluster=meu-cluster \
  --user=joao \
  --namespace=dev
```

### Usar contexto específico só para um comando

```bash
kubectl get pods --context=joao-ctx
```

---

## Múltiplos kubeconfigs

Você pode ter vários arquivos e mesclá-los com `KUBECONFIG`:

```bash
export KUBECONFIG=~/.kube/config:~/.kube/cluster-prod:~/.kube/cluster-hom

# Agora kubectl enxerga todos os clusters/contexts dos três arquivos
kubectl config get-contexts
```

Ou passar direto:
```bash
kubectl --kubeconfig=/path/to/outro-config get pods
```

---

## Como o kubectl autentica — o que acontece por baixo

```
kubectl get pods
        │
        ▼
lê ~/.kube/config
        │
        ├── pega server do cluster atual
        ├── pega CA do cluster (para verificar o cert do apiserver)
        └── pega credenciais do user atual
              ├── se client cert → mTLS (apresenta cert no handshake TLS)
              └── se token → Header: Authorization: Bearer <token>
        │
        ▼
abre conexão TLS com o apiserver
verifica cert do apiserver contra a CA
apresenta credenciais
        │
        ▼
apiserver autentica → RBAC → resposta
```

---

## Segurança do kubeconfig

O arquivo contém as chaves privadas e tokens em base64 (não é criptografia — é só encoding). Quem tem o arquivo tem acesso ao cluster com as permissões do user configurado.

```bash
# Ver o token de um user no kubeconfig (base64 decode)
kubectl config view --raw -o jsonpath='{.users[?(@.name=="admin")].user.client-certificate-data}' | base64 -d | openssl x509 -text -noout | grep Subject
```

**Boas práticas:**
- Nunca commitar kubeconfig em repositórios
- Usar contextos com usuários de permissão mínima para o dia a dia
- Admin context só quando necessário
- Em CI/CD: usar ServiceAccount token com RBAC mínimo, não o kubeconfig de admin
