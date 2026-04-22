# Ataques de Rede

Redes TCP/IP foram desenhadas em ambiente colaborativo acadêmico, décadas antes de se transformar em Internet global. Muitos protocolos carregam premissas de confiança que já não são razoáveis. Ataques de rede exploram essas lacunas — em L2 (link), L3 (network), L4 (transport), L7 (app). Este arquivo cataloga os clássicos e os principais mitigantes.

Complementa `DNS.md`, `TUNNELING.md`, `NETWORK_MONITORING.md`, `NETWORK_DIAGNOSTIC.md`.

## L2 — Camada de Enlace

### ARP Spoofing / ARP Poisoning

ARP (Address Resolution Protocol) resolve IP ↔ MAC na LAN. É sem autenticação: qualquer host pode anunciar "eu tenho IP X" e outros confiam.

- **Ataque**: atacante envia ARP replies afirmando ser o gateway. Tráfego da vítima passa por ele (MITM).
- **Ferramentas**: `arpspoof` (dsniff), `ettercap`, `bettercap`.
- **Detecção**: mudanças suspeitas em tabela ARP; IDS de host (ArpWatch).
- **Mitigação**: **Dynamic ARP Inspection (DAI)** em switches com DHCP snooping; **port security**; 802.1X para autenticação na porta.

### MAC Flooding / CAM Overflow

Switch mantém CAM table (MAC → porta) com capacidade finita. Flood de MACs aleatórios → tabela satura → switch vira hub (envia tudo por todas as portas). Atacante captura tráfego.

- **Ferramentas**: `macof`.
- **Mitigação**: **port security** limita MACs por porta.

### VLAN Hopping

Burlar segmentação VLAN.

- **Double tagging (802.1Q)**: atacante envia pacote com dois tags. Switch strip o primeiro (nativo), encaminha com segundo para outra VLAN.
- **Switch Spoofing (DTP)**: negociar trunk em porta de usuário; atacante acessa todas as VLANs.
- **Mitigação**: desabilitar DTP em portas de acesso, evitar VLAN 1 nativa, explicit trunk configuration, não compartilhar VLAN nativa em trunk.

### DHCP Starvation + Rogue DHCP

Esgotar pool de DHCP legítimo com MACs falsos; subir DHCP próprio; fornecer IP + gateway malicioso → MITM.

- **Ferramentas**: `yersinia`, `dhcpstarv`.
- **Mitigação**: **DHCP Snooping** (trust só em portas configuradas); IP Source Guard; port security.

### STP (Spanning Tree) Attacks

Falso BPDU reivindicando ser root → redesenho da árvore → tráfego desviado.

- **Mitigação**: **BPDU Guard** em portas de acesso; Root Guard em uplinks.

### 802.1X Bypass

Em NAC baseado em MAC authentication bypass, atacante clona MAC autorizado. Protegido por 802.1X com credenciais/cert.

## L3 — Rede

### IP Spoofing

Origem forjada. Em muitas situações, o responsável retorna pacote ao "endereço spoofado"; o atacante nunca recebe. Usado em:

- **DoS/DDoS reflection**: spoofa IP da vítima; dispara pedidos a servidores que respondem em massa à vítima.
- **Blind TCP session hijack**: exige adivinhar números de sequência — muito mais difícil hoje que nos 90s.

**Mitigação**: BCP 38 / **RPF (Reverse Path Forwarding)** em operadoras; bogon filtering.

### ICMP Abuse

- **ICMP Smurf** (obsoleto): spoofa IP da vítima em echo request a broadcast → todos respondem à vítima. Mitigação: default directed-broadcast off.
- **ICMP Tunneling**: exfiltrar dados em payload de echo. Ver `TUNNELING.md`.
- **ICMP Redirect**: enviar `redirect` ao router para redirigir rotas. Frequentemente desabilitado.

### Routing Attacks

Atacar plano de controle:

- **BGP hijacking**: anunciar prefixos alheios. Mudou de curiosidade acadêmica para rotineiro em 2000s. YouTube → Paquistão (2008), Amazon Route 53 (2018), várias exchanges de cripto.
- **BGP leaks**: AS anuncia mal inadvertidamente, desvia tráfego global.
- **Mitigação**: **RPKI** (Resource Public Key Infrastructure) — assinaturas criptográficas de propriedade de prefixos; **ROV** (Route Origin Validation) nos routers.
- **OSPF/IS-IS**: em LAN interna, anúncios falsos. Autenticar OSPF (HMAC).

