# Satellite Hacking

Satélites são **computadores em órbita** — com todas as vulnerabilidades de software, acopladas a limitações físicas extremas (radiação, hostile environment, unreachable hardware) e hoje parte de infra crítica global (comunicações, GPS, observação, militar). Com o **boom LEO** (Starlink, OneWeb, Kuiper, Globalstar, Planet) e custos de lançamento caindo uma ordem de magnitude, a superfície explodiu. Segurança de satélites saiu do nicho estatal para pesquisa aberta.

## Arquitetura de Sistemas Espaciais

Três segmentos, cada um com superfície:

### Space segment

O satélite em si.

- **Bus** (chassis): estrutura, power (painéis solares, baterias), térmico, propulsão, AOCS (attitude control).
- **Payload**: carga útil — transponder de comunicação, câmera, radar SAR, GPS emissor, etc.
- **OBC (On-Board Computer)**: rad-hardened (clássicamente) ou COTS com redundância (novos).
- **TT&C (Telemetry, Tracking & Command)**: link S-band / X-band / Ka-band com a Terra.
- **ISL (Inter-Satellite Links)**: crescente — Starlink, Iridium NEXT. Laser ou RF.

### Ground segment

Estações terrestres, NOC (Network Operations Center), mission control, scheduling.

- **Teleport**: antena + RF chains.
- **Ground modem, baseband**.
- **Backend**: servers, orchestration, OSS/BSS, analytics.
- **Links terrestres**: fibra entre ground stations; often public internet involved.

### User segment

Terminais — modems, antenas VSAT, handsets móveis (Iridium, Starlink dish).

Cada segmento tem ataques próprios. Segurança ponta-a-ponta depende dos três.

## Órbitas

Influenciam ataque e defesa:

- **LEO (Low Earth Orbit, 200-2000 km)**: latência baixa (~5-50 ms), mas cobertura curta por sat (precisa constelação). Starlink, OneWeb, Planet, NOAA, ISS.
- **MEO (Medium, ~8k-20k km)**: GPS, Galileo, GLONASS, BeiDou.
- **GEO (Geostationary, 35,786 km)**: fixo em céu visto da Terra. Broadcast, VSAT, meteorologia, comunicações clássicas. 1 satélite cobre ~1/3 do globo.
- **HEO (Highly Elliptical)**: Molniya — cobertura polar.

## Ataques

### RF (wireless)

Primeiro vetor histórico:

#### Eavesdropping passivo

Downlinks (satélite → Terra) são **broadcast** — qualquer antena na área de beam recebe. Muitos sistemas legados (especialmente SATCOM VSAT) **sem criptografia** ou com crypto fraca.

- **Pesquisa 2020** (Pavur et al., Ruhr-Bochum): ~18 GB de tráfego sensível interceptados com kit COTS de €300. Incluía emails corporativos, autenticação HTTP básica, navegação militar, VoIP, dados industriais marítimos.
- Maioria usava IP sobre SATCOM em claro.

Mitigação: criptografia em user-plane (TLS, IPsec, VPN); depender sempre de crypto aplicação.

#### Jamming

Emissor poderoso transmite em mesma frequência → negação de serviço local.

- **Uplink jamming**: bloqueia comandos ao satélite da estação terrestre.
- **Downlink jamming**: nega recepção por usuários.
- **Cheap**: SDR + amplificador. Geopolítico: Russia em Ucrânia, Oriente Médio.
- **GPS jamming** onipresente em zonas de conflito; voos comerciais recebem alertas regularmente (Báltico, Mar Negro, Oriente Médio).

Mitigação: spread spectrum (DSSS, FHSS), narrow/adaptive beams (phased arrays), multi-constellation user equipment.

#### Spoofing

Transmitir sinal **imitando** autêntico — mais sofisticado que jamming.

- **GPS spoofing**: desvia receptor de posição/tempo. Iran supostamente usou para capturar drone RQ-170 (2011). Pesquisa acadêmica mostra viabilidade (Humphreys).
- **Defesa**: assinatura criptográfica em sinais (Galileo OSNMA em 2024+; GPS L1C/M-code), sensor fusion (IMU, odom, visão), multi-constellation cross-check.

#### Relay / meaconing

Captura sinal real e re-transmite com atraso — posição deslocada. Anti-drone technique; também malicioso.

### Comando / TT&C

Link que controla satélite.

- **Sem authentication**: em satélites legados (especialmente amadores, cubesats cedo), atacante na banda emite comando válido.
- **Command uplink capture**: pesquisa pública demonstrou controle de satélites amadores.
- **Nation-state vs orbital asset**: múltiplos incidentes reportados mas muito classificado.

