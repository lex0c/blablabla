# Antivírus e EDR

Como *antivírus*, *EDR* (Endpoint Detection and Response) e *XDR* realmente funcionam por dentro. Foco em **arquitetura, implementação, e evasão** — não em ranking de produtos comerciais.

Análise de amostras específicas em `MALWARE_ANALYSIS.md`. Reversing em `REVERSE_ENGINEERING.md`. Ameaças em `THREAT_INTEL.md`.

## Evolução (5 gerações em 40 anos)

1. **1987-2000 — Signature AV**: hash/bytes match contra banco de assinaturas. McAfee, Norton, Kaspersky. Funciona enquanto malware é conhecido.
2. **2000-2010 — Heurística + emulação**: roda binário em emulator curto, observa comportamento suspeito (syscalls, API calls). Detecta variantes. Começa arms-race de packing.
3. **2010-2017 — Behavioral + cloud**: telemetria sobe pra cloud, reputação de hash, ML de primeiro geração, sandboxes on-device.
4. **2015-2022 — EDR**: coleta telemetria contínua (process tree, file, network, registry), correlaciona, detection rules (Sigma/EQL), response remoto. CrowdStrike, SentinelOne, Carbon Black.
5. **2020-presente — XDR**: mesmo conceito estendido a email, identity (AD/Entra), cloud, SaaS. Correlation cross-domain. Microsoft Defender XDR, Palo Alto Cortex.

AV puro morreu em mercado enterprise — sobrou em consumer (Windows Defender ainda é signature+heurística+ML no core). EDR é o produto; AV é um feature dentro dele.

## Arquitetura geral

```
┌─ ENDPOINT ──────────────────────────────────────────────────┐
│                                                              │
│  ┌─ Kernel-mode ────────────────────────────────────────┐   │
│  │ • File system filter (real-time scan on open/write)  │   │
│  │ • Process creation callbacks                         │   │
│  │ • Network filter (packet/connection intercept)       │   │
│  │ • Registry callbacks (Windows)                       │   │
│  │ • LSASS protection, memory scan                      │   │
│  │ • Self-protection (anti-tamper)                      │   │
│  └────────────────┬─────────────────────────────────────┘   │
│                   │ events + scan requests                   │
│  ┌────────────────▼─────────────────────────────────────┐   │
│  │ User-mode service                                     │   │
│  │ • Scan engine (signatures, heuristics, ML)           │   │
│  │ • Event buffer & local correlation                   │   │
│  │ • Policy engine                                       │   │
│  │ • Response (kill, quarantine, isolate)               │   │
│  └────────────────┬─────────────────────────────────────┘   │
│                   │ telemetry (batched, compressed)          │
└───────────────────┼──────────────────────────────────────────┘
                    │
         ┌──────────▼──────────────────────────────────────────┐
         │  CLOUD BACKEND                                       │
         │  • Event ingestion (billions/day)                   │
         │  • Correlation & detection (rules + ML + graph)     │
         │  • Threat intel enrichment                          │
         │  • Response orchestration                           │
         │  • Hunting & forensics UI                           │
         └──────────────────────────────────────────────────────┘
```

**Princípio**: kernel coleta e barra; user-mode decide; cloud correlaciona. Cada camada tem blast radius e latência diferentes.

## Real-time file scanning

Coração do "antivírus clássico". Cada file open/write é interceptado, conteúdo analisado, bloqueado se malicioso.

### Windows — Minifilter drivers

Filter Manager expõe **minifilter driver** (`fltmgr.sys`). Driver registra callbacks pra ops de I/O específicas:

```c
FLT_PREOP_CALLBACK_STATUS
PreCreate(PFLT_CALLBACK_DATA Data,
          PCFLT_RELATED_OBJECTS FltObjects,
          PVOID *CompletionContext) {
    // inspeciona Data->Iopb->TargetFileObject
    // retorna FLT_PREOP_SUCCESS_WITH_CALLBACK, ou block
}
```

Altitude (prioridade) registrada com Microsoft — evita ordem indefinida entre múltiplos AVs. Bloqueia ops retornando `STATUS_ACCESS_DENIED` ou `FLT_PREOP_COMPLETE`.

