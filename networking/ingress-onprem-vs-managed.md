# Ingress — On-prem vs Managed

## O que é o Ingress

O Ingress é um objeto Kubernetes que define regras de roteamento HTTP/HTTPS. Sozinho não faz nada — precisa de um **Ingress Controller** rodando no cluster para implementar as regras.

```
Ingress (objeto K8s — só regras)
        │
        ▼
Ingress Controller (pod rodando no cluster — implementa as regras)
        │
        ▼
Load Balancer / proxy real (nginx, traefik, ALB, etc.)
```

---

## On-prem — o problema do LoadBalancer

Em cloud, quando você cria um `Service type: LoadBalancer`, o cloud controller provisiona um LB externo automaticamente. On-prem isso não existe.

```
On-prem sem MetalLB:
Service type: LoadBalancer
EXTERNAL-IP: <pending>    ← fica aqui para sempre
```

### MetalLB — solução on-prem

MetalLB implementa o controller de LoadBalancer para on-prem. Ele anuncia IPs do pool configurado para a rede.

```yaml
# Pool de IPs disponíveis para Services LoadBalancer
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: producao
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.100-192.168.1.110   # range de IPs livres na sua rede
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-advert
  namespace: metallb-system
spec:
  ipAddressPools:
  - producao
```

**Modo L2 (ARP):**
```
Client faz ARP: "quem tem 192.168.1.100?"
MetalLB speaker no node líder responde: "sou eu"
        │
        ▼
Tráfego vai para esse node
kube-proxy faz DNAT para o pod (pode estar em outro node)
```

Limitação: um node recebe todo o tráfego (não é balanceamento real). Se esse node cair, o speaker de outro node assume via ARP gratuitous (failover).

**Modo BGP:**
```
MetalLB anuncia o IP para o roteador via BGP
Roteador distribui o tráfego entre os nodes via ECMP
Balanceamento real — cada node recebe uma fração do tráfego
```

---

## Managed — GKE (como funciona no byxmori)

No GKE você tem duas opções: Ingress clássico e Gateway API.

### Ingress clássico (GKE Ingress Controller)

```
Ingress object no K8s
        │
        ▼
GKE Ingress Controller (gerenciado, roda no control plane)
        │
        ▼
Provisiona Global External ALB no GCP
        │  com: backend services, URL map, forwarding rule
        ▼
Tráfego externo → ALB → NEG (Network Endpoint Group)
                              │
                              └── aponta diretamente para PodIP:Port
```

**NEG (Network Endpoint Group):** no GKE, o ALB não bate no NodePort do Service — ele bate **direto no IP do pod**. Isso é o modo nativo do GKE.

### Gateway API (o que o byxmori usa)

```
Gateway object + HTTPRoute
        │
        ▼
GKE Gateway Controller
        │
        ▼
Global External ALB (mesmo produto, mais controle)
        │
        ▼
NEG → PodIP direto
```

---

## On-prem vs Managed — comparação de fluxo

### On-prem com MetalLB + nginx-ingress

```
Cliente → MetalLB IP → nginx-ingress pod (ClusterIP via kube-proxy)
                               │
                               ▼ (nginx faz proxy)
                        Service ClusterIP
                               │
                               ▼ (kube-proxy DNAT)
                           Pod da app
```

Hops: **cliente → LB IP → nginx → kube-proxy → pod**

### Managed com GKE + NEG

```
Cliente → ALB (GCP) → NEG → PodIP direto
```

Hops: **cliente → ALB → pod**

**Menos hops = menos latência = melhor performance.** O ALB managed bate direto no pod, sem passar pelo nginx intermediário nem pelo kube-proxy.

---

## Quando usar cada abordagem

| Cenário | Solução |
|---|---|
| On-prem, tráfego L2 (mesma rede) | MetalLB modo L2 + nginx-ingress |
| On-prem, roteador BGP disponível | MetalLB modo BGP + nginx-ingress |
| GKE, app simples | GKE Ingress + NEG |
| GKE, controle fino de roteamento | Gateway API + GKE Gateway Controller |
| GKE, múltiplos times gerenciando rotas | Gateway API (Gateway separado das HTTPRoutes) |

---

## Ingress Controller vs Gateway API — diferença de modelo

```
Ingress (modelo antigo):
  Um objeto faz tudo: TLS, regras de host, regras de path
  Annotations específicas de cada controller (não portável)

Gateway API (modelo novo):
  Gateway → quem escuta (porta, TLS, endereço)
  HTTPRoute → regras de roteamento (host, path, headers)
  Separação de responsabilidade: infra configura Gateway, dev configura HTTPRoute
```
