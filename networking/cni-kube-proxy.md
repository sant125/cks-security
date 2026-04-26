# CNI vs kube-proxy — Divisão de Responsabilidades

## Os dois resolvem problemas diferentes

Muita confusão vem de achar que CNI e kube-proxy fazem a mesma coisa. Eles operam em camadas distintas:

| | CNI | kube-proxy |
|--|-----|-----------|
| Responsável por | Pod networking | Service routing |
| Configura | veth pairs, IPs dos pods, rotas, NetworkPolicy | regras DNAT/SNAT para ClusterIP, NodePort, LoadBalancer |
| Gatilho | kubelet chama ao criar/destruir pod | assiste API server por mudanças em Service/Endpoints |
| Conversa com o outro? | não diretamente | não diretamente |

## kube-proxy: Service routing via iptables

Quando você cria um Service (ClusterIP), o kube-proxy:
1. Detecta a mudança no API server
2. Escreve regras DNAT no iptables do node
3. `ClusterIP:porta` → sorteia um pod IP via regras de estatísticas do iptables → DNAT

```
você acessa ClusterIP:80
        ↓
iptables DNAT (kube-proxy escreveu essa regra)
ClusterIP:80 → PodIP:8080  (selecionado aleatoriamente entre os endpoints)
        ↓
CNI roteia o pacote até o pod destino
```

Eles não conversam — o kube-proxy resolve o endereço, a CNI roteia o pacote resultante.

## CNI: pod networking e NetworkPolicy

O kubelet chama o plugin CNI via interface padrão (spec JSON) toda vez que um pod é criado:

```
pod nasce
    ↓
kubelet → chama CNI plugin
    ↓
CNI:
  - cria veth pair (uma ponta no pod, outra no node)
  - atribui IP ao pod via IPAM
  - configura rotas no node
  - programa NetworkPolicy (se suportado)
```

**IPAM** (IP Address Management) é parte da CNI. Cada node tem um CIDR reservado (ex: `10.244.1.0/24`). A CNI gerencia o pool de IPs dentro desse bloco e garante que não há conflito.

## CNIs comuns: o que cada uma faz

| CNI | Pod networking | NetworkPolicy | Service routing |
|-----|---------------|---------------|----------------|
| Flannel | VXLAN overlay | ❌ não implementa | kube-proxy |
| Calico | BGP ou VXLAN | iptables ou eBPF | kube-proxy |
| Weave | overlay mesh | sim | kube-proxy |
| **Cilium** | eBPF | eBPF | **substitui kube-proxy** |

Flannel não implementa NetworkPolicy — se precisar de NetworkPolicy com Flannel, precisa de outro plugin (ex: Calico em modo policy-only).

## Cilium: modo kube-proxy replacement

O Cilium é o único CNI que pode substituir completamente o kube-proxy:

```
Cilium agent assiste o API server diretamente
        ↓
Detecta mudança em Service/Endpoints
        ↓
Programa BPF maps (hash maps, O(1)) em vez de iptables
        ↓
Hooks XDP/TC interceptam pacotes antes do netfilter
```

Com isso:
- kube-proxy não precisa existir no cluster
- Sem regras iptables lineares pra Services
- Performance constante independente do número de Services
- GKE já oferece nativamente (`--dataplane=cilium` em GKE Standard)

## O problema de Calico + kube-proxy juntos

Calico programa iptables pra NetworkPolicy. kube-proxy programa iptables pra Services. Com cluster grande:

```
pacote chega
    ↓
iptables: regras de NetworkPolicy (Calico) + regras de Service (kube-proxy)
    → potencialmente 20k+ regras, avaliadas linearmente
```

Ambos escrevendo no iptables ao mesmo tempo também complica troubleshooting e pode gerar conflitos. Cilium escapa desse problema por design.

## Overlay vs Underlay (VPC-native)

### Overlay

A rede dos pods é uma rede virtual encapsulada em cima da rede do host:

```
pod A → pod B (node diferente)
        ↓
pacote encapsulado em UDP/VXLAN
VPC só vê: node-A → node-B (UDP)
node-B desencapsula, entrega ao pod B
```

- Mais simples de operar — não precisa coordenar com a VPC
- Overhead de encapsulação
- Pod IPs **não são roteáveis** fora do cluster

### Underlay (VPC-native)

A CNI fala com a API da cloud e integra os pod IPs diretamente na VPC:

| Cloud | Mecanismo | Resultado |
|-------|-----------|-----------|
| GKE (`--enable-ip-alias`) | CNI reserva alias IPs na VPC via GCP API | pod IP = IP real da VPC |
| EKS (`amazon-vpc-cni`) | CNI faz attach de ENIs no node via EC2 API | pod IP = IP real da subnet |
| AKS | CNI integra com VNet | pod IP = IP real da VNet |

```
pod A → pod B (node diferente)
        ↓
sem encapsulação
VPC roteia diretamente pelo IP do pod
```

- Pod IPs são **roteáveis** diretamente na VPC
- Outros serviços (RDS, VMs externas) falam com pods pelo IP sem NAT
- Requer planejamento de CIDR: pod CIDR não pode conflitar com VPC CIDR
- Precisa de IPs suficientes na subnet para pods (EKS consome IPs da subnet por node)

### Quando usar cada um

| Cenário | Recomendação |
|---------|-------------|
| Cluster simples, on-prem | Overlay (Flannel/Calico VXLAN) |
| Cloud, precisa integrar com outros serviços da VPC | Underlay (VPC-native) |
| Alta performance, escala | Underlay + Cilium eBPF |
| Segurança + observabilidade profunda | Cilium |

## Referências

- [Kubernetes — Network Plugins (CNI)](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
- [Cilium — kube-proxy replacement](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/)
- [AWS VPC CNI](https://github.com/aws/amazon-vpc-cni-k8s)
- [GKE VPC-native clusters](https://cloud.google.com/kubernetes-engine/docs/concepts/alias-ips)
