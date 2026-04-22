# Engenharia Reversa

*Reverse engineering* (RE) é o processo de extrair compreensão de um artefato a partir do seu produto final — binário, firmware, protocolo — sem código-fonte. Em segurança, RE viabiliza análise de malware, descoberta de vulnerabilidades, bypass de proteções, interoperabilidade.

## Objetivos Comuns

1. **Entender o que um binário faz**: malware, proprietário sem doc, amostra suspeita.
2. **Encontrar vulnerabilidades**: feed para `BINARY_EXPLOITATION.md`.
3. **Reimplementar / interoperar**: wine reimplementa Windows API; Samba fez RE de SMB.
4. **Bypassar verificações**: licensing, anti-cheat, DRM. Dual-use — dentro e fora da lei conforme contexto.
5. **Auditoria de cadeia de suprimentos**: validar que binário distribuído faz só o que deveria (ver `SUPPLY_CHAIN.md`).

## Análise Estática vs Dinâmica

### Estática

Examinar o binário **sem executá-lo**.

**Vantagens**: seguro (não corre código hostil), determinística, pode analisar todos os caminhos.

**Desvantagens**: código ofuscado/packed é opaco; constantes podem ser calculadas em runtime; decidir o fluxo real em binários grandes é lento.

**Ferramentas**:
- **Ghidra** (NSA, gratuito, cross-platform). Inclui decompiler para várias arquiteturas.
- **IDA Pro** (Hex-Rays). Clássico, caro, referência de facto. IDA Free cobre menos.
- **Binary Ninja**. Moderno, API rica, decompiler próprio.
- **radare2 / rizin + cutter**. Open-source, CLI com GUI opcional.
- **objdump, readelf, nm, strings, file**. Básico Unix; chega longe.

### Dinâmica

Executar e observar.

**Vantagens**: vê comportamento real, contorna ofuscação estática (código desempacota em runtime), rápido para entender fluxo principal.

**Desvantagens**: código malicioso pode detectar análise e esconder capacidade (anti-debug, anti-VM); cobre só caminhos executados.

**Ferramentas**:
- **Debuggers**: GDB (Linux), x64dbg / OllyDbg (Windows), WinDbg (kernel), LLDB (macOS).
- **Plugins de contexto**: pwndbg, gef, peda (tornam GDB usável para RE/exploit).
- **Emuladores**: QEMU em user-mode (qemu-user) ou system. Unicorn (lib de emulação baseada em QEMU).
- **Tracers dinâmicos**: strace, ltrace, sysdig, dtrace. Revelam syscalls e chamadas a libs.
- **Instrumentação**: Frida (injeta JS em processos; sustenta RE mobile), DynamoRIO, Pin, TinyInst.
- **Sandboxes**: Cuckoo, CAPE, any.run (comerciais e gratuitos). Executam malware em VM isolada, coletam indicators.

A prática real combina: estática para mapear, dinâmica para confirmar, estática de novo para generalizar.

## Tipos de Artefato

### Binários compilados (ELF, PE, Mach-O)

- **ELF (Linux, Unix)**: header + segments (loadable) + sections (lógicas). `readelf -a binary.elf`.
- **PE (Windows)**: MZ + PE header + sections. `pestudio`, `PE-bear`.
- **Mach-O (macOS, iOS)**: fat binaries podem conter múltiplas arquiteturas. `otool`, `jtool2`.

Campos de interesse: ponto de entrada, imports (APIs chamadas), exports, sections (executável vs gravável), strings, signatures.

### Bytecode

- **Java / `.class` / `.jar`**: `javap -c`, **JD-GUI**, **CFR**, **Procyon**, **Recaf**. Android DEX → smali via **apktool** + **jadx**.
- **.NET / CIL**: **dnSpy** (debugger + decompiler), **ILSpy**. Decompila quase ao código original.
- **Python `.pyc`**: **uncompyle6**, **decompyle3** para versões antigas. **pycdc** para novas.
- **Go**: símbolos são embutidos por default; `go_parser`, `GoReSym`, `redress` extraem nomes.
- **Rust**: demanglers específicos; funções genéricas geram explosão combinatorial.

### JavaScript minificado / WebAssembly

- **Source maps** (se disponíveis) resolvem nomes.
- **wasm-decompile**, **wabt**, Ghidra Wasm para WebAssembly.

### Firmware

- **binwalk** — identifica e extrai sistemas de arquivos embutidos, kernels, configs.
- **unblob** — extrator mais moderno.
- **firmwalker** — busca credenciais em firmware extraído.
- Ver `IOT_FIRMWARE_SECURITY.md`.

## Fluxo Típico

### 1. Triagem

- `file binary` — tipo, arquitetura.
- `strings binary | less` — text strings (URLs, mensagens, paths).
- Hashes (MD5, SHA-256) para search em VirusTotal, MalwareBazaar.
- Entropia (`binwalk -E`) — alta entropia sugere packing/crypto.
- Assinatura digital (se houver): `osslsigncode verify`, `signtool`.

### 2. Carregar no desassemblador

Abre no Ghidra/IDA. Renomeie funções conforme entende. Use o decompiler — nem sempre certo, mas ótimo ponto de partida.

### 3. Identificar APIs

Imports revelam capacidades: `CreateRemoteThread`, `VirtualAllocEx`, `WriteProcessMemory` → injection. `WinHttpOpen`, `send`, `recv` → network. `CryptGenKey`, `BCryptEncrypt` → crypto.

### 4. Mapear controle de fluxo

