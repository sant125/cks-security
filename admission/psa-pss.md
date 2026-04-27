# Pod Security Admission (PSA) e Pod Security Standards (PSS)

## Contexto histórico: PSP → PSA

**PodSecurityPolicy (PSP)** era um admission controller builtin que controlava campos de segurança do spec do Pod. Deprecado no 1.21, removido no 1.25.

Problemas que levaram à remoção:
- Aplicação confusa — fácil de conceder permissões sem querer
- Sem visibilidade de qual PSP estava sendo aplicada a um Pod
- Sem modo audit ou dry-run — impossível testar antes de enforçar
- Difícil de habilitar por padrão no cluster

Alternativas após a remoção:
- **PSA** — builtin, baseado nos Pod Security Standards
- **Policy-as-code** — OPA/Gatekeeper, Kyverno (via webhooks customizados)

---

## Pod Security Standards (PSS)

Define três perfis cumulativos de segurança:

| Nível | Descrição | Uso típico |
|---|---|---|
| `privileged` | Sem restrições | CNI, logging agents, storage drivers |
| `baseline` | Bloqueia privilege escalation conhecida, permite configuração padrão | API servers, workloads gerais |
| `restricted` | Segue hardening best practices, mais restritivo | Apps críticos, dados sensíveis |

### O que o `restricted` exige (em cima do `baseline`):

- `runAsNonRoot: true`
- `allowPrivilegeEscalation: false`
- `seccompProfile.type: RuntimeDefault` ou `Localhost`
- `capabilities.drop: [ALL]`
- `readOnlyRootFilesystem: true` (recomendado)

---

## Pod Security Admission (PSA)

Admission controller builtin (plugin `PodSecurity`) que enforça os PSS por namespace via labels.

### Três modos de operação

| Modo | Comportamento |
|---|---|
| `enforce` | Pod rejeitado se violar a policy |
| `audit` | Pod permitido, violação registrada no audit log |
| `warn` | Pod permitido, aviso retornado para o cliente |

Os três modos são independentes — um namespace pode ter os três simultaneamente com níveis diferentes.

### Labels de namespace

```
pod-security.kubernetes.io/<modo>: <nível>
pod-security.kubernetes.io/<modo>-version: <versão>
```

Exemplo prático:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

- Sem labels: policy padrão do cluster é aplicada (geralmente `privileged`)
- `version` permite fixar o comportamento a uma versão específica do Kubernetes

### Configuração global (admission controller)

Via `--admission-control-config-file` no apiserver:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
  - name: PodSecurity
    configuration:
      apiVersion: pod-security.admission.config.k8s.io/v1
      kind: PodSecurityConfiguration
      defaults:
        enforce: baseline
        enforce-version: latest
        audit: restricted
        warn: restricted
      exemptions:
        usernames: ["system:serviceaccount:kube-system:replicaset-controller"]
        runtimeClassNames: []
        namespaces: ["kube-system", "monitoring"]
```

---

## Exemplos de Pod spec por nível

**Privileged:**
```yaml
spec:
  containers:
    - name: app
      image: nginx
      securityContext:
        privileged: true
```

**Baseline:**
```yaml
spec:
  containers:
    - name: app
      image: nginx
      securityContext:
        allowPrivilegeEscalation: false
```

**Restricted:**
```yaml
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: nginx
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
```

---

## Exemptions

PSA suporta três tipos de isenção — objetos isentos passam sem validação:

| Tipo | Exemplo |
|---|---|
| `usernames` | Service accounts do sistema, controllers |
| `runtimeClassNames` | Runtime classes especiais (ex: kata-containers) |
| `namespaces` | `kube-system`, namespaces de infraestrutura |

Exemptions são configuradas no admission controller (apiserver config) ou no `pod-security-webhook` via ConfigMap.

---

## Migração PSP → PSA

1. **Inventariar PSPs existentes** — mapear quais policies estão em uso e por quem
2. **Habilitar PSA em `audit`** — observar violações sem impacto
3. **Traduzir PSPs para níveis PSS** — ou usar Kyverno/Gatekeeper para policies customizadas
4. **Habilitar `warn`** — feedback imediato nos deploys
5. **Migrar namespace a namespace** para `enforce` — nunca em massa
6. **Remover PSPs** após validar que tudo está coberto

---

## PSA vs Webhooks customizados

| | PSA | Kyverno / OPA Gatekeeper |
|---|---|---|
| Complexidade | Baixa — apenas labels no namespace | Alta — precisa deploy e manutenção |
| Flexibilidade | Limitada aos 3 níveis PSS | Total — qualquer lógica |
| Granularidade | Por namespace | Por namespace, label, campo específico |
| Mutação | Não | Sim (Kyverno suporta) |
| Audit/Warn nativos | Sim | Depende da ferramenta |

PSA cobre a maioria dos clusters. Webhooks customizados entram quando você precisa de políticas além do que os 3 níveis oferecem.
