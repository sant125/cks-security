# Kernel Modules — Restrição e Superfície de Ataque

## O que são módulos

Módulos são extensões do kernel carregadas dinamicamente em runtime, sem precisar reinicializar o sistema. Ficam em `/lib/modules/$(uname -r)/` com extensão `.ko` e são compilados em C — mesma linguagem do kernel.

Quando carregados, rodam em **ring-0**: mesmo nível de privilégio do kernel. Sem sandbox, sem isolamento. Um módulo vulnerável tem acesso irrestrito a qualquer região de memória do sistema.

```bash
lsmod                    # lista módulos carregados
modinfo <modulo>         # metadados do módulo
modprobe <modulo>        # carrega + dependências
modprobe -r <modulo>     # descarrega
```

## Autoload — o vetor principal

Quando um processo tenta usar um protocolo de rede não carregado, o kernel dispara automaticamente:

```
modprobe <protocolo>
```

Isso acontece sem nenhuma interação do usuário ou permissão especial — basta abrir um socket com o protocolo correspondente.

```bash
# processo abre socket SCTP
socket(AF_INET, SOCK_STREAM, IPPROTO_SCTP)
# kernel executa: modprobe sctp
# sctp.ko carrega no kernel do HOST
```

## Por que containers são o risco real

Todos os containers compartilham o kernel do host. Não existe kernel por container.

```
┌─────────────────────────────────────────┐
│         HOST KERNEL  (sctp.ko aqui)     │
├──────────┬──────────┬───────────────────┤
│container │container │    container      │
│    A     │    B     │       C           │
└──────────┴──────────┴───────────────────┘
```

Se `sctp.ko` está carregado no host, qualquer processo em qualquer container pode interagir com ele. Um bug de memória no módulo vira privilege escalation do container pro host.

## CVEs reais nessa família

| CVE | Módulo | Tipo | Resultado |
|-----|--------|------|-----------|
| CVE-2017-6074 | dccp | use-after-free | LPE |
| CVE-2017-7184 | xfrm | out-of-bounds | LPE |
| CVE-2022-0435 | tipc | buffer overflow | RCE remoto no kernel |
| CVE-2019-8956 | sctp | use-after-free | LPE |

Padrão consistente: protocolo de rede obscuro + bug de memória = privilege escalation.

## Como restringir

### blacklist — bloqueia apenas autoload via udev
```bash
# /etc/modprobe.d/blacklist.conf
blacklist sctp
blacklist dccp
```
O módulo ainda pode ser carregado manualmente. Proteção parcial.

### install /bin/false — bloqueia tudo
```bash
# /etc/modprobe.d/disable.conf
install dccp /bin/false
install sctp /bin/false
install rds  /bin/false
install tipc /bin/false
```

Substitui o comando de instalação por `/bin/false`. Qualquer tentativa de carregar — manual ou automática — falha com exit 1. É o controle definitivo.

### Verificação
```bash
modprobe dccp
echo $?          # deve ser != 0
lsmod | grep dccp  # deve retornar vazio
```

## Módulos críticos pra bloquear (CIS Benchmark)

```
dccp   — Datagram Congestion Control Protocol (telecomunicações)
sctp   — Stream Control Transmission Protocol (telecomunicações)
rds    — Reliable Datagram Sockets
tipc   — Transparent Inter-Process Communication
```

Nenhum desses tem uso em workloads web ou Kubernetes convencionais.

## eBPF — alternativa segura a módulos

eBPF permite rodar código no kernel sem os riscos de um módulo convencional. Antes de executar, passa por um **verifier** que rejeita loops infinitos, acessos inválidos de memória e qualquer comportamento inseguro.

```
módulo .ko   → ring-0, sem sandbox, carrega livre
eBPF         → ring-0, mas verifier valida antes de executar
```

Falco e Cilium usam eBPF exatamente por isso: acesso profundo ao kernel com superfície de ataque controlada.
