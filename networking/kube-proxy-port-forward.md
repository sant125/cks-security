# kube-proxy e port-forward — Como o tráfego flui

## kube-proxy

O kube-proxy roda em cada node como DaemonSet e é responsável por implementar os Services do Kubernetes — ou seja, fazer o tráfego para um ClusterIP chegar nos pods certos.

### O que ele faz

```
Você cria um Service
        │
        ▼
apiserver salva no etcd
        │
        ▼
kube-proxy (em cada node) recebe o evento via watch
        │
        ▼
kube-proxy programa regras no kernel do node
        │
        ├── modo iptables (padrão): regras de DNAT
        └── modo ipvs: tabelas de virtual server
```

### Modo iptables (padrão)

```
Pod A → ClusterIP:80
        │
        ▼
iptables PREROUTING
        │  DNAT: ClusterIP:80 → PodIP:8080 (escolhido aleatoriamente entre endpoints)
        ▼
Pod B (destino real)
```

O ClusterIP **não existe como interface de rede** — é uma ficção mantida pelas regras de iptables. O kube-proxy não é um proxy de verdade no caminho dos pacotes — só programa as regras e some.

```bash
# Ver as regras geradas pelo kube-proxy para um service
iptables -t nat -L KUBE-SERVICES -n | grep <ClusterIP>
iptables -t nat -L KUBE-SVC-XXXXX -n   # regras de load balance entre pods
```

---

## kubectl port-forward

O port-forward é diferente — ele realmente é um proxy no caminho dos pacotes, e passa pelo apiserver.

### Fluxo completo

```
kubectl port-forward pod/meu-pod 8080:80
        │
        ▼
kubectl abre conexão WebSocket com apiserver
        │   GET /api/v1/namespaces/default/pods/meu-pod/portforward
        ▼
apiserver verifica RBAC:
        │   verbo: create, recurso: pods/portforward
        ▼
apiserver conecta no kubelet do node onde o pod está
        │   usando apiserver-kubelet-client.crt (mTLS)
        ▼
kubelet abre canal para dentro do container na porta 80
        │
        ▼
kubectl escuta na porta local 8080
        │
        ▼
curl localhost:8080 → tráfego vai por: kubectl → apiserver → kubelet → container
```

### Implicações

- **Latência:** mais hops que acesso direto — não usar para benchmarks
- **RBAC necessário:** `create` em `pods/portforward`
- **Não é produção:** é para debug. Em prod, usar Service + Ingress
- **TLS em todo caminho:** kubectl↔apiserver é TLS, apiserver↔kubelet é mTLS

```bash
# RBAC para permitir port-forward
rules:
- apiGroups: [""]
  resources: ["pods/portforward"]
  verbs: ["create"]
```

---

## Comparação: kube-proxy vs port-forward

| | kube-proxy | port-forward |
|---|---|---|
| Propósito | Implementar Services (produção) | Debug/desenvolvimento |
| Caminho dos pacotes | Kernel (iptables/ipvs) — sem hop extra | kubectl → apiserver → kubelet → pod |
| Latência | Mínima (kernel space) | Alta (múltiplos hops) |
| Autenticação | Sem — qualquer processo no node acessa | RBAC via apiserver |
| Escala | Automático, todos os nodes | Sessão única, processo kubectl |

---

## NodePort e LoadBalancer — o kube-proxy também gerencia

```
NodePort Service
        │
        ▼
kube-proxy programa: qualquer tráfego em NODE_IP:NodePort
        │              → DNAT para ClusterIP → pod
        ▼
Funciona em qualquer node do cluster, mesmo sem o pod nele
```

```bash
# Ver NodePorts ativos
kubectl get svc -A | grep NodePort

# Ver a regra no iptables
iptables -t nat -L KUBE-NODEPORTS -n
```

---

## On-prem: tráfego externo sem cloud LB

Sem um cloud provider, o LoadBalancer type fica em `<pending>`. Por isso no on-prem se usa MetalLB — ele implementa o controller de LoadBalancer e anuncia o IP via ARP (L2) ou BGP (L3).

```
MetalLB speaker (DaemonSet em cada node)
        │
        ├── modo L2: responde ARP para o IP do Service — um node anuncia, faz failover
        └── modo BGP: anuncia o IP para o roteador — balanceamento real por ECMP

Tráfego externo → IP anunciado pelo MetalLB → node que recebeu
        │
        ▼
kube-proxy DNAT → pod (em qualquer node via overlay)
```
