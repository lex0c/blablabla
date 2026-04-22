# Hacking Automotivo

Veículos modernos são **sistemas distribuídos sobre rodas**: 50-150 ECUs (Electronic Control Units), dezenas de milhões de linhas de código, múltiplos barramentos, conectividade celular, Wi-Fi, Bluetooth, GPS, V2X. A superfície de ataque é enorme, o software é legado em muitos casos, e as consequências de bug vão de inconveniência a vidas. Car hacking é disciplina com um pé em segurança embarcada (`IOT_FIRMWARE_SECURITY.md`), um em segurança de redes (`NETWORK_ATTACKS.md`), e um em hardware (`HARDWARE_ATTACKS.md`).

## Arquitetura do Veículo Moderno

Evolução em camadas:

### Zonal / Domain Architecture

Até ~2010, arquiteturas "flat" com muitos ECUs no mesmo barramento. Hoje:

- **Domain-based**: ECUs agrupadas por função (powertrain, chassis, infotainment, ADAS, body).
- **Zonal** (Tesla, VW MEB, novas plataformas): agrupadas por região física; cada zona tem computador central + ECUs locais.
- **Central compute** (Tesla HW3/4, Zeekr, emerging): poucos computadores poderosos substituindo dezenas de ECUs.

Tendência: menos ECUs, mais software, mais conectividade, mais atualização OTA.

## Barramentos In-Vehicle

### CAN (Controller Area Network)

Padrão desde ~1990. Diferencial, multi-master, broadcast. Velocidades:

- **CAN 2.0**: 1 Mbps, ~8 bytes payload.
- **CAN FD**: até 5-8 Mbps, até 64 bytes.
- **CAN XL**: até 20 Mbps (emergente).

**Características críticas para segurança**:

- **Sem autenticação** por padrão. Qualquer nó pode enviar qualquer mensagem.
- **Sem encryption**.
- **Arbitração por ID**: menor ID = prioridade.
- Mensagens são broadcast; todos nós vêem todas.

CAN = LAN insegura dentro do carro. ECU comprometido no infotainment pode enviar mensagens para freios se estiverem no mesmo bus (histórico).

### LIN (Local Interconnect Network)

Barramento lento e barato (até 20 kbps) para subsistemas não-críticos (controles de janela, espelhos).

### FlexRay

Determinístico, tolerante a falha. Usado em chassis/powertrain premium antigos. Substituído em grande parte por Automotive Ethernet.

### MOST (Media Oriented Systems Transport)

Audio/vídeo legacy. Substituído por Ethernet.

### Automotive Ethernet

100BASE-T1, 1000BASE-T1, 10GBASE-T1 em par trançado. Para ADAS cameras, câmeras de ré, infotainment, tráfego alto.

Crescente como backbone. Traz benefícios (banda) e riscos (stack IP). TSN (Time-Sensitive Networking) para real-time.

### SOME/IP, DoIP

- **SOME/IP** (Scalable service-Oriented MiddlewarE over IP): framework de serviços em Ethernet automotivo.
- **DoIP** (Diagnostics over IP): diagnóstico sobre IP.

## OBD-II

Porta padronizada (desde 1996 nos EUA, 2001 UE) sob o volante. Pin-out com CAN, K-Line, outros.

- **Dispositivos plug-in** (dongles de seguradora, rastreadores, scanners) têm acesso direto ao CAN.
- **Comprometer dongle → comprometer carro**. CVEs reais em vários produtos (Progressive, Zubie).

### Protocolos de diagnóstico

- **OBD-II** (emissões).
- **UDS (Unified Diagnostic Services, ISO 14229)**: lê/escreve memória, reprograma ECU, executa rotinas. Base de tudo que scan tools fazem.
- **KWP2000** (legado), **XCP** (calibração).

UDS tem **security access** (seed+key challenge) — mas keys frequentemente são derivadas algoritmicamente e extraíveis de firmware.

## Telemática e Conectividade

Carros modernos são **conectados**:

- **Celular** (LTE/5G) via TCU (Telematics Control Unit). Crucial para OTA, eCall, infotainment.
- **Bluetooth, Wi-Fi hotspot**.
- **GPS**.
- **V2X** (Vehicle-to-Everything): comunicação entre carros, infra, pedestres. DSRC (802.11p) e C-V2X (celular).
- **Keyless entry**: RFID/NFC + rolling codes.
- **Smart key/app**: app no celular controla veículo via cloud.

Cada canal de conectividade = superfície remota.

## Ataques Clássicos

### Miller & Valasek — Jeep Cherokee (2015)

**O caso canônico**. Explorou:

1. Sprint/Harman Uconnect (infotainment) com serviço D-Bus exposto na rede celular via port 6667.
2. RCE no infotainment.
3. Pivot para bus CAN via bridge mal segmentada.
4. Envio de mensagens CAN → controle de steering, brakes, transmissão.