### Linux — fanotify + LSM + BPF-LSM

- **fanotify** (`fanotify_init`/`fanotify_mark`): monitora eventos de FS, pode bloquear (`FAN_ACCESS_PERM`). User-mode daemon recebe fd, lê evento, responde ALLOW/DENY. É o que ClamAV-daemon (clamd) + fanotify helpers usam.
- **LSM** (Linux Security Module): framework de hooks no kernel — SELinux, AppArmor, Tomoyo, Smack implementam. Só um "major" LSM; BPF-LSM permite múltiplos.
- **BPF-LSM** (5.7+): anexar programa eBPF a hooks LSM. Decisões de segurança em eBPF. Base de alguns EDRs Linux modernos.
- **Integrity Measurement Architecture (IMA)**: mede (hash) arquivos antes de execução; compara com policy. Mais appraisal do que detection.

### macOS — EndpointSecurity framework (ES)

Desde macOS 10.15 (2019), substituiu kauth e kernel extensions para AV/EDR. `EndpointSecurity.framework` entrega eventos em user-space via system extension:

```objc
es_new_client(&client, ^(es_client_t *c, const es_message_t *msg) {
    if (msg->event_type == ES_EVENT_TYPE_AUTH_EXEC) {
        es_respond_auth_result(c, msg, ES_AUTH_RESULT_DENY, false);
    }
});
```

Dois tipos de evento: `NOTIFY` (pós-fato, não bloqueia) e `AUTH` (pre-decision, bloqueia). Apple decidiu matar kexts: EDRs modernos em macOS rodam como system extensions notarizadas.

## Process / syscall monitoring

### Windows

- **`PsSetCreateProcessNotifyRoutineEx`**: callback em toda criação/término de processo.
- **`PsSetCreateThreadNotifyRoutine`, `PsSetLoadImageNotifyRoutine`**: thread creation e DLL load.
- **`ObRegisterCallbacks`**: intercepta open handle a processos/threads (usado pra proteger o próprio AV e detectar credential dumping em LSASS).
- **ETW** (Event Tracing for Windows) — especialmente **ETW Threat Intelligence**: trace de syscalls, AMSI, .NET, PowerShell. Muito mais leve que hooks inline.
- **AMSI** (Antimalware Scan Interface): scripts (PowerShell, VBA, JS, etc.) submetem conteúdo ao AV antes de executar. `AmsiScanBuffer()`. Defender e terceiros hookeiam.

### Linux

- **Audit subsystem** (`auditd`): syscalls logadas pra user-space. Barato, mas limitado em granularidade.
- **Tracepoints** (`sched_process_exec`, `sys_enter_*`): markers estáticos no kernel; eBPF pluga.
- **kprobes/uprobes**: instrumentação dinâmica. Mais poder, mais custo.
- **eBPF**: caminho moderno. Programas pequenos verificados rodam em tracepoints, LSM hooks, kprobes. Tetragon, Tracee, Falco são EDRs Linux baseados em eBPF.

### macOS

- `ES_EVENT_TYPE_AUTH_EXEC`, `ES_EVENT_TYPE_NOTIFY_FORK`, `NOTIFY_KEXTLOAD`, ~60 tipos.
- Anteriormente: kauth + kernel hooks (legado, deprecated).

## Técnicas de detecção

### 1. Signatures estáticas

Hash exato (MD5, SHA-1, SHA-256 do arquivo). Falha a qualquer bit flipado.

- **Imphash**: hash da import table (PE). Variantes com mesmas imports têm mesmo imphash. Útil em families.
- **ssdeep, TLSH, sdhash**: **fuzzy hashing**. Similaridade aproximada entre binários. TLSH é o mais usado em AV moderno.
- **Section hashes**: hash de `.text` apenas. Sobrevive a mudança de metadata.

### 2. YARA rules

DSL declarativa para match de strings, byte patterns, condições:

```yara
rule ExampleMalware {
    meta:
        author = "analyst"
    strings:
        $a = "malicious_string"
        $b = { 48 8B 05 ?? ?? ?? ?? E8 }  // hex com wildcards
        $c = /https?:\/\/bad\.example/
    condition:
        ($a or $c) and $b and filesize < 1MB and pe.is_32bit
}
```

