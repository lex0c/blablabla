# Runbook de resposta a incidente Linux

<details>
<summary><b>Índice</b> (clique para expandir)</summary>

- [1. Preparação](#1-preparação)
  - [Snapshot antes de mexer (cloud/VM)](#snapshot-antes-de-mexer-cloudvm)
- [2. Rede](#2-rede)
  - [2.1. Conexões atuais](#21-conexões-atuais)
  - [2.2. Com `lsof`](#22-com-lsof)
  - [2.3. Filtrar por IP suspeito](#23-filtrar-por-ip-suspeito)
  - [2.4. Informações da rede local](#24-informações-da-rede-local)
  - [2.5. Estado do firewall](#25-estado-do-firewall)
  - [2.6. Captura rápida de tráfego](#26-captura-rápida-de-tráfego)
- [3. Processo](#3-processo)
  - [3.1. Informações básicas](#31-informações-básicas)
  - [3.2. Árvore](#32-árvore)
  - [3.3. Executável real](#33-executável-real)
  - [3.4. Diretório atual](#34-diretório-atual)
  - [3.5. Command line real](#35-command-line-real)
  - [3.6. Environment](#36-environment)
  - [3.7. UID/GID/capabilities](#37-uidgidcapabilities)
  - [3.8. File descriptors](#38-file-descriptors)
  - [3.9. Arquivos e sockets abertos](#39-arquivos-e-sockets-abertos)
  - [3.10. Maps e memória mapeada](#310-maps-e-memória-mapeada)
  - [3.11. Cgroups / serviço responsável](#311-cgroups--serviço-responsável)
  - [3.12. Processos do mesmo usuário](#312-processos-do-mesmo-usuário)
  - [3.13. Buscar processos por nome](#313-buscar-processos-por-nome)
  - [3.14. Executáveis deletados ainda em execução](#314-executáveis-deletados-ainda-em-execução)
- [4. Checklist rápido de processo](#4-checklist-rápido-de-processo)
- [5. Arquivo/binário](#5-arquivobinário)
  - [5.1. Metadata](#51-metadata)
  - [5.2. Datas](#52-datas)
  - [5.3. Tipo](#53-tipo)
  - [5.4. Hashes](#54-hashes)
  - [5.5. Strings](#55-strings)
  - [5.6. Binário ELF](#56-binário-elf)
  - [5.7. Hex](#57-hex)
  - [5.8. Quem está usando o arquivo](#58-quem-está-usando-o-arquivo)
  - [5.9. Nunca execute um arquivo suspeito](#59-nunca-execute-um-arquivo-suspeito)
- [6. Preservar arquivo suspeito](#6-preservar-arquivo-suspeito)
  - [Memória do processo (binário packed)](#memória-do-processo-binário-packed)
- [7. Persistência](#7-persistência)
  - [7.1. Cron](#71-cron)
  - [7.2. Systemd](#72-systemd)
  - [7.3. Systemd de usuário](#73-systemd-de-usuário)
  - [7.4. `at`](#74-at)
  - [7.5. SSH](#75-ssh)
  - [7.6. Shell startup](#76-shell-startup)
  - [7.7. rc.local / init scripts](#77-rclocal--init-scripts)
  - [7.8. Dynamic linker / preload](#78-dynamic-linker--preload)
  - [7.9. Usuários e privilégios](#79-usuários-e-privilégios)
  - [7.10. Supervisores de processo (nível de usuário)](#710-supervisores-de-processo-nível-de-usuário)
  - [7.11. Containers](#711-containers)
  - [7.12. Outros pontos de persistência](#712-outros-pontos-de-persistência)
- [8. Arquivos suspeitos pelo filesystem](#8-arquivos-suspeitos-pelo-filesystem)
- [9. Timeline](#9-timeline)
  - [9.1. Correlação de sinais](#91-correlação-de-sinais)
- [10. Logs](#10-logs)
  - [10.1. systemd journal](#101-systemd-journal)
  - [10.2. RHEL/CentOS](#102-rhelcentos)
  - [10.3. Debian/Ubuntu](#103-debianubuntu)
  - [10.4. Cloud — a evidência pode estar OFF-BOX](#104-cloud--a-evidência-pode-estar-off-box)
  - [10.5. Metadata / roubo de credencial de instância (cloud)](#105-metadata--roubo-de-credencial-de-instância-cloud)
- [11. Auditd](#11-auditd)
- [12. Login / SSH e movimentação lateral](#12-login--ssh-e-movimentação-lateral)
  - [12.1. Movimentação lateral — a partir deste host](#121-movimentação-lateral--a-partir-deste-host)
  - [12.2. Túneis / port forwarding](#122-túneis--port-forwarding)
  - [12.3. Credenciais/artefatos que habilitam pivô](#123-credenciaisartefatos-que-habilitam-pivô)
  - [12.4. Mapa de rede interna](#124-mapa-de-rede-interna-para-onde-deu-para-ir)
  - [12.5. Este host foi ALVO de lateral?](#125-este-host-foi-alvo-de-lateral-pivô-de-entrada)
  - [12.6. Movimentação lateral por serviço vulnerável (não-SSH)](#126-movimentação-lateral-por-serviço-vulnerável-não-ssh)
- [13. Bash history](#13-bash-history)
- [14. Identificar serviço vulnerável](#14-identificar-serviço-vulnerável)
  - [Heurística: usuário do malware → estreita o vetor](#heurística-usuário-do-malware--estreita-o-vetor)
- [15. Reverse proxy](#15-reverse-proxy)
  - [Duas armadilhas do proxy](#duas-armadilhas-do-proxy-não-confie-nele-como-proteção)
- [16. Camada de aplicação (agnóstico de linguagem)](#16-camada-de-aplicação-agnóstico-de-linguagem)
  - [Integridade do código](#integridade-do-código-rápido-e-preciso)
  - [Execução de comando (sinks de RCE)](#execução-de-comando-sinks-de-rce-multi-linguagem)
  - [Requisições externas / SSRF](#requisições-externas--ssrf-o-servidor-busca-url-do-cliente)
  - [Escrita de arquivo controlada](#escrita-de-arquivo-controlada-path-traversal--dropoverwrite)
  - [Webshell no docroot](#webshell-no-docroot)
- [17. Indicadores fortes de reverse shell/backdoor](#17-indicadores-fortes-de-reverse-shellbackdoor)
- [18. Contenção](#18-contenção)
  - [18.1. Bloquear C2](#181-bloquear-c2)
  - [18.2. Desabilitar serviço vulnerável](#182-desabilitar-serviço-vulnerável)
- [19. Remover persistência](#19-remover-persistência)
- [20. Matar processo](#20-matar-processo)
- [21. Remover/quarentenar arquivo](#21-removerquarentenar-arquivo)
- [22. Validar depois da limpeza](#22-validar-depois-da-limpeza)
- [23. Procurar segunda persistência](#23-procurar-segunda-persistência)
  - [Varredura da frota (mesmo IOC em outros hosts)](#varredura-da-frota-mesmo-ioc-em-outros-hosts)
- [24. Verificar integridade do sistema](#24-verificar-integridade-do-sistema)
- [25. Capabilities e SUID](#25-capabilities-e-suid)
- [26. Rotação de credenciais](#26-rotação-de-credenciais)
- [27. Quando rebuild é necessário](#27-quando-rebuild-é-necessário)
- [28. Coleta rápida em um único bloco](#28-coleta-rápida-em-um-único-bloco)
- [29. Coleta rápida de um PID](#29-coleta-rápida-de-um-pid)
- [30. Coleta rápida de um arquivo](#30-coleta-rápida-de-um-arquivo)
- [31. Red flags para reconhecer rapidamente](#31-red-flags-para-reconhecer-rapidamente)
- [32. Ferramentas úteis para scan/análise](#32-ferramentas-úteis-para-scananálise)
- [33. Regra principal](#33-regra-principal)
- [34. Reduzir o blast radius (blindagem da VM)](#34-reduzir-o-blast-radius-blindagem-da-vm)
  - [34.1. Sandbox do serviço (systemd)](#341-sandbox-do-serviço-systemd)
  - [34.2. Filesystem sem execução](#342-filesystem-sem-execução)
  - [34.3. Egress default-deny](#343-egress-default-deny)
  - [34.4. Identidade e credenciais](#344-identidade-e-credenciais)
  - [34.5. Segmentação e acesso](#345-segmentação-e-acesso)
  - [34.6. Tempo: hosts descartáveis](#346-tempo-hosts-descartáveis)
  - [34.7. Detecção que reduz raio](#347-detecção-que-reduz-raio)
  - [34.8. Auditoria rápida do estado atual](#348-auditoria-rápida-do-estado-atual)
  - [O que NÃO reduz blast radius](#o-que-não-reduz-blast-radius)
- [35. Comprometimento em nível de kernel](#35-comprometimento-em-nível-de-kernel)
  - [35.1. Antes de tudo: é kernel mesmo?](#351-antes-de-tudo-é-kernel-mesmo)
  - [35.2. Taint e módulos](#352-taint-e-módulos)
  - [35.3. Hooks de função](#353-hooks-de-função-syscall-table-ftrace-kprobes)
  - [35.4. eBPF: o rootkit sem módulo](#354-ebpf-o-rootkit-sem-módulo)
  - [35.5. Método da divergência](#355-método-da-divergência)
  - [35.6. A resposta confiável está fora da caixa](#356-a-resposta-confiável-está-fora-da-caixa)
  - [35.7. Impedir (mais barato que detectar)](#357-impedir-mais-barato-que-detectar)
  - [35.8. Veredito](#358-veredito)
- [36. Escalonamento de privilégio (privesc)](#36-escalonamento-de-privilégio-privesc)
  - [36.1. Varredura rápida](#361-varredura-rápida)
  - [36.2. SUID, SGID e capabilities](#362-suid-sgid-e-capabilities)
  - [36.3. sudo](#363-sudo)
  - [36.4. O que eu escrevo e root executa](#364-o-que-eu-escrevo-e-root-executa)
  - [36.5. Grupos que são root disfarçado](#365-grupos-que-são-root-disfarçado)
  - [36.6. Kernel e componentes com LPE conhecida](#366-kernel-e-componentes-com-lpe-conhecida)
  - [36.7. Ferramentas](#367-ferramentas)
  - [36.8. Mitigação por rota](#368-mitigação-por-rota)
  - [36.9. Higiene contínua](#369-higiene-contínua)

</details>

---

# 1. Preparação

> **Por quê:** toda leitura altera o host (atime, cache, log, journal). Registrar estado e hora **antes** é o que separa análise de contaminação de evidência. **Olhe para:** a hora do host pode estar errada/adulterada (§9) — anote também a hora real de uma fonte externa.

Registre horário e host:

```bash
date -Is
hostname
uname -a
cat /etc/os-release
```

Crie um diretório protegido para coleta:

```bash
IR=/root/ir-$(date +%Y%m%d-%H%M%S)

sudo mkdir -p "$IR"
sudo chmod 700 "$IR"
```

Salve informações básicas:

```bash
{
  date -Is
  hostname
  hostnamectl
  uname -a
  cat /etc/os-release
  uptime
} | sudo tee "$IR/host.txt"
```

> Não coloque esse diretório em local compartilhado. `cmdline`, `environ`, logs e configs podem conter secrets.

## Snapshot antes de mexer (cloud/VM)

Preserva evidência e habilita forense/rebuild. Faça ANTES de conter/limpar:

```bash
# GCP
gcloud compute disks snapshot <DISK> --snapshot-names="ir-$(date +%Y%m%d-%H%M%S)"
# AWS
aws ec2 create-snapshot --volume-id <VOL> --description "ir-$(date +%s)"
```

> OpSec: se o atacante tem shell ativo, ações agressivas (kill/block) podem alertá-lo a destruir evidência. Snapshot + coleta primeiro; contenha em bloco.

---

# 2. Rede

> **Por quê:** um backdoor quase sempre mantém uma conexão de saída (C2). A rede costuma ser o primeiro fio a puxar. **Olhe para:** conexão *outbound* para IP público a partir de um processo/usuário de aplicação inesperado — em especial 443 ou portas altas persistentes.

## 2.1. Conexões atuais

Principal comando de triagem:

```bash
sudo ss -tunap
```

```text
-t TCP   -u UDP   -a todos (listen + established)
-n não resolve DNS/porta  → mais rápido e não gera consulta que avisa o atacante
-p processo dono do socket → precisa de root; sem sudo você vê a conexão mas não quem a abriu
```

> `ss` lê o estado dos sockets direto do kernel (netlink), não do binário do processo. Por isso um processo pode mentir o próprio nome, mas não some daqui — a menos que haja rootkit em kernel (§7.12, `unhide` na §32).

Somente TCP estabelecido:

```bash
sudo ss -tnp state established
```

Com informações adicionais de socket:

```bash
sudo ss -tnpo state established
```

Portas TCP em listen:

```bash
sudo ss -ltnp
```

UDP:

```bash
sudo ss -lunp
```

---

## 2.2. Com `lsof`

Todas as conexões:

```bash
sudo lsof -Pan -i
```

Somente TCP estabelecido:

```bash
sudo lsof -Pan -iTCP -sTCP:ESTABLISHED
```

Listeners:

```bash
sudo lsof -Pan -iTCP -sTCP:LISTEN
```

---

## 2.3. Filtrar por IP suspeito

```bash
sudo ss -tnp | grep '<IP>'
```

```bash
sudo lsof -Pan -i | grep '<IP>'
```

Exemplo:

```text
10.142.0.12:7304 -> 51.91.190.241:443
users:(("defunct",pid=6574,fd=0))
```

Já temos:

```text
IP remoto: 51.91.190.241
porta: 443
PID: 6574
processo: defunct
```

> Porta `443` não significa necessariamente HTTPS. Malware frequentemente usa 443 para parecer tráfego normal.

---

## 2.4. Informações da rede local

```bash
ip addr
ip route
ip neigh
cat /etc/resolv.conf
```

Rota usada para determinado IP:

```bash
ip route get <IP>
```

---

## 2.5. Estado do firewall

### iptables

```bash
sudo iptables -S
```

```bash
sudo iptables -L INPUT -n -v --line-numbers
sudo iptables -L OUTPUT -n -v --line-numbers
sudo iptables -L FORWARD -n -v --line-numbers
```

Salvar configuração:

```bash
sudo iptables-save > "$IR/iptables.txt"
```

### nftables

```bash
sudo nft list ruleset
```

Salvar:

```bash
sudo nft list ruleset > "$IR/nftables.txt"
```

### firewalld

```bash
sudo firewall-cmd --state
sudo firewall-cmd --list-all
sudo firewall-cmd --list-all-zones
```

> O firewall atual não prova qual firewall existia quando o comprometimento ocorreu.

---

## 2.6. Captura rápida de tráfego

Se uma conexão suspeita ainda estiver ativa:

```bash
sudo timeout 30 tcpdump \
  -ni any \
  host <IP> \
  -s0 \
  -w "$IR/suspect.pcap"
```

Ver sem salvar:

```bash
sudo tcpdump -ni any host <IP>
```

Por porta:

```bash
sudo tcpdump -ni any port 443
```

> **O que o pcap responde mesmo com TLS:** você não lê o conteúdo, mas lê o **comportamento** — periodicidade do beacon (intervalo fixo = C2 automatizado), volume e direção (upload grande = exfiltração), e o **SNI**/certificado no handshake, que revela o domínio de destino. Também confirma se a conexão está viva de fato ou é socket órfão.

```bash
sudo tcpdump -ni any -A 'tcp port 443 and tcp[((tcp[12:1] & 0xf0) >> 2):1] = 0x16' | grep -a -o '[a-z0-9.-]*\.[a-z]\{2,\}'   # SNI em claro
```

> `-s0` captura o pacote inteiro (sem truncar) e `-w` salva cru para análise offline no Wireshark. Cuidado: o pcap pode conter dados sensíveis em tráfego não-TLS — trate como o resto da coleta (§1).

---

# 3. Processo

> **Por quê:** confirmar o que o processo suspeito **realmente** é — o `ps` mostra o nome que o processo diz ter, que pode ser forjado. **Olhe para:** exe em `/home`/`/tmp`/`/dev/shm`, nome camuflado de thread de kernel, pai `PID 1` sem motivo, child `sh`/`bash`, PTY + socket remoto.

> **Fundamento:** `ps` mostra `comm`/`cmdline`, campos que o **próprio processo reescreve** (`prctl(PR_SET_NAME)`, `exec -a`, sobrescrita do argv). Já `/proc/<pid>/exe` e `/proc/<pid>/cwd` são links mantidos pelo **kernel** para o inode real — o processo não os controla. Regra: nome mente, `exe` não.

Encontrou um PID suspeito:

```bash
PID=<PID>
```

---

## 3.1. Informações básicas

```bash
sudo ps -o \
pid,ppid,user,group,lstart,etime,stat,args \
-p "$PID"
```

```text
lstart  hora exata de início → âncora para a timeline (§9)
etime   há quanto tempo roda → "3 dias" num processo que você nunca viu = ruim
stat    S dormindo | R rodando | Z zumbi | D I/O ininterrupto | s líder de sessão
        + foreground | l multi-thread | < prioridade alta
```

> `stat` com `s` (líder de sessão) num processo sem terminal, ou `Z` persistente, chama atenção: shells de backdoor costumam criar sessão própria para sobreviver ao pai.

Ver também PID pai:

```bash
PPID=$(ps -o ppid= -p "$PID" | tr -d ' ')
sudo ps -o \
pid,ppid,user,group,lstart,etime,stat,args \
-p "$PID","$PPID"
```

---

## 3.2. Árvore

```bash
sudo pstree -aps "$PID"
```

Árvore geral:

```bash
sudo pstree -ap
```

Outra visão:

```bash
ps -ef --forest
```

> Ler a árvore **de baixo para cima** dá o vetor: quem executou quem. `nginx → php-fpm → sh → curl` é RCE em texto. `PPID 1` significa que o pai já morreu (daemonizou ou saiu de propósito) e o rastro se perdeu — aí quem restaura a origem é o cgroup (§3.11) ou o audit (§11).

---

## 3.3. Executável real

Não confie no nome exibido pelo `ps`.

```bash
sudo readlink -f /proc/$PID/exe
```

Exemplo:

```text
ps:
[card0-crtc8]

/proc/6574/exe:
/home/node/.config/htop/defunct
```

Isso indica masquerading do nome do processo.

---

## 3.4. Diretório atual

```bash
sudo readlink -f /proc/$PID/cwd
```

Locais especialmente suspeitos:

```text
/tmp
/var/tmp
/dev/shm
/home/<user>/.config
/home/<user>/.cache
diretórios temporários em NFS
```

---

## 3.5. Command line real

```bash
sudo sh -c "tr '\0' ' ' < /proc/$PID/cmdline"
echo
```

---

## 3.6. Environment

```bash
sudo sh -c "tr '\0' '\n' < /proc/$PID/environ"
```

Filtrar:

```bash
sudo sh -c "tr '\0' '\n' < /proc/$PID/environ" \
  | grep -E \
  'HOME|USER|PATH|SHELL|NODE|PM2|LD_PRELOAD|GS_|GSOCKET'
```

> `environ` pode conter senhas, tokens, URLs privadas e chaves.

---

## 3.7. UID/GID/capabilities

```bash
sudo grep -E \
'Name|State|Pid|PPid|Uid|Gid|TracerPid|Cap' \
/proc/$PID/status
```

Capabilities:

```bash
sudo grep '^Cap' /proc/$PID/status
```

```text
Uid: real, efetivo, saved, fs → efetivo != real indica setuid/troca de privilégio
TracerPid: != 0 → alguém está com ptrace neste processo (debug, injeção ou hooking)
CapEff: poder REAL agora  → 0000000000000000 = sem capability
        decodifique: capsh --decode=<hex>
```

> Um processo não-root com `CapEff` gordo (ex.: `cap_setuid`, `cap_sys_ptrace`, `cap_net_raw`) é root disfarçado: capability substitui o SUID e passa despercebida em auditoria por UID.

---

## 3.8. File descriptors

```bash
sudo ls -la /proc/$PID/fd
```

Procure:

```text
socket:[...]
/dev/ptmx
/dev/pts/*
arquivos em /tmp
arquivos em /dev/shm
arquivos deletados
pipes
NFS/CIFS
```

> **O sinal mais forte:** `0`, `1` e `2` (stdin/stdout/stderr) apontando para o **mesmo `socket:[...]`**. Isso é a definição de reverse shell — o shell lê comandos da rede e devolve a saída por ela. Some `/dev/pts/*` e você tem shell interativo com TTY do outro lado.

---

## 3.9. Arquivos e sockets abertos

Tudo:

```bash
sudo lsof -Pan -p "$PID"
```

Somente rede:

```bash
sudo lsof -Pan -a -p "$PID" -i
```

---

## 3.10. Maps e memória mapeada

```bash
sudo cat /proc/$PID/maps
```

Filtrar caminhos:

```bash
sudo grep '/' /proc/$PID/maps
```

Útil para encontrar:

* bibliotecas injetadas;
* executáveis deletados;
* arquivos carregados fora de `/usr/lib*`.

---

## 3.11. Cgroups / serviço responsável

```bash
sudo cat /proc/$PID/cgroup
```

Em systemd:

```bash
sudo systemctl status "$PID"
```

Pode revelar diretamente qual service unit iniciou o processo.

---

## 3.12. Processos do mesmo usuário

```bash
ps -fu <USER>
```

Mais detalhado:

```bash
ps -eo \
pid,ppid,user,lstart,etime,stat,args \
| grep '<USER>'
```

---

## 3.13. Buscar processos por nome

```bash
pgrep -af '<NOME>'
```

Por usuário:

```bash
pgrep -a -u <USER> '<NOME>'
```

---

## 3.14. Executáveis deletados ainda em execução

```bash
sudo lsof +L1
```

Filtrar:

```bash
sudo lsof +L1 | grep '(deleted)'
```

Isto é importante porque:

```text
rm malware
```

não mata um processo já em execução.

> **Mecanismo:** no Linux, `rm` só remove o *nome* (link) do diretório. O inode e os blocos só são liberados quando o último link **e** o último file descriptor somem. Enquanto o processo roda, o binário continua íntegro em disco — `+L1` significa "link count 0, ainda aberto". Técnica padrão de malware: executar e apagar em seguida.

Recupere o binário apagado direto do `/proc` (o link `exe` continua válido):

```bash
sudo cp /proc/$PID/exe "$IR/samples/recovered.bin"
```

---

# 4. Checklist rápido de processo

> **Por quê:** roda de uma vez a identidade completa de um PID suspeito. **Olhe para:** exe/cwd/cmdline incoerentes com o nome exibido, fds de socket/PTY, arquivo deletado ainda aberto.

```bash
PID=<PID>

sudo ps -o pid,ppid,user,group,lstart,etime,stat,args -p "$PID"
sudo pstree -aps "$PID"

sudo readlink -f /proc/$PID/exe
sudo readlink -f /proc/$PID/cwd

sudo sh -c "tr '\0' ' ' < /proc/$PID/cmdline"; echo
sudo sh -c "tr '\0' '\n' < /proc/$PID/environ"

sudo grep -E \
'Name|State|Pid|PPid|Uid|Gid|TracerPid|Cap' \
/proc/$PID/status

sudo ls -la /proc/$PID/fd
sudo lsof -Pan -p "$PID"
sudo lsof -Pan -a -p "$PID" -i
```

---

# 5. Arquivo/binário

> **Por quê:** identificar o binário **sem executá-lo** (família, C2, capacidade) e datar quando surgiu. **Olhe para:** `ctime` recente com `mtime` antigo (timestomp), arquivo sem pacote correspondente, strings de C2/shell/base64, ELF em local incomum.

Defina:

```bash
FILE=$(sudo readlink -f /proc/$PID/exe)
```

---

## 5.1. Metadata

```bash
sudo ls -la "$FILE"
sudo stat "$FILE"
```

Informações extras:

```bash
sudo getfacl "$FILE"
sudo lsattr "$FILE"
sudo getcap "$FILE"
```

Filesystem/mount:

```bash
findmnt -T "$FILE"
```

---

## 5.2. Datas

No `stat`, compare:

```text
Access
Modify
Change
Birth
```

Especial atenção ao `ctime` (`Change`).

Exemplo:

```text
Modify: 2018-07-03
Change: 2026-04-30 18:30:33
```

Um `mtime` muito antigo com `ctime` recente pode indicar:

* timestamp preservado;
* timestomping;
* arquivo copiado mantendo mtime.

> **Por que o `ctime` é o timestamp confiável:** `atime`/`mtime` podem ser definidos livremente (`touch -t`, `utimes()`) — é assim que o atacante "envelhece" o artefato. O `ctime` é atualizado pelo **kernel** a cada mudança do inode e **não existe syscall para defini-lo**: só muda mexendo no relógio ou escrevendo no filesystem raw. Ou seja: o `mtime` é o que o arquivo *alega*; o `ctime` é o que o sistema *observou*.

```text
Birth   criação (nem todo fs preenche; ext4 sim, mostrado como '-' quando ausente)
Access  última leitura (pode estar desativado por 'relatime'/'noatime' — não confie)
Modify  último write no CONTEÚDO      → falsificável
Change  última mudança no INODE       → conteúdo, permissão, dono, rename, timestomp
```

> Corolário prático: `touch` para disfarçar o `mtime` **atualiza o `ctime`** e denuncia o próprio disfarce. Por isso a busca da §9 é por `ctime`.

---

## 5.3. Tipo

```bash
sudo file "$FILE"
```

> `file` lê os *magic bytes*, não a extensão. Leia a saída inteira: **`statically linked`** (não depende de libs do host — típico de dropper portátil), **`stripped`** (sem símbolos — dificulta análise, incomum em binário de distro), e a arquitetura (um ELF x86-64 num host ARM = kit genérico jogado às cegas).

---

## 5.4. Hashes

```bash
sudo sha256sum "$FILE"
```

Opcional:

```bash
sudo sha1sum "$FILE"
sudo md5sum "$FILE"
```

Use SHA-256 como referência principal.

> O hash é a **identidade** do artefato: é o que você consulta em VirusTotal/threat intel, o que procura na frota (§23) e o que prova depois que a amostra preservada é a mesma. MD5/SHA-1 só servem para bater com bases antigas — são vulneráveis a colisão.

> **OpSec:** subir o *arquivo* para um serviço público pode entregar seu incidente (o atacante monitora e some/muda de C2, e o binário pode conter secrets do seu ambiente). Consultar só o **hash** não vaza conteúdo.

---

## 5.5. Strings

```bash
sudo strings -a -n 8 "$FILE" | less
```

Filtrar indicadores:

```bash
sudo strings -a "$FILE" \
  | grep -Ei \
  'http|https|socket|connect|shell|bash|sh|curl|wget|ssl|tls|auth|key|password'
```

Pode revelar:

* família do malware;
* nomes de bibliotecas;
* C2;
* parâmetros;
* variáveis de ambiente;
* paths;
* domínio/IP.

> **Se vier quase nada legível, isso é o achado:** binário normal tem centenas de strings (mensagens de erro, nomes de símbolo, paths). Pouquíssimas strings + seções estranhas = **packed/criptografado** (UPX e derivados). Nesse caso o disco não vai contar a história — o C2 e a config só existem **em memória**, descompactados, enquanto o processo roda: vá para §6.

---

## 5.6. Binário ELF

Header:

```bash
sudo readelf -h "$FILE"
```

Program headers:

```bash
sudo readelf -l "$FILE"
```

Sections:

```bash
sudo readelf -S "$FILE"
```

Dynamic section:

```bash
sudo readelf -d "$FILE"
```

Symbols:

```bash
sudo readelf -s "$FILE"
```

Informações gerais:

```bash
sudo objdump -p "$FILE"
```

O que cada saída responde:

```text
-h  tipo (EXEC x DYN), arquitetura, entry point
-d  NEEDED = libs de que depende | RPATH/RUNPATH = onde procura libs (vetor de hijack)
-S  seções: poucas/sem nome, ou uma seção enorme e gravável+executável = packer
-s  símbolos: vazio (stripped) é comum em malware, raro em pacote de distro
-l  segmentos carregados; RWX no mesmo segmento = código automodificável
```

> Dois achados clássicos aqui: `RUNPATH` apontando para diretório gravável (o binário carrega a lib do atacante) e entry point fora da `.text` (descompactador de packer).

---

## 5.7. Hex

```bash
sudo xxd "$FILE" | head -50
```

ou:

```bash
sudo hexdump -C "$FILE" | head -50
```

---

## 5.8. Quem está usando o arquivo

```bash
sudo lsof "$FILE"
```

ou:

```bash
sudo fuser -v "$FILE"
```

---

## 5.9. Nunca execute um arquivo suspeito

Não faça:

```bash
./malware
sudo ./malware
```

Evite também:

```bash
ldd malware
```

> **Por que `ldd` é perigoso:** ele não "lê" o binário — na prática pede ao *loader* que o carregue para resolver as dependências. Com um ELF preparado (ou `LD_TRACE_LOADED_OBJECTS` manipulado), isso **executa código**. Use `readelf -d` (§5.6), que só faz parsing.

> Mesmo raciocínio para o resto: só rode ferramenta que **lê bytes**. E se precisar detonar de verdade, faça em sandbox descartável e sem rota para sua rede.

Prefira análise estática:

```text
file
sha256sum
strings
readelf
objdump
xxd
```

---

# 6. Preservar arquivo suspeito

> **Por quê:** guardar prova antes de qualquer alteração (post-mortem, análise, possível uso legal). Matar/apagar sem preservar destrói evidência. **Olhe para:** copiar com `-a` (preserva metadata), registrar hash, e capturar a **memória** se o binário for packed.

Antes de remover:

```bash
sudo mkdir -p "$IR/samples"
sudo chmod 700 "$IR/samples"
```

Copiar:

```bash
sudo cp -a "$FILE" "$IR/samples/"
```

Metadata:

```bash
sudo stat "$FILE" \
  > "$IR/samples/stat.txt"
```

Hash:

```bash
sudo sha256sum "$FILE" \
  > "$IR/samples/sha256.txt"
```

## Memória do processo (binário packed)

Malware como GSocket costuma vir **empacotado**: `strings` no disco rende pouco, mas a config/segredo/C2 estão **em memória** enquanto roda. Capture ANTES de matar:

```bash
sudo gcore -o "$IR/samples/proc.core" "$PID"        # ou: sudo cp /proc/$PID/mem ...
sudo strings -n 8 "$IR/samples/proc.core"* \
  | grep -iE 'gs-netcat|gsocket|GS_|\.onion|relay|/bin/sh|password|token'
```

---

# 7. Persistência

> **Por quê:** matar o processo sem remover a persistência é enxugar gelo — cron, systemd ou supervisor relançam o malware em minutos, e agora o atacante sabe que você viu. **Olhe para:** qualquer coisa que o sistema executa **sozinho** (por tempo, por boot, por login, por evento). **Conceito:** persistência = um gatilho automático + um payload. Você precisa achar o *gatilho*; o payload você já tem.

Procure persistência **antes de matar o processo**.

Os gatilhos, em ordem de frequência real:

```text
tempo    cron, systemd timer, at, anacron            §7.1 §7.2 §7.4
boot     unit systemd, rc.local, init, módulo kernel  §7.2 §7.7 §7.12
login    authorized_keys, .bashrc, .ssh/rc, PAM, MOTD §7.5 §7.6 §7.12
sempre   supervisor (pm2/supervisord), container restart:always  §7.10 §7.11
sombra   LD_PRELOAD, ld.so.preload, udev, hook de pacote        §7.8 §7.12
```

> **Presuma mais de um.** Operador competente deixa persistência barulhenta (que você acha) e uma silenciosa (que você não acha). Só considere limpo depois da §22 e do re-scan em 24–48h.

---

# 7.1. Cron

Usuário:

```bash
sudo crontab -u <USER> -l
```

Root:

```bash
sudo crontab -l
```

Arquivos:

### RHEL/CentOS

```bash
sudo ls -la /var/spool/cron/
```

### Debian/Ubuntu

```bash
sudo ls -la /var/spool/cron/crontabs/
```

Sistema:

```bash
sudo cat /etc/crontab
sudo ls -la /etc/cron.d/
sudo ls -la /etc/cron.hourly/
sudo ls -la /etc/cron.daily/
```

---

## Buscar padrões suspeitos

```bash
sudo grep -RInaE \
'curl|wget|base64|bash -c|sh -c|exec -a|/tmp|/dev/shm|\.config|\.cache' \
/var/spool/cron \
/etc/cron.d \
/etc/crontab \
2>/dev/null
```

Indicadores fortes:

```text
base64 | bash
curl | sh
wget
exec -a
arquivo ELF no home
comentário tentando parecer legítimo
```

> **Por que esses padrões:** o atacante quer o payload **fora do disco** (baixado a cada execução — muda quando ele quiser, e o que fica na sua máquina é só uma linha inocente) e **ilegível** (`base64` derrota o `grep` do defensor). `exec -a` renomeia o processo na hora do exec: é como um cron vira `[kworker/0:2]` no `ps`.

> Leia o cron **como o kernel lê**: os 5 campos são `min hora dia mês dia-da-semana`. `*/7 * * * *` (a cada 7 min) e `@reboot` são os favoritos — beacon frequente e sobrevivência a restart.

---

# 7.2. Systemd

Units habilitadas:

```bash
sudo systemctl list-unit-files --state=enabled
```

Services:

```bash
sudo systemctl list-units --type=service --all
```

Timers:

```bash
sudo systemctl list-timers --all
```

> Onde olhar dentro da unit: `ExecStart`/`ExecStartPre` (o payload), `Restart=always` (o systemd vira o *supervisor do malware* — ele ressuscita o que você matar), `User=` (com qual identidade roda) e `WantedBy=` (quando dispara). Compare também o **caminho**: unit legítima de pacote fica em `/usr/lib/systemd/system`; o que o atacante escreve costuma cair em `/etc/systemd/system` (que tem precedência e pode *sobrescrever* uma unit legítima com o mesmo nome).

Pesquisar conteúdo:

```bash
sudo grep -RInaE \
'curl|wget|bash|sh|/tmp|/dev/shm|/home' \
/etc/systemd/system \
/usr/lib/systemd/system \
/lib/systemd/system \
2>/dev/null
```

---

# 7.3. Systemd de usuário

```bash
sudo find /home \
  -path '*/.config/systemd/user/*' \
  -type f \
  -print
```

> **Ponto cego clássico:** unit de usuário **não aparece** em `systemctl list-units` de root — só em `systemctl --user` daquele usuário. Pior: com `loginctl enable-linger <USER>` ativo, ela roda **sem ninguém logado**, no boot. Persistência completa sem nunca tocar em `/etc` nem precisar de root.

```bash
loginctl show-user <USER> | grep -i linger
sudo -iu <USER> systemctl --user list-units --type=service
```

---

# 7.4. `at`

```bash
atq
```

Root:

```bash
sudo atq
```

---

# 7.5. SSH

Encontrar chaves:

```bash
sudo find /home /root \
  -path '*/.ssh/authorized_keys' \
  -type f \
  -print
```

Ver metadata:

```bash
sudo stat /home/<USER>/.ssh/authorized_keys
```

Conteúdo:

```bash
sudo cat /home/<USER>/.ssh/authorized_keys
```

Como ler a linha:

```text
[options] <tipo> <chave base64> <comentário>
```

* **comentário** (`user@host` no fim): quem gerou. Um comentário estranho é o nome da máquina do atacante — IOC de graça.
* **options** no início (`command=`, `from=`, `no-pty`): raro em uso normal. `command="..."` executa algo a **cada** login — é backdoor com gatilho.
* **posição**: chave acrescentada no fim, sem quebra de linha antes, é a assinatura do `echo >> authorized_keys`.
* O `ctime` do arquivo (§5.2) data a inserção mesmo que o conteúdo pareça antigo.

> Também vale conferir `sshd_config`: `AuthorizedKeysFile` apontando para outro caminho (ex.: `/etc/ssh/keys/%u`) esconde a chave fora do `~/.ssh` que todo mundo olha.

```bash
sudo sshd -T 2>/dev/null | grep -iE 'authorizedkeys|permitroot|passwordauth|port|allowusers'
```

---

# 7.6. Shell startup

Arquivos importantes:

```text
~/.bashrc
~/.bash_profile
~/.profile
~/.zshrc
/etc/profile
/etc/profile.d/*
```

Pesquisar:

```bash
sudo grep -RInaE \
'curl|wget|bash -c|sh -c|exec -a|/tmp|/dev/shm' \
/home \
/etc/profile \
/etc/profile.d \
2>/dev/null
```

---

# 7.7. rc.local / init scripts

```bash
sudo cat /etc/rc.local 2>/dev/null
sudo ls -la /etc/init.d/
```

Pesquisar:

```bash
sudo grep -RInaE \
'/tmp|/dev/shm|curl|wget|bash|sh' \
/etc/init.d \
/etc/rc.local \
2>/dev/null
```

---

# 7.8. Dynamic linker / preload

```bash
sudo cat /etc/ld.so.preload 2>/dev/null
```

Pesquisar `LD_PRELOAD`:

```bash
sudo grep -RIna 'LD_PRELOAD' \
/etc \
/home \
2>/dev/null
```

> **Mecanismo (é o rootkit de userland mais comum):** o loader dinâmico carrega as libs listadas em `/etc/ld.so.preload` (global) ou `$LD_PRELOAD` **antes** da libc, em **todo** processo dinâmico. Quem chega primeiro vence a resolução do símbolo. Reescrevendo `readdir()`, `open()` ou `accept()`, a lib esconde arquivos, processos e conexões de `ls`, `ps` e `ss` — **sem tocar no kernel**.

Como furar o disfarce:

```bash
sudo cat /etc/ld.so.preload            # normalmente NÃO EXISTE; se existe, leia cada linha
sudo grep -l 'ld.so.preload' /proc/*/maps 2>/dev/null   # quem carregou a lib
ls /proc/*/exe 2>/dev/null | wc -l ; ps -e | wc -l      # divergência = processo escondido
```

> Binário **estático** (`file` diz `statically linked`) e `busybox` ignoram o preload — por isso servem de "segunda opinião" num host suspeito. Se `ls` e `busybox ls` discordam sobre o mesmo diretório, você tem rootkit.

---

# 7.9. Usuários e privilégios

Usuários UID 0:

```bash
awk -F: '$3 == 0 {print}' /etc/passwd
```

> **É o UID que define o poder, não o nome.** O kernel só compara números: qualquer conta com `uid=0` **é** root, chame-se `backup`, `systemd-net` ou `ftp`. Por isso a busca é por `$3 == 0` e não por "root".

Outros ângulos rápidos:

```bash
awk -F: '$2 == "" {print $1": SEM SENHA"}' /etc/shadow          # login sem autenticação
awk -F: '$3 >= 1000 && $7 !~ /nologin|false/ {print}' /etc/passwd  # conta de serviço com shell
getent group sudo wheel docker lxd adm 2>/dev/null              # grupos = root indireto
sudo stat -c '%n %z' /etc/passwd /etc/shadow /etc/sudoers        # ctime na janela do incidente?
```

> `docker`, `lxd` e `disk` equivalem a root: quem monta o filesystem do host num container lê e escreve tudo. Conta de serviço que ganhou shell (`/bin/bash` onde era `/usr/sbin/nologin`) é alteração deliberada.

Todos:

```bash
getent passwd
```

Sudoers:

```bash
sudo grep -RIna \
'' \
/etc/sudoers \
/etc/sudoers.d \
2>/dev/null
```

Validar:

```bash
sudo visudo -c
```

---

# 7.10. Supervisores de processo (nível de usuário)

Além do systemd, apps costumam ser mantidas por supervisores que reiniciam
processos — ótimos pontos de persistência (um app "extra" ou script adulterado
que relança o malware). Verifique o que estiver em uso:

```bash
# supervisord
sudo grep -RIna '' /etc/supervisor* 2>/dev/null; sudo supervisorctl status 2>/dev/null
# pm2 (Node), forever, etc. — rode como o usuário dono
sudo -iu <USER> pm2 list 2>/dev/null; sudo -iu <USER> pm2 jlist 2>/dev/null
sudo ls -la /home/<USER>/.pm2/ 2>/dev/null          # dump.pm2 = o que é ressuscitado
# crontab @reboot / user services já cobertos em §7.1–§7.3
```

Procure: apps/scripts desconhecidos, paths em `/tmp` `/dev/shm` `~/.config`,
entradas de startup persistente, um app em crash-loop (pode ser o loader).

---

# 7.11. Containers

Docker:

```bash
sudo docker ps -a --no-trunc
```

Imagens:

```bash
sudo docker images
```

Inspect:

```bash
sudo docker inspect <CONTAINER>
```

Restart policy:

```bash
sudo docker inspect \
  --format '{{.HostConfig.RestartPolicy.Name}}' \
  <CONTAINER>
```

> `restart: always`/`unless-stopped` transforma o Docker no supervisor do malware: você mata o processo, o daemon sobe outro. Pare o **container**, não o processo.

O que procurar no `inspect` (e por quê):

```text
Privileged: true        → sem isolamento; escapar para o host é trivial
Binds/Mounts: /  ou /etc, /root, /var/run/docker.sock → escrita no host = root no host
NetworkMode: host       → o container usa a rede do host direto
CapAdd: SYS_ADMIN...    → capabilities extras devolvem poder de root
Image: sem tag conhecida / registry estranho → imagem plantada
```

> **Fronteira importante:** container não é máquina virtual — é processo isolado por namespace/cgroup no **mesmo kernel**. Comprometimento com `--privileged` ou com o socket do Docker montado é comprometimento **do host**, e a §27 se aplica ao host inteiro. Os PIDs também aparecem no host: `sudo cat /proc/<PID>/cgroup` (§3.11) diz de qual container o processo veio.

---

# 7.12. Outros pontos de persistência

Vetores menos óbvios, todos abusados na prática. Defina a janela do incidente para filtrar por recém-criados:

```bash
D='2026-04-30'   # data suspeita (ajuste)
```

## PAM (backdoor de autenticação)

```bash
sudo grep -RIna 'pam_exec|pam_.*\.so' /etc/pam.d 2>/dev/null
sudo find /lib*/security /usr/lib*/security -name 'pam_*.so' -newermt "$D" 2>/dev/null
```

## udev (executa em evento de device)

```bash
sudo grep -RInaE 'RUN|PROGRAM' /etc/udev/rules.d 2>/dev/null
```

## Módulos de kernel (rootkit / autoload)

```bash
sudo grep -RIna '' /etc/modules-load.d /etc/modprobe.d 2>/dev/null
sudo lsmod            # compare com baseline; módulos fora de árvore são suspeitos
cat /proc/sys/kernel/tainted   # != 0 pode indicar módulo não assinado/rootkit (decodifique os bits)
sudo dmesg | grep -iE 'taint|module verification failed'
# processos/portas escondidos por LKM: compare visões (ver 'unhide' na §32)
```

## Scripts de login SSH (rodam a cada login)

```bash
sudo find /home /root -path '*/.ssh/rc' 2>/dev/null
sudo cat /etc/ssh/sshrc 2>/dev/null
```

## MOTD dinâmico (Ubuntu; roda no login)

```bash
sudo ls -la /etc/update-motd.d/ 2>/dev/null
sudo grep -RInaE 'curl|wget|base64|/tmp|/dev/shm' /etc/update-motd.d 2>/dev/null
```

## Cron restante (além da §7.1)

```bash
sudo ls -la /etc/cron.weekly/ /etc/cron.monthly/ 2>/dev/null
sudo cat /etc/anacrontab 2>/dev/null
```

## Hooks de gerenciador de pacote (rodam em update)

```bash
sudo grep -RInaE 'curl|wget|bash|/tmp|/dev/shm' \
  /etc/apt/apt.conf.d /etc/dnf/plugins /etc/yum/pluginconf.d \
  2>/dev/null
```

---

# 8. Arquivos suspeitos pelo filesystem

> **Por quê:** achar o que foi plantado, mesmo sem um processo rodando no momento. **Olhe para:** ELF/executáveis em `~/.config`, `~/.cache`, `/tmp`, `/dev/shm`, e arquivos criados na janela do incidente.

Executáveis em locais incomuns:

```bash
sudo find \
/home \
/tmp \
/var/tmp \
/dev/shm \
-type f \
-perm /111 \
-ls \
2>/dev/null
```

---

## ELFs em home/tmp

```bash
sudo find \
/home \
/tmp \
/var/tmp \
/dev/shm \
-type f \
-exec sh -c '
  file "$1" | grep -q ELF && echo "$1"
' _ {} \; \
2>/dev/null
```

---

## Arquivos recentemente alterados

Últimas 24 horas:

```bash
sudo find \
/home \
/tmp \
/var/tmp \
/dev/shm \
/data \
-ctime -1 \
-ls \
2>/dev/null
```

Últimos 7 dias:

```bash
sudo find \
/home \
/tmp \
/var/tmp \
/dev/shm \
/data \
-ctime -7 \
-ls \
2>/dev/null
```

---

# 9. Timeline

> **Confie no relógio antes de confiar na timeline.** Toda análise por `ctime`/`mtime` assume que a hora do host está correta. Se o atacante mexeu no relógio (ou o NTP falhou), os timestamps mentem. Verifique e, na dúvida, use os **timestamps off-box (cloud/log central)** como verdade-âncora — são independentes do host.

```bash
timedatectl                      # timezone, sincronização NTP, offset
sudo journalctl -u systemd-timesyncd -u chronyd -u ntpd 2>/dev/null | grep -iE 'step|jump|adjust|change'
last -x | grep -iE 'time change'   # mudanças de hora registradas no wtmp
```

Defina uma janela:

```bash
START='2026-04-30 18:20:00'
END='2026-04-30 18:40:00'
```

Buscar por `ctime`:

```bash
sudo find \
/home \
/tmp \
/var/tmp \
/dev/shm \
/data \
-xdev \
-newerct "$START" \
! -newerct "$END" \
-printf '%CY-%Cm-%Cd %CT %u %g %m %s %p\n' \
2>/dev/null \
| sort
```

Buscar por `mtime`:

```bash
sudo find \
/home \
/tmp \
/var/tmp \
/dev/shm \
/data \
-xdev \
-newermt "$START" \
! -newermt "$END" \
-printf '%TY-%Tm-%Td %TT %u %g %m %s %p\n' \
2>/dev/null \
| sort
```

> Em incidente, normalmente vale procurar primeiro por `ctime`.

```text
-xdev       não atravessa mounts → evita varrer NFS/procfs e travar (veja abaixo)
-newerct X  ctime mais recente que X
! -newerct Y  ... e mais antigo que Y  → o par delimita a janela
-printf     imprime o timestamp junto, para o 'sort' ordenar de fato
```

> **Por que a timeline funciona:** uma intrusão é uma sequência de escritas em segundos. Ordenados por tempo, os artefatos contam a história sozinhos — diretório criado, binário escrito, `chmod +x`, cron alterado, log tocado. Se você tem só **um** timestamp confiável (o do artefato), ele é o âncora: leve-o para todas as outras fontes (§9.1).

> Comece com a janela **estreita** (minutos em volta do âncora) e vá abrindo. Janela larga em `/home` retorna milhares de linhas de ruído legítimo e você perde o sinal.

---

## Filesystems remotos

Ver mounts:

```bash
findmnt
```

Se houver NFS/CIFS/Gluster, pesquise separadamente:

```bash
sudo find /backup \
-newerct "$START" \
! -newerct "$END" \
-printf '%CY-%Cm-%Cd %CT %u %g %m %s %p\n' \
2>/dev/null \
| sort
```

---

## 9.1. Correlação de sinais

Um ataque deixa rastro em VÁRIAS fontes no mesmo instante. A técnica: achar um **âncora temporal** e pivotar para todas as outras fontes na mesma janela. Muita coisa que parece "operacional" é, na verdade, tentativa/scan/exploração.

### Sinais que podem ser ataque (não só bug)

```text
restart de serviço fora do padrão   → exploit malformado derruba o processo; supervisor relança
muitos restarts / crash-loop         → fuzzing, DoS, ou exploração instável
pico no tamanho/volume de log        → tráfego anômalo (scan, brute, exploração)
rajada de 4xx (401/403/404)          → scan de rotas / brute force / enumeração
rajada de 5xx / erros de parse       → payloads malformados (tentativa de exploração)
segfault / OOM-killer                → exploração de memória / DoS
conexões recusadas/timeouts em massa → varredura de portas
```

### Achar o âncora (quando algo "estranho" aconteceu)

```bash
# restarts de serviço (systemd): quantos e quando
systemctl show <SVC> -p NRestarts
sudo journalctl -u <SVC> | grep -iE 'started|stopped|main process exited|failed|signal'
# crashes no kernel/sistema
sudo journalctl -k | grep -iE 'segfault|oom|killed process'
sudo grep -iE 'segfault|oom-killer|panic|traceback' /var/log/messages* 2>/dev/null
```

### Padrão-base vs desvio (o desvio é o âncora)

```bash
# VOLUME de log por dia: um salto = anomalia (aquele dia teve algo)
for f in /var/log/<SVC>*; do printf '%s\t' "$(stat -c %s "$f")"; echo "$f"; done | sort -n
# meses de ~250 bytes/dia e de repente 6 KB num dia -> investigue esse dia

# rajada de erros HTTP no access log (scan/fuzz)
sudo awk '{print $9}' /var/log/nginx/access.log 2>/dev/null | sort | uniq -c | sort -rn
# 4xx/5xx agrupados por minuto (pico = varredura)
sudo grep -E ' (4[0-9]{2}|5[0-9]{2}) ' /var/log/nginx/access.log 2>/dev/null \
  | awk '{print $4}' | cut -d: -f1-3 | uniq -c | sort -rn | head
```

### Com a janela definida, cruze TODAS as fontes

```bash
START='2026-04-30 18:20:00'; END='2026-04-30 18:40:00'
```

```text
filesystem (ctime)   → §9      arquivos escritos no instante = o kit dropado
auth / SSH           → §12     login/pivô no mesmo minuto?
audit (EXECVE)       → §11     comando executado + processo pai (o vetor)
web/access log       → §10     a requisição de exploração (método, rota, payload, IP)
restart do serviço   → 9.1     o processo caiu/reiniciou no instante?
rede                 → §2      conexão de saída (C2) logo após?
cloud (off-box)      → §10.4   se o log local rotacionou
```

> Convergência é prova: se o `ctime` de um artefato, um restart do serviço, uma
> requisição anômala no access log e um `EXECVE` no audit caem no **mesmo segundo**,
> a cadeia está amarrada. Um sinal isolado é pista; a convergência é o laudo.

---

# 10. Logs

> **Por quê:** achar a requisição de entrada e a atividade do atacante no tempo. **Olhe para:** rajada de 4xx/5xx (scan/fuzz), `POST` estranho na hora do incidente, erros de auth. **Cheque a retenção primeiro** — se o log local já rotacionou, a evidência pode estar off-box (§10.4).

---

## 10.1. systemd journal

Últimas horas:

```bash
sudo journalctl --since '2 hours ago'
```

Por período:

```bash
sudo journalctl \
  --since '2026-04-30 18:20:00' \
  --until '2026-04-30 18:40:00'
```

Por service:

```bash
sudo journalctl -u <SERVICE>
```

Por PID:

```bash
sudo journalctl _PID=<PID>
```

Integridade (anti-forense — logs adulterados/truncados):

```bash
sudo journalctl --verify 2>&1 | tail        # journal corrompido/alterado?
ls -la /var/log/* | grep ' 0 '              # arquivos zerados (truncados)
sudo journalctl --list-boots                # gaps de boot inesperados
```

> Gaps, arquivos zerados ou `--verify` com erro = possível limpeza de log. A cópia off-box (§10.4) não é alterável pelo atacante — prefira-a.

---

## 10.2. RHEL/CentOS

Principais:

```text
/var/log/secure
/var/log/messages
/var/log/cron
/var/log/audit/
/var/log/nginx/
/var/log/httpd/
```

Buscar IOC:

```bash
sudo zgrep -Hni '<IOC>' \
/var/log/secure* \
/var/log/messages* \
/var/log/cron* \
/var/log/nginx/* \
/var/log/httpd/* \
2>/dev/null
```

---

## 10.3. Debian/Ubuntu

Principais:

```text
/var/log/auth.log
/var/log/syslog
/var/log/nginx/
/var/log/apache2/
```

```bash
sudo zgrep -Hni '<IOC>' \
/var/log/auth.log* \
/var/log/syslog* \
/var/log/nginx/* \
/var/log/apache2/* \
2>/dev/null
```

---

## 10.4. Cloud — a evidência pode estar OFF-BOX

Logs locais rotacionam rápido (às vezes 2–5 dias). Se o incidente é antigo, o local pode ter sumido — mas o cloud retém mais e é à prova de adulteração pelo atacante.

**Primeiro: cheque a retenção local (decide se ainda existe aqui).**

```bash
ls -la /var/log/cron* /var/log/messages* /var/log/secure* /var/log/audit/ 2>/dev/null
# até qual data os .gz vão? Se não cobre o incidente -> vá para o cloud.
```

**Há agente de logging? (define o que foi para o cloud)**

```bash
systemctl status google-cloud-ops-agent stackdriver-agent \
  amazon-cloudwatch-agent rsyslog 2>/dev/null
cat /etc/google-cloud-ops-agent/config.yaml 2>/dev/null   # quais arquivos são enviados
```

**Evidência off-box (retenção longa). Lembre: cloud usa UTC.**

```bash
# GCP — Cloud Logging (syslog/audit shipado):
gcloud logging read \
  'timestamp>="2026-04-30T21:20:00Z" AND timestamp<="2026-04-30T21:40:00Z"' \
  --limit 300 --freshness=200d
# GCP — Cloud Audit / Admin Activity (SEMPRE ligado, ~400 dias):
#   abuso de credencial da service-account (via SSRF ao metadata) aparece aqui
# GCP — VPC Flow Logs (se habilitado): conexão de entrada na porta + saída do C2
# AWS — CloudTrail (90d+), VPC Flow Logs, CloudWatch Logs
```

## 10.5. Metadata / roubo de credencial de instância (cloud)

Se a app comprometida tem SSRF, o atacante alcança o metadata e rouba o token da service-account/instance-role → **pivô para todo o projeto/conta**. Verifique e trate como exposto:

```text
GCP:  http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/token
AWS:  http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

* Rotacione/reveja a **service-account da instância** (não só os `.env`).
* Revise o audit do cloud por uso anômalo do token **após** a data do incidente.
* Bloqueie egress ao `169.254.169.254` para o usuário da app (ver §18).

---

# 11. Auditd

> **Por quê:** se o auditd estiver ativo, é a fonte mais forte — registra `execve` e escrita de arquivo **com o processo pai e o usuário**, amarrando o vetor. **Olhe para:** `sh`/`curl`/`bash`/`base64` cujo pai é o processo da aplicação, na janela do incidente. (Se estiver desligado, pule.)

Está ligado?

```bash
sudo systemctl is-active auditd ; sudo auditctl -s ; sudo auditctl -l
```

> **O que torna o auditd único:** ele grava no **kernel**, no momento da syscall, com `ppid`, `auid` e `exe` juntos. O `auid` (*audit UID*) é o carimbo do login original e **não muda** com `su`/`sudo` — é o que liga uma ação de `root` de volta à pessoa que entrou. Nenhuma outra fonte do host faz isso.

Por período:

```bash
sudo ausearch \
-ts '04/30/2026 18:20:00' \
-te '04/30/2026 18:40:00'
```

Por UID:

```bash
sudo ausearch \
-ts '04/30/2026 18:20:00' \
-te '04/30/2026 18:40:00' \
-ua <UID> \
-i
```

Executáveis:

```bash
sudo ausearch -m EXECVE -i
```

Por arquivo (quem tocou/escreveu) e por PID:

```bash
sudo ausearch -f /home/<USER>/.config/htop/defunct -i   # quem criou/acessou o arquivo
sudo ausearch -p <PID> -i                                # tudo do processo
```

> Amarra o vetor: um `EXECVE` de `sh`/`curl`/`bash` cujo processo pai é o processo da aplicação (o serviço que escuta a porta) prova o pulo "RCE → comando".

Se estava **desligado**, ligue agora — o próximo movimento do atacante (ou a reincidência da §22) fica gravado:

```bash
sudo auditctl -w /etc/passwd -p wa -k ir_passwd
sudo auditctl -w /home/<USER>/.ssh -p wa -k ir_ssh
sudo auditctl -a always,exit -F arch=b64 -S execve -F auid>=1000 -k ir_exec
```

> Regras via `auditctl` valem até o reboot; para permanentes, `/etc/audit/rules.d/`. Custo: `execve` sem filtro gera volume alto — use `-F` para restringir a usuário/diretório de interesse.

---

# 12. Login / SSH e movimentação lateral

> **Por quê:** ver acesso interativo e se o atacante saltou de/para outros hosts. **Olhe para:** login de IP incomum na hora do incidente, `authorized_keys` novo, túneis SSH (`-L/-R/-D`), agente encaminhado (`SSH_AUTH_SOCK`), conexões de saída para :22 interno.

Usuários atuais:

```bash
who
w
```

Histórico:

```bash
last -Fai
```

Último login:

```bash
lastlog
```

```text
last      lê /var/log/wtmp   (histórico de login/logout)
lastb     lê /var/log/btmp   (tentativas FALHAS → brute force)
lastlog   lê /var/log/lastlog (último login por usuário)
who/w     leem /var/run/utmp  (sessões AGORA)
```

> **Esses arquivos são binários e graváveis por root** — apagar a própria sessão do `wtmp` (com `utmpdump`, editar, recarregar) é anti-forense básica. Sinais: `last` com salto temporal, arquivo com tamanho 0, `ctime` recente sem motivo. **A prova real está no `sshd` do log de auth** (`/var/log/secure`, `auth.log`, journal), que registra IP, chave/fingerprint usada e método — e melhor ainda na cópia off-box (§10.4).

```bash
sudo lastb | head -20                                    # brute force antes do sucesso?
sudo stat /var/log/wtmp /var/log/btmp /var/log/lastlog   # zerado ou tocado?
```

Conexões SSH atuais:

```bash
sudo ss -tnp | grep ':22'
```

---

## 12.1. Movimentação lateral — a partir deste host

Descobrir se o atacante saltou daqui para OUTRAS máquinas.

### SSH de saída (este host como cliente)

```bash
# conexões de saída para :22 (este host conectando em outros)
sudo ss -tnp | awk '$5 ~ /:22$/'
# processos ssh/scp/sftp ativos e seus argumentos (revela destino e túneis)
sudo ps -eo pid,user,args | grep -E '\bssh\b|scp|sftp|rsync' | grep -v grep
```

### Alvos que este host alcançou

```bash
# hosts para onde já se conectou (mesmo com HashKnownHosts, o nº de entradas mostra alcance)
sudo find /home /root -path '*/.ssh/known_hosts' -exec sh -c 'echo "== $1 =="; wc -l "$1"' _ {} \;
# atalhos/config: ProxyJump, ForwardAgent, hosts pré-configurados
sudo find /home /root -path '*/.ssh/config' -exec cat {} \;
```

### Agent forwarding sequestrado (pivô sem roubar a chave)

Se um SSH com `-A`/`ForwardAgent` chegou aqui, o atacante usa o **socket do agente** para saltar para onde a chave tem acesso, sem nunca ver a chave privada.

```bash
# sockets de agente ativos
sudo ls -la /tmp/ssh-*/agent.* 2>/dev/null
# processos com SSH_AUTH_SOCK no environ (podem reusar o agente)
for p in $(pgrep -u '<USER>' .); do
  sudo grep -qa SSH_AUTH_SOCK /proc/$p/environ 2>/dev/null && echo "PID $p usa SSH_AUTH_SOCK"
done
```

### Chaves privadas expostas (roubo para pivô)

```bash
sudo find /home /root /tmp /dev/shm -name 'id_*' ! -name '*.pub' -type f 2>/dev/null
sudo find / -xdev -name 'authorized_keys' -newermt "$D" 2>/dev/null   # chave nova plantada
```

## 12.2. Túneis / port forwarding

```bash
# túneis SSH: -L (local), -R (reverse), -D (SOCKS), -w (vpn)
sudo ps -eo pid,user,args | grep -E 'ssh.* -[LRDW]' | grep -v grep
# ferramentas comuns de túnel/proxy usadas em pivô
sudo ps -eo pid,user,args | grep -E 'socat|chisel|ngrok|frpc?|gost|iodine|sshuttle|3proxy|proxychains' | grep -v grep
# listeners inesperados em 0.0.0.0 (podem ser forwards/proxy)
sudo ss -ltnp | grep -vE '127\.0\.0\.1|::1'
```

## 12.3. Credenciais/artefatos que habilitam pivô

O que este usuário conseguia ler pode ter sido usado para se mover:

```bash
sudo find /home /root -maxdepth 3 2>/dev/null \( \
  -name '.netrc' -o -name '.pgpass' -o -name '.git-credentials' \
  -o -path '*/.aws/credentials' -o -path '*/.kube/config' \
  -o -path '*/.docker/config.json' -o -name '*.pem' -o -name '*.ppk' \) -print
ls -la /tmp/krb5cc_* 2>/dev/null                 # tickets kerberos
env | grep -iE 'TOKEN|KEY|SECRET|PASS' 2>/dev/null
```

Credenciais de cloud (metadata/instância) → pivô no projeto: ver §10.5.

## 12.4. Mapa de rede interna (para onde deu para ir)

```bash
ip neigh                    # ARP: hosts com quem falou recentemente
ip route                    # sub-redes alcançáveis
sudo conntrack -L 2>/dev/null | grep -vE '127\.0\.0\.1'   # conexões internas
# sinais de varredura interna (muitos SYN de saída para :22/:445/:3306):
sudo ss -tan | awk '$1=="SYN-SENT"{print $5}' | sed 's/:[0-9]*$//' | sort | uniq -c | sort -rn | head
```

## 12.5. Este host foi ALVO de lateral? (pivô de entrada)

```bash
# logins SSH de entrada vindos de IPs INTERNOS (RHEL: secure; Debian: auth.log)
sudo zgrep -aE 'Accepted (password|publickey)' /var/log/secure* /var/log/auth.log* 2>/dev/null \
  | grep -E '10\.|172\.(1[6-9]|2[0-9]|3[01])\.|192\.168\.' | tail -40
# o serviço vulnerável era alcançável só internamente? (então veio de dentro)
```

> Lógica: se o backdoor roda como um usuário de app e a entrada foi por um serviço exposto (§14/§16), provavelmente **não** houve lateral de entrada — foi exploração direta. Confirme cruzando IP de origem (interno vs externo) com a data do incidente.

## 12.6. Movimentação lateral por serviço vulnerável (não-SSH)

Nem toda lateral é por SSH. Um atacante em outro host interno pode explorar uma **porta de serviço** exposta na VPC (backend de app, DB, painel admin) batendo **direto na porta** — contornando qualquer proxy/WAF que só existe na borda. Bater direto no backend = mesmo efeito que `localhost` (sem a normalização do proxy).

> **Por quê:** confirmar se a entrada veio de uma VM/host interno (lateral) em vez de fora. **Olhe para:** conexões na porta vulnerável vindas de **IP interno da VPC**, e o firewall interno que libera essa porta entre hosts.

### Quem alcança a porta vulnerável (superfície interna)

```bash
# a porta escuta em todas as interfaces? (0.0.0.0/:: = alcançável por outros hosts, não só localhost)
sudo ss -ltnp | grep ':<PORTA>'
# firewall da cloud: a porta é liberada DENTRO da VPC?
gcloud compute firewall-rules list --filter="allowed.ports:<PORTA>" \
  --format='table(name,sourceRanges.list(),allowed[].ports)'        # GCP
# AWS: aws ec2 describe-security-groups --filters Name=ip-permission.to-port,Values=<PORTA>
```

### A app foi acessada por origem interna?

```bash
# conexões na porta vindas de IP RFC1918 (interno)
sudo ss -tnp | grep ':<PORTA>' | grep -E '10\.|172\.(1[6-9]|2[0-9]|3[01])\.|192\.168\.'
sudo conntrack -L 2>/dev/null | grep '<PORTA>'
# logs do serviço/proxy: IP de origem interno na janela do incidente
# VPC Flow Logs (cloud): conexões de OUTRAS VMs -> :<PORTA> em torno do incidente
```

### O inverso: este host atacou serviços de OUTRAS VMs?

```bash
# varredura/exploração de saída para serviços internos (comportamento worm-like)
sudo ss -tan | awk '$1=="SYN-SENT"{print $5}' | sed 's/:[0-9]*$//' | sort | uniq -c | sort -rn | head
sudo conntrack -L 2>/dev/null | grep -vE '127\.0\.0\.1' | head
```

> **Distinção final:** IP de origem **externo** (no log do proxy/borda) = entrada externa; IP de origem **interno** direto na porta (conntrack/VPC Flow) = **lateral de entrada** a partir de outra VM. Um proxy/WAF na borda **não protege** a porta se ela for alcançável direto dentro da rede — feche a porta no firewall interno e faça bind em `127.0.0.1` quando possível.

---

# 13. Bash history

> **Por quê:** às vezes o atacante deixa os comandos que rodou. **Olhe para:** `curl|bash`, `base64 -d`, edição de cron, `chmod +x`, download de binário. E cheque anti-forense: history apontando para `/dev/null` ou zerado. **Ausência de comandos ≠ ausência de ataque** — RCE não passa por shell interativo.

```bash
sudo cat /home/<USER>/.bash_history
```

Filtrar:

```bash
sudo grep -niE \
'curl|wget|base64|cron|chmod|bash|sh|tmp|ssh|scp' \
/home/<USER>/.bash_history
```

Cheque anti-forense comum (history desviado para o nada):

```bash
ls -la /home/<USER>/.bash_history   # symlink para /dev/null ou tamanho 0 = suspeito
```

Não trate ausência de comandos no history como evidência de ausência de ataque.

RCE normalmente não passa por shell interativo.

---

# 14. Identificar serviço vulnerável

> **Por quê:** ligar a porta/serviço exposto ao processo e ao usuário — é o vetor de entrada. **Olhe para:** qual aplicação escuta a porta atacada, sob qual usuário; cruze com o usuário do backdoor (a heurística abaixo estreita o vetor).

Liste listeners:

```bash
sudo ss -ltnp
```

Exemplo:

```text
*:4025 users:(("node",pid=3173,...))
```

Pegue o PID:

```bash
PID=3173
```

Descubra aplicação:

```bash
sudo readlink -f /proc/$PID/exe
sudo readlink -f /proc/$PID/cwd
sudo sh -c "tr '\0' ' ' < /proc/$PID/cmdline"
```

## Heurística: usuário do malware → estreita o vetor

O backdoor rodou como um usuário específico. **Só serviços que rodam como ESSE usuário podem ser o vetor** — o resto provavelmente não é a entrada (e ajuda a descartar "movimentação lateral").

```bash
USER_MAL=$(sudo stat -c %U /proc/$PID/exe 2>/dev/null || echo '<user>')
# quais serviços/portas rodam com esse usuário?
ps -u "$USER_MAL" -o pid,comm,args
sudo ss -ltnp | grep "\"$USER_MAL\"" 2>/dev/null || sudo ss -ltnp   # cruze usuário x listener
```

Exemplo: backdoor roda como `app-user`; o servidor web PHP roda como `www-data`. Um RCE no PHP daria `www-data`, não `app-user` → **PHP não é o vetor**; olhe os serviços que rodam como `app-user`.

---

# 15. Reverse proxy

> **Por quê:** mapear o caminho Internet → app e onde o payload realmente passa. **Olhe para:** rotas/`proxy_pass` para o serviço atacado, se o proxy **não** filtra o corpo, e se o backend está exposto direto (bypass do proxy).

## nginx

Config completa:

```bash
sudo nginx -T
```

Filtrar:

```bash
sudo nginx -T 2>/dev/null \
  | grep -nE \
  'server_name|listen|location|proxy_pass|fastcgi_pass'
```

---

## Apache

```bash
sudo apachectl -S
```

```bash
sudo grep -RInE \
'ProxyPass|ProxyPassReverse|RewriteRule.*http' \
/etc/httpd \
/etc/apache2 \
2>/dev/null
```

Objetivo:

```text
Internet
↓
LB / proxy
↓
nginx/apache
↓
porta interna
↓
aplicação
```

## Duas armadilhas do proxy (não confie nele como proteção)

1. **O proxy repassa o CORPO da requisição.** Ele pode normalizar/bloquear a URL, mas o payload no body (JSON/form) chega intacto à app. Filtro de WAF/regra na URL **não** protege sinks alimentados pelo corpo (SSRF, deserialização, template injection).

2. **A porta interna pode estar exposta direto** — o atacante pula o proxy:

```bash
# a porta do backend responde SEM passar pelo proxy?
sudo ss -ltnp | grep -vE '127\.0\.0\.1|::1'      # backend em 0.0.0.0/:: = alcançável direto
curl -s -m5 http://<IP_PUBLICO>:<PORTA_BACKEND>/  # de fora: se responder, o proxy é bypassável
```

Correção: bind do backend em `127.0.0.1`, firewall na porta interna, e proteção na camada que a app realmente lê (não só na URL do proxy).

---

# 16. Camada de aplicação (agnóstico de linguagem)

> **Por quê:** o binário e a persistência são o *efeito*; aqui está a **causa** — a linha de código que deixou entrada externa virar execução. Sem fechá-la, tudo se repete. **Olhe para:** um `sink` perigoso alimentado por dado que veio da requisição.

**O modelo mental (source → sink):** vulnerabilidade = dado controlado pelo atacante (*source*: URL, header, body, upload, nome de arquivo) chegando a uma função poderosa (*sink*: executa comando, resolve URL, escreve arquivo, desserializa) sem validação no meio. Os `grep` abaixo acham os **sinks** — poucos e nomeáveis em qualquer linguagem. Achado o sink, siga o dado para trás até a rota HTTP: se o caminho existe, é o vetor.

Achar o serviço vulnerável e o "sink" que virou RCE — sem assumir a stack (PHP, Python, Node, Ruby, Go, Java, shell). Ajuste os diretórios de código:

```bash
APP='/srv /opt /var/www /data /home'
EXC='node_modules|vendor|/\.git/|site-packages'
```

## Integridade do código (rápido e preciso)

Apps versionadas em git delatam adulteração na hora:

```bash
for b in /srv /opt /var/www /data /home; do
  find "$b" -maxdepth 5 -name .git -type d 2>/dev/null | while read g; do
    r=$(dirname "$g")
    git -C "$r" status --short 2>/dev/null | grep -q . && { echo "== ALTERADO: $r =="; git -C "$r" diff --stat 2>/dev/null; }
  done
done
```

Sem git: use timeline por `ctime` (§9) e o gerenciador de pacotes (§24).

## Execução de comando (sinks de RCE, multi-linguagem)

```bash
sudo grep -RInaE \
  'system\(|popen\(|shell_exec|proc_open|passthru|exec[lvpe]*\(|child_process|spawn|subprocess|os\.system|Runtime\.getRuntime|ProcessBuilder|eval\(|assert\(|unserialize\(|pickle\.loads|Marshal\.load|yaml\.load\(' \
  $APP 2>/dev/null | grep -vE "$EXC"
```

## Invocação de shell

```bash
sudo grep -RInaE '/bin/bash|/bin/sh|bash -c|sh -c|exec -a' $APP 2>/dev/null | grep -vE "$EXC"
```

## Requisições externas / SSRF (o servidor busca URL do cliente)

```bash
sudo grep -RInaE 'curl|wget|file_get_contents|fopen|http\.get|https\.get|fetch\(|requests\.get|urlopen|HttpClient|WebClient' $APP 2>/dev/null | grep -vE "$EXC"
```

## Escrita de arquivo controlada (path traversal → drop/overwrite)

```bash
sudo grep -RInaE 'file_put_contents|fwrite|writeFile|createWriteStream|move_uploaded_file|shutil\.copy|os\.rename|copyFile' $APP 2>/dev/null | grep -vE "$EXC"
```

## Upload

```bash
sudo grep -RInaE 'multipart|form-data|move_uploaded_file|\$_FILES|multer|MultipartFile|FileUpload' $APP 2>/dev/null | grep -vE "$EXC"
```

## Componentes desatualizados (EOL/CVE) — manifests de qualquer stack

```bash
sudo find $APP -maxdepth 4 \( -name package.json -o -name composer.json \
  -o -name requirements.txt -o -name go.mod -o -name pom.xml -o -name Gemfile \) 2>/dev/null
```

## Webshell no docroot

Arquivos plantados na raiz web que executam comando (backdoor clássico). Ajuste `WEBROOT`:

```bash
WEBROOT='/var/www /srv/www /usr/share/nginx /opt/*/public'
# recém-criados na janela do incidente:
sudo find $WEBROOT -type f -newermt "$D" 2>/dev/null
# padrões de webshell (agnóstico):
sudo grep -RIlnaE \
  'eval\(|assert\(|system\(|shell_exec|passthru|base64_decode\(.*\)|popen\(|proc_open|`.*\$_(GET|POST|REQUEST)|FromBase64String|Runtime\.exec' \
  $WEBROOT 2>/dev/null
```

> A porta que a app escuta (§14) + o sink encontrado aqui = o vetor. Correlacione com o `ctime` do artefato dropado (§9) e o processo pai do `EXECVE` no audit (§11).

---

# 17. Indicadores fortes de reverse shell/backdoor

> **Por quê:** cada sinal isolado tem explicação inocente; a **combinação** não tem. **Olhe para:** a conjunção abaixo — é a assinatura estrutural de shell remoto, independente de família de malware.

**A lógica:** um shell reverso precisa, por construção, de (1) um canal de saída para o operador, (2) um interpretador para executar o que chega e (3) um TTY para o operador digitar confortavelmente. Você não está reconhecendo um malware específico — está reconhecendo o **formato** de qualquer um deles. É por isso que esse checklist envelhece bem.

Combinação especialmente suspeita:

```text
processo desconhecido
+
socket outbound para IP público
+
child bash/sh
+
PTY (/dev/ptmx)
```

Exemplo:

```text
systemd
└─defunct
   └─defunct
      └─bash
```

e:

```text
/home/node/.config/htop/defunct
→ IP_PUBLICO:443
→ /dev/ptmx
```

Trate como comprometimento confirmado até prova em contrário.

> **Por que "reverse" e não bind shell:** firewall e NAT bloqueiam conexão de *entrada*, mas quase todo host pode **sair** para a internet — em especial na 443, que ninguém bloqueia e onde o tráfego se mistura ao HTTPS legítimo. Daí a regra prática: a defesa que realmente dói para o atacante é **egress default-deny** (§18.1), não mais uma regra de INPUT.

> Consequência para a caça: procure pela conexão **de saída** iniciada pelo host, não por porta aberta esperando. E lembre que o *beacon* pode ser periódico — um C2 que fala de 10 em 10 minutos não aparece num `ss` tirado no minuto errado; repita a coleta ou use `conntrack`/flow logs (§12.4, §10.4).

---

# 18. Contenção

> **Por quê:** cortar o acesso do atacante — mas só depois de coletar o mínimo (senão você destrói evidência). **Olhe para:** ordem — bloquear C2, isolar a porta/serviço vulnerável, e lembrar que C2 por relay não para bloqueando 1 IP (use egress).

Depois de coletar o mínimo necessário:

```text
processo
rede
exe
ctime
hash
persistência
```

contenha.

---

## 18.1. Bloquear C2

### iptables

```bash
sudo iptables \
-I OUTPUT 1 \
-d <IOC_IP> \
-j DROP
```

Confirmar:

```bash
sudo iptables \
-L OUTPUT \
-n -v \
--line-numbers
```

Idealmente bloqueie também no firewall da cloud/rede.

> **C2 por relay não para bloqueando 1 IP.** Backdoors como GSocket/gs-netcat, Tor ou domain-fronting saem por uma rede de relays sem IP fixo. Nesses casos, prefira **egress default-deny** (libere só o necessário) e bloqueie o metadata:

```bash
sudo iptables -A OUTPUT -m owner --uid-owner <USER> -d 169.254.169.254 -j REJECT
sudo iptables -A OUTPUT -m owner --uid-owner <USER> \
  -d 10.0.0.0/8,172.16.0.0/12,192.168.0.0/16 -j REJECT
# e, se possível, DROP de saída do usuário da app exceto destinos aprovados
```

---

## 18.2. Desabilitar serviço vulnerável

Systemd:

```bash
sudo systemctl stop <SERVICE>
sudo systemctl disable <SERVICE>
```

PM2:

```bash
sudo -iu <USER> pm2 stop <APP>
```

Nginx/backend:

* retire rota;
* remova backend do load balancer;
* restrinja firewall;
* coloque autenticação emergencial, se necessário.

---

# 19. Remover persistência

> **Por quê:** a ordem é o que faz a limpeza funcionar. **Olhe para:** remover gatilho → matar processo → apagar binário. Invertido, o supervisor relança o malware antes de você chegar no `rm` — e você entrega que foi detectado.

Faça isso **antes de matar o malware**, para evitar relaunch.

```text
1. coletar/preservar   §6    depois não tem volta
2. remover gatilhos    §19   cron, unit, supervisor, chave
3. matar processo      §20   árvore inteira, de uma vez
4. remover binário     §21   já com hash e cópia guardados
5. validar             §22   e re-scan em 24-48h
```

> Registre **cada** remoção (o quê, onde, hora, conteúdo original). Um cron apagado sem cópia é evidência perdida — e se o serviço quebrar depois, você não sabe se era legítimo.

---

## Cron

```bash
sudo crontab -u <USER> -l
```

Editar:

```bash
sudo crontab -u <USER> -e
```

---

## Systemd

```bash
sudo systemctl disable --now <UNIT>
```

Depois:

```bash
sudo rm -f /etc/systemd/system/<UNIT>
sudo systemctl daemon-reload
```

Somente se a unit foi confirmada como maliciosa.

---

## SSH

Remova somente chave confirmada como maliciosa de:

```text
~/.ssh/authorized_keys
```

---

# 20. Matar processo

> **Por quê:** derrubar o malware — mas só **depois** de coletar (§6) e remover a persistência (§19), senão o supervisor/cron o relança. **Olhe para:** matar a árvore inteira (pai + filhos) e confirmar que não reapareceu.

Primeiro:

```bash
sudo kill -TERM <PID>
```

Aguarde:

```bash
sleep 2
```

Verifique:

```bash
ps -p <PID>
```

Se continuar:

```bash
sudo kill -KILL <PID>
```

> **TERM x KILL:** `SIGTERM` (15) é capturável — o processo pode ignorar, limpar rastro, avisar o C2 ou reexecutar-se em outro lugar. `SIGKILL` (9) é entregue pelo kernel e não passa pelo programa. Em incidente, se há suspeita de handler, vá **direto de KILL**; o TERM educado só faz sentido em serviço legítimo que você quer parar limpo.

> **Se o KILL não mata**, o processo está em estado `D` (I/O ininterrupto — travado em rede/NFS) ou é thread de kernel. Nesses casos, corte a fonte: bloqueie o C2 (§18.1) ou desconecte a interface; o processo sai do `D` e morre.

---

## Parent + child

Liste a árvore:

```bash
sudo pstree -aps <PID>
```

Mate os PIDs confirmados:

```bash
sudo kill -TERM <PID1> <PID2> <PID3>
sleep 2
```

> **Mate a árvore num único comando.** Só o filho: o pai o recria. Só o pai: o filho é **reparentado para o PID 1** e continua rodando — agora sem o pai que te levaria até ele. Um comando com todos os PIDs elimina a janela em que um recria o outro.

Se necessário:

```bash
sudo kill -KILL <PID1> <PID2> <PID3>
```

---

# 21. Remover/quarentenar arquivo

> **Por quê:** eliminar o binário — só depois de hash + cópia + persistência removida + processo morto. **Olhe para:** confirmar que a cópia preservada (§6) existe antes do `rm`.

Depois de:

```text
hash
stat
cópia
persistência removida
processo morto
```

Remova:

```bash
sudo rm -f <FILE>
```

---

# 22. Validar depois da limpeza

> **Por quê:** confirmar que a limpeza pegou tudo. **Olhe para:** sem a conexão de C2, sem o listener, sem o processo, sem o arquivo, e nada em "deletado-em-execução". Se algo persiste, você não removeu toda a persistência.

Conexões:

```bash
sudo ss -tnp state established
```

Listeners:

```bash
sudo ss -ltnp
```

Processos:

```bash
sudo pgrep -af '<IOC>'
```

Arquivo:

```bash
sudo find /home /tmp /var/tmp /dev/shm \
-name '<IOC>' \
-print \
2>/dev/null
```

Executável deletado:

```bash
sudo lsof +L1
```

Rede:

```bash
sudo lsof -Pan -i
```

> **Re-scan em 24–48h.** Atacante costuma voltar (persistência que você não achou, ou re-exploração do vetor se ele não foi fechado). Repita §2 (rede), §3.14 (deletados), §8 (ELF em home/tmp) e confira se o listener/C2 reapareceu. Se voltou, ainda há persistência ou o vetor segue aberto (§18).

---

# 23. Procurar segunda persistência

> **Por quê:** quem investe em invadir investe em voltar. A primeira persistência costuma ser a descartável — ela existe para ser achada. **Olhe para:** os mesmos IOCs em lugares que você ainda não olhou, e nos **outros hosts**.

Vire a busca do avesso: em vez de procurar "malware", procure **o seu IOC** (hash, nome, IP, chave, usuário) em todo lugar onde ele possa ter sido referenciado.

```bash
sudo grep -RInaE \
'<IOC>|exec -a|base64|curl|wget|/tmp|/dev/shm' \
/home \
/etc \
/var/spool/cron \
/tmp \
/var/tmp \
/dev/shm \
/data \
2>/dev/null
```

## Varredura da frota (mesmo IOC em outros hosts)

Um comprometimento raramente é de 1 host só. Com os IOCs deste host (hash do binário, nome do arquivo, IP/domínio de C2, chave SSH plantada, usuário criado), **varra os demais servidores** — o atacante pode ter movido lateralmente (§12) ou atingido vários pela mesma exposição.

```bash
# rode nos outros hosts (ssh/ansible/parallel):
sha256sum <ARQUIVO_IOC> 2>/dev/null                      # mesmo binário?
sudo find / -xdev -name '<NOME_IOC>' 2>/dev/null          # mesmo nome/artefato
sudo grep -RIna '<CHAVE_SSH_ATACANTE>' /home /root 2>/dev/null   # mesma chave plantada
sudo ss -tnp | grep '<C2_IP>'                             # mesma conexão de C2
```

> Compare também: mesma porta/serviço exposto (o vetor), mesmo usuário criado, mesma janela de `ctime`. Um host "limpo" que compartilha o vetor ainda está em risco até você fechá-lo.

---

# 24. Verificar integridade do sistema

> **Por quê:** detectar binários de sistema adulterados (trojanização de `ls`, `ps`, `ss`, `sshd`…). **Olhe para:** checksum divergente em `/bin`, `/usr/bin`, `/sbin`, `/lib*`, `/etc`. Adulteração aqui = alto risco → considere rebuild (§27).

## RHEL/CentOS

```bash
sudo rpm -Va
```

## Debian/Ubuntu

```bash
sudo dpkg -V
```

Como ler a saída (mesmo formato nos dois, por arquivo alterado):

```text
S.5....T.  c /etc/foo.conf
│ │    │   └ tipo: c=config (mudança esperada), vazio=binário/lib (NÃO esperado)
│ │    └ T  mtime mudou
│ └ 5  checksum mudou   ← o campo que importa
└ S    tamanho mudou
outros: M modo/permissão | U dono | G grupo | L link simbólico | D device
'missing' = arquivo do pacote sumiu   |  '?' = não verificável
```

> **Regra de leitura:** `5` em arquivo **não-config** dentro de `/bin`, `/usr/bin`, `/sbin`, `/lib*` = binário do sistema adulterado (trojan de `ls`, `ps`, `ss`, `sshd`). Diferenças em `/etc/*.conf` são normais — você mesmo edita config.

> **Limite honesto:** esta verificação compara com a base de dados **local** de pacotes (RPM/dpkg). Um atacante root pode reescrever essa base junto com o binário, e nada aqui cobre arquivos que não vieram de pacote. Divergência aqui é prova forte de comprometimento; ausência de divergência **não** é prova de limpeza. Para garantia real, compare com um host-espelho ou use baseline offline (AIDE/Tripwire, §32).

Procure alterações inesperadas principalmente em:

```text
/bin
/sbin
/usr/bin
/usr/sbin
/lib*
/etc
```

---

# 25. Capabilities e SUID

> **Por quê:** achar caminhos de escalonamento persistente deixados pelo atacante. **Olhe para:** capability/SUID inesperado (ex.: `cap_setuid` num binário estranho, SUID em `/tmp`/home) — compare com um baseline conhecido.

Capabilities:

```bash
sudo getcap -r / 2>/dev/null
```

SUID:

```bash
sudo find / \
-xdev \
-type f \
-perm -4000 \
-ls \
2>/dev/null
```

Compare com baseline conhecido.

> **O que cada mecanismo dá:** SUID faz o binário rodar como o **dono do arquivo** (SUID root = root). Capability dá um **pedaço** do poder de root ao binário — mais discreto, porque não aparece num `find -perm -4000`. Ambos são caminho de escalonamento *persistente*: ficam no disco esperando, mesmo depois de você matar o processo.

```text
cap_setuid       vira qualquer usuário       → root direto
cap_dac_override ignora permissões de arquivo → lê/escreve tudo
cap_sys_ptrace   injeta em processos alheios  → rouba credencial da memória
cap_sys_module   carrega módulo de kernel     → rootkit
cap_net_raw      sniffing/spoofing
```

> Suspeite de: SUID em `/tmp`, `/home` ou `/dev/shm` (nunca legítimo), SUID em cópia de shell/interpretador (`bash`, `python`, `perl`, `find`, `cp`) — é a receita pronta de escalonamento — e capability em binário que não é do sistema. `cp` de um binário SUID **não** preserva o bit sem `-a`/root, então um SUID em home foi posto ali de propósito.

---

# 26. Rotação de credenciais

> **Por quê:** o atacante teve tempo de leitura no host; qualquer segredo alcançável saiu de lá. Limpar o malware não invalida o que ele copiou. **Olhe para:** o **raio de alcance do usuário comprometido**, não só o que você acha que ele usou.

**Presuma cópia, não uso.** O uso pode vir semanas depois, de outro IP, contra outro sistema — e aí não parece mais incidente, parece acesso normal. Por isso rotação não é opcional nem "quando der".

Se um processo de aplicação foi comprometido, considere exposto tudo que aquele usuário conseguia ler:

```text
.env
variáveis PM2
PostgreSQL
MongoDB
Redis
RabbitMQ
API keys
tokens
GCP/AWS credentials
service accounts
SSH keys
certificados
cookies/tokens internos
```

Rotacione fora do host comprometido.

> **Fora do host** e nesta ordem: primeiro **revogue** a credencial antiga (senão a nova convive com a velha ainda válida), depois emita a nova, depois distribua. Rotacionar de dentro da máquina comprometida entrega a credencial nova ao mesmo atacante.

> Não esqueça o que não é senha: **tokens de sessão/refresh e cookies continuam válidos** depois da troca de senha — invalide as sessões. E a credencial da instância na cloud (§10.5) não fica em nenhum `.env`: rotacione a service-account/role também.

---

# 27. Quando rebuild é necessário

Rebuild da máquina a partir de imagem confiável quando:

* atacante teve root;
* malware foi executado como root;
* houve alteração de binários do sistema;
* não é possível garantir integridade;
* houve rootkit/kernel module;
* não há baseline confiável;
* a máquina é crítica e o impacto justifica rebuild.

Limpar:

```text
processo
arquivo
cron
```

é contenção.

Não é garantia de que o host voltou a ser confiável.

> **O raciocínio:** limpeza exige enumerar **tudo** que o atacante fez — e você só remove o que encontrou. Rebuild inverte o problema: parte de um estado sabidamente bom e não depende da sua enumeração ter sido completa. Com root, a assimetria é definitiva: as próprias ferramentas que você usaria para verificar podem estar mentindo.

> **Rebuild não fecha o incidente.** Se o vetor de entrada (§14/§16) continuar aberto, a máquina nova é comprometida de novo — às vezes em minutos. Ordem: fechar o vetor → rebuild a partir de imagem confiável → restaurar dados **validados** (backup anterior ao incidente; §9 datou isso) → rotacionar credenciais (§26) → varrer a frota (§23).

---

# 28. Coleta rápida em um único bloco

> **Por quê:** evidência tem prazo de validade. **Olhe para:** coletar na **ordem de volatilidade** — o que morre primeiro, primeiro. Um reboot (ou o kill da §20) apaga para sempre o que só existia em memória.

```text
mais volátil  memória, processos, conexões, sockets, /proc   segundos
              tabela de rotas/ARP, conntrack                 minutos
              arquivos temporários, /tmp, /dev/shm           horas
              disco, logs locais                             dias (rotação!)
menos volátil snapshot, logs off-box/cloud                   meses
```

> Consequência prática: `ss`/`ps`/`lsof` **antes** de qualquer `find` demorado. E o log local pode ser mais volátil que o disco — se a rotação for de 2 dias, ele expira antes do arquivo (§10.4).

Para uma triagem inicial:

```bash
IR=/root/ir-$(date +%Y%m%d-%H%M%S)

sudo mkdir -p "$IR"
sudo chmod 700 "$IR"

date -Is | sudo tee "$IR/date.txt"

sudo ss -tunap \
  > "$IR/ss-tunap.txt"

sudo ss -tnp state established \
  > "$IR/ss-established.txt"

sudo ss -ltnp \
  > "$IR/ss-listen.txt"

sudo lsof -Pan -i \
  > "$IR/lsof-network.txt"

sudo ps auxww \
  > "$IR/ps.txt"

sudo pstree -ap \
  > "$IR/pstree.txt"

sudo lsof +L1 \
  > "$IR/deleted-open-files.txt"

sudo systemctl list-unit-files \
  > "$IR/systemd-units.txt"

sudo systemctl list-timers --all \
  > "$IR/systemd-timers.txt"

sudo iptables-save \
  > "$IR/iptables.txt" \
  2>/dev/null || true

sudo nft list ruleset \
  > "$IR/nftables.txt" \
  2>/dev/null || true
```

---

# 29. Coleta rápida de um PID

```bash
PID=<PID>

sudo ps -o \
pid,ppid,user,group,lstart,etime,stat,args \
-p "$PID"

sudo pstree -aps "$PID"

sudo readlink -f /proc/$PID/exe
sudo readlink -f /proc/$PID/cwd

sudo sh -c \
"tr '\0' ' ' < /proc/$PID/cmdline"
echo

sudo sh -c \
"tr '\0' '\n' < /proc/$PID/environ"

sudo grep -E \
'Name|State|Pid|PPid|Uid|Gid|TracerPid|Cap' \
/proc/$PID/status

sudo ls -la /proc/$PID/fd

sudo lsof -Pan -p "$PID"

sudo lsof -Pan -a -p "$PID" -i
```

---

# 30. Coleta rápida de um arquivo

```bash
FILE=<FILE>

sudo stat "$FILE"
sudo file "$FILE"
sudo sha256sum "$FILE"

sudo getfacl "$FILE"
sudo lsattr "$FILE"
sudo getcap "$FILE"

sudo strings -a -n 8 "$FILE" \
  | head -200

sudo readelf -h "$FILE"
sudo readelf -d "$FILE"

sudo objdump -p "$FILE"

sudo lsof "$FILE"
sudo fuser -v "$FILE"
```

---

# 31. Red flags para reconhecer rapidamente

> **Como usar:** nenhum item abaixo é prova sozinho — cada um tem explicação legítima possível. O valor está em ser **anomalia barata de checar**: o custo de olhar é segundos, o custo de não olhar é o incidente inteiro. Dois ou três juntos, no mesmo processo ou na mesma janela de tempo, deixam de ser coincidência.

**O princípio por trás de todos:** software legítimo é **previsível** — vem de pacote, mora onde a distro põe, roda com o usuário esperado, fala com quem sempre falou. Red flag é toda quebra dessa previsibilidade. Por isso conhecer o **normal do seu host** vale mais que qualquer lista.

## Rede

```text
processo desconhecido → IP público
```

```text
usuário de aplicação → conexão outbound inesperada
```

```text
conexão persistente em 443 sem processo esperado
```

---

## Processo

```text
nome tipo kernel mas executável em /home
```

```text
PPID 1 sem motivo conhecido
```

```text
processo em /tmp ou /dev/shm
```

```text
child bash/sh de processo desconhecido
```

```text
PTY + socket remoto
```

---

## Arquivo

```text
ELF em ~/.config
```

```text
ELF em ~/.cache
```

```text
ELF em /tmp
```

```text
mtime antigo + ctime recente
```

```text
arquivo sem pacote correspondente
```

---

## Persistência

```text
base64 | bash
```

```text
curl | sh
```

```text
exec -a
```

```text
cron com nome tentando parecer kernel
```

```text
unit systemd apontando para /home ou /tmp
```

```text
authorized_keys alterado sem mudança conhecida
```

---

# 32. Ferramentas úteis para scan/análise

Os comandos deste runbook resolvem a triagem sem instalar nada. Ferramenta entra quando você precisa de **escala** (varrer tudo/vários hosts), **profundidade** (memória, binário, disco cru) ou **segunda opinião** (baseline, host não confiável).

> ⚠️ **Num host comprometido, binário local pode estar adulterado** — rootkit esconde do scanner que roda em cima dele. Prefira **snapshot/cópia offline** (§1) ou binário estático trazido de fora. Instalar pacote no host suspeito também custa: mexe na base de pacotes (que é sua evidência na §24), gera tráfego e pode avisar o atacante.

## Como escolher

```text
achar o que eu JÁ SEI que existe (IOC)    → yara, Loki, clamav, osquery
achar o que eu NÃO SEI                    → pspy/execsnoop (ao vivo), plaso (timeline), lynis
provar que binário do sistema mudou       → debsums / rpm -Va, AIDE, hashdeep
binário sem strings / packed              → AVML + Volatility, capa, upx -d
o arquivo foi APAGADO                     → sleuthkit (fls/icat), Volatility (bash na RAM)
o mesmo comprometimento em N hosts        → osquery, ansible, Fenrir
fechar o vetor                            → trivy/osv-scanner, nmap/nuclei (de fora)
```

## Antes de instalar nada (já está no host)

```text
ss, lsof, ps, pstree     §2, §3      estado ao vivo
ausearch/auditctl        §11         a fonte mais forte, se estiver ligada
rpm -Va / dpkg -V        §24         integridade dos pacotes
getcap, find -perm       §25         escalonamento deixado para trás
strings/readelf/objdump  §5          análise estática do binário
gcore                    §6          dump de memória de UM processo
tcpdump                  §2.6        tráfego
capsh --decode=<hex>     §3.7        traduz CapEff
busybox                  §7.8        segunda opinião contra LD_PRELOAD
```

## Monitorar ao vivo (auditd desligado, ou pegar no flagrante)

```text
pspy              # execs e cron em tempo real SEM root e sem instalar nada
bcc-tools/bpftrace# execsnoop, opensnoop, tcpconnect — via eBPF, custo baixo
sysdig / falco    # captura de syscall com filtro; falco = regras de detecção
```
```bash
./pspy64                                   # binário único, sem deps: vê o cron relançar o malware
sudo execsnoop-bpfcc                       # todo execve, com PPID  (RHEL: /usr/share/bcc/tools/)
sudo tcpconnect-bpfcc                      # toda conexão de saída + PID → pega o beacon periódico
sudo sysdig -p'%evt.time %proc.pname %proc.name %evt.args' evt.type=execve
```

> Serve para dois momentos: **antes** de mexer (o C2 é periódico? o beacon aparece de 10 em 10 min?) e **depois** da limpeza (§22) — algo ainda tenta relançar? É o substituto do auditd quando ele não estava ligado.

## Rootkit / objetos escondidos

```text
rkhunter          # checa rootkits, binários alterados, portas conhecidas
chkrootkit        # idem, heurísticas clássicas
unhide            # processos/portas escondidos (compara ps/ss x /proc x syscalls)
```
```bash
sudo rkhunter --update && sudo rkhunter --check --sk    # --sk = não pedir ENTER a cada bloco
sudo chkrootkit -q                                      # -q = só o que é anômalo
sudo unhide quick                                       # PID visível em /proc mas não no ps
sudo unhide-tcp                                         # porta em LISTEN que o ss não mostra
```

> ⚠️ **Nunca rode `rkhunter --propupd` num host suspeito**: ele grava o estado **atual** como baseline — ou seja, carimba o trojan como legítimo e você perde a comparação para sempre.

> Falso positivo aqui é regra, não exceção (esses scanners são heurísticos e desatualizam). Trate o resultado como **pista para investigar** com §3/§5, nunca como veredito. Já o `unhide` é diferente: divergência entre duas visões do mesmo kernel é sinal técnico forte.

## Malware / assinatura / IOC

```text
clamav (clamscan) # antivírus; scan recursivo de paths suspeitos
maldet (LMD)      # Linux Malware Detect; forte em webshell/backdoor de docroot
yara              # regras de IOC (família, C2, strings) — o formato padrão da indústria
Loki / THOR-Lite  # scanner pronto: YARA + hash + nome de arquivo + regras — bom 1º pass
Fenrir            # mesma ideia em bash puro, sem dependência — host minimalista/container
```
```bash
sudo freshclam && sudo clamscan -r -i /home /tmp /var/tmp /dev/shm /data   # -i = só infectados
sudo maldet -a /var/www                       # scan sob demanda do docroot (§16)
sudo yara -r -w regras.yar /home /tmp         # rulesets: Yara-Rules, Neo23x0/signature-base
python3 loki.py -p / --noprocscan             # ajuste o escopo; roda pesado em / inteiro
sudo ./fenrir.sh /var/www
```

> **Assinatura só acha o conhecido: resultado negativo não inocenta o host.** O valor está em varrer volume rápido (docroot, `/home`, `/tmp`) e em rodar **suas** regras YARA com os IOCs deste incidente — foi para isso que você tirou hash e strings na §5.

Regra YARA mínima com o que você já tem (serve para a frota, §23):

```text
rule ioc_incidente {
  strings: $c2 = "51.91.190.241"  $n = "defunct"
  condition: any of them or hash.sha256(0, filesize) == "<SHA256>"
}
```

## Coleta padronizada (quando o caso vai escalar)

```text
UAC     # Unix-like Artifacts Collector — coleta completa e reproduzível em 1 pacote
CyLR    # coletor rápido de artefatos
```
```bash
sudo ./uac -p ir_triage /evidence     # perfis: ir_triage, full; saída .tar.gz + hash
```

> Use quando terceiro/jurídico/seguro vai olhar depois: saída padronizada e documentada vale mais que a sua coleta ad-hoc da §28 — e evita a acusação de "manipulou a evidência".

## Análise do binário (sem executar — §5.9)

```text
binwalk        # payload/arquivo embutido dentro do binário
upx            # detecta e desempacota o packer mais comum
capa           # traduz o binário em CAPACIDADES ("abre socket", "persiste via cron")
radare2/rizin  # análise/desmontagem interativa
ghidra         # descompilador, quando precisa entender o protocolo de C2
vt (VirusTotal CLI)
```
```bash
binwalk -e "$FILE"                          # extrai o que estiver embutido
upx -t "$FILE" && upx -d -o unpacked "$FILE"   # se for UPX, resolve o "sem strings" da §5.5
capa "$FILE"                                # melhor custo/benefício: entende sem ler assembly
r2 -A "$FILE"                               # dentro: 'iz' strings, 'afl' funções, 'ii' imports
vt file <SHA256>                            # consulta por HASH — não suba o arquivo (§5.4)
```

## Integridade / baseline

```text
debsums           # (Debian) checksum dos pacotes — complementa dpkg -V
AIDE / Tripwire   # baseline de integridade (compara pré x pós)
hashdeep          # hash recursivo + modo de auditoria contra lista conhecida
osquery           # o host como SQL (também serve para frota, abaixo)
Wazuh / OSSEC     # HIDS: FIM, rootkit, correlação de log — o que evita o próximo incidente
lynis / openscap  # auditoria de hardening e compliance/CVE
```
```bash
sudo debsums -c                                     # lista só os arquivos alterados
sudo aide --check                                   # exige DB criada ANTES e guardada FORA do host
hashdeep -r -c sha256 /usr/bin /bin /sbin > now.txt
hashdeep -r -a -k baseline.txt /usr/bin             # -a audita contra a baseline; sai != 0 se divergir
sudo lynis audit system                             # o que deixou o vetor possível, em ordem de risco
```

> Baseline só existe se foi feita **antes** e mora **fora** do host — senão o atacante a atualiza junto. Sem baseline, o substituto prático é comparar hashes com uma **máquina-espelho** da mesma imagem e versão de pacote.

## Memória (binário packed, C2 em RAM)

```text
AVML              # aquisição em Linux cloud (Microsoft) — binário estático, sem compilar módulo
LiME              # aquisição via módulo de kernel (precisa dos headers da versão exata)
Volatility 3      # análise do dump: processos, conexões, injeção, history
```
```bash
sudo ./avml /evidence/mem.lime                       # capturar ANTES de matar (§6/§20)
vol -f mem.lime linux.pslist.PsList                  # processos como o KERNEL os vê
vol -f mem.lime linux.lsof.Lsof                      # fds/sockets do dump
vol -f mem.lime linux.malfind.Malfind                # região RWX anônima = código injetado
vol -f mem.lime linux.bash.Bash                      # history recuperado da RAM
vol -f mem.lime linux.check_syscall.Check_syscall    # syscall table hookada = rootkit de kernel
```

> **Gotcha do Volatility em Linux:** ele precisa do **símbolo (ISF) do kernel exato** daquele host — sem isso, nenhum plugin roda. Gere com `dwarf2json` a partir do `vmlinux`/debug symbols da mesma versão, e faça isso **enquanto a máquina existe**.

> Dois motivos para não pular a memória: o `linux.bash.Bash` recupera comandos mesmo com `~/.bash_history` apagado ou apontando para `/dev/null` (§13), e o binário packed só está descompactado **ali** (§5.5).

## Disco: arquivo apagado, timeline completa

```text
sleuthkit (fls, icat, mactime)  # lê o filesystem CRU: enxerga o que foi deletado
extundelete / photorec          # recuperação de conteúdo apagado
plaso / log2timeline            # super-timeline: filesystem + logs + artefatos, tudo ordenado
mac_robber + mactime            # timeline clássica, leve
```
```bash
sudo dd if=/dev/sdX of=/evidence/disk.img bs=4M status=progress   # ou use o snapshot da §1
fls -r -d disk.img                          # -d = só ENTRADAS DELETADAS, com inode
icat disk.img <INODE> > recuperado.bin      # recupera o conteúdo pelo inode
mac_robber /mnt/evidence | mactime -d -b - > timeline.csv
log2timeline.py --storage_file plaso.db disk.img && psort.py -o dynamic plaso.db
```

> **Por que isso vai além da §9:** `find -newerct` só enxerga arquivo que **existe**. O dropper que o atacante usou e apagou não aparece lá — mas o inode ainda está no filesystem, e `fls`/`icat` o trazem de volta. Rode sempre sobre a **imagem/snapshot**, nunca no disco montado em escrita.

## Componentes vulneráveis (fechar o vetor — §16)

```text
trivy         # CVE em filesystem, código, imagem de container e IaC
grype         # CVE a partir de SBOM/imagem
osv-scanner   # CVE via base OSV, direto dos manifests (Google)
npm audit / pip-audit / bundler-audit / govulncheck   # nativo de cada stack
```
```bash
trivy fs --scanners vuln /srv/app        # casa com os manifests achados na §16
trivy image <IMAGEM>                     # o container já subiu vulnerável?
osv-scanner -r /srv/app
```

> Isto responde a pergunta que fecha o incidente: **por onde entrou**. Cruze a data do CVE e a versão instalada com a janela da §9 — se bate, você tem a hipótese do vetor, não só o efeito.

## Frota: a mesma pergunta em N hosts (§23)

```text
osquery       # transforma o host em SQL; osqueryi local ou fleet centralizado
ansible/pssh  # rodar o mesmo comando/IOC em todos os hosts
Fenrir/Loki   # scanner de IOC portátil para distribuir
```
```bash
osqueryi "SELECT pid,name,path,uid FROM processes WHERE path LIKE '/tmp/%' OR path LIKE '/dev/shm/%';"
osqueryi "SELECT DISTINCT p.name,p.path,l.port,l.address FROM listening_ports l JOIN processes p USING(pid);"
osqueryi "SELECT * FROM users WHERE uid=0;"
osqueryi "SELECT path,sha256 FROM hash WHERE path='/usr/bin/ss';"
ansible all -m shell -a "ss -tnp | grep -F '<C2_IP>'; sha256sum <ARQ> 2>/dev/null"
```

> osquery é a forma prática de fazer a §23 sem escrever script por host: mesma pergunta, resposta tabelada, comparável entre máquinas. É também a melhor forma de **levantar baseline** antes do próximo incidente.

## Rede: identificar/atribuir o C2 (passivo)

```text
dig, whois, ipinfo   # dono do bloco, ASN, país, PTR — sem tocar no atacante
ngrep, tshark        # inspeção de pacote a partir do pcap (§2.6)
zeek                 # transforma pcap em logs de SESSÃO (conn/ssl/dns/http)
```
```bash
dig -x <IP> +short ; whois <IP> | grep -iE 'netname|orgname|country|abuse'
curl -s https://ipinfo.io/<IP>
tshark -r suspect.pcap -Y 'tls.handshake.type==1' \
  -T fields -e ip.dst -e tls.handshake.extensions_server_name   # SNI = domínio do C2
zeek -r suspect.pcap && cat conn.log ssl.log dns.log
```

> ⚠️ **Consulta passiva (DNS/whois/ipinfo) é segura; conectar de volta ao C2 não.** `curl`, `nmap` ou browser contra o IP do atacante confirma para ele que foi detectado — e ele reage apagando rastro e trocando de infraestrutura. Nunca "vá ver o que tem lá".

## Validar SUA superfície (externo, só com autorização)

Confirmar o que está exposto e reproduzir o vetor — rode de fora, contra o próprio ativo:

```text
nmap / naabu      # portas e serviços expostos (ex.: um backend que devia ser interno)
testssl.sh        # TLS/config do endpoint
nuclei            # scan de vulnerabilidades por templates (CVE, exposições)
ffuf / gobuster   # enumeração de rotas/parametros de uma app web
nikto             # checks web genéricos
```
```bash
nmap -Pn -p- --min-rate 1000 <IP>         # a porta interna responde de fora? (o bypass da §15)
nmap -sV -p <PORTA> <IP>                  # versão do serviço → casa com CVE conhecida
testssl.sh https://<host>
nuclei -u https://<host>                  # templates de exposição/CVE
ffuf -u https://<host>/FUZZ -w wordlist.txt -mc 200,301,403   # rota esquecida/painel exposto
```

> O achado que mais importa aqui é simples: **um serviço que devia ser interno respondendo do lado de fora**. Compare o resultado do `nmap` externo com o `ss -ltnp` local (§14) — o que aparece nos dois é sua superfície real.

> Scan ativo é intrusivo e pode gerar tráfego/alertas — faça só em ambiente/ativo autorizado e, de preferência, avisando o time.

---

# 33. Regra principal

Não comece procurando uma CVE.

Comece pelo que está acontecendo agora:

```text
IP
↓
PID
↓
PPID
↓
executável real
↓
usuário
↓
arquivos abertos
↓
arquivo/binário
↓
persistência
↓
timeline
↓
aplicação responsável
↓
vetor de entrada
```

Cada passo responde uma pergunta, e a resposta é a entrada do próximo:

```text
o que fala com fora?      → conexão
quem abriu isso?          → PID
quem executou esse?       → PPID / árvore
o que ele É de verdade?   → /proc/<pid>/exe
com que poder roda?       → usuário, capabilities
quando apareceu?          → ctime → janela → todas as fontes (§9.1)
por onde ele entrou?      → serviço na porta + sink na app (§14/§16)
```

> **CVE é conclusão, não ponto de partida.** Procurar CVE primeiro é adivinhar; a máquina já está te contando o que aconteceu — leia o estado atual e deixe a hipótese aparecer no fim. Enquanto o **vetor** não estiver fechado, limpar é adiar: o host volta a ser comprometido pelo mesmo caminho.

> E encerre respondendo três coisas por escrito: **como entrou**, **o que fez** (e alcançou), **como impedir que volte**. Sem as três, o incidente não está fechado — só está quieto.

A terceira resposta é a §34.

---

# 34. Reduzir o blast radius (blindagem da VM)

> **Por quê:** as §1–§33 assumem que já deu errado. Esta assume que **vai** dar errado de novo — e trata do que muda quando dá. **Olhe para:** cada controle abaixo transformando um comprometimento *total* num comprometimento *contido*.

**A troca de pergunta:** de "como impedir a invasão?" (impossível garantir) para "**quando um processo da app for controlado pelo atacante, o que exatamente ele alcança?**". Esse alcance tem quatro dimensões — e cada uma se corta separadamente:

```text
identidade  o que o processo PODE fazer         → usuário, capabilities, sandbox
rede        com quem ele PODE falar             → egress, segmentação
dados       o que ele PODE ler                  → secrets, escopo da credencial
tempo       por quanto tempo ele fica           → host descartável, rotação, detecção
```

> **Toda a §7 (persistência) precisa de duas coisas: escrever em algum lugar e ser executada por algo.** Quase tudo aqui existe para tirar uma dessas duas.

---

## 34.1. Sandbox do serviço (systemd)

O maior ganho por linha editada — e não precisa mudar o código da app. Aplique na unit do serviço exposto:

```ini
[Service]
User=app                       # nunca root (o óbvio que quase sempre falta)
NoNewPrivileges=yes            # mata SUID e capabilities: §25 deixa de ser rota
ProtectSystem=strict           # todo o filesystem read-only...
ReadWritePaths=/var/lib/app    # ...exceto o que a app realmente escreve
ProtectHome=yes                # /home invisível → mata o drop em ~/.config e ~/.cache (§8)
PrivateTmp=yes                 # /tmp e /dev/shm privados → mata o drop em /tmp (§8)
CapabilityBoundingSet=         # vazio = nenhuma capability, nunca (§3.7)
RestrictSUIDSGID=yes
MemoryDenyWriteExecute=yes     # sem página W+X → quebra packer e injeção (§5.5, §3.10)
LockPersonality=yes
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX    # sem AF_NETLINK/AF_PACKET
SystemCallFilter=@system-service
SystemCallFilter=~@module @mount @debug @swap       # sem carregar módulo, montar, ptrace
ProtectKernelModules=yes
ProtectKernelTunables=yes
ProtectProc=invisible          # a app não enxerga /proc dos outros processos (§3)
IPAddressDeny=any              # egress default-deny no cgroup do serviço
IPAddressAllow=10.0.0.0/8      # ...só o que precisa
```

Meça antes e depois — o systemd dá nota:

```bash
systemd-analyze security <SERVICE>          # 0 (ótimo) a 10 (exposto), item a item
sudo systemctl edit <SERVICE>               # aplica como override, sem tocar na unit do pacote
sudo systemctl daemon-reload && sudo systemctl restart <SERVICE>
```

> **Aplique incrementalmente.** `ProtectSystem=strict` e `SystemCallFilter` quebram app que escreve fora do esperado — suba um por vez, teste, e leia o `journalctl -u <SERVICE>` para ver o que foi negado. Vale o trabalho: com isso, o mesmo RCE que hoje deixa um binário em `~/.config` e abre C2 passa a não ter onde escrever nem para onde falar.

---

## 34.2. Filesystem sem execução

Onde o atacante escreve não deve executar; onde executa não deve ser gravável.

```bash
# /etc/fstab
/tmp        tmpfs  defaults,noexec,nosuid,nodev,size=1G  0 0
/var/tmp    ext4   defaults,noexec,nosuid,nodev          0 0
/dev/shm    tmpfs  defaults,noexec,nosuid,nodev          0 0
/home       ext4   defaults,nosuid,nodev                 0 0
```

```bash
findmnt -o TARGET,OPTIONS /tmp /var/tmp /dev/shm /home    # confere o que está valendo AGORA
```

> **Limite honesto:** `noexec` não é barreira absoluta — um interpretador contorna (`sh < script`, `python arquivo`) e o próprio loader também (`/lib64/ld-linux-x86-64.so.2 ./binario`). O que ele faz é **encarecer e barulhar**: quebra dropper preguiçoso, força o atacante a técnicas mais visíveis, e combina com `MemoryDenyWriteExecute` (34.1) para fechar a alternativa em memória.

---

## 34.3. Egress default-deny

**O controle de maior alavancagem do arquivo inteiro.** Quase todo backdoor precisa sair (§17); quase nenhum servidor precisa falar com a internet inteira.

```bash
# saída do usuário da app: só o aprovado
sudo iptables -A OUTPUT -m owner --uid-owner app -d <API_APROVADA> -j ACCEPT
sudo iptables -A OUTPUT -m owner --uid-owner app -d 169.254.169.254 -j REJECT   # metadata (§10.5)
sudo iptables -A OUTPUT -m owner --uid-owner app -j REJECT
```

* No firewall da **cloud** também (regra de egress na VPC) — o do host cai junto com o host.
* Bloqueie o **metadata** para todo processo que não precisa dele: é o pivô da §10.5. Na AWS, exija **IMDSv2** e reduza o hop limit; no GKE, ative *metadata concealment*/Workload Identity.
* Serviço interno faz `bind` em `127.0.0.1`, não em `0.0.0.0` (§15) — e a porta interna fecha no firewall da VPC (§12.6).

> Isto é o que derruba C2 por relay (GSocket, Tor, domain fronting), que **não** para bloqueando IP (§18.1): sem lista de saída permitida, não há IP para bloquear.

---

## 34.4. Identidade e credenciais

O raio de um vazamento é o raio da credencial vazada.

```text
uma identidade por serviço      → o comprometimento de um não move o outro
permissão mínima e escopada     → sem role ampla de projeto/conta na service-account da VM
credencial de vida curta        → token que expira vale menos que .env eterno
secret manager, não .env        → busca em runtime, com audit de acesso e rotação
conta de serviço SEM shell      → /usr/sbin/nologin (§7.9)
home da app sem escrita         → sem ~/.ssh, sem ~/.config graváveis (mata §7.3/§7.5/§7.6)
```

```bash
sudo usermod -s /usr/sbin/nologin app
sudo chown root:root /home/app && sudo chmod 755 /home/app     # app não escreve no próprio home
```

> A pergunta de teste: *"se o token desta VM vazar agora, o que ele abre?"*. Se a resposta for "o projeto inteiro", o blast radius é o projeto inteiro — e nenhum hardening no host muda isso.

---

## 34.5. Segmentação e acesso

```text
um serviço por host             → o RCE não alcança o banco que roda ao lado
firewall ENTRE tiers na VPC     → :22, :3306, :6379 não abertos host-a-host (§12.6)
SSH só via bastion              → ProxyJump; nada de porta 22 exposta em cada VM
chave por pessoa/host           → chave compartilhada = lateral de graça
sem ForwardAgent (-A)           → use ProxyJump: o agente encaminhado é pivô pronto (§12.1)
```

```bash
# ~/.ssh/config — no SEU cliente
Host bastion
  ForwardAgent no
Host app-*
  ProxyJump bastion
  ForwardAgent no
```

---

## 34.6. Tempo: hosts descartáveis

A dimensão que quase todo mundo esquece — e a mais eficaz contra persistência.

```text
deploy por imagem, não por SSH   → mudança feita à mão no host é anomalia detectável
redeploy periódico               → o host morre em dias; a persistência da §7 morre junto
root filesystem read-only        → não há onde plantar
sem SSH em produção              → sem shell interativo, o §13 fica vazio de propósito
```

> Um host que vive 7 dias limita o *dwell time* a 7 dias, sem você precisar detectar nada. É a §27 (rebuild) transformada em rotina, e não em emergência.

---

## 34.7. Detecção que reduz raio

Detectar cedo é reduzir raio no eixo tempo. Ligue **antes** de precisar:

```text
auditd ligado com regras         §11    sem ele, o vetor fica sem prova
log enviado para fora na hora    §10.4  o atacante não apaga o que não está no host
FIM (Wazuh/OSSEC/AIDE)           §32    alerta em escrita em /etc, /usr/bin, authorized_keys
alerta de egress inesperado      §2     conexão de saída do usuário da app = alarme
baseline (hash, portas, SUID)    §24/25 sem "normal" registrado, tudo parece normal
```

```bash
sudo auditctl -w /etc/passwd -p wa -k id_change
sudo auditctl -w /root/.ssh -p wa -k ssh_keys
sudo auditctl -a always,exit -F arch=b64 -S execve -F auid>=1000 -k exec
# baseline enquanto o host está limpo — guarde FORA dele:
sudo ss -ltnp > baseline-ports.txt
sudo find / -xdev -perm -4000 -ls > baseline-suid.txt
sudo getcap -r / 2>/dev/null > baseline-caps.txt
hashdeep -r -c sha256 /usr/bin /bin /sbin > baseline-hashes.txt
```

---

## 34.8. Auditoria rápida do estado atual

Rode agora e veja seu raio real:

```bash
systemd-analyze security --no-pager | head -20        # serviços mais expostos, ordenados
sudo ss -ltnp | grep -vE '127\.0\.0\.1|::1'           # o que escuta além do localhost?
ps -eo user,comm --no-headers | sort -u | grep '^root' # o que roda como root sem precisar?
findmnt -o TARGET,OPTIONS /tmp /var/tmp /dev/shm      # noexec/nosuid estão lá?
awk -F: '$3>=1000 && $7!~/nologin|false/ {print $1,$7}' /etc/passwd   # contas com shell
sudo iptables -S OUTPUT | tail -5                     # existe política de egress?
sudo getcap -r / 2>/dev/null                          # capabilities soltas (§25)
sudo lynis audit system                               # relatório completo, com prioridade
```

---

## O que NÃO reduz blast radius

```text
antivírus no host          assinatura não pega o que é feito sob medida (§32)
WAF só na borda            não vê o corpo da requisição nem a porta direta (§15)
senha forte de root        irrelevante se a entrada é RCE na app (§16)
"a porta está atrás do LB" ela responde direto dentro da VPC (§12.6)
fail2ban sozinho           trata brute force; não trata exploração na 1ª requisição
```

> O padrão: todos protegem a **entrada**. Blast radius é sobre o **depois** da entrada — é aí que a maioria dos ambientes não tem nada.

---

# 35. Comprometimento em nível de kernel

> **Por quê:** todo o resto deste runbook pergunta ao kernel ("quais processos existem?", "quais conexões?"). Se o **kernel** foi comprometido, todas as respostas são fornecidas pelo atacante. **Olhe para:** não o objeto escondido (você não vai vê-lo), mas a **incoerência** entre duas formas de perguntar a mesma coisa.

**O problema fundamental:** `ps`, `ss`, `ls`, `lsof` e até ler `/proc` são syscalls. Um rootkit de kernel intercepta a syscall e devolve a realidade editada. Não existe ferramenta *dentro* da caixa que seja imune — inclusive as da §32.

Daí três consequências práticas:

```text
1. ausência de sinal NÃO é prova de limpeza     (você está perguntando ao suspeito)
2. o que denuncia é a INCOERÊNCIA               (o rootkit mente numa via e esquece a outra)
3. a resposta confiável vem de FORA             (memória via hypervisor, rede a montante)
```

---

## 35.1. Antes de tudo: é kernel mesmo?

Quase toda suspeita de "rootkit de kernel" acaba sendo outra coisa. Descarte na ordem — é mais rápido e mais provável:

```text
1. LD_PRELOAD / ld.so.preload      §7.8   rootkit de USERLAND, muito mais comum
2. binário de sistema trocado      §24    'ls'/'ps' trojanizados, sem kernel envolvido
3. processo só camuflado           §3     nome forjado; /proc/<pid>/exe entrega
4. aí sim: kernel                  §35
```

> **Pré-requisito lógico:** carregar módulo exige **root** (`CAP_SYS_MODULE`), e explorar o kernel exige um LPE. Ou seja, rootkit de kernel é **segundo estágio** — só acontece depois que o atacante já era root. Se você não achou comprometimento de root (§7.9, §25), a hipótese de kernel é fraca. Se achou, ela é obrigatória.

---

## 35.2. Taint e módulos

O kernel registra permanentemente quando algo "sujo" foi carregado:

```bash
T=$(cat /proc/sys/kernel/tainted); echo "tainted=$T"
for i in $(seq 0 18); do [ $(( (T>>i) & 1 )) -eq 1 ] && echo "  bit $i"; done
sudo dmesg | grep -iE 'taint|module verification failed|out-of-tree|lockdown'
```

Bits que importam aqui:

```text
bit 0  (1)     módulo proprietário
bit 1  (2)     módulo carregado à FORÇA (--force) ← anômalo
bit 3  (8)     módulo descarregado à força        ← anômalo
bit 12 (4096)  módulo out-of-tree (não veio do kernel da distro)
bit 13 (8192)  módulo NÃO ASSINADO               ← o mais relevante
bit 15 (32768) kernel live-patched
```

> **Falso positivo comum:** NVIDIA, VirtualBox, ZFS e qualquer DKMS sujam o kernel legitimamente (bits 0/12/13). O que interessa é `tainted != 0` **sem que você tenha instalado driver nenhum** — e o `dmesg` diz o **nome** do módulo que sujou.

Módulo escondido do `lsmod` mas presente no `/sys`:

```bash
for m in /sys/module/*/; do
  n=$(basename "$m")
  [ -f "$m/initstate" ] && ! lsmod | grep -q "^$n " && echo "OCULTO? $n"
done
```

Módulo carregado sem arquivo correspondente em disco:

```bash
for m in $(lsmod | awk 'NR>1{print $1}'); do
  modinfo -F filename "$m" 2>/dev/null | grep -q '^/' || echo "SEM ARQUIVO EM DISCO: $m"
done
```

> `initstate` só existe em módulo **carregável** — built-in do kernel aparece em `/sys/module` sem ele, e por isso não gera ruído no teste acima. Um módulo ativo sem arquivo em disco significa que ele foi carregado e o arquivo apagado: mesma lógica da §3.14, um nível abaixo.

---

## 35.3. Hooks de função (syscall table, ftrace, kprobes)

Patch direto na syscall table é técnica antiga — hoje ela é read-only (`CONFIG_STRICT_KERNEL_RWX`). O rootkit moderno usa a infraestrutura de tracing do próprio kernel:

```bash
sudo cat /sys/kernel/debug/tracing/enabled_functions   # (ou /sys/kernel/tracing/...)
sudo cat /sys/kernel/debug/kprobes/list
sudo cat /sys/kernel/debug/kprobes/enabled
```

> **Como ler:** `enabled_functions` normalmente está **vazio** num host que não roda ferramenta de tracing. Cada linha lista a função hookada e o **callback** — se o callback aponta para um módulo que você não reconhece (ou para função sem nome), é hook ativo. Alvos favoritos: `getdents64` (esconder arquivo), `tcp4_seq_show` (esconder conexão), `kill` (comando mágico via sinal), `sys_read`.

Comparar o conjunto de símbolos com o do kernel instalado:

```bash
diff <(sudo awk '{print $3}' /proc/kallsyms | sort -u) \
     <(awk '{print $3}' /boot/System.map-$(uname -r) | sort -u) | head -40
```

> Compare **nomes**, não endereços: o KASLR desloca tudo por um offset a cada boot, então endereço diferente é normal. Símbolo que existe em `/proc/kallsyms` e não no `System.map` = código que entrou depois. E se todo endereço aparecer `0000000000000000`, é `kptr_restrict` (proteção normal), não rootkit.

---

## 35.4. eBPF: o rootkit sem módulo

O ponto cego atual. eBPF roda **dentro** do kernel, intercepta syscalls e tráfego, **não carrega módulo e não suja o `tainted`** — todos os testes da 35.2 passam limpos.

```bash
sudo bpftool prog show          # programas carregados: tipo, nome, PID que carregou
sudo bpftool link show          # ANEXOS ativos (kprobe/tracepoint/lsm/xdp/tc)
sudo bpftool map show           # mapas — onde o rootkit guarda o que esconder
sudo ls -la /sys/fs/bpf/        # objetos "pinados" = persistência (some da §7!)
sudo bpftool prog dump xlated id <ID>    # o que o programa realmente faz
```

> **Sinais:** programa `kprobe`/`tracepoint`/`lsm` que você não carregou; programa cujo `pid` carregador não existe mais (foi carregado e o processo saiu — o hook fica); objeto pinado em `/sys/fs/bpf` sem dono conhecido; programa `xdp`/`tc` numa interface, que pode filtrar pacote **antes** do `tcpdump` ver.

> **Consequência importante:** se há eBPF hostil em `xdp`/`tc`, até a §2.6 mente — o pacote é escondido antes da captura. Aí só a captura **fora da VM** (VPC Flow, espelhamento) vale.

---

## 35.5. Método da divergência

O princípio geral: pergunte a mesma coisa por **dois caminhos diferentes**. O rootkit mantém a mentira coerente numa via e esquece a outra.

```bash
# processos: /proc x ps
ls -d /proc/[0-9]* 2>/dev/null | wc -l ; ps -e --no-headers | wc -l

# diretórios escondidos: link count do diretório = 2 + nº de subdiretórios
stat -c '%h %n' /etc ; ls -d /etc/*/ 2>/dev/null | wc -l

# espaço "sumido": df (superbloco) x du (percorre entradas)
df -h / ; du -shx / 2>/dev/null

# rede: conexão que o ss não mostra mas o pacote sai
sudo ss -tnp state established ; sudo tcpdump -ni any -c 200 'not port 22'

# ferramentas que já fazem isso
sudo unhide quick ; sudo unhide-tcp ; sudo unhide proc sys brute
```

> Cada divergência tem explicação inocente possível (processo nasceu entre os dois comandos, `du` sem permissão, tráfego de terceiro). O que vale é a **divergência que persiste e é grande** — 30 PIDs de diferença não é corrida de tempo.

> Vale também a via "binário que não passa pela libc": `busybox` e binário estático furam LD_PRELOAD (§7.8), mas **não** furam rootkit de kernel — se `busybox ls` e `ls` concordam e ainda assim há divergência de `du`/`df`, o problema está abaixo da libc.

---

## 35.6. A resposta confiável está fora da caixa

É aqui que **VM ajuda**: o hypervisor enxerga a memória sem pedir licença ao kernel convidado.

```bash
# 1) snapshot / dump de memória SEM confiar no host (§1, §32)
gcloud compute disks snapshot <DISK> --snapshot-names="ir-kernel"     # disco
sudo ./avml /evidence/mem.lime                                        # memória

# 2) analisar OFFLINE — plugins de rootkit do Volatility 3
vol -f mem.lime linux.check_syscall.Check_syscall     # entrada da syscall table desviada
vol -f mem.lime linux.check_modules.Check_modules     # módulo na lista interna x /proc/modules
vol -f mem.lime linux.hidden_modules.Hidden_modules   # varre a memória atrás de módulo oculto
vol -f mem.lime linux.check_afinfo.Check_afinfo       # operações de rede hookadas
vol -f mem.lime linux.check_creds.Check_creds         # processos compartilhando cred = root dado
vol -f mem.lime linux.malfind.Malfind
```

> **Por que funciona:** o Volatility lê a estrutura de dados do kernel diretamente do dump; ele não chama syscall, então não há como o rootkit mentir para ele. É a única categoria de verificação que não depende da boa-fé do suspeito. (Ver o *gotcha* do ISF na §32 — sem o símbolo do kernel exato, nada roda.)

Outras vantagens de fora:

```text
VPC Flow Logs / espelhamento   §10.4   o tráfego que o host esconde, a rede mostra
snapshot de disco montado RO   §32     fls/icat leem o filesystem sem o kernel comprometido
métricas do hypervisor                 CPU/rede que não batem com o que o host relata
```

---

## 35.7. Impedir (mais barato que detectar)

```bash
mokutil --sb-state                       # Secure Boot ligado?
cat /sys/kernel/security/lockdown        # [none] integrity confidentiality
sysctl kernel.modules_disabled kernel.unprivileged_bpf_disabled \
       kernel.kptr_restrict kernel.dmesg_restrict
```

```bash
# depois do boot, quando todos os módulos legítimos já carregaram (via sysctl/rc):
sudo sysctl -w kernel.modules_disabled=1        # NINGUÉM mais carrega módulo até o reboot
sudo sysctl -w kernel.unprivileged_bpf_disabled=1
sudo sysctl -w kernel.kptr_restrict=2 kernel.dmesg_restrict=1
```

> `kernel.modules_disabled=1` é **irreversível sem reboot** — é o controle mais forte e mais simples desta seção. Combine com Secure Boot + assinatura obrigatória de módulo, e a via "carregar rootkit" fecha; sobra só exploração de kernel, que é ordem de grandeza mais cara. Casa com a §34.1 (`ProtectKernelModules=yes`, `SystemCallFilter=~@module`).

---

## 35.8. Veredito

```text
tainted por módulo não assinado que você não instalou   → forte
módulo ativo sem arquivo em disco                       → forte
hook em getdents64/tcp4_seq_show apontando p/ módulo    → forte
eBPF kprobe/lsm sem dono, pinado em /sys/fs/bpf         → forte
divergência grande e estável entre duas visões          → forte
Volatility acusando syscall/módulo oculto no dump       → conclusivo
```

> **Confirmado kernel = rebuild (§27), sem discussão.** Não existe "remover rootkit de kernel": você não consegue verificar a remoção com ferramentas que rodam sobre o kernel removido. Faça snapshot (§1) para análise, levante substituto a partir de imagem confiável, rotacione tudo (§26) e trate os outros hosts como suspeitos (§23) — quem chegou a root aqui provavelmente chegou em mais lugares.

> **E o inverso:** todos os testes limparem **não** absolve o host. Significa apenas que, perguntando de dentro, nada apareceu. Se a suspeita nasceu de um sinal externo (tráfego que a rede vê e o host nega), o sinal externo vence.

---

# 36. Escalonamento de privilégio (privesc)

> **Por quê:** a §16 responde "o atacante consegue executar código como o usuário da app?". Esta responde a pergunta seguinte, que decide o tamanho do estrago: **"e desse usuário, ele vira root?"**. É a ponte entre o vetor (§14/§16) e o blast radius (§34). **Olhe para:** rotas, não exploits — você corrige a rota sem nunca explorá-la.

**Privesc não é infinito: são seis famílias de rota.** Auditar é percorrer as seis.

```text
1. binário com poder     SUID/SGID, capabilities            §36.2
2. delegação mal feita   sudo, polkit                       §36.3
3. permissão frouxa      arquivo/dir gravável que root usa  §36.4
4. grupo privilegiado    docker, lxd, disk, adm             §36.5
5. software desatualizado LPE de kernel, sudo, polkit, glibc §36.6
6. credencial reaproveitada  senha/chave em arquivo legível §12.3
```

> **Você não precisa explorar para corrigir.** Identificar a rota já basta para fechá-la — e explorar em produção é arriscado e ruidoso. Se for validar, faça numa cópia/staging e com autorização.

---

## 36.1. Varredura rápida

Rode **como o usuário que você quer avaliar** (tipicamente o da app), não como root — o objetivo é ver o que *ele* alcança:

```bash
id; groups
sudo -l 2>/dev/null                         # o que este usuário pode rodar como root

# binários com poder
sudo find / -xdev -type f \( -perm -4000 -o -perm -2000 \) -printf '%m %u %g %p\n' 2>/dev/null
sudo getcap -r / 2>/dev/null

# gravável para mim, usado por root
echo "$PATH" | tr ':' '\n' | while read -r d; do [ -w "$d" ] && echo "PATH GRAVÁVEL: $d"; done
sudo find / -xdev -type d -perm -0002 ! -perm -1000 -ls 2>/dev/null    # dir world-writable SEM sticky
sudo find / -xdev -type f -perm -0002 ! -path '/proc/*' ! -path '/sys/*' -ls 2>/dev/null | head -40

# permissões de arquivos que definem privilégio
stat -c '%a %U:%G %n' /etc/passwd /etc/shadow /etc/sudoers /etc/ld.so.conf 2>/dev/null
ls -la /etc/ld.so.conf.d/ /etc/sudoers.d/ 2>/dev/null

# superfície de versão
uname -r; grep PRETTY /etc/os-release; sudo -V | head -1
```

---

## 36.2. SUID, SGID e capabilities

A enumeração está na §25 — aqui é **como interpretar**.

**Baseline esperado** num Linux comum (qualquer coisa fora disso merece pergunta):

```text
/usr/bin/{su,sudo,passwd,chsh,chfn,newgrp,gpasswd,mount,umount,pkexec,crontab,at}
/usr/lib/{openssh/ssh-keysign,dbus-1.0/dbus-daemon-launch-helper,polkit-1/polkit-agent-helper-1}
```

**A regra de decisão:** um binário SUID root é rota de privesc se ele conseguir fazer **uma** destas três coisas:

```text
executar comando arbitrário   → shell, editor, pager, interpretador, "!cmd", --exec
ler/escrever arquivo qualquer → lê /etc/shadow, escreve /etc/passwd ou uma unit
carregar código               → plugin, LD_*, módulo, biblioteca configurável
```

> Consulte **GTFOBins** para confirmar se aquele binário específico tem uma dessas primitivas — é a tabela de referência do assunto. Suspeitos clássicos aparecendo com SUID: `find`, `vim`, `nano`, `less`, `more`, `awk`, `python`, `perl`, `tar`, `cp`, `bash`, `env`, `nmap`, `docker`.

**Correção:**

```bash
sudo chmod u-s /caminho/do/binario          # remove SUID
sudo setcap -r /caminho/do/binario          # remove capabilities
```

Prevenção estrutural: `nosuid` nos mounts de escrita (§34.2) e `NoNewPrivileges=yes` + `CapabilityBoundingSet=` na unit (§34.1) — com isso o bit SUID deixa de valer para aquele serviço, mesmo que exista.

---

## 36.3. sudo

```bash
sudo -l                                     # como o próprio usuário
sudo -l -U <USER>                           # como root, avaliando outro usuário
sudo grep -RIn '' /etc/sudoers /etc/sudoers.d 2>/dev/null | grep -vE ':\s*#|^\s*$'
sudo visudo -c
```

O que é rota, na regra do sudoers:

```text
NOPASSWD para binário com shell/leitura     vim, less, awk, find, python, tar, git, systemctl
ALL=(ALL) ALL sem restrição de comando      é root com um passo a mais
curinga no argumento  (/bin/cat /var/log/*) traversal: ../../etc/shadow
comando por caminho RELATIVO                PATH hijack
env_keep com LD_PRELOAD / LD_LIBRARY_PATH   carrega lib do atacante como root
(ALL, !root) para "excluir" root            bypass conhecido (CVE-2019-14287)
sudo desatualizado                          Baron Samedit (CVE-2021-3156) e afins
```

**Correção:** comando sempre por **caminho absoluto**, sem curinga, `NOPASSWD` só para binário que não abre shell nem lê arquivo arbitrário, e no topo do sudoers:

```text
Defaults env_reset
Defaults secure_path="/usr/sbin:/usr/bin:/sbin:/bin"
Defaults use_pty
Defaults logfile=/var/log/sudo.log
```

> `use_pty` corta o sequestro do TTY por processo do mesmo usuário; `log_output`/`logfile` transforma sudo em evidência para a §11.

---

## 36.4. O que eu escrevo e root executa

**A rota mais comum na prática** — e a mais fácil de corrigir. Não é bug de software, é permissão.

```bash
# scripts citados em cron/systemd que meu usuário consegue escrever
sudo grep -RhoE '/[A-Za-z0-9_./-]+\.(sh|py|pl|rb)' \
  /etc/cron* /etc/systemd/system /usr/lib/systemd/system /etc/init.d 2>/dev/null \
  | sort -u | while read -r f; do [ -w "$f" ] && echo "GRAVÁVEL E RODA COMO ROOT: $f"; done

# a própria unit / o cron gravável
find /etc/systemd/system /etc/cron.d /etc/cron.daily -writable 2>/dev/null
# config do loader: lib carregada em TODO processo (§7.8)
find /etc/ld.so.conf.d /etc/ld.so.conf -writable 2>/dev/null
```

Duas variantes que passam despercebidas:

```text
PATH hijack       script de root chama 'tar'/'ps' sem caminho absoluto, e um diretório
                  gravável vem antes no PATH → você planta o binário com aquele nome
wildcard injection cron de root faz 'tar czf bkp.tgz *' num diretório que você escreve;
                  um arquivo com nome de flag (--checkpoint-action=...) vira argumento
```

**Correção:** dono `root:root` e modo `644`/`755` em tudo que root executa; caminho **absoluto** dentro dos scripts; `--` ou `./` explícito antes de curinga; diretório de trabalho de job de root nunca gravável por app.

---

## 36.5. Grupos que são root disfarçado

```bash
getent group docker lxd disk adm shadow sudo wheel video kvm 2>/dev/null
id <USER>
```

```text
docker / lxd   monta '/' do host num container → root imediato, sem exploit
disk           acesso cru ao dispositivo → lê e escreve o filesystem inteiro
shadow / adm   lê hashes de senha e logs
sudo / wheel   depende do sudoers (§36.3)
```

> Colocar alguém no grupo `docker` é **equivalente a dar root** — só que sem aparecer em nenhuma auditoria de sudoers. Se precisar, use socket rootless, `podman` sem daemon, ou uma regra sudo específica e auditada.

---

## 36.6. Kernel e componentes com LPE conhecida

```bash
uname -r; grep PRETTY /etc/os-release
sudo dnf updateinfo list security 2>/dev/null || \
  { sudo apt update -qq && apt list --upgradable 2>/dev/null | grep -ci security; }
sysctl kernel.unprivileged_userns_clone user.max_user_namespaces 2>/dev/null
```

Componentes onde LPE aparece com regularidade — vale checar versão contra o patch da distro:

```text
kernel   (overlayfs, nf_tables, io_uring, dirty pipe)
sudo     polkit/pkexec     glibc (ld.so)     systemd     dbus
```

> **Amplificador que quase ninguém desliga:** *user namespaces* sem privilégio. Boa parte das LPEs de kernel dos últimos anos precisa deles para alcançar a superfície vulnerável. Se nada no host usa container sem root:

```bash
sudo sysctl -w kernel.unprivileged_userns_clone=0     # Debian/Ubuntu
sudo sysctl -w user.max_user_namespaces=0             # genérico (cuidado: quebra podman/flatpak)
```

Outros `sysctl` que fecham primitivas usadas em privesc:

```bash
sudo sysctl -w kernel.yama.ptrace_scope=1     # processo não injeta em irmão do mesmo usuário
sudo sysctl -w fs.protected_symlinks=1 fs.protected_hardlinks=1   # ataques de link em /tmp
sudo sysctl -w fs.protected_fifos=2 fs.protected_regular=2
sudo sysctl -w fs.suid_dumpable=0             # sem core dump de processo SUID
sudo sysctl -w kernel.perf_event_paranoid=3
```

---

## 36.7. Ferramentas

```text
linpeas / LinEnum          # enumeração automatizada — cobre as 6 famílias de uma vez
unix-privesc-check         # checagem clássica de permissões
linux-exploit-suggester-2  # cruza 'uname -r' com LPEs públicas conhecidas
pspy                       # vê o que ROOT executa sem você ser root (acha o cron da §36.4)
lynis                      # hardening geral, com prioridade
GTFOBins / LOLBAS          # tabela: este binário SUID/sudo tem primitiva explorável?
```
```bash
./linpeas.sh -a > /tmp/pe.txt      # rode como o usuário SUSPEITO — é o ponto de vista que importa
./les.sh -k "$(uname -r)"
./pspy64                            # deixe rodando alguns minutos: pega o job periódico de root
```

> Duas ressalvas: (1) esses scripts são **barulhentos** e frequentemente marcados por EDR/AV — avise o time antes; (2) eles listam *candidatos*, com muito falso positivo. A validação é a regra de decisão da §36.2 e a leitura do sudoers da §36.3.

---

## 36.8. Mitigação por rota

```text
SUID/SGID desnecessário   chmod u-s  +  mount nosuid  +  NoNewPrivileges=yes      §34.1/§34.2
capability solta          setcap -r  +  CapabilityBoundingSet= (vazio)            §34.1
sudo permissivo           caminho absoluto, sem curinga, env_reset, use_pty       §36.3
arquivo gravável          root:root 644/755 em tudo que root executa              §36.4
PATH relativo             caminho absoluto nos scripts + secure_path no sudo      §36.4
grupo docker/lxd/disk     remover; rootless/podman ou regra sudo específica       §36.5
LPE conhecida             patch + userns desligado + sysctl de hardening          §36.6
credencial reusada        secret manager, sem senha em arquivo/env                §26/§34.4
tudo acima de uma vez     um serviço por host + sandbox systemd                   §34
```

> **O atalho honesto:** a §34.1 neutraliza sozinha as famílias 1, 2 e boa parte da 3 **para o serviço exposto** — `NoNewPrivileges` + `CapabilityBoundingSet=` + `ProtectSystem=strict` fazem o SUID e a capability deixarem de existir do ponto de vista daquele processo. Corrigir rota por rota é necessário; a sandbox é o que segura o que você não achou.

---

## 36.9. Higiene contínua

```bash
# baseline enquanto o host está limpo — guarde FORA dele (§34.7)
sudo find / -xdev -type f \( -perm -4000 -o -perm -2000 \) -printf '%m %u %g %p\n' 2>/dev/null | sort > suid.base
sudo getcap -r / 2>/dev/null | sort > caps.base
sudo -l -U <USER> > sudo.base 2>/dev/null

# depois, periodicamente:
diff suid.base <(sudo find / -xdev -type f \( -perm -4000 -o -perm -2000 \) -printf '%m %u %g %p\n' 2>/dev/null | sort)

# e alerte em tempo real (§11)
sudo auditctl -a always,exit -F arch=b64 -S chmod,fchmod,fchmodat -F a1\&04000 -k suid_set
sudo auditctl -w /etc/sudoers -p wa -k sudoers
sudo auditctl -w /etc/sudoers.d -p wa -k sudoers
```

> Privesc raramente aparece de uma vez: entra junto com um pacote novo, um deploy que mudou permissão, alguém adicionado ao grupo `docker` "só para testar". Sem baseline e diff, isso acumula em silêncio — e o próximo RCE encontra a rota pronta.