Resultado: Wired publicou demo vivo na estrada. Recall de 1.4M veículos. Catalisou indústria a levar a sério. Charlie Miller e Chris Valasek tornaram-se figuras emblemáticas; hoje na indústria (Cruise, Didi).

### Tesla — Keen Labs, Pwn2Own

- **Tesla Model S** (Keen 2016): chain de web browser → kernel → CAN.
- **Model 3** (Pwn2Own 2019): Fluoroacetate team ganhou carro.
- **Model 3 (2023)**: Synacktiv ganhou Pwn2Own Vancouver.
- **Bluetooth attacks** em outros fabricantes (Kia, Hyundai).

Tesla responde relativamente rápido + OTA; vantagem estrutural sobre fabricantes tradicionais.

### Keyless Entry Relay

RSA de relay:

- Atacante 1 próximo à casa (onde está a chave).
- Atacante 2 próximo ao carro.
- Dispositivos relay amplificam sinal curto de alcance.
- Carro acredita que chave está por perto → destrava + ignition.

Comum em crimes de roubo. Mitigação: chaves com motion sensor (dormem se paradas), Ultra-Wideband (UWB) ranging preciso, apps com presença fora de casa.

### RollJam / Roll-Back

Rolling codes em keyless — atacante grava + injeta jamming no último pressionamento do fob → capturar código não usado → reexecutar depois.

### CAN Injection via Headlight

2023+ onda de roubo: acessar o CAN através da conexão do **farol** (facil remoção). Pacote injetado simula comando de chave. Toyota, Lexus, Honda entre alvos.

Mitigação: separar farol de CAN principal (gateway), adicionar autenticação de mensagem (SecOC), comunicação cifrada em domínios sensíveis.

### UDS Abuse

Scan tool + conhecimento de UDS → muitas ECUs respondem comandos sensíveis sem autenticação robusta. Reprogramação → fraude de odômetro, bypass de immobilizer, chipping de motor.

### Tire Pressure Monitoring (TPMS)

TPMS sensors transmitem em 315/433 MHz. Identificáveis, trackeáveis. Ataques: injeção de leituras falsas para causar alerta.

### Infotainment → CAN

Infotainment é frequentemente Linux/Android baseado. Superfície grande, updates lentos (historicamente). Comprometê-lo e pivotar foi o padrão Jeep.

Mitigação moderna: **gateway** com filtering entre infotainment e CANs críticos, message validation, firewall.

### EV Charging

- **CCS, CHAdeMO, Type 2**: protocolos de carga.
- **ISO 15118**: Plug and Charge, com certificados.
- Atacar charger, MITM entre veículo-charger, roubar créditos, negar serviço.
- Pesquisa crescente de Sandia, etc.

### OTA

Crítico mas vetor poderoso. Se assinatura fraca ou chain compromise → push de firmware malicioso para frota. Todos os grandes fabricantes sofreram scrutiny de pesquisadores; em geral a assinatura é robusta mas a distribuição de chaves, HSM, revogação são pontos de atenção.

## Metodologia de Pesquisa Automotiva

### Preparação

- **Veículo dedicado** (pesquisa não em carro diário).
- **Ferramentas safety**: macacão para EV, EPP, descarga de baterias, first-aid.
- **Bancada**: ECUs individualmente em bench rig simula carro em laboratório com custo menor.
- **Ambiente seguro**: área fechada para testes dinâmicos.

### Ferramentas

| Categoria | Ferramenta |
|---|---|
| Bus sniff/inject | CANtact, Macchina M2, CANable, Kvaser, Peak PCAN |
| Analysis | SavvyCAN, can-utils (Linux socketcan), Wireshark com CAN dissector |
| Full toolkit | Vector CANoe, dSPACE, Intrepid (comerciais caros) |
| OBD/UDS | ELM327 clones (basic), CaringCaribou, UDSim, ICSim (simulation) |
| RF | HackRF, RTL-SDR, YARD Stick One, Flipper Zero |
| Keyless | Bishop Fox's RollJam-like setups, BLE sniffers |
| Firmware | Ghidra, IDA (com ARM, Tricore plugins), binwalk |
| Hardware | Logic analyzer (Saleae), J-Link, ChipWhisperer para fault |
| Simulation | CARLA (ADAS), LGSVL, MATLAB/Simulink |

### Fluxo

1. **OSINT** em OEM: docs, patches, CVEs históricos, FCC filings de módulos.
2. **Hardware extraction**: ECU, flash dump (SPI/NAND), JTAG/SWD, chip-off se necessário.
3. **Firmware RE**: Ghidra/IDA; identificar funções UDS, handlers CAN, routines de crypto.
4. **Sniff** do bus em carro rodando: baseline de tráfego normal.
5. **Replay**: injeta mensagens gravadas.
6. **Fuzz**: variar IDs/payloads; medir reações.
7. **Exploração**: chain de primitives até objetivo.
8. **Disclose**: ISAC-Auto, OEM PSIRT, Auto-ISAC. Muitos OEMs têm bug bounty (Tesla, GM, Ford).