Usado em: ClamAV, YARA-L (Chronicle), virtualmente todo threat hunter. Rápido, expressivo, compilável.

### 3. Heurística

Regras em cima de features: "PE com section de alta entropia + import de `VirtualAlloc`+`WriteProcessMemory` + TLS callback + strings em Cirílico". Score ponderado; acima de threshold = suspeito.

### 4. Emulation / dynamic unpacking

Binário executado em **emulator CPU** interno (não VM, mais leve). Executa até N instructions, observa comportamento. Unpack natural (UPX se unpacka sozinho). Windows Defender **MpEngine** tem emulator notório — ataques exploraram bugs dele (Travis Ormandy, 2017).

### 5. Sandbox

Execução em VM isolada (Cuckoo, CAPE, Any.Run, comerciais). Roda binário por 1-5 min, grava syscalls, network, mutex, file ops. Muito mais profundo que emulator, muito mais lento. Usado em cloud analysis, não on-device real-time.

### 6. ML classifiers

Features comuns em PE files:
- Entropy por section
- Número de sections
- Tamanho e ratios
- Imports hash, imports count
- Caracteristics flags
- Presença de TLS, overlay, digital signature
- Strings extraídas (URLs, paths, registry keys)
- Opcode n-grams

Modelos: gradient boosting (LightGBM) pra tabular; CNN/LSTM em byte sequences (Ember, MalConv); BERT-family sobre disassembly (recente).

**Ember** (Endgame Malware BEnchmark for Research): dataset público de features PE + labels. Baseline comum.

### 7. Behavioral / ATT&CK-mapped detection

Não olha para *o que é*, olha para *o que faz*. "Processo `winword.exe` executando `powershell.exe -enc <base64>`" = suspeito independente do binário.

Framework: **MITRE ATT&CK**. Táticas (Initial Access, Execution, Persistence, ...), técnicas (T1055 Process Injection, T1003 OS Credential Dumping, ...), sub-técnicas.

Detection rules em:
- **Sigma** (vendor-neutral YAML): compila pra Splunk SPL, Elastic KQL, Sentinel KQL, etc.
- **Elastic EQL** (Event Query Language): sequence queries sobre process tree.
- **Microsoft KQL** em Defender XDR.
- **CrowdStrike CQL**, SentinelOne STAR.

### 8. Memory scanning

Malware unpackado vive em memória (`VirtualAlloc` + `RWX` ou via `WriteProcessMemory` em processo legítimo). Scanner periodicamente hasheia regiões `PRIVATE+EXECUTE`, compara com YARA, signatures.

- **In-process scan** (Defender AMSI scan após unpack).
- **Cross-process scan**: AV abre handle pra outros processos, lê memória. Bloqueado se alvo é PPL.

## AMSI em detalhe (Windows-specific, importante)

Script engines (PowerShell, VBScript, JScript, VBA, WMIC, HTA) chamam `AmsiScanBuffer()` antes de executar código dinamicamente. AV registrado como AMSI provider recebe buffer, escaneia, devolve veredito.

Antes de AMSI: obfuscação Base64/XOR + `Invoke-Expression` era trivial. Depois de AMSI: conteúdo decoded é entregue ao AV **antes** de executar.

Evasão AMSI (ver seção de evasão): patching em memória, DLL load order abuse, downgrade forçado.

## EDR — o que muda

AV clássico: *preveno/bloqueio* na detecção. EDR: *coleta tudo*, deixa correlation resolver depois.

### Telemetria coletada

- Process tree (PID, PPID, cmdline, hash de image, integrity level, token, user SID)
- File events (create, write, delete, rename, ACL change)
- Network (conn open/close, DNS queries, TLS SNI, TLS JA3/JA4)
- Registry (key create, value set, especialmente RUN keys)
- Module loads (DLLs, drivers)
- Logon events (4624, 4625 no Windows; PAM/SSH em Linux)
- WMI, scheduled tasks, services
- Script content (via AMSI/PowerShell module logging)
- ETW providers: kernel-process, .NET, DNS, RPC, WMI

