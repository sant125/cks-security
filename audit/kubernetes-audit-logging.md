# Kubernetes Audit Logging

O kube-apiserver registra toda requisição que passa por ele — quem fez, o quê, quando, em qual recurso. É a principal fonte forense em um incidente.

Não é um objeto Kubernetes — é configuração direta do processo do apiserver via flags e arquivo de policy.

---

## Estágios de auditoria

Toda requisição pode ser capturada em 4 momentos:

| Stage | Quando |
|---|---|
| `RequestReceived` | Requisição chegou, antes de qualquer processamento |
| `ResponseStarted` | Headers da resposta enviados, body ainda não |
| `ResponseComplete` | Resposta completa enviada |
| `Panic` | Erro interno no apiserver |

---

## Níveis de detalhe

| Level | O que registra |
|---|---|
| `None` | Nada — ignora a requisição |
| `Metadata` | Quem, o quê, quando — sem body |
| `Request` | Metadata + body da requisição |
| `RequestResponse` | Metadata + body da requisição + body da resposta |

---

## Self-managed (kubeadm)

### 1. Criar a policy

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Ignorar ruído do sistema
- level: None
  users: ["system:kube-proxy"]
  verbs: ["watch"]
  resources:
  - group: ""
    resources: ["endpoints", "services"]

- level: None
  resources:
  - group: ""
    resources: ["events"]

# Leitura de Secrets — body completo (investigação de vazamento)
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]

# Tudo mais — só metadata
- level: Metadata
```

### 2. Configurar o apiserver

Editar `/etc/kubernetes/manifests/kube-apiserver.yaml`:

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit.log
    - --audit-log-maxage=30        # retenção em dias
    - --audit-log-maxbackup=10     # número de arquivos rotacionados
    - --audit-log-maxsize=100      # tamanho máximo em MB antes de rotacionar
```

O apiserver reinicia automaticamente ao salvar (é um static pod).

---

## Cloud providers

### EKS (AWS)

Habilitado via CLI — a AWS configura o apiserver por você:

```bash
aws eks update-cluster-config \
  --name meu-cluster \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'
```

Logs vão para CloudWatch Logs em `/aws/eks/<cluster-name>/cluster`.

**Custo:** você paga ingestão e storage do CloudWatch. Desde set/2023 os logs do EKS são **Vended Logs** com desconto por volume. Sem configurar retenção, os logs acumulam custo indefinidamente — configurar retenção é obrigatório.

```bash
# Configurar retenção de 30 dias no grupo de logs
aws logs put-retention-policy \
  --log-group-name /aws/eks/meu-cluster/cluster \
  --retention-in-days 30
```

### GKE (Google Cloud)

Admin Activity audit logs habilitados por padrão, sem custo adicional.

```
Cloud Logging → Logs Explorer:
resource.type="k8s_cluster"
logName="projects/PROJECT/logs/cloudaudit.googleapis.com%2Factivity"
```

**Custo:** gratuito até 50 GiB/mês de ingestão. Acima disso: $0.50/GiB. Storage gratuito por 30 dias.

### AKS (Azure)

```bash
az monitor diagnostic-settings create \
  --resource <cluster-resource-id> \
  --name audit-logs \
  --logs '[{"category":"kube-audit","enabled":true}]' \
  --workspace <log-analytics-workspace-id>
```

Logs vão para Log Analytics Workspace.

---

## Comparativo

| | Self-managed | EKS | GKE | AKS |
|---|---|---|---|---|
| Configuração | Flag no apiserver | `aws eks update-cluster-config` | Padrão habilitado | Diagnostic Settings |
| Destino | Arquivo no node | CloudWatch Logs | Cloud Logging | Log Analytics |
| Policy customizável | Sim — total | Não | Não | Não |
| Custo adicional | Não | Sim (CloudWatch) | Não até 50GiB/mês | Sim (Log Analytics) |

---

## Investigação — queries úteis

```bash
# Self-managed — quem leu um Secret?
grep '"secrets"' /var/log/kubernetes/audit.log \
  | grep '"get"' \
  | jq '{user: .user.username, time: .requestReceivedTimestamp, secret: .objectRef.name}'

# Acesso anônimo
grep '"system:anonymous"' /var/log/kubernetes/audit.log

# Pod criado com hostNetwork
grep '"pods"' audit.log \
  | grep '"create"' \
  | jq 'select(.requestObject.spec.hostNetwork==true)'

# Quem deletou um deployment?
grep '"deployments"' audit.log \
  | grep '"delete"' \
  | jq '{user: .user.username, time: .requestReceivedTimestamp, name: .objectRef.name}'
```

---

## Por que é importante para CKS

O audit log é a única fonte que responde:
- **Quem** fez o quê no cluster
- **Quando** um Secret foi acessado
- **De onde** veio uma requisição suspeita
- **O que** um ServiceAccount comprometido fez

Em um incidente onde o atacante tem token de SA (ver `serviceaccount/token-lifecycle.md`), o audit log mostra exatamente quais chamadas foram feitas com aquele token.

---

## Referências

- [EKS Control Plane Logs — AWS Docs](https://docs.aws.amazon.com/eks/latest/userguide/control-plane-logs.html)
- [GKE Audit Logging — Google Cloud Docs](https://cloud.google.com/kubernetes-engine/docs/how-to/audit-logging-container)
- [Kubernetes Audit — kubernetes.io](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/)
