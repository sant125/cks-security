# RBAC — Roles, ClusterRoles e Bindings

RBAC (Role-Based Access Control) controla quem pode fazer o quê no cluster. É o mecanismo de autorização padrão do Kubernetes.

---

## Os 4 objetos

```
┌──────────────────────────────────────────────────────────────┐
│  PERMISSÕES                    VINCULAÇÃO                    │
│                                                              │
│  Role              →  RoleBinding                           │
│  (namespaced)         (namespaced)                          │
│                                                              │
│  ClusterRole       →  ClusterRoleBinding                    │
│  (cluster-wide)       (cluster-wide)                        │
│                                                              │
│  ClusterRole       →  RoleBinding                           │
│  (reutilizado)        (namespaced — restringe ao namespace)  │
└──────────────────────────────────────────────────────────────┘
```

---

## Role — permissões dentro de um namespace

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev             # só vale nesse namespace
rules:
- apiGroups: [""]            # core group
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]    # sub-resource para kubectl logs
  verbs: ["get"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: joao-pod-reader
  namespace: dev
subjects:
- kind: User
  name: joao                 # CN do certificado ou claim do OIDC
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## ClusterRole — permissões em todo o cluster

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader          # sem namespace = cluster-wide
rules:
- apiGroups: [""]
  resources: ["nodes"]       # nodes são cluster-scoped (não têm namespace)
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["get", "list"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ops-node-reader
subjects:
- kind: Group
  name: ops-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## ClusterRole + RoleBinding — o caso especial

Um ClusterRole pode ser reutilizado em múltiplos namespaces via RoleBinding. O acesso fica restrito ao namespace do RoleBinding.

```yaml
# Cria uma vez o ClusterRole
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
---
# Reutiliza em namespaces específicos
kind: RoleBinding
metadata:
  name: dev-secret-reader
  namespace: dev              # acesso só em "dev"
subjects:
- kind: User
  name: maria
roleRef:
  kind: ClusterRole           # referencia ClusterRole
  name: secret-reader
```

Vantagem: define a regra uma vez, vincula em vários namespaces.

---

## Subjects — quem recebe a permissão

| Kind | Exemplo | Quando usar |
|---|---|---|
| `User` | `joao` | Usuário humano (CN do cert ou OIDC claim) |
| `Group` | `dev-team` | Grupo (O do cert ou OIDC groups claim) |
| `ServiceAccount` | `minha-sa` | Pod/processo dentro do cluster |

```yaml
subjects:
# Usuário
- kind: User
  name: joao
  apiGroup: rbac.authorization.k8s.io

# Grupo
- kind: Group
  name: dev-team
  apiGroup: rbac.authorization.k8s.io

# ServiceAccount
- kind: ServiceAccount
  name: minha-app
  namespace: producao        # namespace é obrigatório para SA
```

---

## Verbs disponíveis

| Verb | Operação HTTP | Exemplo |
|---|---|---|
| `get` | GET por nome | `kubectl get pod meu-pod` |
| `list` | GET (coleção) | `kubectl get pods` |
| `watch` | GET com watch | Streams de eventos |
| `create` | POST | `kubectl apply` (novo) |
| `update` | PUT | `kubectl apply` (existente) |
| `patch` | PATCH | `kubectl patch` |
| `delete` | DELETE | `kubectl delete` |
| `deletecollection` | DELETE (coleção) | `kubectl delete pods --all` |

---

## Verificar permissões

```bash
# O que eu posso fazer?
kubectl auth can-i --list

# Posso deletar pods?
kubectl auth can-i delete pods

# Posso criar deployments no namespace dev?
kubectl auth can-i create deployments -n dev

# O que o usuário "joao" pode fazer? (admin verificando)
kubectl auth can-i --list --as=joao

# O que a SA minha-app pode fazer?
kubectl auth can-i --list --as=system:serviceaccount:default:minha-app
```

---

## Recursos cluster-scoped vs namespaced

Recursos cluster-scoped **nunca** são controlados por Role — só por ClusterRole.

| Cluster-scoped | Namespaced |
|---|---|
| nodes | pods |
| persistentvolumes | deployments |
| namespaces | services |
| clusterroles | roles |
| storageclasses | secrets |
| certificatesigningrequests | configmaps |

---

## Privilege escalation — o que o RBAC impede

O Kubernetes impede que um usuário conceda permissões que ele mesmo não tem:

```bash
# Joao não tem permissão de deletar pods
# Joao NÃO consegue criar um Role que dê permissão de deletar pods para outro usuário
# mesmo que seja admin de RBAC
```

Exceto se tiver `escalate` verb no recurso `roles` — raramente concedido.

---

## Padrão mínimo de segurança

```yaml
# SA da aplicação — só o que ela precisa
apiVersion: v1
kind: ServiceAccount
metadata:
  name: minha-app
automountServiceAccountToken: false   # desabilita se não precisar da API
---
# Se precisar da API, Role mínimo
kind: Role
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["minha-config"]    # só esse configmap específico
  verbs: ["get"]
```