Volume típico: **10k-100k eventos/minuto/endpoint**. Compressão + batching agressivos.

### Correlação

**Local** (on-agent): rules simples, primeiro filtro. Reduz volume enviado.

**Cloud**:
- **Graph-based**: nós = entidades (process, file, host, user, IP), arestas = eventos. Query "processo que baixou, escreveu arquivo, spawneou child com `rundll32`".
- **Statistical anomaly**: comportamento inédito relativo a baseline por usuário/host/AD OU.
- **Correlation across endpoints**: mesmo IP externo contactado por N endpoints em M minutos → C2 beacon.
- **Threat intel enrichment**: hash/IP/domain joinado contra feeds.

### Response

- Kill process (por PID ou imagem)
- Quarantine file (mover pra zona restrita)
- Network isolation (host só fala com console EDR)
- Remediation script (PowerShell remoto, osquery action)
- Memory dump pra forense
- Live response shell (console)

Tudo auditado, idealmente aprovado por analista (ou automático em alta confiança).

## Sensors / agents open-source (referência)

- **osquery** (Meta, agora Linux Foundation): sistema operacional como tabelas SQL. `SELECT * FROM processes WHERE name='powershell.exe'`. Coleta, não detecta nem bloqueia. Base de muitos EDRs comerciais.
- **Sysmon** (Windows): sysinternals. Log detalhado de process/network/file. Configuração via XML (swiftonsecurity/sysmon-config é canônico). Input crítico pra SIEM/EDR.
- **auditd** (Linux): legado mas ubíquo.
- **Falco** (CNCF): detection rules em YAML sobre eventos kernel via eBPF. Origem cloud-native (containers).
- **Tetragon** (Isovalent/Cilium): observability + enforcement via eBPF. Blocking em syscall-level.
- **Tracee** (Aqua): eBPF-based tracing + detection.
- **Wazuh**: fork/spinoff OSSEC, HIDS completo (log analysis, FIM, rootcheck, response).
- **Velociraptor**: DFIR at scale, VQL query language, endpoint artifact collection. Não-substituto de EDR mas complementar.
- **LimaCharlie**: infraestrutura EDR "bring your own detection" — plataforma.
- **Elastic Defend / Elastic Security**: stack aberta (agent + engine + SIEM).

## ClamAV — AV open source como referência

Arquitetura:

1. **libclamav**: engine. Recebe arquivo, aplica matchers em cascata.
2. **Signature DBs**:
   - `.hdb`/`.hsb` — MD5/SHA-1/SHA-256 hashes.
   - `.ndb` — byte patterns com offsets e wildcards.
   - `.ldb` — logical signatures (combinação de sub-signatures).
   - `.cvd`/`.cld` — databases assinadas, atualizadas via freshclam.
   - `.yara` — suporte YARA.
3. **Unpackers**: UPX, FSG, PETite, NSIS, Aspack, MEW — desempacota antes de scan.
4. **Parsers**: PE, ELF, Mach-O, OLE2, PDF, HTML, archives (zip/rar/7z/tar), email (MIME).
5. **clamscan**: on-demand CLI.
6. **clamd**: daemon; `clamdscan`, `clamav-milter` (email gateway).
7. **Real-time no Linux**: clamd + fanotify helper (clamav-unofficial-sigs).

ClamAV é sólido em email gateway, fraco em endpoint real-time comparado a EDRs modernos. Mas é o melhor código pra entender como um AV funciona por dentro.

## Windows Defender — arquitetura

- **MsMpEng.exe**: service user-mode, roda como `NT AUTHORITY\SYSTEM` dentro de Protected Process Light (PPL). Hosta o MpEngine.
- **MpEngine.dll**: engine de scan. Tem emulator próprio (JavaScript, PowerShell, x86/x64), AV signatures, heuristics, ML. Historicamente atacável — Ormandy achou RCE via emulator JS em 2017.
- **WdFilter.sys**: minifilter de FS.
- **WdBoot.sys**: ELAM driver (anti-rootkit em boot).
- **WdNisDrv.sys** + **MpNetworkMonitor**: network inspection.
- **Cloud Protection**: telemetria + lookup em real-time (MAPS). Decide sob N ms.
- **SmartScreen**: reputation pra downloads/installs.
- **Controlled Folder Access**: whitelist apps que podem escrever em Documents/Desktop etc.
- **Exploit Protection**: mitigations (DEP, CFG, ACG, ASR rules).

