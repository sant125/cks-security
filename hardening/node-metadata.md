# Protegendo o Node Metadata

O serviço de metadata do cloud provider (AWS, GCP, Azure) fica acessível em `169.254.169.254` — um IP link-local alcançável de qualquer processo no node, incluindo pods.

---

## O problema

```
Pod malicioso (ou comprometido)
        │
        ▼
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
        │
        ▼
Credenciais temporárias da IAM Role do node
        │
        ▼
Atacante age com as permissões do node no AWS/GCP
```

A IAM Role do node tipicamente tem permissões de:
- Pull de imagens do ECR/Artifact Registry
- Acesso a volumes (EBS/PD)
- Às vezes Secrets Manager, SSM, etc.

---

## Mitigação via NetworkPolicy

Bloquear acesso ao metadata server para todos os pods (exceto os que realmente precisam):

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-metadata
  namespace: default
spec:
  podSelector: {}          # aplica em todos os pods do namespace
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 169.254.169.254/32   # bloqueia só o metadata server
```

**Atenção:** NetworkPolicy é aditiva. Você precisa garantir que o restante do egress esteja permitido (o `cidr: 0.0.0.0/0` com except faz isso).

---

## GCP — Workload Identity (solução preferida)

Em vez de bloquear, a solução correta no GKE é **não ter credenciais no metadata server**.

Com Workload Identity:
- O pod usa uma Kubernetes ServiceAccount
- A KSA é vinculada a uma GCP Service Account
- O pod recebe tokens escopados via Projected Volume — não precisa do metadata server
- Mesmo que acesse `169.254.169.254`, não há credenciais lá para roubar

```bash
# Habilitar no GKE
gcloud container clusters update meu-cluster \
  --workload-pool=PROJECT_ID.svc.id.goog

# Vincular KSA → GSA
kubectl annotate serviceaccount minha-ksa \
  iam.gke.io/gcp-service-account=minha-gsa@PROJECT_ID.iam.gserviceaccount.com
```

---

## AWS — IRSA (IAM Roles for Service Accounts)

Equivalente ao Workload Identity no EKS. O pod recebe token OIDC via Projected Volume, troca por credenciais temporárias via STS — sem depender do metadata do node.

---

## Verificar se um pod consegue acessar o metadata

```bash
# De dentro do pod
kubectl exec -it meu-pod -- curl -s http://169.254.169.254/latest/meta-data/ --max-time 2
# Se retornar dados → metadata acessível → mitigar
# Se der timeout → NetworkPolicy funcionando ou Workload Identity ativo
```

---

## No CKS — o que cai na prova

- Criar NetworkPolicy que nega egress para `169.254.169.254/32`
- Entender que o risco é a role do node, não do pod
- Saber a diferença entre bloquear (NetworkPolicy) e remover o risco (Workload Identity/IRSA)