Mitigação: authentication em comando (HMAC, assinatura), cadeia de key management com HSM, command authorization multi-pessoa, command lockdown em modos.

### Ground segment — cyber

Frequentemente o **vetor mais prático e barato**:

- Ground stations rodam Windows/Linux ordinários → vulneráveis a phishing, malware, lateral movement.
- Teleports conectados a Internet para ops.
- **Hack-and-control**: controlar o satélite via network interna em vez de tentar RF.
- **OT/IT convergência**: mesmo problema que ICS/SCADA.

Exemplos:
- **NASA Goddard (2011)**: afirmado acesso a satélites Landsat-7 e Terra por atacantes não-identificados. Nunca totalmente confirmado; relatório inspetor-geral.
- **ViaSat KA-SAT (fev 2022)**: wiper AcidRain em modems viasat atacando lado terra — ~30k modems inutilizados, Europa inteira afetada (incluindo turbinas eólicas alemãs usando conectividade satélite). Atribuído a Sandworm (GRU Unit 74455). Primeira ação espacial de cyberwar documentada em nation-state conflict.

### Supply chain

Hardware: chips, pré-lançamento firmware, contratos terceirizados.

- **Long lead times** e poucos vendors: concentração.
- **Comprometimento em bench** antes de lançar = sem recuperação pós-orbit.
- **DoD, NASA, ESA** têm programas rigorosos; newspace varia.

### Ataques físicos e pseudo-físicos

- **ASAT (Anti-satellite weapons)**: cinéticos (missiles ground-based — EUA 1985, China 2007, Índia 2019, Rússia 2021) ou co-orbital. Geopolítica ativa.
- **High-power laser dazzling**: cega sensores (não destrói necessariamente).
- **Microwave directed energy**: disrupção eletrônica.
- **Rendezvous e inspection**: satélites manobradores chegando perto.
- **Cyber via ransomware em operadora**: Viasat indireto; AWS-hosted ops também alvo possível.

## Pesquisa em Segurança Espacial

### Trabalhos acadêmicos

- **Pavur, Moser et al. (Oxford)**: "A Tale of Sea and Sky" - interceptação VSAT.
- **James Pavur**: tese completa em SATCOM.
- **SpaceSec** (Stony Brook, Ruhr), **CHESS (UCSD)**.
- **Mathilde Ollivier (ANSSI, Eviden)**: research em SATCOM.

### Hacking de ground equipment

- Dishy (Starlink UT): Lennert Wouters (KU Leuven) apresentou bypass em Black Hat 2022 via glitching voltage em modchip. Acesso root; SpaceX patched.

### Conferências / Events

- **DEF CON Aerospace Village**.
- **Hack-A-Sat**: CTF oficial da USAF (Space Force) 2020-2023; versões: simulado → satélite real em órbita (2023). Reviveu o field para pesquisa aberta.
- **CySat (conferência europeia)**.
- **IEEE Aerospace, AIAA, ESA cyber days**.

### CubeSats educacionais

Acesso barato para experimentação (partial). Kits de pesquisa crescem.

## Constelações LEO

### Starlink

- **~6k satélites** em órbita (2025), alvo 12k-40k.
- **Laser inter-satellite links** para roteamento off-ground.
- **Phased array** user terminals (Dishy).
- **Vermelho** em Ucrânia (usado desde 2022; Musk controlou geograficamente em pelo menos uma ocasião, politicamente controverso).
- **Security posture**: segredo; assinatura de comandos confirmada; bug bounty existente.
- **Dishy hacks**: Wouters (2022).
- **Risk de dependência**: monopolístico em LEO capaz; alternativas em anos.

### OneWeb, Kuiper, outros

OneWeb menor. Amazon Kuiper em deployment. Chinese Guowang, Qianfan (SpaceSail), planejadas megas. Multipolar orbit.

### ISLs laser

Reduz dependência de ground stations (especialmente sobre oceano, polar). Também **nega visibilidade** de comunicação a observadores em terra — mais difícil SIGINT.

## GPS e PNT

### Vulnerabilidade central

GPS civil L1 C/A é **não-autenticado**. Spoofing é demonstrado barato. Impactos enormes:

- **Navegação** aérea, marítima, terrestre.
- **Timing** — bolsas de valores, telecom, power grid dependem de sinal GPS para sincronização precisa.
- **Aplicações financeiras** (MiFID II exige timestamping preciso).
- **Agriculture autônoma, autonomous vehicles, logística**.

