# Cluster Upgrade — kubeadm

Upgrade de um cluster kubeadm segue uma ordem específica: control plane primeiro, workers depois. Nunca pula versão minor (1.28 → 1.29 → 1.30).

---

## Ordem obrigatória

```
1. Upgrade do control plane
   └── kubeadm → apiserver → controller-manager → scheduler → etcd

2. Upgrade dos workers (um por vez)
   └── drain → upgrade kubelet → uncordon
```

---

## Upgrade do Control Plane

### 1. Atualizar o kubeadm

```bash
# Ver versão disponível
apt-cache madison kubeadm

# Atualizar o pacote kubeadm
apt-mark unhold kubeadm
apt-get install -y kubeadm=1.29.0-00
apt-mark hold kubeadm

# Verificar
kubeadm version
```

### 2. Verificar o plano de upgrade

```bash
kubeadm upgrade plan
# Mostra: versão atual, versão disponível, componentes que serão atualizados
```

### 3. Aplicar o upgrade

```bash
kubeadm upgrade apply v1.29.0
# Atualiza: apiserver, controller-manager, scheduler, etcd, kube-proxy
# Gera novos manifestos estáticos em /etc/kubernetes/manifests/
```

### 4. Atualizar kubelet e kubectl no control plane

```bash
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.29.0-00 kubectl=1.29.0-00
apt-mark hold kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet
```

---

## Upgrade dos Workers

Repetir para cada worker, um por vez:

### 1. Drenar o node (no control plane)

```bash
# Remove pods do node e impede agendamento de novos
kubectl drain node-worker-1 --ignore-daemonsets --delete-emptydir-data

# --ignore-daemonsets: DaemonSet pods não podem ser removidos, só ignorados
# --delete-emptydir-data: pods com emptyDir perdem os dados (esperado)
```

O node fica com status `SchedulingDisabled` (cordon implícito).

### 2. Atualizar no próprio worker

```bash
# SSH no worker
ssh worker-1

# Atualizar kubeadm
apt-mark unhold kubeadm
apt-get install -y kubeadm=1.29.0-00
apt-mark hold kubeadm

# Upgrade do node (só atualiza o kubelet config, não os componentes do control plane)
kubeadm upgrade node

# Atualizar kubelet
apt-mark unhold kubelet
apt-get install -y kubelet=1.29.0-00
apt-mark hold kubelet

systemctl daemon-reload
systemctl restart kubelet
```

### 3. Recolocar o node no cluster (no control plane)

```bash
kubectl uncordon node-worker-1
# Node volta a receber pods
```

### 4. Verificar

```bash
kubectl get nodes
# NAME            STATUS   ROLES           AGE   VERSION
# control-plane   Ready    control-plane   10d   v1.29.0
# worker-1        Ready    <none>          10d   v1.29.0
```

---

## drain vs cordon

| Comando | O que faz |
|---|---|
| `kubectl cordon NODE` | Marca como `SchedulingDisabled` — novos pods não chegam, pods atuais ficam |
| `kubectl drain NODE` | Cordon + remove pods existentes do node |
| `kubectl uncordon NODE` | Remove `SchedulingDisabled` — node volta a receber pods |

---

## Pontos de atenção

**PodDisruptionBudget:** se um deployment tiver PDB com `minAvailable`, o drain pode travar esperando que outros pods estejam saudáveis antes de remover os do node.

```bash
# Se o drain travar, verificar PDBs
kubectl get pdb -A
```

**StatefulSets:** pods com dados locais (emptyDir) perdem dados no drain — esperado e correto para stateless. StatefulSets com PVCs mantêm os dados (PVC persiste).

**etcd:** em clusters com etcd externo, o etcd não é atualizado pelo `kubeadm upgrade apply` — precisa de upgrade separado.

---

## Resumo visual

```
Control plane
  ├── apt install kubeadm=1.29
  ├── kubeadm upgrade apply v1.29.0
  └── apt install kubelet=1.29 kubectl=1.29 + restart kubelet

Worker (repete para cada um)
  ├── kubectl drain worker-X (no control plane)
  ├── apt install kubeadm=1.29 (no worker)
  ├── kubeadm upgrade node (no worker)
  ├── apt install kubelet=1.29 + restart kubelet (no worker)
  └── kubectl uncordon worker-X (no control plane)
```
