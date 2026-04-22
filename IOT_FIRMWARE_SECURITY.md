# Segurança de IoT e Firmware

Dispositivos embarcados — roteadores, câmeras, smart devices, ICS, carros, implantes médicos — constituem a "Internet of Things". Diferenças estruturais em relação a sistemas tradicionais:

- **Recursos limitados**: pouca RAM, CPU lenta, às vezes sem MMU.
- **Atualização escassa**: muitos dispositivos nunca recebem patch.
- **Longa vida útil**: 10-20 anos em campo não é raro (infraestrutura).
- **Sistema fechado**: código fechado, builds únicos, diagnóstico difícil.
- **Superfície física acessível**: atacante pode ter posse do device.

Resultado: firmware é, sistematicamente, um dos softwares mais vulneráveis em uso.

## Arquitetura Típica

- **CPU**: ARM (Cortex-M para micro; Cortex-A para SoC com Linux), MIPS (roteadores antigos), RISC-V crescente, Xtensa (ESP32).
- **Armazenamento**: NAND/NOR flash, eMMC, SPI flash externo.
- **Boot ROM**: código imutável de fábrica, carrega bootloader.
- **Bootloader**: U-Boot (Linux embarcado), proprietários. Verifica e carrega OS.
- **OS**: Linux embarcado (OpenWrt, Buildroot, Yocto), RTOS (FreeRTOS, Zephyr, VxWorks), ou bare-metal.
- **RTOS features**: multitask, real-time scheduling, redes (LwIP).

## Obtendo Firmware

### Download oficial

Muitos fabricantes publicam updates. Baixar, descompactar. Arquivos frequentemente criptografados — mas a chave pode estar no binário do updater do próprio fabricante.

### Dump do hardware

Quando firmware não está disponível:

- **SPI flash** — chip externo, dumpar com programador (CH341A, Dediprog) ou via SOIC-8 clip (sem dessoldar).
- **eMMC** — dessoldar e ler; ferramentas como Easy-Jtag.
- **UART** — terminal serial do bootloader muitas vezes dá shell; `dd` dumpa.
- **JTAG/SWD** — acesso direto a CPU/memória se não desabilitado (fusíveis).
- **Chip-off** — último recurso; desmonta.

### UART

Header de pinos no PCB. Identificar TX/RX/GND com multímetro/osciloscópio. Conectar adaptador USB-serial (FTDI, CP2102) em taxa certa (comum: 115200, às vezes 57600). Bootloader imprime, frequentemente oferece prompt.

### JTAG / SWD

Interface de debug. Com sondas (J-Link, Black Magic Probe, Glasgow), se habilitado, permite dump de RAM/flash, breakpoints, leitura de registradores. **Fusíveis de produção** frequentemente desabilitam JTAG — mas nem sempre.

## Análise de Firmware

### Extração

- **binwalk**: procura magic bytes e extrai. `binwalk -e firmware.bin`.
- **unblob**: mais novo, modular, cobre mais formatos.
- **firmware-mod-kit**: antigo mas útil para alguns.

Após extração: filesystem (squashfs, ubifs, jffs2, cramfs) montável.

### Exploração

- **`file`**, **`strings`** — tipo, idioma, URLs.
- **firmwalker**: bash script que procura credenciais, chaves, URLs em filesystem extraído.
- **EMBA**: framework completo de análise automatizada.
- **FACT (Firmware Analysis and Comparison Tool)**.

### O que procurar

- **Credenciais hardcoded**: `/etc/passwd`, scripts init, telnetd config.
- **Chaves privadas**: TLS certs, SSH keys, JWT secrets.
- **Backdoors**: telnetd, senhas fixas, "magic IPs".
- **URLs**: update endpoint, C2, telemetry.
- **Binários de interesse**: `httpd`, `telnetd`, `ubusd`, `dropbear`.
- **Scripts CGI**: entrada clássica de CVEs em roteadores.

### RE do binário

Ver `REVERSE_ENGINEERING.md`. Ghidra + processor-specific plugin (MIPS, ARM) → decompilar CGI; procurar buffer overflow em parse de parâmetros, command injection em `system("foo " + user_input)`, auth bypass.

## Emulação

Rodar firmware sem o hardware — rápido iteração:

- **QEMU user-mode**: emula binários isolados (ex.: `qemu-mips-static ./httpd`).
- **QEMU system-mode**: emula sistema inteiro. Funciona se kernel + initramfs corretos.
- **FirmAE, Firmadyne**: frameworks que automatizam emulação de firmware Linux embarcado.
- **Renode, QEMU + AVATAR2**: emulação mista hardware/software para firmware com perifericos.
- **Unicorn + afl-unicorn**: fuzzing de funções isoladas.

## Ataques Típicos em Dispositivos

1. **Web interface**: CGI com command injection, XSS, CSRF. Roteadores têm histórico terrível.
2. **Telnetd / SSH com senha default**: botnets como Mirai varreram a Internet por esses.
3. **UPnP aberto**: abre portas no roteador para dispositivos internos.
4. **Firmware update sem assinatura**: atacante substitui firmware.
5. **Boot sem chain of trust**: arbitrary firmware aceito.
6. **Debug interfaces expostos**.
7. **Protocolos proprietários**: sem criptografia, sem autenticação.
8. **OTA com TLS quebrado**: cert mal validado, MITM da atualização.

## Secure Boot

Cadeia de confiança:

