# Firewall, IDS/IPS e Filtragem de Rede

Como firewall funciona por dentro: hooks no kernel, stateful tracking, DPI, IDS/IPS, e as arquiteturas modernas que substituem o "perímetro" clássico. Foco em **implementação**, não em comparação de produto.

Rede em geral em `NETWORK_DIAGNOSTIC.md`, `NETWORK_MONITORING.md`. Ataques em `NETWORK_ATTACKS.md`. Arquitetura sem perímetro em `ZERO_TRUST.md`.

## 5 gerações (história condensada)

1. **1988 — Packet filter sem estado**: examina headers isolados. Cisco ACL, `ipfwadm` early Linux.
2. **1993 — Stateful inspection** (Check Point FireWall-1, patenteado): acompanha conexões; "return traffic" de conexão iniciada por dentro é permitido. Virou default.
3. **1999 — Application proxies**: firewall termina conexão e re-inicia. SOCKS, circuit proxies, application-aware (HTTP proxies).
4. **2005+ — UTM (Unified Threat Management)**: firewall + IDS/IPS + AV + filtering em um appliance. Fortinet, SonicWall.
5. **2010+ — NGFW (Next-Gen Firewall)**: app-aware (reconhece HTTP/WhatsApp/Netflix por fingerprint, não porta), user-aware (integra AD/LDAP), IPS inline, SSL inspection, sandbox. Palo Alto popularizou. **É o produto enterprise atual**.
6. **2020+ — SASE/SSE/ZTNA**: firewall é cloud service, não caixa; acesso é identity-based, não IP-based. Zscaler, Netskope, Cloudflare. Perímetro morto.

Firewall tradicional não sumiu — ainda dominante em data center east-west, OT/ICS, home. Mas narrativa enterprise migrou pra Zero Trust.

## Fundamentos: o que firewall filtra

### 5-tuple (conexão)

`(src_ip, src_port, dst_ip, dst_port, protocol)`. Chave canônica de flow.

### Layers relevantes (TCP/IP)

- **L2 (Ethernet)**: MAC, VLAN tag. Filtragem L2 rara em firewall clássico, comum em switches (port security, 802.1x).
- **L3 (IP)**: IP src/dst, protocol number, TTL, flags (DF, MF).
- **L4 (TCP/UDP/ICMP)**: ports, flags (SYN, ACK, FIN, RST), sequence, ICMP type/code.
- **L5-L7 (applicação)**: HTTP headers, TLS SNI, DNS query, SSH banner, etc. Exige DPI (Deep Packet Inspection).

### Stateless vs stateful

- **Stateless**: cada pacote avaliado isolado. Rápido, simples. Problema: ACL precisa permitir resposta manualmente (duas regras por direção). Vulnerável a spoofing, fragmented attacks.
- **Stateful**: mantém tabela de conexões ativas. Pacote é parte de conexão conhecida (ESTABLISHED) ou nova (NEW) ou inválido. Regra para inbound "deny NEW" + allow ESTABLISHED → tráfego retorno flui automaticamente.

## Netfilter (Linux kernel) — o padrão de referência

Framework no kernel Linux pra intercept/mutation/drop de pacotes. **iptables/nftables** são ferramentas user-space que falam com netfilter via netlink.

### Hooks

Pacote passa por pontos específicos no stack IP:

```
                    ┌──> local process
                    │
   IN ─► PREROUTING ─► routing ─► INPUT (local) ─► output local
                            │
                            └─► FORWARD ─► POSTROUTING ─► OUT
                                                  ▲
              local process ─► OUTPUT ────────────┘
```

Hooks: `PREROUTING`, `INPUT`, `FORWARD`, `OUTPUT`, `POSTROUTING`. Conntrack, NAT, firewall rules registram callbacks em hooks específicos.

### Tables (iptables)

- **filter**: default; `INPUT`, `OUTPUT`, `FORWARD`. Decide ACCEPT/DROP.
- **nat**: `PREROUTING` (DNAT), `POSTROUTING` (SNAT/MASQUERADE), `OUTPUT`.
- **mangle**: modifica pacote (TTL, TOS, marks).
- **raw**: antes de conntrack; usado pra NOTRACK.
- **security**: MAC (Mandatory Access Control), integra com SELinux.

