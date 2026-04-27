# CKS Security

Estudos para CKS (Certified Kubernetes Security Specialist).

---

## Estrutura

### Cluster Setup
- [On-prem vs Managed](cluster-setup/onprem-vs-managed.md) — responsabilidades, autoupgrade, superfície de ataque

### Autenticação
- [Métodos do API Server](authentication/api-server-methods.md) — mTLS, bearer token, OIDC, bootstrap, webhook, anonymous
- [Kubelet — Portas e Segurança](authentication/kubelet-ports.md) — porta 10250 vs 10255, anonymous auth, fluxo do kubectl exec

### ServiceAccount
- [Token Lifecycle](serviceaccount/token-lifecycle.md) — projected volume, TokenRequest API, automount
- [Tokens e Usuários no Kubeconfig](serviceaccount/kubeconfig-tokens.md) — gerar tokens, configurar usuários

### TLS
- [Arquitetura TLS do Cluster](tls/cluster-tls-architecture.md) — quem tem qual certificado, hierarquia de CAs
- [PKI Deep Dive](tls/pki-deep-dive.md)
- [Certificate API e CSR](tls/certificate-api-csr.md) — fluxo de CSR, signers, controller-manager

### Kubeconfig
- [Deep Dive](kubeconfig/deep-dive.md) — estrutura, contextos, múltiplos configs, como o kubectl autentica

### API Groups
- [Core vs Named](api-groups/overview.md) — /api/v1 vs /apis/, impacto no RBAC

### RBAC
- [Roles e Bindings](rbac/roles-and-bindings.md) — Role, ClusterRole, RoleBinding, ClusterRoleBinding, privilege escalation

### Admission
- [Webhooks](admission/webhooks.md) — plugins builtin, ValidatingWebhook, MutatingWebhook, failurePolicy, exemplo em Go, OPA/Kyverno

### Kubelet
- [Segurança](authentication/kubelet-ports.md) — portas, anonymous auth, modo webhook

### Networking
- [kube-proxy e port-forward](networking/kube-proxy-port-forward.md) — iptables, fluxo do port-forward, NodePort
- [Ingress — On-prem vs Managed](networking/ingress-onprem-vs-managed.md) — MetalLB, GKE NEG, Gateway API
- [CNI vs kube-proxy](networking/cni-kube-proxy.md) — divisão de responsabilidades, Cilium kube-proxy replacement, overlay vs underlay, VPC-native

### Hardening
- [Verificar Binários](hardening/verify-binaries.md) — sha256sum, sha512sum, imagens com digest
- [Cluster Upgrade](hardening/cluster-upgrade.md) — drain, uncordon, kubeadm step-by-step
- [Node Metadata](hardening/node-metadata.md) — 169.254.169.254, NetworkPolicy, Workload Identity
- [Seccomp](hardening/seccomp.md) — syscalls, modos BPF, ações, runtimes, perfis no k8s, managed clusters, Tracee
- [eBPF](hardening/ebpf.md) — verifier, hooks, BPF maps, Cilium, Falco/Tracee, conexão com seccomp
- [AppArmor](hardening/apparmor.md) — LSM path-based, profiles, aa-genprof, k8s, managed clusters
- [Capabilities](hardening/capabilities.md) — conjuntos (Bounding/Permitted/Effective), runtime defaults, por que non-root importa

### Audit
- [Kubernetes Audit Logging](audit/kubernetes-audit-logging.md) — on-prem vs managed, policy, backends

---

## Incidentes de referência

- [2026-04-20 — RCE em container Next.js (IPGC)](incidents/2026-04-20-ipgc-rce.md)
