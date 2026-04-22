# Ataques de Hardware

Ataques em nível de hardware exploram o **mundo físico** que as abstrações de software tentam ignorar. Enquanto software assume "1 + 1 = 2 sempre", hardware vive em um universo de ruído, tensão, tempo e radiação. Quando atacante tem acesso físico (ou quase), abstrações de isolamento caem.

## Modelo de Ameaça

- **Acesso físico**: atacante com o dispositivo em mãos. Pior caso: smartcards, HSMs, wallets de cripto, pacemakers.
- **Acesso próximo**: laser, EM, acoustic. Sem tocar.
- **Acesso remoto à microarquitetura**: Rowhammer, Spectre — sem hardware dedicado.

Ataques de hardware são, geralmente, **caros** (equipamento, tempo) mas frequentemente resultam em bypass total de criptografia/autenticação.

## Side Channels

Informação vazada por canal secundário — tempo, energia, EM, luz, som.

### Timing

Já coberto em `CRYPTO_ADVANCED.md`. Em hardware:

- Algoritmos com branching em segredo.
- Cache hits/misses variam acesso.
- Multiplicação modular em microcode pode variar com valores.

### Power Analysis

Consumo de corrente do chip varia com operações.

- **SPA (Simple Power Analysis)**: observar traço de consumo; picos revelam sequência de operações. Extração de chave RSA observando exponenciação square-and-multiply.
- **DPA (Differential Power Analysis)**: estatística sobre muitos traços. Paul Kocher (1998). Devastador em smartcards sem contramedidas.
- **CPA (Correlation Power Analysis)**: variante com modelo de Hamming weight ou Hamming distance.

Contramedidas:

- **Masking**: operar em valor XOR-mascarado; consumo não correlaciona com segredo.
- **Hiding**: adicionar ruído (dummy ops, clock jitter).
- **Constant-time, constant-power** implementações.

Ferramentas: **ChipWhisperer** (NewAE) — plataforma open-source para side-channel + glitching.

### Electromagnetic Analysis (EMA)

Similar a power mas por radiação EM de vizinhança do chip. Sonda próxima ao die capta sinal; análise análoga a DPA.

### Acoustic

Chaves RSA extraídas ouvindo coil whine do laptop (Genkin, Shamir, Tromer, 2013). Possível com microfone de celular próximo.

### Optical

LEDs de atividade de disco/CPU correlacionam com operações. Side channel surpreendentemente prático a distância (van Eck phreaking moderno).

### TEMPEST

Captura remota de radiação EM de monitores/cabos reconstrói conteúdo. NSA tem padrões TEMPEST há décadas; pesquisa aberta (Kuhn, 2000s).

## Fault Injection (Glitching)

Induzir erro na execução para pular uma instrução — frequentemente a verificação de autenticação.

### Métodos

- **Voltage glitching**: pulso breve na Vcc causa instruções falharem. Ferramentas: ChipWhisperer, PicoEMP.
- **Clock glitching**: pulso extra ou reduzido no clock. Instrução executa parcialmente.
- **EM pulse**: bobina próxima a área do chip induz falha localizada. Ferramenta: ChipSHOUTER.
- **Laser**: laser focalizado em transistores específicos. Precisão cirúrgica; requer decap.
- **Temperature**: underclock + frio extremo causa flips.
- **Optical (UV/visible)**: EEPROMs antigos zerados com UV.

### Alvos típicos

- **Bootloader**: pular verificação de assinatura.
- **Fuses de segurança**: pular leitura.
- **PIN check** em smartcard, wallet de hardware.
- **Kernel check** em processador seguro.

### Contramedidas

- **Dupla verificação**: verificar duas vezes com caminhos diferentes.
- **Delay aleatório, dummy rounds**.
- **Integrity check em código em execução**.
- **Sensores**: glitch detectors em silício.
- **Redundant logic** em hardware crítico.

## Chip Decap e Microscopia