### Rule anatomy (iptables)

```bash
iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/8 -m conntrack --ctstate NEW -j ACCEPT
```

- `-A INPUT`: append à chain INPUT
- `-p tcp --dport 22`: match TCP destination 22
- `-s 10.0.0.0/8`: source IP range
- `-m conntrack --ctstate NEW`: module match (conntrack, state NEW)
- `-j ACCEPT`: target

Targets: ACCEPT, DROP, REJECT, LOG, DNAT, SNAT, MASQUERADE, MARK, REDIRECT, custom chain.

### nftables — sucessor moderno

Replace de iptables (mesmo kernel backend via netfilter). Sintaxe unificada, atômica, mais expressiva:

```
table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;
        ct state established,related accept
        iif lo accept
        tcp dport { 22, 80, 443 } accept
        icmp type echo-request accept
        log prefix "drop: "
    }
}
```

Vantagens sobre iptables:
- **Atomic rule replace**: nftables aplica ruleset todo ou nada (iptables tem flush+apply com janela insegura).
- **Sets** nativos (lista de IPs, ports com O(1) lookup). Em iptables seria N regras.
- **Maps** (chave → valor), útil pra NAT dinâmico.
- **Inet family**: IPv4 + IPv6 no mesmo ruleset.
- **Variables, conditionals**.
- **Counter/quota** built-in.

Linux moderno usa **nft** internamente; `iptables-nft` é wrapper compat. Comece projetos novos em nftables.

### conntrack — o coração do stateful

`nf_conntrack` mantém tabela de conexões em kernel. Estados TCP:

- **NEW**: primeiro pacote (SYN)
- **ESTABLISHED**: handshake completo
- **RELATED**: conexão filha (FTP data channel, ICMP errors)
- **INVALID**: não corresponde a nenhuma conexão válida

Ver tabela: `/proc/net/nf_conntrack` ou `conntrack -L`. Limites (`/proc/sys/net/netfilter/nf_conntrack_max`) em servidor com muitas conexões (proxies, NAT) precisa tunar — overflow = pacotes dropados.

Timeouts por estado: `TIME_WAIT` (2MSL), `CLOSE_WAIT`, `FIN_WAIT`, UDP (2 min), ICMP (30s). Tunable em `/proc/sys/net/netfilter/nf_conntrack_*_timeout_*`.

### NAT — implementação

- **SNAT** (Source NAT): reescreve src IP/port no POSTROUTING. Usado quando LAN privada sai pra internet.
- **DNAT** (Destination NAT): reescreve dst IP/port em PREROUTING. Port forwarding, load balancer básico.
- **MASQUERADE**: SNAT automático usando IP da interface. Para conexões dinâmicas (dial-up, DHCP).
- **PAT** (Port Address Translation): reescreve porta junto. Permite N hosts por 1 IP público (CGNAT moderno).

Conntrack mantém mapeamento: `(internal_src, internal_port) ↔ (external_src, external_port)`. Pacote de retorno é reescrito de volta.

## eBPF / XDP — filtragem no fast path

**XDP** (eXpress Data Path): hook mais cedo possível no kernel, **antes de alocar `sk_buff`**. Programa eBPF pode `XDP_DROP`, `XDP_PASS`, `XDP_REDIRECT`, `XDP_TX`. Performance: 10-40 Mpps por core.

```c
SEC("xdp")
int drop_tcp_22(struct xdp_md *ctx) {
    void *data = (void *)(long)ctx->data;
    void *end = (void *)(long)ctx->data_end;
    struct ethhdr *eth = data;
    if (data + sizeof(*eth) > end) return XDP_PASS;
    if (eth->h_proto != bpf_htons(ETH_P_IP)) return XDP_PASS;
    // ... parse IP, TCP, check port
    return XDP_DROP;
}
```