## Defesa Automotiva

### Design

- **Gateway segmentation**: infotainment e domínios críticos separados por gateway com filtering.
- **SecOC (Secure Onboard Communication)** — ISO 17356: MAC (CMAC AES-128) em mensagens CAN críticas. Padrão em AUTOSAR moderno.
- **Crypto no ECU**: HSM dedicado (SHE, EVITA profiles).
- **Secure boot** em ECUs.
- **OTA seguro** com assinatura e rollback protection.
- **Fail-safe defaults**: ECU que perde conectividade com DR é prior seguro.
- **Defense in depth**: assumir comprometimento de algum ECU; minimizar blast radius.

### Detecção em produção

- **IDS em CAN** (comportamental, anomaly baseado em timing/frequência).
- **SIEM veicular + backend** (VSOC — Vehicle SOC). OEMs grandes operam.
- **Logs** e telemetria cloud para detection pós-fato.

### Padrões e regulação

- **ISO/SAE 21434**: cybersecurity engineering em automotivo.
- **UNECE R155**: cybersecurity management system obrigatório para homologação em Europa, Japão, Coreia (2024+). Obriga OEM a ter processos.
- **UNECE R156**: software update management.
- **ISO 24089**: software update engineering.
- **Purdue Model for automotive** (emergente).

Fabricante sem CSMS não homologa em UE — elevou bar materialmente em 2024.

## ADAS e Autônomos

- **Sensores**: câmeras, radar, LIDAR, ultrasom, GPS, IMU. Ver `ROBOTICS.md`.
- **Ataques**:
  - **Adversarial patches** em placas de trânsito (Eykholt et al., Tencent Keen) fazendo Tesla autopilot errar.
  - **Laser spoofing** em LIDAR.
  - **GPS spoofing** desvia de rota.
  - **Acoustic** em MEMS gyros (Trippel et al.) perturba IMU.
  - **Projetor em estrada** enganou Mobileye.
- **Defense**: fusão sensorial, sanity checks, redundância física.

## Safety vs Security

Em automotivo, não são a mesma coisa:

- **Safety** (segurança física): ISO 26262, ASIL levels. Aborda falha aleatória e sistemática.
- **Security**: ISO/SAE 21434. Aborda ataque intencional.

Convergência: **CAL (Cybersecurity Assurance Levels)** estão se alinhando com ASIL. Security pode causar safety issue — e safety feature mal-implementada pode ser vetor.

## Casos que Marcaram

- **2010**: Koscher et al. "Experimental Security Analysis of a Modern Automobile" — primeira demonstração acadêmica ampla.
- **2015**: Jeep Hack.
- **2015-2020**: Tesla hacks, Keen Labs expõe várias vulns.
- **2019-2020**: BMW ConnectedDrive RCE (Vulnerability Lab).
- **2022**: Yuga Labs exposed chain em Honda/Nissan/Infiniti/BMW/Ford/... via Spireon.
- **2022**: CAN injection mass roubo (Toyota/Lexus via headlight).
- **2023**: Sam Curry & Shubham Shah "Web Hackers vs. The Auto Industry" — dezenas de OEMs com vulns críticas em portais/APIs.
- **2024**: Kia Connect portal bug — any car by plate lookup.

Padrão consistente: portais/APIs de companion apps são frequentemente mais fracos que o veículo em si.

## Recursos

- **"The Car Hacker's Handbook"** — Craig Smith. Referência canônica.
- **"Hands-On Connected Car Cybersecurity"** — Vidal, Campos.
- **DEF CON Car Hacking Village**.
- **Escar (Embedded Security in Cars)** conferência.
- **Auto-ISAC**: informação setorial.
- **Upstream Security Global Automotive Report**: compilação anual de incidentes.
- **OpenGarages** (comunidade).
- **Sam Curry blog**, **Yuga Labs research**.
- **OEM PSIRT pages, bug bounty scopes**.

## Princípios

1. **CAN é LAN insegura**. Assume qualquer ECU que acesso ao bus pode enviar qualquer coisa — a menos que SecOC esteja implementado.
2. **Superfície remota > local**. TCU celular, BT, Wi-Fi, apps, backend APIs recebem foco primário.
3. **Gateway segmentation é crítico**. Infotainment comprometido não pode tocar brakes.
4. **OTA é aliado + risco**. Bem feito, mitiga vulnerabilidades; mal feito, vetor massivo.
5. **Safety e security convergem**. Design moderno integra.
6. **Testing in bench > na estrada**. Reduzir risco físico durante pesquisa.
7. **Disclosure responsável**. OEMs reagem melhor com canal aberto; publicar sem coordenação é anti-ético em vida humana em jogo.
8. **Legal**: jurisdicional. Pesquisa sem autorização em veículo próprio ≠ em veículo alheio. DMCA safe harbor para pesquisa de segurança em EUA (desde 2022).