### Mitigação

- **Galileo OSNMA** (Open Service Navigation Message Authentication): assinado. Roll-out 2024+.
- **GPS M-code** (militar), **L1C**.
- **Multi-constellation** (GPS + Galileo + GLONASS + BeiDou).
- **Sensor fusion**.
- **Alternate PNT**: eLORAN, INS puro, star tracker, celestial nav, visual odometry.
- **Chip-scale atomic clocks (CSAC)**: autonomia por dias a semanas sem sinal.

Governos investem em "GPS backup" (UK, EUA, EU) dada vulnerabilidade estratégica.

## Legais e Regulatórios

- **FCC (EUA)**: autoriza operações comerciais.
- **ITU**: coordenação internacional de espectro e órbita.
- **NOAA**: autoriza sensores de imageamento.
- **Outer Space Treaty (1967)**: estado responsável por atividades de seus cidadãos em espaço.
- **Computer Fraud and Abuse Act** e equivalentes aplicam a operações em satélite a partir do território.
- **Export controls** (ITAR, EAR) — muito equipamento espacial restrito.

Operar sem licença em banda licenciada → crime; operar em satélite alheio → crime extra.

## Tendências

1. **Proliferação**: mais satélites, mais operadoras, baselines variáveis.
2. **COTS on orbit**: componentes comerciais, menos rad-hardened; mais software flexibilidade, mais updates possíveis.
3. **Software-defined satellites**: payload reconfigurable em órbita — mais poderoso, mais atacável.
4. **Quantum key distribution** (Chinese Mozi/Micius satélite): QKD ground-sat, limitado mas real.
5. **Mega-constelações competindo**: mais surface globally, mais infra crítica rolando em espaço.
6. **Military space ops**: Space Force (EUA), Forces Spatiales (FR), space commands de vários. Cyber + ASAT integrados.
7. **Standards emergentes**: CCSDS (Consultative Committee for Space Data Systems) com SDLS (Space Data Link Security); NIST para satélites; ISO/IEC.
8. **Supply chain scrutiny** após eventos.

## Boas Práticas Defensivas

### Projeto

- **Encryption** em telemetria, comando, user plane.
- **Authenticated commands** (HMAC ou digital signature).
- **Cryptographic keys** rotáveis em órbita.
- **Secure boot** em OBC.
- **Partition** entre funções críticas (command) e não (payload data).
- **Redundância** tanto funcional (múltiplos ECUs) quanto de conectividade.

### Operação

- **Ground segment endurecido**: MFA, network seg, EDR, patch management.
- **Runbooks** para incidentes.
- **NOC com 24/7 monitoring**.
- **Zero-trust** em arquitetura ground ↔ satélite.
- **Pentest regulares** em ground segment e apps; responsible disclosure para pesquisadores.
- **SIEM** com baselining de telemetria.

### Resiliência

- **Capacidade de autonomia temporária** (satélite opera em modo seguro sem comandos por dias).
- **Failover** entre ground stations.
- **Backup PNT** para clientes.

## Recursos

- **"Handbook of Space Security"** — Schrogl et al.
- **CCSDS (Consultative Committee) docs**: SDLS (Space Data Link Security), KMS (Key Management).
- **NIST SP 800-series** sobre TT&C security (em desenvolvimento).
- **Aerospace Corporation, RAND reports** sobre space security.
- **SWF (Secure World Foundation)** publicações.
- **DEF CON Aerospace Village**, **Hack-A-Sat**, **CySat** talks.
- **Moonlighter**: satélite-CTF 2023 (primeiro CTF em hardware real em órbita).
- **"Satellite Cyber Security: A New Frontier"** — Matt Buttner etc.
- **Pavur thesis "Whispers Among the Stars"**.

## Princípios

1. **Se não é cifrado, é lido**. RF é broadcast; assume-se intercepção.
2. **Ground segment é o vetor mais provável**. Comprometer operator é mais barato que comprometer satélite.
3. **Comandos autenticados, sempre**. Sem compromise.
4. **Sensor fusion em PNT** — nunca depender de GPS como única verdade em missão crítica.
5. **Resiliência > prevenção absoluta**. Espere comprometimento; projete para degradação graciosa.
6. **Patch = OTA** orbital; planeja desde dia 0.
7. **Cadeia de supply é profunda**. Componentes, fornecedores, ground tools, cloud backends.
8. **Acessibilidade à pesquisa cresce**. Bug bounty, responsible disclosure, comunidade aberta. Quem opera deve se preparar.