**tc-bpf**: hook no traffic control, depois de XDP. Mais features (egress também).

Usado em:
- **Cilium**: CNI Kubernetes baseado em eBPF; policies L3-L7, service mesh, load balancing.
- **Cloudflare**: DDoS mitigation de primeira linha em XDP.
- **Katran** (Meta): L4 load balancer eBPF.
- **Falco, Tetragon, Tracee**: observability/security.

eBPF é a direção que Linux networking está indo — iptables/nftables ainda dominam políticas simples, mas eBPF domina quando perf ou lógica complexa importa.

## Windows — WFP (Windows Filtering Platform)

Substituiu TDI/NDIS filter em Vista+. APIs em user-mode (filter engine) + kernel-mode (callouts).

Layers (similar a hooks netfilter): ALE (Application Layer Enforcement) at auth_connect, auth_recv_accept, etc. Stream, datagram, MAC, transport layers.

Callouts registrados por drivers: Windows Firewall, IPSec, Defender Network Protection, EDR network filters.

APIs: `FwpmEngineOpen`, `FwpmFilterAdd`, `FwpmProviderAdd`. netsh advfirewall como CLI legada; `Set-NetFirewallRule` PowerShell.

## pf (OpenBSD/FreeBSD/macOS)

Firewall do OpenBSD (Henning Brauer et al.), portado pra FreeBSD, pfSense/OPNsense, macOS (legado, desde Lion). Sintaxe declarativa limpa:

```
set skip on lo
block all
pass in  on egress proto tcp from any to any port { 22 80 443 } flags S/SA keep state
pass out on egress from (egress:network) to any keep state
```

Features fortes: CARP (high-availability), altq (QoS), tables (sets), anchors (includes). Mais simples que iptables, mas menos programável que nftables/eBPF em 2026. pfSense/OPNsense são frontends populares.

## Application layer — L7

Firewall L3/L4 não distingue HTTP de SSH na mesma porta 80. Para policy por *aplicação*, exige parsing L7.

### Proxies

- **Transparent proxy**: cliente não sabe. Firewall redireciona (DNAT/REDIRECT) pra proxy local que intercepta.
- **Explicit proxy**: cliente configurado; `HTTP CONNECT` tunnels TLS.
- **Reverse proxy**: do lado do servidor (nginx, HAProxy, Envoy, Caddy).

### WAF (Web Application Firewall)

L7 para HTTP/HTTPS com regras de detecção de ataques de aplicação (SQLi, XSS, path traversal, etc).

- **ModSecurity** (open, WAF engine; roda em nginx/Apache/IIS/libmodsecurity).
- **OWASP Core Rule Set (CRS)**: regras canônicas.
- **Cloudflare WAF, AWS WAF, Imperva, F5, Akamai Kona**: comerciais.
- **Coraza**: reescrita ModSecurity em Go.

Detecção: regex de payload, anomaly scoring, virtual patching (bloqueia CVE antes de patch real).

### DPI (Deep Packet Inspection)

Parser profundo de L7. Reconhece protocolo mesmo em porta não-padrão, faz policy por app.

**Técnicas**:
- **Port-based**: pobre (tudo pode ser 443).
- **Payload signature**: string match em bytes iniciais ("GET ", "SSH-2.0", etc).
- **Statistical / behavioral**: tamanho de pacotes, timing, entropia. Identifica VPN/Tor mesmo obfuscated.
- **TLS fingerprinting**: JA3 (client hello hash), JA4 (versão moderna), JA3S (server hello). Distingue Firefox vs curl vs malware C2 sem decriptar.

### TLS inspection (SSL bump / MITM)

Enterprise comum. Firewall:

1. Intercepta TLS handshake saindo.
2. Gera cert dinâmico pro FQDN alvo, assinado por **CA própria instalada em todos os endpoints**.
3. Apresenta cert falso ao cliente; firewall se conecta ao servidor real e faz a outra ponta do TLS.
4. Decripta, inspeciona, re-criptografa.

