# AppArmor — MAC por Path de Executável

## O que é

AppArmor é um **LSM (Linux Security Module)** — framework do kernel que implementa MAC (Mandatory Access Control) em cima do DAC normal do Linux. Vem compilado no kernel ou como módulo. No Ubuntu vem ativo por padrão.

O `apparmor_parser` (binário) carrega profiles no kernel. O AppArmor em si roda via hooks LSM no kernel — não é um daemon, não é um processo separado.

```bash
aa-status                                          # modo atual e profiles carregados
systemctl enable apparmor                          # carrega profiles no boot
apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx  # recarrega profile
```

## Modos

| Modo | Comportamento |
|------|--------------|
| `enforce` | bloqueia e loga violações |
| `complain` | só loga, não bloqueia — usado pra aprender o comportamento |
| `disabled` | desativado para aquele processo |

```bash
aa-enforce /etc/apparmor.d/usr.sbin.nginx   # muda pra enforce
aa-complain /etc/apparmor.d/usr.sbin.nginx  # muda pra complain
```

## Como AppArmor se posiciona — relação com seccomp

São camadas complementares:

```
processo chama open("/etc/shadow", O_RDONLY)
        ↓
1. Seccomp: syscall openat está na allowlist?
   → não: bloqueia, processo nem chega no kernel
   → sim: continua
        ↓
2. LSM hooks (AppArmor): esse executável tem permissão de ler /etc/shadow?
   → não: EACCES
   → sim: executa
```

| | Seccomp | AppArmor |
|--|---------|---------|
| Filtra | qual syscall (número) | qual recurso (path, rede, capability) |
| Pergunta | "pode chamar `openat`?" | "pode acessar `/etc/shadow`?" |

## O nome do profile — por que `usr.sbin.nginx`

AppArmor associa profiles ao **path do executável**. O nome do arquivo é o path com `/` substituído por `.` e sem a barra inicial:

```
/usr/sbin/nginx  →  /etc/apparmor.d/usr.sbin.nginx
/usr/bin/python3 →  /etc/apparmor.d/usr.bin.python3
```

Quando o kernel executa `/usr/sbin/nginx`, busca o profile com aquele path e aplica as restrições. É **path-based** — diferente do SELinux que usa labels em todos os objetos.

## Estrutura de um profile

```
#include <tunables/global>

/usr/sbin/nginx {
  #include <abstractions/base>
  #include <abstractions/nameservice>

  # capabilities permitidas
  capability net_bind_service,
  capability setuid,
  capability setgid,

  # arquivos que pode ler
  /etc/nginx/** r,
  /var/log/nginx/** w,
  /var/www/html/** r,

  # rede
  network inet stream,

  # deny explícito (opcional, default já nega)
  deny /etc/shadow r,
  deny /proc/*/mem rw,
}
```

## Geração de profile — aa-genprof / aa-logprof

```bash
aa-genprof nginx
# coloca nginx em complain, você faz requests reais
# ao final sugere regras baseado no que foi acessado
# você aprova/rejeita cada uma
# gera /etc/apparmor.d/usr.sbin.nginx
```

```bash
aa-logprof
# lê /var/log/syslog ou audit.log
# sugere atualizações no profile baseado em violações recentes
# útil após deploy de nova versão que acessa paths novos
```

Workflow real:

```
1. deploy em complain mode
2. tráfego real por um período
3. aa-logprof → revisa sugestões → aprova
4. muda pra enforce
5. monitora logs de violação
```

## Overhead de CPU

Em operação normal com profile carregado: **1-3%**, desprezível.

O que explode CPU é o modo **complain com logging intensivo** — `aa-genprof` / `aa-logprof` geram um evento de auditoria pra cada operação do processo. Em nginx sob carga em instância pequena (t3.small, e2-small), isso é o suficiente pra saturar CPU. Uma vez em enforce, o overhead volta ao normal.

## Kubernetes — como aplicar

Profiles ficam nos nodes em `/etc/apparmor.d/`. Precisam estar presentes em todos os nodes onde o pod pode ser schedulado.

### GA desde k8s 1.30 (securityContext)

```yaml
securityContext:
  appArmorProfile:
    type: RuntimeDefault          # profile default da runtime
```

```yaml
securityContext:
  appArmorProfile:
    type: Localhost
    localhostProfile: nginx-restrito   # /etc/apparmor.d/nginx-restrito
```

```yaml
securityContext:
  appArmorProfile:
    type: Unconfined              # sem AppArmor
```

### Antes do k8s 1.30 (annotation)

```yaml
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/<nome-container>: localhost/nginx-restrito
```

## AppArmor em clusters gerenciados

| Cluster | Node OS default | LSM disponível |
|---------|----------------|---------------|
| GKE (COS ou Ubuntu) | Container-Optimized OS | **AppArmor ativo** |
| EKS (Amazon Linux 2) | Amazon Linux 2 | **SELinux** — não AppArmor |
| EKS (Bottlerocket) | Bottlerocket | modelo próprio |
| AKS (Ubuntu) | Ubuntu | **AppArmor ativo** |

EKS com Amazon Linux 2 usa SELinux. Aplicar profile AppArmor nesses nodes não vai funcionar — precisa usar Ubuntu como node OS ou adaptar pra SELinux.

GKE Autopilot enforça profiles AppArmor nos componentes de sistema por padrão.

## Referências

- [AppArmor — Ubuntu docs](https://ubuntu.com/server/docs/security-apparmor)
- [Kubernetes — AppArmor](https://kubernetes.io/docs/tutorials/security/apparmor/)
- [AppArmor profile language](https://gitlab.com/apparmor/apparmor/-/wikis/QuickProfileLanguage)