1. **Boot ROM** (imutável) verifica bootloader via assinatura.
2. **Bootloader** verifica kernel/OS.
3. **OS** verifica aplicações.

Cada estágio valida o próximo. Quebrar em qualquer ponto permite rodar firmware customizado. Dispositivos de consumo usam Secure Boot para garantir firmware oficial; pesquisadores e jailbreakers buscam a quebra.

## TEE e Enclaves

- **ARM TrustZone**: dois mundos — secure world para crypto/DRM; normal world para OS geral. Comunicação via SMC. OPTEE, QSEE (Qualcomm), Trusty (Android).
- **Intel SGX**: enclaves user-mode; comprometido por várias vulnerabilidades de arquitetura.
- **AMD SEV / SEV-SNP**: encryption de VM, atestação.

Para IoT, TrustZone é onde chave privada do device vive — resistente a dump por FS.

## OTA (Over-the-Air Updates)

Vetor crítico. Princípios:

- **Assinatura obrigatória** e verificada.
- **Rollback protection**: não aceitar versão mais velha (counter monotônico, anti-rollback em hardware).
- **Dual-bank / A-B partitions**: roll-back funcional se update falha.
- **Confidencialidade**: criptografia útil para proteger IP; integridade/assinatura é o essencial.
- **Canário/staged rollout**: % por vez.

## Protocolos IoT

- **MQTT, CoAP, AMQP**: mensageria; MQTT comumente sem auth ou com "topic ACL" frouxo.
- **Zigbee, Z-Wave, Thread, Bluetooth LE**: wireless locais. Cada um com seus CVEs históricos.
- **LoRaWAN, Sigfox, NB-IoT**: LPWAN.
- **Modbus, DNP3, IEC-104** (ICS): sem autenticação histórica; segmentação de rede é defesa.

## ICS / SCADA / OT

Industrial Control Systems têm particularidades:

- **Disponibilidade > confidencialidade** (reversão das prioridades tradicionais).
- **Máquinas que não podem parar** → sem patch, exposição longa.
- **Protocolos legados** (Modbus, Profinet, S7).
- **Air gap mítico**: Stuxnet provou que raramente é real.

Ataques-marco: **Stuxnet** (2010, centrífugas de urânio), **BlackEnergy/Industroyer** (Ucrânia, grid), **Triton/Trisis** (safety systems), **Colonial Pipeline** (2021, ransomware em IT mas impactando OT).

Padrões: **IEC 62443**, **NIST SP 800-82**, **Purdue Model** (segmentação em níveis 0-5).

## Car / Automotive

Veículos modernos têm 100+ ECUs, CAN bus, Ethernet automotive, cellular, Wi-Fi, Bluetooth.

- **CAN bus** sem autenticação — acesso físico controla tudo.
- **TPMS, Keyless Entry**: ataques de relay clássicos.
- **Infotainment → gateway → ECU**: Jeep Cherokee 2015 (Miller & Valasek).
- **UDS / ISO 14229**: diagnóstico. Comandos para reprogramar ECUs.

Padrões: **ISO/SAE 21434** (cybersecurity automotive), **UNECE R155**.

## Medical Devices

- **Marcapassos, bombas de insulina, MRI** — comprometimento tem consequência letal.
- **FDA / MDR EU** exigem análise de segurança.
- **Pacemaker com Bluetooth** sem encryption foi demonstrado remotamente exploitable (Jay Radcliffe, DEF CON 2011).

## Hardware Hacking

Intersecta com `HARDWARE_ATTACKS.md`. Técnicas básicas:

- **Logic analyzer** (Saleae, DSLogic): capturar SPI/I2C/UART.
- **Bus Pirate / Shikra / Glasgow** — tools multi-protocolo.
- **Osciloscópio** para sinais analógicos.
- **Microscópio** para leitura de chip.
- **Dessolda com ar quente** para chip-off.

## Organizações

- **OWASP IoT Top 10**.
- **IoT Security Foundation**.
- **Cyber Resilience Act (EU)**: regulamentação 2024+ exige segurança por design em produtos conectados.

## Ferramentas

| Categoria | Ferramenta |
|---|---|
| Extração | binwalk, unblob, firmware-mod-kit |
| Análise | EMBA, FACT, firmwalker |
| Emulação | QEMU, FirmAE, Firmadyne, Renode |
| Reverse | Ghidra, IDA (com processor modules) |
| Fuzzing | AFL++ + QEMU, afl-unicorn, Peach |
| Hardware | Saleae, Bus Pirate, J-Link, CH341A, Glasgow |
| Scan | nmap, banner grabbing |
| Wireless | Wireshark + AirPcap, Ubertooth, HackRF |

## Princípios

1. **Secure boot + signed updates**. Sem isso, tudo é opcional.
2. **Sem credenciais default únicas**. Senha por device ou obrigatoriedade de setup.
3. **Superfície mínima**: desabilitar telnet, UART em produção, JTAG fusível.
4. **TLS + autenticação** em protocolos de rede.
5. **Atualização realista**: plano de OTA por 10+ anos.
6. **TEE para segredos**: chave privada nunca em flash legível.
7. **Componentes com SBOM**: Vulnerable Component Analysis em firmware-level.
8. **Segmentação**: IoT em VLAN própria; acesso restrito a serviços específicos.
9. **Monitoramento**: dispositivo "normalmente calado" batendo em servidor estranho = IOC.