Consequências:
- **Certificate pinning** quebra. Alguns apps (banking, Signal) recusam conexão. Firewall tem lista de "no inspect".
- **HSTS + HPKP** (legado) bloqueia.
- **Certificate Transparency logs** podem denunciar — enterprise CAs geralmente não se logam publicamente.
- **Perfomance**: +10-30% overhead, crypto escala mal.
- **Privacidade / compliance**: logs expõem tudo que user vê, incluindo banca/saúde. Policy + segregation obrigatórios.
- **TLS 1.3 + ECH (Encrypted Client Hello)**: dificulta SNI-based inspection sem MITM.

## IDS/IPS

**IDS** (Intrusion Detection System): observa, alerta, não bloqueia. Tipicamente SPAN/mirror port ou TAP — out-of-band.

**IPS** (Intrusion Prevention System): inline, bloqueia. Mesma engine de detecção, caminho diferente.

**NIDS** (network-based) vs **HIDS** (host-based; HIDS moderno é EDR — ver `AV_EDR.md`).

### Engines principais

- **Snort** (Cisco/community): signature-based, padrão histórico. Rules em DSL própria. Snort 3 é rewrite multi-thread.
- **Suricata** (OISF): Snort-compatible rules + próprias, multi-thread desde dia 1, melhor em performance. **É o que se usa em IDS moderno**.
- **Zeek** (ex-Bro): não é signature-based; **comportamental**. Scripts Zeek (domain-specific) descrevem eventos e correlações. Extrai metadados ricos (HTTP logs, DNS logs, SSL logs, conn logs) — fonte padrão em security analytics.

### Detection methods

- **Signature/rule-based**: match em payload ("pattern X em HTTP body"). Snort/Suricata rules. Rápido, mas não pega zero-day.
- **Anomaly-based**: baseline estatística (volume, ratio SYN/ACK, DNS entropy). Pega desvio, alto FP.
- **Heuristic**: combinação de features conhecidas.
- **Protocol anomaly**: parser estrito; tráfego que desvia de RFC é suspeito (malformed HTTP para evasion).
- **Flow-based** (NetFlow, sFlow, IPFIX): agrega sem ver payload. Bom pra volume, fraco pra content.

### Snort/Suricata rule exemplo

```
alert tcp $EXTERNAL_NET any -> $HOME_NET 80 (msg:"SQL injection attempt";
    content:"UNION SELECT"; nocase; http_uri;
    classtype:web-application-attack; sid:1000001; rev:1;)
```

Rulesets mantidos: **Emerging Threats (ET)**, **Proofpoint ETPRO**, **Snort Subscriber (Talos VRT)**. Atualizados diariamente.

### Posicionamento

- **Inline IPS**: no caminho do tráfego. Bloqueia. Risco de false positive derrubar serviço; fail-open vs fail-closed é decisão de arquitetura.
- **IDS em SPAN/TAP**: out-of-band. Detect-only. Zero risco operacional, sem prevention.
- **Distribuição**: perímetro + segmentos internos (data center east-west). Um IDS só na borda perde tudo lateral.

### Placement típico

```
Internet ──► edge FW ──► edge IPS ──► DMZ ──► internal FW ──► internal IPS ──► data center
                                       │                                        │
                                       └── WAF ──► web servers                  └── SIEM/SOAR
```

### Flow-based security analytics

- **NetFlow/IPFIX exports**: router/switch envia estatística de fluxos. Sampling em altíssimo volume.
- **Zeek logs**: `conn.log`, `dns.log`, `http.log`, `ssl.log`, etc.
- **SIEM** (Splunk, Elastic, Chronicle, Sentinel): ingere, correlaciona, investiga.
- **NDR** (Network Detection and Response): ML sobre flows, anomaly detection. Darktrace, Vectra, Awake. Opinião dividida — bom em alguns casos, marketing em outros.

## NGFW — o que "next-gen" significa

Palo Alto definiu em ~2008. Features que distinguem de firewall L4:

