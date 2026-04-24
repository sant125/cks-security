# Sockets, Protocolos L4 e Superfície de Ataque

## Socket não é TCP

Socket é uma abstração do kernel pra comunicação — uma API genérica. O protocolo é parâmetro, não parte do socket em si:

```c
socket(domain, type, protocol)

socket(AF_INET, SOCK_STREAM, 0)             // TCP
socket(AF_INET, SOCK_DGRAM, 0)             // UDP
socket(AF_INET, SOCK_STREAM, IPPROTO_SCTP) // SCTP
socket(AF_INET, SOCK_DCCP,  IPPROTO_DCCP) // DCCP
```

## Todos os protocolos de transporte vivem em L4

```
L7  aplicação     HTTP, gRPC, DNS
L4  transporte    TCP / UDP / SCTP / DCCP / RDS / TIPC  ← todos aqui
L3  rede          IP (IPv4, IPv6)
L2  enlace        Ethernet, WiFi
```

Diferenças de comportamento:

| Protocolo | Conexão | Confiável | Ordenado | Caso de uso real |
|-----------|---------|-----------|----------|------------------|
| TCP | sim | sim | sim | HTTP, SSH, banco de dados |
| UDP | não | não | não | DNS, streaming, jogos |
| SCTP | sim | sim | sim | SS7 telecom, VoIP de operadora |
| DCCP | sim | não | não | VoIP com controle de congestionamento |

## Por que TCP não é módulo e SCTP é

TCP está compilado diretamente no kernel (`built-in`). É pré-requisito do boot — sem TCP o sistema não sobe. Não existe `rmmod tcp`.

SCTP, DCCP, RDS, TIPC são opcionais. Ficam em disco como `.ko` e só ocupam memória se alguém pedir. Pode verificar:

```bash
grep CONFIG_IP_DCCP /boot/config-$(uname -r)
# CONFIG_IP_DCCP=m    ← m = module, carregável

grep CONFIG_TCP /boot/config-$(uname -r)
# CONFIG_TCP_CONG_CUBIC=y   ← y = built-in, sempre presente
```

## O mecanismo de autoload é o vetor

Quando você chama `socket()` com um protocolo não carregado, o kernel executa automaticamente `modprobe <protocolo>` antes de retornar. Sem nenhuma permissão especial.

```
socket(AF_INET, SOCK_STREAM, IPPROTO_SCTP)
    └── kernel: "sctp não está carregado"
    └── kernel executa: modprobe sctp
        ├── sem /bin/false → sctp.ko carrega no HOST, bug fica exposto
        └── com /bin/false → falha aqui, processo recebe EPROTONOSUPPORT
```

Demonstração prática:
```bash
lsmod | grep sctp    # vazio

# qualquer processo roda:
python3 -c "import socket; socket.socket(socket.AF_INET, socket.SOCK_STREAM, socket.IPPROTO_SCTP)"

lsmod | grep sctp
# sctp    421888  0    ← carregou sozinho pelo autoload
```

## Encadeamento do ataque em Kubernetes

```
Pod malicioso
    └── processo abre socket SCTP
        └── kernel do NODE carrega sctp.ko via autoload
            └── exploit (ex: CVE-2019-8956 use-after-free)
                └── root no node
                    ├── /var/lib/kubelet/pods/  (secrets montados)
                    ├── credenciais do kubelet
                    ├── outros pods no mesmo node
                    └── lateral movement pro control plane
```

## Portas abertas no Kubernetes — ss vs netstat

`ss` é o substituto moderno do `netstat` (net-tools está deprecated):

```bash
ss -tlnp    # tcp listening, numeric, com processo
ss -ulnp    # udp
ss -tlnp | grep 6443
```

Flags:
```
-t   tcp
-u   udp
-l   listening
-n   numeric (não resolve nomes)
-p   processo/pid dono do socket
```

### Portas esperadas por componente

```
control plane
  6443        kube-apiserver       (toda comunicação entra aqui)
  2379-2380   etcd                 (só etcd members + apiserver)
  10250       kubelet
  10257       kube-controller-manager
  10259       kube-scheduler

worker node
  10250       kubelet
  30000-32767 NodePort range
```

## Managed vs onprem — camada de autenticação

Em clouds managed (EKS, GKE, AKS), existe uma camada de autenticação antes do apiserver:

```
kubectl → endpoint gerenciado do cloud provider
              └── IAM/OIDC da cloud (autentica aqui)
                  └── kube-apiserver (você nunca acessa direto)
```

No EKS: `aws-iam-authenticator` valida o token AWS antes de chegar no apiserver. O RBAC do Kubernetes ainda existe, mas a primeira camada é IAM.

No kubeadm onprem:
```
kubectl → :6443 direto no apiserver → certificado client ou token
```

Sem camada na frente — você é responsável por firewall, VPN, bastion. Por isso hardening de portas importa mais em onprem.

## Parâmetros de kernel pra Kubernetes (sysctl)

Módulos e parâmetros precisam persistir entre reboots:

```bash
# /etc/modules-load.d/k8s.conf — carrega módulos no boot
overlay
br_netfilter

# /etc/sysctl.d/k8s.conf — parâmetros do kernel
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
```

`br_netfilter` precisa estar carregado para o parâmetro `bridge-nf-call-iptables` existir em `/proc/sys/`. O módulo expõe o parâmetro — sem ele, o sysctl não tem onde escrever.