Entry point → main → subrotinas. Grafo de chamadas ajuda. Funções importantes: parser de config, networking, loop principal.

### 5. Decompile e renomeie

Dê nomes significativos a variáveis e funções. `sub_401000` não ajuda; `parse_config_blob` sim. O conhecimento que você ganhou é o que vai reusar amanhã.

### 6. Confirme dinamicamente

Breakpoint em ponto suspeito; rode; inspecione estado. Muitas vezes a hipótese da estática se confirma ou cai aqui.

### 7. Documente

Markdown com funções identificadas, string → uso, protocolo, IOCs. Tempo investido em RE sem nota é tempo perdido em 6 meses.

## Anti-Análise — e Contra-Medidas

Malware tenta detectar que está sendo analisado:

### Anti-debug

- `IsDebuggerPresent`, `CheckRemoteDebuggerPresent` (Windows).
- Timing: `rdtsc` antes/depois; se muito tempo passou, está em debugger.
- `PEB.BeingDebugged` flag. `NtQueryInformationProcess`.
- Exceções: lançar exceção; debugger captura, programa normal segue.
- Self-debug: chamar `DebugSelf` — se já debugado, falha.

**Contra**: patch das checks (plugins como ScyllaHide); hooking. Ou rode num debugger que esconda presença.

### Anti-VM

- Detecta VMware/VirtualBox via registry keys, MAC addresses, instruções específicas (`cpuid` reporta hypervisor), drivers montados.

**Contra**: VM endurecida — scripts que apagam artefatos de VM. Ou bare-metal sandbox (caro).

### Anti-análise comportamental

- **Sleep de minutos**: sandboxes timeout antes do payload executar. Mitigação: hook de sleeps.
- **Requisita interação**: espera movimentos de mouse, teclado.
- **Geofencing**: só roda se IP/timezone/locale específicos.
- **Domain fronting / DGA**: evita IOCs estáticos.

### Packing e Crypting

Código empacotado é descomprimido/decodificado em runtime. Ferramentas:

- **UPX** (comum, legítimo e malicioso). `upx -d` desempacota.
- Packers customizados/comerciais (Themida, VMProtect) — muito difíceis. Usam VM virtualization: cada instrução original vira bytecode próprio do packer.
- **Unpacking manual**: rodar em debugger, colocar breakpoint no momento em que a entropia cai / código original está em memória, dump. Ferramentas: Scylla (reconstrução de IAT), OllyDumpEx.

### Obfuscation de código

- **Control flow flattening**: switch enorme com cada bloco.
- **Opaque predicates**: branches que sempre tomam mesmo caminho mas parecem condicionais.
- **MBA (Mixed Boolean-Arithmetic)**: expressões equivalentes a operações simples, mas sintaticamente complexas.
- **String encryption**: strings em runtime, não legíveis estaticamente.

**Contra**: **execução simbólica** (angr, Triton), **taint analysis**, deofuscadores específicos (miasm, sidefx), ou trabalhar no nível decompilado.

## Execução Simbólica

Substitui valores concretos por símbolos; o solver SMT decide quais inputs levam a qual caminho. Útil para:

- Achar input que atinge branch específico.
- Resolver "crackme" (inserir serial válido).
- Encontrar caminhos que levam a assertions/crashes.

**Ferramentas**: **angr** (Python, em cima de VEX), **Triton** (C++/Python, taint + simbólica), **KLEE** (LLVM IR), **Manticore**.

Limitação: *path explosion*. Em binários grandes, árvore de caminhos explode. Combinar com *concolic execution* (guia com execução concreta) ou heurísticas.

## RE de Protocolos

Quando o alvo é um protocolo de rede:

- **Capturar tráfego**: `tcpdump`, **Wireshark**. Dissectors customizados em Lua.
- **MITM**: `mitmproxy`, `Burp Suite`, `fiddler`. Para TLS: interceptar + reassinar (exige instalar CA no cliente).
- **Identificar estrutura**: bytes fixos, counters, offsets. Diff entre mensagens.
- **Replay**: tentar reexecutar para entender efeitos.
- Para binários closed-source: instrumentar API de rede (`send`, `recv`) em runtime.

## Crackmes e Treino

A forma mais rápida de aprender:

- **crackmes.one** — repositório com milhares de níveis.
- **Flare-On** — CTF anual da Mandiant, focado em RE.
- **pwn.college, HackTheBox, RootMe** — trilhas dedicadas.
- **Plataformas reais com bug bounty** — uma vez confortável.

## Aspectos Legais

- **DMCA anti-circumvention** (EUA) criminaliza contornar DRM em alguns contextos; exceções para segurança/pesquisa foram expandidas mas são específicas.
- **Cláusulas de EULA**: muitos produtos proíbem RE por contrato. Aplicabilidade varia por jurisdição.
- **Pesquisa acadêmica** geralmente protegida; produto comercializado exige cuidado.
- **Software livre**: sem restrição.

## Princípios

1. **Renomeie enquanto entende**. A primeira hora vale um dia futuro.
2. **Combine estática e dinâmica**. Uma valida a outra.
3. **Conheça o ABI e APIs do sistema**. Windows API, POSIX, ARM calling conv — vocabulário básico.
4. **Automatize repetitivo**. Scripts de Ghidra/IDA, Frida para tasks recorrentes.
5. **Documente ou esqueça**. Notas, screenshots, diagrams. Próxima amostra reaproveita.
6. **Tempo é a restrição real**. Priorize funções prováveis de conter o que você quer (tendem a ser centrais no grafo, com muitas strings, chamando APIs relevantes).
