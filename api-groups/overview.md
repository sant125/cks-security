# API Groups — Core vs Named

O Kubernetes organiza seus recursos em grupos de API. Isso impacta diretamente como você escreve RBAC e como chama a API REST.

---

## A divisão

```
API do Kubernetes
├── Core Group (legado, sem nome de grupo)
│     └── /api/v1
│           ├── pods
│           ├── services
│           ├── configmaps
│           ├── secrets
│           ├── namespaces
│           ├── nodes
│           ├── persistentvolumes
│           ├── persistentvolumeclaims
│           └── serviceaccounts
│
└── Named Groups
      └── /apis/<group>/<version>
            ├── apps/v1
            │     ├── deployments
            │     ├── statefulsets
            │     ├── daemonsets
            │     └── replicasets
            ├── batch/v1
            │     ├── jobs
            │     └── cronjobs
            ├── rbac.authorization.k8s.io/v1
            │     ├── roles
            │     ├── clusterroles
            │     ├── rolebindings
            │     └── clusterrolebindings
            ├── networking.k8s.io/v1
            │     ├── ingresses
            │     └── networkpolicies
            ├── storage.k8s.io/v1
            │     └── storageclasses
            └── certificates.k8s.io/v1
                  └── certificatesigningrequests
```

---

## Como descobrir o grupo de um recurso

```bash
# Listar todos os recursos e seus grupos
kubectl api-resources

# Saída:
NAME                    SHORTNAMES  APIVERSION                    NAMESPACED  KIND
pods                    po          v1                            true        Pod
deployments             deploy      apps/v1                       true        Deployment
ingresses               ing         networking.k8s.io/v1          true        Ingress
clusterroles                        rbac.authorization.k8s.io/v1  false       ClusterRole
```

O campo `APIVERSION` mostra o grupo:
- `v1` → core group (sem grupo)
- `apps/v1` → grupo `apps`, versão `v1`
- `rbac.authorization.k8s.io/v1` → grupo `rbac.authorization.k8s.io`, versão `v1`

---

## Impacto no RBAC

No RBAC, o `apiGroups` no Role reflete exatamente essa divisão:

```yaml
rules:
# Recursos do core group → apiGroups: [""]
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list"]

# Recursos do grupo apps → apiGroups: ["apps"]
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "watch"]

# Recursos de RBAC → apiGroups: ["rbac.authorization.k8s.io"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["roles", "rolebindings"]
  verbs: ["get", "list"]
```

**Core group é string vazia `""`** — erro comum é colocar `"v1"` ou `"core"` que não funcionam.

---

## Chamando a API REST diretamente

```bash
# Proxy local para facilitar
kubectl proxy &

# Core group — /api/v1
curl http://localhost:8001/api/v1/namespaces/default/pods

# Named group — /apis/<group>/<version>
curl http://localhost:8001/apis/apps/v1/namespaces/default/deployments
curl http://localhost:8001/apis/networking.k8s.io/v1/namespaces/default/ingresses

# Listar todos os grupos disponíveis
curl http://localhost:8001/apis
```

---

## Versões e estabilidade

Cada grupo pode ter múltiplas versões em paralelo:

| Versão | Estabilidade | Exemplo |
|---|---|---|
| `v1` | Stable (GA) | `v1` no core group |
| `v1beta1`, `v1beta2` | Beta — pode mudar | `networking.k8s.io/v1beta1` |
| `v1alpha1` | Alpha — pode sumir | Recursos experimentais |

```bash
# Ver quais versões um grupo oferece
kubectl api-versions | grep networking
# networking.k8s.io/v1
```

---

## Por que isso importa para CKS

1. **RBAC errado por `apiGroups` incorreto** é erro silencioso — o Role parece certo mas não concede nada
2. **Audit logs** mostram o group/resource/verb de cada request — saber mapear ajuda na análise
3. **NetworkPolicy** fica em `networking.k8s.io/v1` — relevante para isolamento de pods
4. **CertificateSigningRequest** fica em `certificates.k8s.io/v1` — relevante para RBAC de aprovação de CSRs