Para ataques invasivos extremos:

- **Decap**: remover encapsulamento — fuming nitric acid para plástico; calor para certos packages.
- **Microscopia óptica / SEM**: fotografar die.
- **FIB (Focused Ion Beam)**: cortar/reconectar trilhas com precisão nm.
- **Probe station**: contato físico com sinais internos.

Ataques derivados:

- **Read ROM mask**: conteúdo gravado fisicamente na topologia.
- **Observe internal buses**: dump de memória em runtime.
- **Modify circuits**: adicionar backdoor persistente.

Custo: dezenas a centenas de milhares de dólares em equipamento. Dentro do alcance de estados e grandes cibercriminosos.

## Rowhammer

Acessar repetidamente uma linha de DRAM causa bit flips em linhas adjacentes — fenômeno físico (acoplamento elétrico) não previsto em design de DDR3.

- **Original paper** (Kim et al., 2014).
- **Exploits**: Project Zero (2015) — escalar de user para root pulando em linhas que contêm page tables.
- **Variantes**: Throwhammer (sobre RDMA), Drammer (Android), RAMBleed (leak vs flip).

**Mitigações**:

- **TRR (Target Row Refresh)**: DDR4+. Contornada por padrões multi-sided.
- **ECC**: DDR4 ECC detecta/corrige muitos flips, mas não todos (estudos mostram que pode ser contornado).
- **DDR5**: mitigações adicionais mas Rowhammer adaptado ainda afeta.

## Cold Boot Attack

RAM retém dados por segundos-minutos após desligar — mais se congelada com spray de ar comprimido invertido (−50°C). Atacante desliga máquina com FDE (Full Disk Encryption) unlocked, puxa DIMM, insere em máquina controlada, dumpa → recupera chaves.

**Mitigações**:

- **Chaves em registradores/CPU-only** (ex.: TRESOR, Loop-Amnesia).
- **RAM soldada**.
- **Memory scrubbing** em suspend.
- Dispositivos modernos com **memory encryption** (SEV, TME) — dados em DIMM já criptografados.

## Evil Maid

Alguém (camareira de hotel) com acesso físico breve compromete o dispositivo:

- **Flash firmware comprometido** do bootloader.
- **Implante físico** (USB, cabo).
- **Keylogger** hardware entre teclado e computador.
- **Thunderclap, Thunderspy**: DMA via Thunderbolt/PCIe externo, contorna bloqueio de tela.

**Mitigações**:

- **Secure Boot + firmware signing**.
- **TPM + PCR measurement**: detectar mudança em firmware.
- **Kernel DMA Protection**.
- **Physical tamper evidence** (selos, resina epóxi).
- **Não deixe dispositivos desbloqueados em locais inseguros**. Básico mas subestimado.

## TPM e Ataques a TPM

**TPM** (Trusted Platform Module) armazena chaves, faz atestação, derrete em caso de tamper. Ainda:

- **TPM Sniffing**: comunicação TPM-CPU sobre LPC ou SPI sem encryption pode ser sniffada. **Bitlocker** sem PIN extra foi explorado assim (Dolos group, 2021).
- **fTPM bugs**: AMD fTPM com RC4 fraco em early boot.
- **Mitigação**: TPM 2.0 com session encryption + PIN obrigatório.

## USB Attacks

- **BadUSB** (2014): firmware de pendrive pode declarar-se teclado/ethernet. Digitar comandos, hijack DNS.
- **Rubber Ducky, Bash Bunny, OMG Cable** — devices comerciais.
- **Juice Jacking**: carregador USB público com data line — copia dados, injeta malware.
- **P4wnP1**: Raspberry Pi Zero como HID.

**Mitigação**:

- **USB Condom** / power-only cables.
- **Desabilitar USB** quando não usado (GPO).
- **USBGuard** (Linux): allowlist por device class/VID/PID.
- **USB Keyboard allowlisting** (Windows).

## DMA Attacks