Desde Windows 10, Defender ficou sério (top tier em MITRE ATT&CK evals). É praticamente um EDR (Defender for Endpoint estende).

## Evasão — o outro lado

EDR é adversarial. Profissionais (red team, APT) têm toolkit amplo.

### Packing / crypting

- **Packers** (UPX, ASProtect, Themida, VMProtect): comprimem/criptografam payload; stub unpacka em runtime. Quebra signatures estáticas, **não** sobrevive emulator/sandbox/memory scan.
- **Crypters custom**: mesma ideia, assinatura única.
- **Polymorphic**: código muta a cada cópia. Stubs gerados em runtime.
- **Metamorphic**: binário reescreve-se; raríssimo hoje.

### In-memory execution

- **Reflective DLL injection** (Stephen Fewer): DLL carregada sem `LoadLibrary` (alloc + map + call). Não aparece em module list.
- **Process Hollowing**: criar processo suspenso, unmap da imagem original, mapear payload no lugar, resume. Persistência masquerading.
- **Process Doppelgänging** (Liberman/Kotler): usa NTFS transactions pra comitar imagem fake e rodar.
- **Module Stomping**: sobrescrever seção `.text` de DLL legítima já carregada.
- **Atom Bombing**: globals via atom tables pra inject.
- **Early Bird APC, Extra Window Memory, Thread Hijacking**: variações.

### AMSI bypass

- **Patch `AmsiScanBuffer`** em memória (substitui pelos primeiros bytes por `xor eax, eax; ret` → sempre "clean").
- **`amsi.dll` forced fail** via registry `HKCU\Software\Microsoft\Windows Script\Settings`.
- **DLL hijacking** de `amsi.dll` em load order.
- **Obfuscação** que derrota signatures mesmo após decode.

### EDR unhooking

Muitos EDRs fazem **inline hooks** em `ntdll.dll` (usermode, antes do syscall). Payload re-mapa `ntdll.dll` fresco do disco → hooks desaparecem. Ou:

- **Direct syscalls**: emit `syscall` opcode com SSN (syscall number) correto, pulando `ntdll` inteira. **Hell's Gate** (extrai SSN do ntdll), **Halo's Gate, Tartarus Gate** (variantes).
- **Indirect syscalls**: `call <ntdll_address>` pra instruction `syscall` original (frustra heuristics que checam retorno).
- **Sysmon/ETW unhook**: disable ETW providers que EDR monitora, patch `EtwEventWrite`.

Contramedida EDR: kernel callbacks > usermode hooks (unhooking não atinge kernel). ETW TI (Threat Intelligence) é kernel-side, resistente.

### Living off the land (LOLBin/LOLBAS)

Usar binários legítimos pra ação maliciosa. `rundll32.exe`, `regsvr32.exe`, `certutil.exe`, `mshta.exe`, `wmic.exe`, `bitsadmin.exe`, PowerShell, `msbuild.exe`. Assinado pela Microsoft, difícil bloquear sem false positives.

Lista canônica: **lolbas-project.github.io** (Windows), **gtfobins.github.io** (Linux).

Contramedida: behavioral detection + AppLocker/WDAC allowlist + reduzir superfície (ASR rules).

### Sleep obfuscation / beacon stealth

C2 implants que dormem longos períodos com stack + heap encrypted (só decriptam pra ativar). **Ekko, Foliage, Nightingale**. Frustra memory scan.

### Byovd — Bring Your Own Vulnerable Driver

Attacker carrega driver legítimo assinado (mas vulnerável) e explora. Capsula (tamper AV), RTCore64.sys, Gdrv.sys — listas curadas. Microsoft mantém block list; lenta pra atualizar.

### Contramedida sistema: EDR vs evasion