## L4 — Transporte

### TCP SYN Flood

Abrir meio-conexões em massa para esgotar tabela de `SYN_RECV` no servidor.

- **Mitigação**: **SYN cookies** (Linux por default); aumentar backlog; deixar ao firewall/WAF na borda.

### TCP Session Hijacking / Reset

Injetar pacote com sequência válida encerra ou sequestra sessão.

- Em redes locais, sniff + inject é fácil.
- Com TLS, hijack torna-se injeção de RST (terminação) — ainda DoS, não roubo de sessão.
- **Mitigação**: TLS everywhere. **TCP-AO** (RFC 5925) para sessões críticas (BGP).

### UDP Amplification / Reflection DDoS

Spoofa IP da vítima; envia pequeno request a servidores UDP que respondem com tamanho muito maior → amplificação.

Vetores históricos:

- **DNS** (amp ~50x): query ANY a resolvers abertos.
- **NTP** (monlist, amp ~556x).
- **Memcached** (amp ~51000x). Usado no ataque ao GitHub em 2018.
- **SSDP, SNMP, RIPv1, Chargen, QOTD**.

**Mitigação**: BCP 38 em operadoras; fechar serviços UDP abertos à Internet; response rate limiting (RRL) em DNS.

## DNS

Ver `DNS.md`. Ataques adicionais:

### DNS Cache Poisoning

Injetar registros falsos no cache do resolver.

- **Kaminsky attack** (2008): em vez de "chutar" uma resposta para `www.bank.com` (o resolver cache por TTL), consulta randomkey123.bank.com + injetar glue apontando `www.bank.com` → NS controlado. Probabilidade alta em 2^16.
- **Mitigação**: **source port randomization** (mais entropia), **DNSSEC** (assinatura de zona), **0x20 encoding** (case-randomization aumenta entropia).

### DNS Rebinding

Servidor DNS controlado responde:
1. Primeira query: IP público.
2. TTL expira muito curto.
3. Segunda query: IP privado (127.0.0.1 ou rede interna).

Navegador da vítima, pela same-origin policy, continua enviando à "mesma origem" — agora apontando para rede interna. Pode acessar admin panels locais.

**Mitigação**: verificar `Host` header no servidor; **DNS pinning** (browser mantém IP); bloquear resoluções privadas em resolvers externos.

### DNS Tunneling

Exfiltrar dados em queries/respostas. Ver `TUNNELING.md`.

### DNS over HTTPS / over TLS (DoH/DoT)

Criptografam queries DNS. Para usuário, proteção de privacidade; para defensor, cegueira em um log historicamente rico. Política organizacional: forçar DNS interno, bloquear DoH.

### Subdomain Takeover

CNAME ou NS aponta para recurso cloud (github.io, azurewebsites, cloudfront) que foi deletado. Atacante cria recurso com mesmo nome → controle do subdomínio. Usado em phishing e evasão de auth (cookies com `Domain=.example.com`).

**Ferramentas**: `subjack`, `takeover`, `dnsReaper`.

**Mitigação**: inventário de registros DNS versus recursos; alerta em registros "dangling".

## TLS / HTTPS

### MITM sem controle de CA

Em rede corporativa com proxy SSL-intercept, o proxy emite cert on-the-fly assinado por CA instalada no endpoint. Para atacante externo, é mais difícil.

### Downgrade

Forçar TLS mais antigo (1.0, SSLv3) com ataques conhecidos (POODLE, BEAST, CRIME, FREAK, Logjam).

**Mitigação**: desabilitar < TLS 1.2; idealmente só 1.3. Ver `CRYPTO_ADVANCED.md`.

### Certificate Attacks

- **Comprometimento de CA** (DigiNotar 2011): emite certs fraudulentos.
- **Mis-issuance**: cliente negligente emite cert errado.
- **Mitigação**: **Certificate Transparency** (CT logs, obrigatório para CAs acreditadas), monitoramento (crt.sh); **HPKP** foi depreciado — use **CAA record** no DNS.

### TLS Stripping (sslstrip)

Interceptar HTTP inicial; reescrever links HTTPS → HTTP; mantém HTTPS só no backend. Vítima não vê HTTPS na barra.

