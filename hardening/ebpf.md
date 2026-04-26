# eBPF — Programas no Kernel com Sandbox

## Origem: BPF clássico (1992)

O problema original era simples: o `tcpdump` precisava filtrar pacotes de rede, mas copiar todos os pacotes pro userspace pra filtrar lá era caro demais. A solução foi rodar o filtro *dentro do kernel*, onde os pacotes já estão.

O **BPF (Berkeley Packet Filter)** era uma mini VM no kernel que executava programas de filtragem. O comando `tcpdump -i eth0 port 80` compila um filtro BPF e o injeta no kernel — nenhum pacote precisa sair do kernel space pra ser avaliado.

## eBPF: generalizando o conceito

**eBPF (extended BPF, ~2014+)** pegou essa ideia e generalizou: e se você pudesse rodar programas arbitrários dentro do kernel, não só pra filtrar pacotes, mas pra qualquer evento?

Isso abre a possibilidade de instrumentar o kernel sem modificar o código-fonte dele e sem o risco de um módulo `.ko`.

## O mental model correto

eBPF **não roda abaixo do kernel** — roda *dentro* do kernel, mas num ambiente sandboxado.

```
userspace (ring-3)
────────────────────────────────────
kernel space (ring-0)
  ├── subsistemas normais (VFS, TCP/IP, netfilter...)
  ├── eBPF verifier
  ├── eBPF JIT compiler
  └── eBPF programs (sandboxados, rodando em ring-0)
────────────────────────────────────
hardware
```

A comparação com módulo `.ko`:

| | módulo `.ko` | eBPF program |
|--|-------------|-------------|
| Ring | 0 (kernel) | 0 (kernel) |
| Sandbox | nenhum | verifier garante segurança |
| Bug → | kernel panic, privilege escalation | rejeitado antes de rodar |
| Carregamento | `modprobe` | syscall `bpf()` |

## O Verifier: o coração do eBPF

Antes de qualquer programa eBPF rodar, o kernel executa uma análise estática do bytecode:

- **Sem loops infinitos** — execução deve ser bounded (loops limitados permitidos desde kernel 5.3)
- **Sem acesso inválido a memória** — cada ponteiro é verificado
- **Só helpers aprovados** — não pode chamar funções arbitrárias do kernel, só a lista de helpers permitidos
- **Sem chamadas de retorno não inicializadas** — todos os registradores usados devem ter valor definido

Se passar no verifier → JIT compiler traduz pra código nativo da arquitetura → executa.  
Se não passar → rejeitado, `bpf()` retorna erro.

## Hooks: onde os programas se anexam

eBPF funciona anexando programas a pontos específicos do kernel:

```
evento acontece no kernel
         ↓
hook dispara
         ↓
eBPF program executa (kernel space, sandboxado)
         ↓
lê/escreve em BPF maps
         ↓
userspace daemon lê os maps e age
```

| Hook | Ponto de interceptação | Quem usa |
|------|----------------------|----------|
| `kprobe` / `kretprobe` | entrada/saída de qualquer função do kernel | Tracee, Falco |
| `tracepoint` | pontos estáveis definidos pelo kernel (syscall enter/exit) | Falco, Tracee |
| `XDP` (eXpress Data Path) | antes do pacote entrar no stack de rede | Cilium |
| `TC` (Traffic Control) | após XDP, antes do roteamento | Cilium |
| `LSM` (Linux Security Module) | hooks de segurança (open, exec, mmap...) | Falco |
| `uprobe` | entrada de função em userspace | profiling |
| `cgroup` | operações em nível de cgroup | Cilium policy |

## BPF Maps: comunicação entre kernel e userspace

BPF maps são estruturas de dados compartilhadas entre programas eBPF e o userspace:

```
eBPF program (kernel) → escreve evento no map
                              ↓
userspace daemon → lê o map → alerta, roteia, bloqueia
```

Tipos comuns:
- `BPF_MAP_TYPE_HASH` — hash map, O(1) lookup ← o que o Cilium usa pra endpoints
- `BPF_MAP_TYPE_ARRAY` — array indexado por inteiro
- `BPF_MAP_TYPE_RINGBUF` — ring buffer eficiente pra eventos de alta frequência
- `BPF_MAP_TYPE_LRU_HASH` — hash com eviction automático

## Como o Cilium usa eBPF

O problema do iptables/kube-proxy: regras armazenadas como lista linear. Com 1000 Services = ~10k regras. Cada pacote percorre a lista. O(n).

O Cilium substitui isso com BPF maps (hash maps, O(1)) e hooks XDP/TC:

```
pacote chega na NIC
        ↓
XDP hook (antes de netfilter, antes do stack de rede)
        ↓
eBPF program: "destino?" → lookup em BPF map → O(1)
        ↓
encaminha direto ou dropa
        ↓
(netfilter nem é consultado pra boa parte dos fluxos)
```

Com kube-proxy replacement mode, o Cilium também assiste o API server diretamente, programa BPF maps pra Services/Endpoints, e o kube-proxy não precisa existir.

## Como Falco e Tracee usam eBPF

Observabilidade sem overhead por processo:

```
processo faz syscall execve("/bin/sh")
        ↓
kernel executa a syscall
        ↓
tracepoint sys_enter_execve dispara
        ↓
eBPF program captura: pid, uid, args, container_id, timestamp
        ↓
escreve no ring buffer (BPF map)
        ↓
daemon userspace lê → compara com regras → alerta
```

Sem `ptrace`, sem parar o processo, sem overhead por container. Um único programa eBPF observa todos os processos do sistema.

## A conexão com seccomp

Seccomp modo 2 (filter) usa BPF clássico internamente — o filtro seccomp *é* um programa BPF que o kernel executa em cada syscall. eBPF é a evolução natural desse mecanismo, com mais poder expressivo e os mesmos princípios de segurança do verifier.

## Referências

- [kernel.org — eBPF documentation](https://ebpf.io/what-is-ebpf/)
- [Cilium — eBPF datapath](https://docs.cilium.io/en/stable/network/ebpf/)
- [Brendan Gregg — BPF Performance Tools](https://www.brendangregg.com/bpf-performance-tools-book.html)
