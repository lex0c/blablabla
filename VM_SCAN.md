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

</details>

---

# 1. Preparação

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

---

# 3. Processo

> **Por quê:** confirmar o que o processo suspeito **realmente** é — o `ps` mostra o nome que o processo diz ter, que pode ser forjado. **Olhe para:** exe em `/home`/`/tmp`/`/dev/shm`, nome camuflado de thread de kernel, pai `PID 1` sem motivo, child `sh`/`bash`, PTY + socket remoto.

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

---

## 5.3. Tipo

```bash
sudo file "$FILE"
```

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

Procure persistência **antes de matar o processo**.

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

---

# 7.9. Usuários e privilégios

Usuários UID 0:

```bash
awk -F: '$3 == 0 {print}' /etc/passwd
```

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

Faça isso **antes de matar o malware**, para evitar relaunch.

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

---

# 26. Rotação de credenciais

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

---

# 28. Coleta rápida em um único bloco

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

Os comandos deste runbook resolvem a triagem sem instalar nada. Quando quiser ir além (varredura ampla, IOC, memória), estas ferramentas ajudam.

> ⚠️ **Num host comprometido, binários/ferramentas locais podem estar adulterados** (rootkit esconde de scanner rodando no próprio host). Sempre que possível, **analise um snapshot/cópia offline** (§1), ou rode a partir de mídia confiável. Instale scanners de repositório confiável.

## Rootkit / objetos escondidos

```text
rkhunter          # checa rootkits, binários alterados, portas conhecidas
chkrootkit        # idem, heurísticas clássicas
unhide            # processos/portas escondidos (compara ps/ss x /proc x syscalls)
```
```bash
sudo rkhunter --check --sk ; sudo unhide quick ; sudo unhide-tcp
```

## Malware / assinatura

```text
clamav (clamscan) # antivírus; scan recursivo de paths suspeitos
maldet (LMD)      # Linux Malware Detect; bom em webshells/backdoors
yara              # regras de IOC customizadas (família de malware, C2, strings)
```
```bash
sudo clamscan -r -i /home /tmp /var/tmp /dev/shm /data
sudo yara -r regras.yar /home /tmp        # com rulesets (ex.: Yara-Rules, Neo23x0)
```

## Frameworks de IOC / coleta (triagem forense)

```text
Loki / THOR-Lite  # scanner de IOC (YARA + hash + nome + regras) — ótimo p/ 1º pass
Fenrir            # scanner IOC em bash puro (sem deps) — bom p/ host minimalista
UAC               # Unix-like Artifacts Collector — coleta padronizada p/ análise
CyLR              # coletor rápido de artefatos
```

## Auditoria / hardening / baseline

```text
lynis             # auditoria de segurança e hardening do host
openscap          # compliance/CVE
AIDE / Tripwire   # baseline de integridade (compare pré x pós)
Wazuh / OSSEC     # HIDS: FIM, deteção de rootkit, correlação de logs
debsums           # (Debian) verifica checksums dos pacotes — complementa dpkg -V
```

## Memória (binário packed, C2 em RAM)

```text
AVML              # aquisição de memória em Linux cloud (Microsoft) — 1 binário estático
LiME              # aquisição via módulo de kernel
Volatility 3      # análise do dump de memória (processos, conexões, injeção)
```
```bash
sudo ./avml /evidence/mem.lime      # capturar; analisar offline com Volatility
```

## Timeline / super-timeline

```text
plaso / log2timeline   # super-timeline de todo o filesystem + logs
mac_robber + mactime   # timeline clássica (Sleuth Kit)
```

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
nmap -Pn -p- --min-rate 1000 <IP>        # a porta interna responde de fora?
nuclei -u https://<host>                  # templates de exposição/CVE
```

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