- **App-ID**: identificação de aplicação por DPI+heurística+decoders (não porta). Policy `permit facebook deny tiktok`.
- **User-ID**: integração com AD/LDAP, policy por usuário/grupo, não por IP.
- **Content-ID**: IPS + AV + URL filtering + file blocking integrados.
- **Decryption (TLS MITM)**: inspect tráfego criptografado.
- **WildFire / sandbox**: arquivos suspeitos enviados pra sandbox cloud, veredito propagado.
- **Threat intel feeds**: blocos de IP/domain/URL atualizados.
- **Zones**: construto de trust (untrust/dmz/trust) em vez de interfaces brutas.

Vendors: Palo Alto, Fortinet, Check Point, Cisco Firepower, Juniper SRX, Forcepoint.

## SASE / SSE / ZTNA — o firewall que virou cloud

**SASE** (Secure Access Service Edge, Gartner 2019): convergência de SD-WAN + network security as cloud service. Componente:

- **ZTNA** (Zero Trust Network Access): substituto de VPN. User autentica; policy decide; túnel criado pro recurso específico (não rede inteira).
- **SWG** (Secure Web Gateway): proxy web em cloud.
- **CASB** (Cloud Access Security Broker): visibility/policy sobre SaaS (Office 365, Salesforce).
- **FWaaS** (Firewall-as-a-Service): NGFW policy em cloud.

Modelo: user laptop tem agent → conecta a PoP cloud → policy aplicada → tráfego sai pra internet/SaaS/app interno.

Vendors: **Zscaler, Netskope, Palo Alto Prisma, Cloudflare One, Cato Networks**. Microsoft Entra Private Access.

Por que substitui VPN:
- VPN dá acesso à **rede** (muito amplo).
- ZTNA dá acesso ao **recurso** (granular, identity-based).
- VPN precisa concentrator; SASE é PoP global.
- Performance: hop direto user → PoP → SaaS, sem triangular via sede.

Ver `ZERO_TRUST.md` pra arquitetura de fundo.

## Rate limiting e DoS protection

### Técnicas

- **SYN cookies** (kernel Linux: `net.ipv4.tcp_syncookies`): sob SYN flood, kernel codifica estado em ISN e não aloca state até ACK retornar. Sobrevive floods enormes.
- **Conntrack limits**: por src IP.
- **iptables `hashlimit`, `connlimit`, `limit` modules**.
- **nftables `limit`, `flow`** expressions.
- **fail2ban**: user-space; tail de logs de auth/web, banimento via iptables por N min.
- **Commercial**: Cloudflare (anycast + scrubbing), AWS Shield, Akamai Prolexic, Arbor. Fundamentalmente: rede de PoPs enormes absorvendo e filtrando antes do origin.

### DDoS — mitigação moderna

- **L3/4 volumetric**: filtered em XDP por rede com banda massiva. Cloudflare, Akamai.
- **L7 (HTTP flood, slowloris)**: rate limiting, JS challenge, hCaptcha/Turnstile, bot scoring.
- **Amplification attacks** (DNS, NTP, memcached): bloqueio de src port spoofing (BCP 38 — ainda longe de universal).

## Testes e verificação

- **nmap**: scan portas/OS/serviços. `nmap -sS -p- -T4 target`.
- **hping3**: packet crafter, útil em teste de ACLs (`hping3 -S -p 80 target`).
- **scapy**: Python toolkit pra crafting pacote custom.
- **nuclei**: templates pra scan de CVEs em massa.
- **fraggler, fragroute**: evasão por fragmentação IP/TCP.
- **firewalk**: TTL-based firewall mapping.
- **Testar IDS**: replay de PCAP de malware (malware-traffic-analysis.net), Atomic Red Team network tests.

## Logging e observabilidade

- **netfilter LOG**: envia pra kernel log (dmesg). NFLOG envia pra user-space (ulogd).
- **conntrack events**: userland via `conntrack -E`.
- **nftables meta** + log.
- **Flow export**: nfnetlink, ipt_NETFLOW, softflowd.
- **pfsense/OPNsense**: syslog built-in.

Padrão enterprise: tudo pra SIEM (Splunk, Elastic, Sentinel, Chronicle). Correlaciona com endpoint telemetry (EDR) + identity (AD/Entra) + cloud (CloudTrail).

