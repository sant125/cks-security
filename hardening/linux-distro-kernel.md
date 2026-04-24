# Linux: Kernel, Distribuições e Escolhas de Stack

## Kernel vs Distribuição

**Linux é só o kernel** — gerenciador de recursos (CPU, memória, I/O, rede). Por si só não tem shell, gerenciador de pacotes, nem nada útil pro usuário. O que o kernel faz:
- Agenda processos
- Gerencia memória
- Fala com hardware via drivers/módulos
- Expõe syscalls pra userspace

**Distribuição = kernel + userland**. Uma distro pega o kernel e monta tudo em cima:

```
┌─────────────────────────────────────────────┐
│  aplicações       nginx, postgres, kubectl  │
├─────────────────────────────────────────────┤
│  ferramentas      bash, coreutils, systemd  │
├─────────────────────────────────────────────┤
│  gerenciador      apt / dnf / pacman / apk  │
├─────────────────────────────────────────────┤
│  libc             glibc / musl              │
├─────────────────────────────────────────────┤
│  KERNEL           Linux                     │
└─────────────────────────────────────────────┘
```

## Escolhas que definem uma distro

### Init system — PID 1
```
systemd   → Ubuntu, Debian, Fedora, RHEL, Arch
OpenRC    → Alpine, Gentoo
```
systemd virou muito mais que init: logging (journald), rede (networkd), DNS (resolved). Alpine usa OpenRC por minimalismo — por isso imagens Alpine são menores e mais comuns em containers.

### libc — biblioteca C fundamental
```
glibc  → Ubuntu, Debian, Fedora, RHEL  (completa, ~2MB)
musl   → Alpine                         (minimalista, ~1MB)
```
Binários compilados pra glibc podem não rodar em Alpine (musl). Causa comum de "funciona na imagem Ubuntu, quebra na Alpine".

### Gerenciador de pacotes
```
apt/dpkg  → Debian/Ubuntu  (.deb)
dnf/rpm   → Fedora/RHEL    (.rpm)
apk       → Alpine
pacman    → Arch
```

### LSM (Linux Security Module) — impacta hardening
```
AppArmor  → Ubuntu, Debian  (perfis por caminho de arquivo)
SELinux   → RHEL, Fedora    (políticas por contexto/label)
```
O CKS cobra os dois. A filosofia de configuração é completamente diferente.

## Firewall: ufw vs firewalld

Ambos são frontends pro mesmo subsistema do kernel: **netfilter**.

```
ufw / firewalld / iptables
        ↓
   nftables (kernel)
        ↓
   netfilter
        ↓
   pacotes de rede
```

**ufw** (Ubuntu/Debian) — simples, comandos diretos, sem zonas:
```bash
ufw allow 22/tcp
ufw enable
```

**firewalld** (RHEL/Fedora) — orientado a zonas (`public`, `internal`, `trusted`):
```bash
firewall-cmd --permanent --add-service=ssh
firewall-cmd --reload
```

Qualquer um que você configure, as regras reais chegam no kernel via nftables:
```bash
nft list ruleset   # ver o que realmente tá no kernel
```

## Filesystems: ext4 vs XFS

### ext4
- Default histórico Debian/Ubuntu
- Bom pra arquivos pequenos/médios e uso geral
- `fsck` lento em volumes grandes
- Suporta resize online (crescer), não encolhe

### XFS
- Default RHEL/Fedora/Amazon Linux 2
- Projetado pra I/O paralelo e arquivos grandes
- `xfs_repair` rápido mesmo em volumes multi-TB
- Ruim pra muitos arquivos pequenos
- **Nunca encolhe** — pra reduzir, formata de novo

```
ext4 → uso geral, containers, workloads genéricos
XFS  → banco de dados, Kafka, Elasticsearch, volumes grandes
```

No Kubernetes, EBS da AWS com Amazon Linux 2 vem XFS por default. PersistentVolumes pra Postgres ou workloads de escrita pesada → XFS + gp3/io2.

## O kernel como sistema de plugins

O kernel expõe interfaces bem definidas pra cada subsistema:

```
subsistema          interface
────────────────────────────────────────────
processos         → /proc/
dispositivos      → /dev/
parâmetros        → /proc/sys/  (sysctl)
firewall          → netfilter hooks
segurança         → LSM hooks
módulos           → syscall init_module
filesystem        → VFS
observabilidade   → tracepoints (eBPF pluga aqui)
```

A distro escolhe o que plugar em cada hook. O kernel não impõe as escolhas — só fornece os pontos de encaixe. Por isso Linux roda desde smartwatch até supercomputador com o mesmo kernel base.
