# Admission Webhooks

## Fluxo no API server

Toda requisição passa por:

```
authn → authz → admission controllers → etcd
```

Admission controllers ficam entre autorização e persistência. Podem **rejeitar** ou **mutar** o objeto antes de salvar.

---

## Plugins builtin

Compilados no binário do `kube-apiserver`. Habilitados/desabilitados via flag:

```
--enable-admission-plugins=NodeRestriction,PodSecurity,...
--disable-admission-plugins=DefaultStorageClass
```

Relevantes pro CKS:

| Plugin | O que faz |
|---|---|
| `NodeRestriction` | Impede kubelet de modificar objetos de outros nodes |
| `PodSecurity` | Enforça Pod Security Standards por namespace |
| `AlwaysPullImages` | Força pull toda vez — impede uso de imagem cacheada sem auth |
| `DenyServiceExternalIPs` | Bloqueia `externalIPs` em Services (vetor de interceptação de tráfego) |

No self-hosted, esses plugins são configuráveis editando o manifest estático em `/etc/kubernetes/manifests/kube-apiserver.yaml`. No managed (GKE/EKS/AKS), o control plane não é acessível — o provider define quais estão ativos.

---

## Webhooks — extensão dinâmica

Dois tipos, configurados via recursos da API sem restart do apiserver:

- **ValidatingWebhookConfiguration** — só aprova ou rejeita, não altera o objeto
- **MutatingWebhookConfiguration** — pode modificar o objeto (injetar sidecars, defaults, labels)

Mutating sempre roda antes de Validating.

### Fluxo

```
kubectl apply → apiserver → HTTPS POST para o webhook
                                      ↓
                           AdmissionReview com o objeto
                                      ↓
                           webhook responde: allowed + patch opcional
```

### AdmissionReview

Request que o apiserver envia:

```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "request": {
    "uid": "abc-123",
    "kind": {"group": "", "version": "v1", "kind": "Pod"},
    "operation": "CREATE",
    "object": { }
  }
}
```

Resposta de rejeição:

```json
{
  "response": {
    "uid": "abc-123",
    "allowed": false,
    "status": {"message": "runAsRoot not allowed"}
  }
}
```

Resposta de mutação (JSON Patch em base64):

```json
{
  "response": {
    "uid": "abc-123",
    "allowed": true,
    "patchType": "JSONPatch",
    "patch": "W3sib3AiOiJhZGQiLCJwYXRoIjoiL3NwZWMvc2VjdXJpdHlDb250ZXh0L3J1bkFzTm9uUm9vdCIsInZhbHVlIjp0cnVlfV0="
  }
}
```

O patch decodificado:

```json
[{"op": "add", "path": "/spec/securityContext/runAsNonRoot", "value": true}]
```

---

## Configuração do recurso

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: pod-security-check
webhooks:
  - name: check.security.io
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
    clientConfig:
      service:
        name: webhook-svc
        namespace: security
        path: /validate
      caBundle: <base64 CA cert>
    admissionReviewVersions: ["v1"]
    sideEffects: None
    failurePolicy: Fail
```

Campos críticos:

- `failurePolicy: Fail` — se o webhook cair, todas as requisições bloqueiam. `Ignore` deixa passar. Tradeoff segurança x disponibilidade.
- `namespaceSelector` / `objectSelector` — limita quais namespaces/objetos o webhook intercepta.
- `sideEffects: None` — necessário para dry-run (`kubectl apply --dry-run`) funcionar corretamente.
- `caBundle` — o apiserver usa para verificar o TLS do webhook. TLS é obrigatório.

---

## Webhook em Go

Estrutura mínima de um servidor que valida pods:

```go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"

	admissionv1 "k8s.io/api/admission/v1"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

func validate(w http.ResponseWriter, r *http.Request) {
	var review admissionv1.AdmissionReview
	if err := json.NewDecoder(r.Body).Decode(&review); err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}

	var pod corev1.Pod
	if err := json.Unmarshal(review.Request.Object.Raw, &pod); err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}

	allowed := true
	msg := ""

	sc := pod.Spec.SecurityContext
	if sc == nil || sc.RunAsNonRoot == nil || !*sc.RunAsNonRoot {
		allowed = false
		msg = "pod must set securityContext.runAsNonRoot: true"
	}

	review.Response = &admissionv1.AdmissionResponse{
		UID:     review.Request.UID,
		Allowed: allowed,
	}
	if !allowed {
		review.Response.Result = &metav1.Status{Message: msg}
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(review)
}

func main() {
	http.HandleFunc("/validate", validate)
	// TLS obrigatório — apiserver só chama via HTTPS
	fmt.Println("listening on :8443")
	if err := http.ListenAndServeTLS(":8443", "tls.crt", "tls.key", nil); err != nil {
		panic(err)
	}
}
```

O cert que você passa no `ListenAndServeTLS` é o mesmo cuja CA vai no `caBundle` do webhook resource.

---

## Self-hosted vs Managed

| | Self-hosted | Managed |
|---|---|---|
| Admission plugins builtin | Configura via flag no manifest estático | Provider define, sem acesso ao binário |
| Webhooks customizados | Liberdade total | Liberdade total |
| Vetor de ataque extra | Acesso ao manifest do apiserver permite desabilitar plugins | Control plane inacessível — superfície menor |

---

## Ferramentas que usam webhooks

- **OPA/Gatekeeper** — policy as code via ValidatingWebhook, políticas em Rego
- **Kyverno** — policy engine nativa Kubernetes, usa ambos Validating e Mutating
- **Istio** — injeta sidecar Envoy via MutatingWebhook