Thunderbolt e PCIe externos permitem DMA — read/write de RAM sem passar pelo OS.

- **FireWire**: histórico; quase aposentado.
- **Thunderbolt**: Thunderspy (2020) demonstrou bypass com ferramenta ~$200.
- **PCILeech**: framework de DMA attacks.

**Mitigação**:

- **IOMMU / Intel VT-d / AMD-Vi**: bloqueia DMA arbitrário.
- **Kernel DMA Protection** (Windows 10+).
- **Desabilitar Thunderbolt** em equipamento sensível.

## Implantes e Supply Chain Física

- **Bloomberg Chinese Chip** (2018): alegou implantes em placas Supermicro; negado por todas as partes; evidência fraca. Mas tema é real — alvo de estados.
- **Chip-level backdoor**: dopante malicioso que ativa em condição específica. Pesquisa acadêmica (Becker et al., 2013) demonstrou viabilidade.
- **Interdição**: tradução: NSA já interceptou envios, adicionou implantes em firmware, reenviou (Snowden leaks).

## Contra-medidas Gerais

- **Tamper-evident + tamper-resistant**: selos, resinas, mesh metal em hardware de alta segurança (HSM).
- **Anti-tamper sensors**: detectam decap, temperatura fora da faixa, intrusão. Destruição de chave crítica.
- **Secure elements** (Infineon, NXP, STMicro): co-processadores endurecidos para side channel e glitching.
- **HSM certificados**: FIPS 140-2/3, Common Criteria EAL.
- **Redundancy**: threshold signatures, MPC — chave nunca existe em um local só (ver `CRYPTO_ADVANCED.md`).

## Ferramentas

- **ChipWhisperer**: SCA + glitching, open.
- **PicoEMP**: EM fault injection acessível.
- **GreatFET, HackRF, YARD Stick One**: plataformas RF/hardware hacking.
- **Glasgow**: Swiss-army knife em FPGA.
- **Bus Pirate, Shikra**: protocolos serial.
- **Saleae Logic**: analisador lógico.
- **ProxMark3**: RFID/NFC.
- **Flipper Zero**: multi-protocolo acessível; bom para aprender.

## Cenário onde importa

1. **Cripto wallets** (Ledger, Trezor): alvo direto. Empresas investem em anti-glitch.
2. **Smartcards** (bancários, SIM): DPA é ameaça histórica; contramedidas maduras.
3. **HSMs**: mesmo.
4. **Consoles de jogos**: pesquisa de jailbreak é escola aberta de SCA/glitch.
5. **Celulares high-end**: Secure Enclave (Apple), Titan M (Google). Cada geração fecha vetores.
6. **Carros**: ECUs, chaves FOB.
7. **Medical devices, infra crítica**: segurança física é requisito.

## Lições

1. **Física vence abstração**. Isolamento lógico não basta contra acesso físico competente.
2. **Custo é a barreira**. Atacante estatal tem recursos para decap; um criminoso comum, não — mas ChipWhisperer está no alcance de qualquer pesquisador hoje.
3. **Defesa em camadas** também em hardware: secure boot + TPM + tamper sensors + encryption em RAM + kernel DMA protection.
4. **Threat model realista**: dispositivo de consumo não precisa resistir a FIB com laser; HSM bancário sim.
5. **Transparência e auditoria**: hardware fechado é caixa-preta; projetos open-hardware (OpenTitan, Google) crescem.

## Recursos

- **"Hardware Hacker"** — Andrew "bunnie" Huang.
- **"The Hardware Hacking Handbook"** — Jasper van Woudenberg, Colin O'Flynn.
- **"A Practical Introduction to Hardware/Software Codesign"**.
- **DEF CON Hardware Hacking Village, HITB, Black Hat** talks.
- **Wrongbaud.ninja**, **J Grand blog**, **Joe Fitz** — blogs.
- **ChipWhisperer docs + tutorials** — SCA prática acessível.