O gap nunca fecha. EDR confia em:
1. **Kernel visibility** (process/thread create callbacks — attacker não desregistra trivialmente sem exploit).
2. **Telemetria variada** (se um canal é mudo, outro ainda fala).
3. **Behavioral detection** (ação final é o que importa).
4. **Cloud correlation** (coisas que não fazem sentido isoladas fazem em conjunto).
5. **Time-to-detect, não time-to-block**: assumir comprometimento, reduzir dwell time.

## Self-protection / anti-tamper

- **PPL / PP** (Protected Process Light / Protected Process): Windows nega OpenProcess com write rights a processos protegidos, mesmo SYSTEM. Defender roda PPL.
- **Driver signing enforcement**: kernel module precisa ser assinado. EDR driver é Microsoft-signed, attacker precisa de BYOVD ou exploit.
- **ELAM** (Early Launch Anti-Malware): driver iniciado antes de boot drivers não-essenciais. Inspeciona outros drivers.
- **HVCI** (Hypervisor-protected Code Integrity / Memory Integrity): via Hyper-V, kernel code signing enforcement. Mesmo com exploit, difícil patchar kernel.
- **Service recovery / watchdog**: se agent crasha, outro service restart.
- **Tamper protection UI**: admin local não pode desligar Defender sem MDM/Intune approval.

Qualquer EDR sério tem essas camadas. Sem elas, primeiro malware com SYSTEM mata o agente.

## Testing e evaluation

- **MITRE ATT&CK Evaluations** (`attackevals.mitre-engenuity.org`): simulação de APT real; produtos são avaliados em cobertura (detectou? qual contexto? tagged corretamente?). Referência mais honesta do mercado.
- **Atomic Red Team** (Red Canary): biblioteca open de atomic tests por técnica ATT&CK. `atomics/T1055/T1055.md`.
- **Caldera** (MITRE): automation de adversary emulation.
- **PurpleSharp, Stratus Red Team** (cloud).
- **AV-Test, AV-Comparatives**: tests consumer, mais markeitng-friendly.

## Princípios

1. **AV ≠ EDR**. AV preveno, EDR investiga. Moderno é visibility-first.
2. **Defender depth**: nenhuma camada sozinha para attacker motivado. Kernel + usermode + cloud + IR.
3. **Behavioral > signature** pra threat moderna. Signatures ainda matam lixo em massa.
4. **Telemetry é o ativo, não a detection**. Com dados bons, rule nova detecta ameaça antiga. Sem dados, rule perfeita é inútil.
5. **ATT&CK é lingua franca**. Descreva tudo em técnicas; mapeia detections; evita vendor-speak.
6. **Kernel visibility é crítica**. Usermode hooks são joia frágil; kernel callbacks são ancoradouro.
7. **False positives matam credibilidade**. Boa detection tem precision > 0.9; ruim desaparece em whitelist.
8. **Assume breach**: questão é detectar rápido, não prevenir sempre. Dwell time (time-to-detect) é métrica-chave.
9. **EDR vendors hook inline; attackers unhook**. Conforme kernel mitigations fortalecem, evasion vai pra LOLBin/BYOVD/supply chain.
10. **Logs sem retenção = sem investigation**. Armazenar 90d+ é pré-requisito de IR.

## Recursos

- **"Practical Malware Analysis"** — Sikorski & Honig. Base.
- **"The Art of Memory Forensics"** — Ligh, Case, Levy, Walters.
- **"Windows Internals 7e"** — Russinovich et al.
- **"Evading EDR"** — Matt Hand (No Starch, 2023). Estado da arte em bypass.
- **"Designing Secure Systems"** — Dr. Ross Anderson — Security Engineering 3e.
- **MITRE ATT&CK** — `attack.mitre.org`.
- **Sigma HQ** — `github.com/SigmaHQ/sigma`. Rule repo.
- **Atomic Red Team** — `atomicredteam.io`.
- **Elastic Security docs** — open, transparente.
- **CrowdStrike falcon OverWatch reports** — annual threat landscape.
- **HexRays, NSA Ghidra, radare2** — reversing tooling.
- **Connor McGarr's blog**, **Sektor7** (crypter school), **MalDev Academy** — evasion atual.
- **Andrea Fortuna, MalwareTech, Didier Stevens** — blogs clássicos.
