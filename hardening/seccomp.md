# Seccomp — Syscall Filtering no Kernel

## Syscalls: a interface universal

Tudo que roda no sistema — shell, binário compilado (Go, C), interpretado (Python) — precisa do kernel pra interagir com hardware. Essa comunicação é feita via **syscall**: o processo para, passa o pedido pro kernel (ring-0), o kernel executa e devolve o resultado.

```
processo (ring-3, userspace)
    ↓  syscall: open("/etc/passwd", ...)
kernel (ring-0, kernel space)
    ↓  acessa disco, devolve file descriptor
processo recebe resultado
```

O bash que você abre faz syscalls. O binário que você executa faz syscalls. Toda operação com rede, arquivo, stdout, memória — é syscall. Existem ~300+ syscalls no kernel Linux.

## Por que filtrar syscalls?

Algumas syscalls são vetores diretos de ataque em ambientes containerizados:

| Syscall | Risco |
|---------|-------|
| `ptrace` | injeção em processos, bypass de sandbox |
| `mount` | montar filesystems arbitrários no host |
| `init_module` | carregar módulo `.ko` no kernel do host |
| `create_module` | idem |
| `kexec_load` | substituir o kernel em execução |
| `unshare` | criar novos namespaces (privilege escalation) |
| `clone` com flags específicas | escape de namespace |

Lembre: todos os containers compartilham o kernel do host. Um container que chama `init_module` está carregando um módulo no kernel do host — se esse módulo tem CVE, é game over.

## Seccomp: o mecanismo

**Seccomp** (secure computing mode) é uma feature do kernel Linux que permite filtrar syscalls de um processo antes que o kernel as execute. Opera em kernel space, sem overhead de context switch.

### Três modos

| Modo | Comportamento |
|------|--------------|
| `mode 0` | desabilitado (default) |
| `mode 1` | strict — só permite `read`, `write`, `exit`, `sigreturn`. Inutilizável pra containers |
| `mode 2` | filter — programa BPF define o que é permitido. É esse que as runtimes usam |

### Ações disponíveis no modo filter

```
SECCOMP_RET_ALLOW       → executa normalmente
SECCOMP_RET_ERRNO       → retorna erro ao processo (não mata, só falha a syscall)
SECCOMP_RET_KILL_PROCESS → mata o processo inteiro
SECCOMP_RET_TRAP        → envia SIGSYS ao processo
SECCOMP_RET_LOG         → loga sem bloquear ← útil pra construir perfis
SECCOMP_RET_TRACE       → notifica um tracer via eBPF/ptrace
```

`SECCOMP_RET_LOG` é importante: permite observar quais syscalls um processo faz em produção sem impacto, antes de criar um perfil restritivo.

### Como um perfil é definido (JSON)

```json
{
  "defaultAction": "SCCOMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64", "SCMP_ARCH_AARCH64"],
  "syscalls": [
    {
      "names": ["read", "write", "open", "close", "exit", "mmap", "brk"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

`defaultAction` bloqueia tudo. A lista `syscalls` define as exceções. Isso é **allowlist** — o modelo mais seguro.

## Container runtimes e seus defaults

As runtimes implementam seccomp mode 2 por padrão, com um perfil que bloqueia ~44 syscalls perigosas sem quebrar workloads normais.

| Runtime | Default seccomp |
|---------|----------------|
| Docker | enforça por padrão (perfil próprio) |
| containerd | RuntimeDefault habilitado |
| CRI-O | RuntimeDefault habilitado |

Docker aceita um perfil customizado via `--security-opt seccomp=perfil.json`.

O perfil default das runtimes cobre a grande maioria dos workloads: webservices, APIs, workers — eles não precisam de `ptrace`, `init_module` ou `kexec_load`. O default já é seguro e não prejudica apps comuns.

## Kubernetes: como funciona

### Default do Kubernetes

O Kubernetes por padrão roda pods com seccomp **Unconfined** — sem filtragem. Diferente do Docker, ele não aplica o default da runtime automaticamente (exceto se `SeccompDefault` feature gate estiver ativo, GA desde k8s 1.27).

### Tipos de perfil no pod spec

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault    # usa o default da runtime (containerd/crio)
```

```yaml
securityContext:
  seccompProfile:
    type: Localhost
    localhostProfile: profiles/meu-app.json   # relativo ao diretório do kubelet
```

```yaml
securityContext:
  seccompProfile:
    type: Unconfined        # sem filtragem (default atual)
```

### Diretório de perfis no kubelet

Perfis `Localhost` ficam em:
```
/var/lib/kubelet/seccomp/
└── profiles/
    └── meu-app.json
```

O campo `localhostProfile` é relativo a esse diretório.

### Pod-level vs container-level

Pode definir no pod inteiro ou sobrescrever por container:

```yaml
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault     # default pra todos os containers
  containers:
  - name: app
    securityContext:
      seccompProfile:
        type: Localhost
        localhostProfile: profiles/app-restrito.json   # sobrescreve só esse
```

## Managed clusters (GKE, EKS)

| Plataforma | Comportamento |
|-----------|--------------|
| GKE Autopilot | enforça `RuntimeDefault` por padrão, não configurável |
| GKE Standard | segue default k8s (`Unconfined`), você configura |
| EKS | segue default k8s (`Unconfined`) |
| Ambos | aceitam `SeccompDefault` feature gate via configuração do cluster |

Em clusters gerenciados o acesso aos workers é restrito — o controle de seccomp é feito via spec dos pods, não configurando o kubelet diretamente.

## Workflow: construir um perfil customizado

Para apps específicas (VOIP, sistemas que usam raw sockets, apps com acesso a hardware):

```
1. Deploy com SECCOMP_RET_LOG ou Tracee rodando
         ↓
2. App roda em produção, syscalls são logadas (sem bloqueio)
         ↓
3. Coleta as syscalls utilizadas → gera allowlist
         ↓
4. Cria perfil JSON com defaultAction: ERRNO + allowlist
         ↓
5. Aplica via Localhost profile, testa
         ↓
6. Monitora com SECCOMP_RET_LOG antes de mudar pra ERRNO/KILL
```

## Tracee vs Seccomp

| | Tracee | Seccomp |
|--|--------|---------|
| Função | observabilidade (loga syscalls via eBPF) | enforcement (filtra/bloqueia syscalls) |
| Mecanismo | eBPF tracepoints | BPF filter no kernel |
| Bloqueia? | não | sim |
| Usa como? | descobrir o que um app faz, detectar comportamento suspeito | restringir o que um app pode fazer |

Tracee é a ferramenta pra **observar**. Seccomp é o mecanismo de **controle**. O workflow ideal usa os dois: Tracee pra mapear, seccomp pra enforçar.

---

## Referências

- [Linux Kernel docs — Seccomp BPF](https://www.kernel.org/doc/html/latest/userspace-api/seccomp_filter.html)
- [Kubernetes — Seccomp](https://kubernetes.io/docs/tutorials/security/seccomp/)
- [Docker default seccomp profile](https://github.com/moby/moby/blob/master/profiles/seccomp/default.json)
