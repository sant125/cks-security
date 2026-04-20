# ServiceAccount Token — Ciclo de vida completo

## O padrão antigo (< K8s 1.20)

O token era um **Secret estático** montado no pod:

```
/var/run/secrets/kubernetes.io/serviceaccount/
├── token      ← JWT sem expiração
├── ca.crt     ← CA do cluster
└── namespace  ← namespace do pod
```

**Problema:** token nunca expirava. Se vazasse, o atacante tinha acesso para sempre.

---

## O padrão atual — Projected Volume + TokenRequest API

A partir do K8s 1.20 o mesmo path é montado, mas agora via **Projected Volume** com token renovável.

### Como funciona

```
Pod criado
  └── kubelet lê spec.serviceAccountName
        └── chama TokenRequest API (kube-apiserver)
              └── passa: SA name, expirationSeconds, audiences
                    └── recebe: JWT assinado com TTL
                          └── monta via Projected Volume no pod
                                └── renova automaticamente em 80% do TTL
```

### O que fica em /var/run/secrets/kubernetes.io/serviceaccount/

O conteúdo parece igual ao antigo, mas é completamente diferente por baixo:

| Arquivo | O que é | Antes | Agora |
|---|---|---|---|
| `token` | JWT de autenticação | Secret estático, sem TTL | TokenRequest, TTL 1h, renovável |
| `ca.crt` | CA do cluster para validar o apiserver | Igual | Igual |
| `namespace` | Namespace do pod | Igual | Igual |

### O Projected Volume por baixo

O kubelet cria automaticamente esse volume (você não precisa declarar):

```yaml
volumes:
- name: kube-api-access-xxxxx
  projected:
    defaultMode: 420
    sources:
    - serviceAccountToken:
        expirationSeconds: 3607   # ~1 hora (default)
        path: token
    - configMap:
        items:
        - key: ca.crt
          path: ca.crt
        name: kube-root-ca.crt
    - downwardAPI:
        items:
        - fieldRef:
            apiVersion: v1
            fieldPath: metadata.namespace
          path: namespace
```

### TTL e renovação

- **Default:** 1 hora (3607 segundos) — [fonte oficial](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)
- **Renovação:** kubelet renova quando 80% do TTL decorreu (~48min)
- **Invalidação:** token é invalidado automaticamente quando o pod é deletado
- **Configurável:** via `--service-account-max-token-expiration` no apiserver

---

## Por que ainda é risco de segurança

Mesmo com TTL de 1h, se o pod tiver RCE:

```bash
# Atacante lê o token
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
APISERVER=https://kubernetes.default.svc

# Chama a API do cluster
curl -sk $APISERVER/api/v1/namespaces/default/secrets \
  -H "Authorization: Bearer $TOKEN"
```

Com permissões permissivas na SA, o atacante consegue:
- Listar/ler Secrets do namespace
- Criar pods com volumes hostPath (escalar para o node)
- Fazer deploy de containers maliciosos

---

## Como mitigar

### 1. automountServiceAccountToken: false

Para pods que não precisam chamar a API do K8s (a maioria das aplicações web):

```yaml
apiVersion: v1
kind: Pod
spec:
  automountServiceAccountToken: false
```

Ou na própria ServiceAccount para aplicar em todos os pods que a usam:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: minha-app
automountServiceAccountToken: false
```

O kubelet **não monta nenhum volume** — o token simplesmente não existe no pod.

### 2. RBAC mínimo

Se o pod precisar do token, limitar ao mínimo necessário:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: leitura-configmaps
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
  resourceNames: ["meu-config"]  # só esse configmap específico
```

### 3. Projected Volume com audience e TTL customizados

Para casos onde precisa do token mas quer mais controle:

```yaml
volumes:
- name: token-customizado
  projected:
    sources:
    - serviceAccountToken:
        audience: meu-servico-interno   # só válido para esse audience
        expirationSeconds: 600          # 10 minutos
        path: token
```

### 4. Falco — detectar leitura do token

O Falco detecta quando um processo tenta ler o token de dentro do container:

```
Read environment variable from /proc files
```

E também leitura suspeita em `/var/run/secrets`.

---

## Resumo visual

```
Pod com automount (padrão)          Pod com automount: false
┌─────────────────────────┐         ┌─────────────────────────┐
│ /var/run/secrets/...    │         │                         │
│   token  ← JWT 1h       │         │   sem volume montado    │
│   ca.crt                │         │                         │
│   namespace             │         │                         │
│                         │         │                         │
│ RCE → lê token →        │         │ RCE → sem token →       │
│ chama API cluster  ✗    │         │ não consegue chamar ✓   │
└─────────────────────────┘         └─────────────────────────┘
```

---

## Referências

- [Managing Service Accounts — kubernetes.io](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)
- [Configure Service Accounts for Pods — kubernetes.io](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
- [TokenRequest API — kubernetes.io](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-request-v1/)