**Mitigação**: **HSTS** (com preload); HTTPS-only por default em browsers modernos.

### Heartbleed / BEAST / POODLE / Bleichenbacher / Lucky13...

Vulnerabilidades específicas em implementações ou modos. Cada uma rende um CVE famoso. Lição: **não implemente TLS**; use libs atualizadas.

## L7 — Aplicação (rápido)

Ver `WEB_SECURITY.md` para SQLi, XSS, CSRF, SSRF.

Nesta camada de rede pura, os vetores relevantes incluem:

### HTTP Request Smuggling

Front-end e back-end interpretam `Content-Length` e `Transfer-Encoding` de forma diferente (CL.TE, TE.CL, TE.TE). Atacante injeta "requisição extra" que o front-end pula mas o back-end processa.

- **Descoberto em larga escala por James Kettle (PortSwigger, 2019)**.
- **Ferramentas**: `smuggler.py`, Burp scanner.
- **Mitigação**: HTTP/2 end-to-end; normalizar headers no proxy; atualizar implementações.

### Server-Side Template Injection, SSRF, Deserialization

Ver `WEB_SECURITY.md`.

### WebSocket Hijacking

Analogous CSRF para conexões WS. Verificar Origin no handshake; usar tokens.

## Wireless

- **WEP**: quebrado desde sempre.
- **WPA/WPA2-PSK**: handshake capturável; bruteforce offline (`hashcat -m 22000`). Senhas fortes + SSIDs não dicionário mitigam.
- **KRACK** (2017): reinstalação de chave; afeta WPA2. Patch.
- **WPA3**: SAE (Dragonfly) resiste a offline attack; Dragonblood em 2019 revelou fraquezas, corrigidas em updates.
- **Evil Twin**: AP com mesmo SSID que legítimo, mais perto/mais forte. Roubo de credenciais em captive portals.
- **Deauth attacks**: pacotes de desassociação forçam reconnect. Útil para captura de handshakes.
- **Mitigação corporativa**: WPA3-Enterprise com 802.1X + certificados cliente.

## VPN

- **IPsec**: IKEv1 tem ataques (aggressive mode PSK). Use IKEv2 com certificados.
- **WireGuard**: moderno, pequeno, rápido, usa Noise Protocol.
- **OpenVPN**: TLS-based; dependente de confs — defaults recentes são seguros.
- **Split tunneling**: conveniente, vetor de comprometimento (dispositivo em rede hostil + rede interna).

## Ferramentas de Análise / Ataque

| Função | Ferramenta |
|---|---|
| Sniffing | Wireshark, tcpdump, tshark |
| MITM | ettercap, bettercap, mitmproxy |
| Spoofing | arpspoof, yersinia |
| Crackers Wi-Fi | aircrack-ng, hashcat |
| Scanner | nmap, masscan |
| Frameworks | Impacket, Responder, NetExec |
| DDoS testing (autorizado!) | hping3, slowloris, goldeneye |
| Traffic generation | scapy (programático) |

## Defesas em Camadas

1. **Segmentação** de rede por função/tier/confiança.
2. **Firewall stateful** perimetral + host-based.
3. **IDS/IPS** em pontos-chave (Suricata, Snort, Zeek).
4. **TLS everywhere**, inclusive intra-rede (mTLS, service mesh — ver `ZERO_TRUST.md`).
5. **802.1X** em cabo e wireless.
6. **DHCP snooping, DAI, IP source guard** em switches.
7. **Port security, BPDU Guard, Storm Control**.
8. **NetFlow / Zeek logs** para visibilidade.
9. **DNS sinkholing** de domínios maliciosos conhecidos.
10. **Monitoramento ativo** com detecção (ver `DETECTION_ENGINEERING.md`).

## Princípios

1. **Zero trust em rede**: autenticidade dos endpoints, não só localização.
2. **Criptografe por default**: confidencialidade + integridade em trânsito.
3. **Minimize superfície**: portas fechadas; serviços não-usados desabilitados.
4. **Separe planos**: dado, controle, gestão — canais distintos.
5. **Visibilidade é pré-requisito**: sem logs/NetFlow, defesa é cega.
6. **Atualize firmware**: switches, routers, APs. Bugs em plano de controle levam a comprometimento infra.
