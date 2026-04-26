# Linux Capabilities — Dividindo os Superpoderes do Root

## O problema

O modelo Unix clássico é binário: root tem tudo, não-root tem nada. Um processo que precisa só de uma operação privilegiada (bindar porta 80, usar raw socket) teria que rodar como root inteiro — com poder de carregar módulos no kernel, matar qualquer processo, etc.

**Capabilities** dividem os privilégios do root em ~40 unidades independentes. Você concede só o que o processo precisa.

## Capabilities relevantes

| Capability | O que permite |
|-----------|--------------|
| `CAP_NET_BIND_SERVICE` | bindar portas < 1024 |
| `CAP_NET_RAW` | raw sockets — ping, sniffing, ARP |
| `CAP_NET_ADMIN` | configurar interfaces, iptables, rotas |
| `CAP_SYS_ADMIN` | mount, ptrace, muitos outros — o "canivete suíço" |
| `CAP_SYS_MODULE` | carregar/descarregar módulos `.ko` no kernel |
| `CAP_SYS_PTRACE` | ptrace em qualquer processo |
| `CAP_DAC_OVERRIDE` | ignora permissões de arquivo (lê/escreve qualquer coisa) |
| `CAP_SETUID` / `CAP_SETGID` | muda UIDs/GIDs arbitrariamente |
| `CAP_KILL` | envia signal pra qualquer processo |
| `CAP_SYS_CHROOT` | chroot arbitrário |

## Os conjuntos — mental model

Cada processo carrega 3 conjuntos principais empilhados:

```
┌─────────────────────────────────────────────┐
│  BOUNDING                                   │  ← teto absoluto. Ninguém passa disso.
│  definido pelo pai, filho não pode aumentar │     Nem root ultrapassa.
├─────────────────────────────────────────────┤
│  PERMITTED                                  │  ← pool disponível pro processo
│  subconjunto do Bounding                    │     pode mover caps daqui pro Effective
├─────────────────────────────────────────────┤
│  EFFECTIVE                                  │  ← o kernel verifica ESSE
│  o que está ativo agora                     │     é daqui que vem allow/deny
└─────────────────────────────────────────────┘
```

**Effective** — o único que o kernel consulta. Processo tenta bindar porta 80 → kernel verifica se `CAP_NET_BIND_SERVICE` está em Effective. Não está → `EPERM`.

**Permitted** — o pool. O processo pode mover caps do Permitted pro Effective via código (`capset()`). Nunca pode adicionar ao Permitted o que não estava lá.

**Bounding** — o teto inviolável. Cap fora do Bounding nunca entra no Permitted, nunca entra no Effective. Definido pelo processo pai. Filho não tem como aumentar.

### Os outros dois conjuntos (execve)

**Inheritable** — caps que você quer passar pro filho via execve, mas o filho também precisa ter a cap no próprio Inheritable (via file cap). Os dois precisam concordar.

**Ambient** — caps que passam automaticamente pro filho via execve sem precisar de file caps. Filho herda sem fazer nada. (kernel 4.3+)

## Root no host vs root no container

```
root no host:
  Bounding:  conjunto completo (systemd herda de init, filho herda de pai)
  Effective: conjunto completo
  → pode tudo

root no container (runtime padrão):
  Bounding:  subset — runtime dropou ~15 caps antes de criar o container
  Effective: esse mesmo subset
  → root mas limitado pelo Bounding
```

Root dentro do container tentando adicionar `CAP_SYS_MODULE`:
```
não está no Bounding → não pode entrar no Permitted
não está no Permitted → não pode entrar no Effective
→ impossível, mesmo sendo root
```

## Caps que o runtime deixa por padrão (Docker/containerd)

**Incluídas:**
```
CAP_CHOWN, CAP_DAC_OVERRIDE, CAP_FOWNER, CAP_FSETID,
CAP_KILL, CAP_NET_BIND_SERVICE, CAP_NET_RAW,
CAP_SETGID, CAP_SETUID, CAP_SETFCAP, CAP_SETPCAP,
CAP_SYS_CHROOT, CAP_AUDIT_WRITE
```

**Removidas do Bounding (mesmo sendo root):**
```
CAP_SYS_ADMIN, CAP_SYS_MODULE, CAP_SYS_PTRACE,
CAP_NET_ADMIN, CAP_SYS_BOOT, CAP_MAC_ADMIN, ...
```

## Por que não rodar como root mesmo com caps dropadas

O runtime já faz bastante — mas não-root é a camada que protege quando as anteriores falham.

**1. Runtime tem CVEs**

Escapes de container reais exigem ou são muito mais fáceis sendo root:
- CVE-2019-5736 (runc) — root no container reescrevia o binário runc no host
- CVE-2022-0185 (kernel) — privilege escalation
- CVE-2022-0811 (CRI-O) — path traversal

Root no container + escape bem-sucedido = root no host imediatamente.  
Não-root no container + escape = ainda precisa escalar localmente primeiro.

**2. Caps "seguras" do default que não são triviais**

```
CAP_NET_RAW    → raw sockets → ARP spoofing, sniffing na rede do pod
CAP_DAC_OVERRIDE → ignora permissões de arquivo dentro do container
CAP_SETUID     → muda para qualquer UID
```

Processo não-root não ativa essas caps automaticamente mesmo que estejam no Bounding. Processo root ativa.

**3. SUID binaries**

Imagens frequentemente têm binários SUID (`su`, `passwd`, `newgrp`). Root executando SUID → cap do arquivo ativa diretamente. Não-root → geralmente não ganha nada.

**4. allowPrivilegeEscalation**

Default do k8s é `allowPrivilegeEscalation: true` — processo pode ganhar mais caps via execve de SUID. Não-root + `allowPrivilegeEscalation: false` fecha esse vetor completamente.

**5. RCE na aplicação**

```
RCE como root:   attacker já tem root no container → tenta escape direto
RCE como uid 1000: attacker precisa escalar pra root dentro do container primeiro
```

Cada passo extra é uma chance a mais de detectar ou o ataque falhar.

## Kubernetes — como controlar

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]                    # começa do zero
    add: ["NET_BIND_SERVICE"]        # adiciona só o necessário
```

`drop: ["ALL"]` zera o Permitted e o Bounding do container. `add` adiciona de volta só o que precisa. É o modelo mais seguro.

## Hardening de serviço no host (systemd)

Para serviços rodando diretamente no host, você pode restringir o Bounding via unit:

```ini
[Service]
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
```

Nginx pode bindar porta 80 sem precisar de root, com Bounding limitado a uma cap só.

## Como inspecionar caps de um processo

```bash
cat /proc/$$/status | grep Cap
# CapInh: 0000000000000000
# CapPrm: 0000003fffffffff
# CapEff: 0000003fffffffff
# CapBnd: 0000003fffffffff
# CapAmb: 0000000000000000

capsh --decode=0000003fffffffff   # decodifica o hex pra nomes legíveis
```

## As camadas juntas

```
Seccomp      → filtra quais syscalls o processo pode chamar
Capabilities → filtra quais superpoderes o processo pode usar
Non-root     → garante que caps ativas sejam mínimas mesmo com falha nas camadas acima
AppArmor     → filtra quais arquivos/paths/rede o processo pode tocar
```

Cada camada independente. Se uma falha ou tem CVE, as outras ainda estão de pé.

## Referências

- [Linux man — capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [Kubernetes — Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Docker — Runtime privilege and capabilities](https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities)
