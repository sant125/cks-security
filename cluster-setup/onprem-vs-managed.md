# On-prem vs Managed — Visão de Segurança

## O que muda entre os dois modelos

```
ON-PREM (kubeadm)                    MANAGED (GKE, EKS, AKS)
┌─────────────────────┐              ┌─────────────────────────────┐
│  control plane      │              │  control plane              │
│  ┌───────────────┐  │              │  ┌───────────────────────┐  │
│  │  apiserver    │  │              │  │  apiserver            │  │
│  │  etcd         │  │              │  │  etcd                 │  │
│  │  scheduler    │  │              │  │  controller-manager   │  │
│  │  controller   │  │              │  └───────────────────────┘  │
│  └───────────────┘  │              │  gerenciado pelo provedor   │
│  você gerencia      │              │  você NÃO loga nesses nós   │
├─────────────────────┤              ├─────────────────────────────┤
│  workers            │              │  workers                    │
│  você acessa via SSH│              │  você pode acessar via SSH  │
│  você gerencia OS   │              │  mas o provedor gerencia OS │
└─────────────────────┘              └─────────────────────────────┘
```

---

## Responsabilidades por modelo

| Responsabilidade | On-prem | Managed |
|---|---|---|
| Atualizar control plane | Você | Provedor |
| Atualizar workers | Você | Você (ou automático) |
| Certificados do cluster | Você | Provedor |
| Logs do apiserver | Você configura | Provedor oferece integrado |
| Acesso SSH ao control plane | Sim | Não |
| etcd backup | Você | Provedor |
| Auditing | Você configura | Configurável via console/API |

---

## Node Autoupgrade (GKE)

No GKE, os nodes podem ser atualizados automaticamente quando o control plane é atualizado.

```
Control plane atualizado pelo GKE
          ↓
  Node autoupgrade habilitado?
          ├── Sim → GKE drena o node, atualiza, reconecta
          └── Não → node fica na versão antiga até você atualizar manualmente
```

**Como funciona o drain automático:**
1. GKE marca o node como `SchedulingDisabled`
2. Move os pods para outros nodes (respeitando PodDisruptionBudget)
3. Atualiza o node OS + kubelet
4. Recoloca o node no cluster (`uncordon`)

**Implicação de segurança:** com autoupgrade ativo, patches de segurança do kubelet e do OS chegam automaticamente sem intervenção manual — reduz janela de exposição.

---

## Acesso aos nodes em managed

No GKE, você **não tem SSH** para o control plane. Para workers:

```bash
# GKE — acesso ao worker via IAP (Identity-Aware Proxy)
gcloud compute ssh NODE_NAME --tunnel-through-iap

# Ou via kubectl exec em um pod privilegiado (não recomendado em prod)
```

**Implicação:** sem acesso ao control plane, você não consegue:
- Ler `/etc/kubernetes/pki/` diretamente
- Modificar manifestos estáticos dos componentes
- Acessar etcd diretamente

Isso reduz superfície de ataque — mas também reduz visibilidade. O CKS cobra os dois cenários.

---

## Superfície de ataque comparada

| Vetor | On-prem | Managed |
|---|---|---|
| etcd exposto | Risco se mal configurado | Isolado pelo provedor |
| apiserver público | Você controla | Você controla (endpoint público/privado) |
| SSH no control plane | Possível — risco se comprometido | Não disponível |
| Kubelet porta 10250 | Você configura | Geralmente protegido |
| Node metadata API | Você protege | Geralmente protegido por padrão |