## Open-source production-grade

- **iptables / nftables**: kernel Linux.
- **pfSense, OPNsense**: FreeBSD-based; full firewall appliance OSS.
- **VyOS**: router + firewall; Vyatta fork.
- **OpenWrt**: roteadores consumer, base de muitas soluções home/SOHO.
- **Cilium**: CNI Kubernetes + network security via eBPF.
- **Calico**: similar, policy-focused.
- **Suricata, Snort, Zeek**: IDS.
- **ModSecurity + CRS, Coraza**: WAF.
- **Caddy, HAProxy, nginx, Envoy**: proxies com security features.
- **CrowdSec**: community-driven fail2ban moderno com threat intel compartilhado.

## Pitfalls / common mistakes

1. **"Default allow"**: policy começa permitindo tudo, adiciona denies. Primeira nova app = buraco. Default deny + explicit allow.
2. **Regras nunca removidas**: cemitério de regras de projeto morto. Audit regular.
3. **Single firewall = single point of failure**: HA pair (ativo-passivo ou ativo-ativo) pra produção.
4. **Inspect tudo, perf cai**: TLS MITM em 10Gbps exige hardware sério. Calibre com realismo.
5. **Confiar em zone internal cegamente**: lateral movement passa por firewall interno; segmentação importa (ver Zero Trust).
6. **Bloquear ICMP totalmente**: quebra path MTU discovery. Permita types específicos (3, 4, 11, 12) + echo se quiser ping.
7. **conntrack cheio**: hosts com muitas conexões (proxy, LB) OOM em tabela. Tune `nf_conntrack_max`.
8. **Asymmetric routing + stateful**: pacote volta por path diferente, conntrack não vê, dropa como INVALID. Comum em multi-homed.
9. **IDS sem tuning**: ruleset default tem 50k regras, 90% irrelevante pro ambiente — gera FP em massa. Curadoria obrigatória.
10. **Log flood**: LOG em DROP de internet scans enche disco em horas. Rate-limit log.

## Princípios

1. **Default deny**. Allow explícito. Toda vez.
2. **Defense in depth**. Edge FW + internal FW + host firewall + EDR. Nenhum para sozinho.
3. **Log e audit**. Regra sem log é invisível. Tráfego sem log é indefensável em IR.
4. **HA não é opcional** em prod. Failover testado, não assumido.
5. **Stateful por default**. Stateless só em casos específicos (DDoS frontal em XDP, p.ex.).
6. **Layer L3/4 + L7**. Um sem outro é buraco.
7. **Identity > IP**. User-aware policy vence IP-based em workforce móvel/remote.
8. **Zero Trust complementa, não substitui totalmente**. Firewalls internos + ZTNA convivem por anos.
9. **DPI/TLS inspection é trade-off**. Decripta tudo = privacy compliance e performance custam. Decripta seletivamente.
10. **IDS sem resposta é alarm fatigue**. Precisa de SOC, SOAR, playbooks. Senão vira dashboard decorativo.

## Recursos

- **"The Practice of Network Security Monitoring"** — Richard Bejtlich. Base de NSM.
- **"TCP/IP Illustrated v1"** — Stevens. Protocol fundamentals.
- **"Linux Firewalls"** — Steve Suehring (Packt, atualizado pra nftables).
- **"Network Security with OpenSSL"** — Viega et al.
- **"Learning eBPF"** — Liz Rice (2023). Moderno.
- **Netfilter.org docs**, **nftables wiki**.
- **Suricata docs**, **Zeek docs** — tutoriais de NSM.
- **SANS ISC** — `isc.sans.edu` — diário de ameaças de rede.
- **Cloudflare blog** — arquitetura de mitigation em escala.
- **Palo Alto Unit 42** — reports de threat + firewall context.
- **NIST SP 800-41** — Guidelines on Firewalls and Firewall Policy.
- **MITRE D3FEND** — contramedidas mapeadas (complemento ao ATT&CK).
