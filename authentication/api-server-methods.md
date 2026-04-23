# Métodos de Autenticação do API Server

O API server tenta cada método em ordem até algum autenticar. Se nenhum funcionar, o request vira `anonymous`.

```
Request chega no apiserver
        │
        ▼
┌─── Client Certificate? ──────────────────────────────────────┐
│    mTLS: client apresenta cert assinado pela CA do cluster    │
│    Usado por: kubectl (kubeconfig), kubelet, controller, etc. │
└──────────────────────────────────────────────────────────────┘
        │ não autenticou
        ▼
┌─── Bearer Token? ────────────────────────────────────────────┐
│    Header: Authorization: Bearer <token>                      │
│    Tipos: ServiceAccount token, OIDC token                    │
└──────────────────────────────────────────────────────────────┘
        │ não autenticou
        ▼
┌─── Bootstrap Token? ─────────────────────────────────────────┐
│    Usado quando kubelet ainda não tem cert                    │
│    Durante o kubeadm join de um novo node                    │
└──────────────────────────────────────────────────────────────┘
        │ não autenticou
        ▼
┌─── Webhook Token? ───────────────────────────────────────────┐
│    Delega validação para serviço externo                      │
│    Usado com SSO, LDAP, etc.                                 │
└──────────────────────────────────────────────────────────────┘
        │ não autenticou
        ▼
   anonymous (system:anonymous, group: system:unauthenticated)
   por padrão sem permissão para nada
```

---

## 1. Client Certificate (mTLS)

O canal TLS já está estabelecido. O client apresenta um certificado junto.

```bash
# kubectl por baixo dos panos
curl https://apiserver:6443/api/v1/pods \
  --cacert /etc/kubernetes/pki/ca.crt \
  --cert /etc/kubernetes/pki/admin.crt \
  --key /etc/kubernetes/pki/admin.key
```

O apiserver extrai do certificado:
- **Subject CN** → username (ex: `kubernetes-admin`)
- **Subject O** → groups (ex: `system:masters`)

Não precisa de nenhuma configuração extra no apiserver — é nativo do TLS.

---

## 2. Bearer Token — ServiceAccount

Token JWT gerado pela TokenRequest API. O pod recebe montado em:

```
/var/run/secrets/kubernetes.io/serviceaccount/token
```

```bash
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

curl https://kubernetes.default.svc/api/v1/namespaces/default/pods \
  --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  -H "Authorization: Bearer $TOKEN"
```

---

## 3. Bearer Token — OIDC

Para integração com provedores de identidade (Google, Okta, Dex, etc.).

```
Usuário faz login no OIDC provider → recebe JWT
             ↓
kubectl usa o JWT como Bearer token
             ↓
apiserver valida assinatura do JWT contra o OIDC provider
             ↓
extrai username e groups do payload do JWT
```

Configuração no apiserver:
```
--oidc-issuer-url=https://accounts.google.com
--oidc-client-id=meu-cluster
--oidc-username-claim=email
--oidc-groups-claim=groups
```

---

## 4. Bootstrap Token

Usado apenas durante o `kubeadm join`. O kubelet ainda não tem certificado — usa um token temporário para se autenticar pela primeira vez e solicitar um CSR.

```
kubeadm join --token <bootstrap-token> --discovery-token-ca-cert-hash sha256:<hash>
                    ↓
kubelet autentica com o bootstrap token
                    ↓
cria CSR (Certificate Signing Request)
                    ↓
controller-manager aprova e assina
                    ↓
kubelet passa a usar o certificado próprio
```

Bootstrap tokens ficam em Secrets no namespace `kube-system`:
```bash
kubectl get secrets -n kube-system | grep bootstrap-token
```

---

## 5. Webhook Token

O apiserver envia o token para um endpoint externo que retorna se é válido ou não.

```
Request com Bearer token
        ↓
apiserver chama webhook: POST /validate-token
        ↓
webhook responde: { "authenticated": true, "user": { "username": "...", "groups": [...] } }
        ↓
apiserver continua com o username/groups retornados
```

---

## Resumo de uso

| Método | Quem usa | Tipo |
|---|---|---|
| Client certificate | kubectl, kubelet, controller-manager, scheduler | mTLS |
| Bearer SA token | Pods chamando a API do cluster | JWT com TTL |
| Bearer OIDC | Usuários humanos via SSO | JWT externo |
| Bootstrap token | kubelet durante kubeadm join | Temporário |
| Webhook | Integrações customizadas | Delegado |

---

## Anonymous access

Se nenhum método autenticar, o request chega como:
- Username: `system:anonymous`
- Group: `system:unauthenticated`

Por padrão, nenhum ClusterRole dá permissão a esses grupos. Mas se existir um RoleBinding concedendo acesso a `system:unauthenticated`, anonymous requests conseguem agir.

Para desabilitar completamente:
```
--anonymous-auth=false  # flag do apiserver
```

**CKS:** saber desabilitar anonymous e entender o impacto (health checks podem usar anonymous — verificar antes de desabilitar).
