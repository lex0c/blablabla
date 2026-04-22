# Forense Digital

**Digital Forensics (DFIR é a junção com IR)** é a disciplina que recupera, preserva, analisa e apresenta dados digitais como evidência. Aplica-se a investigações criminais, litígio civil, resposta a incidentes internos, e descoberta eletrônica. Características distintivas vs análise "normal":

- **Preservação**: a evidência não pode ser alterada — cadeia de custódia, hashes.
- **Repetibilidade**: outro analista, com os mesmos artefatos, deve chegar aos mesmos achados.
- **Defensabilidade**: o método suporta escrutínio de tribunal ou auditor.

## Princípios

1. **Ordem de volatilidade**: capture primeiro o que some primeiro. Ordem RFC 3227 (do mais volátil ao menos):
   1. Registradores, cache CPU.
   2. Memória RAM.
   3. Estado de rede ativo, tabelas ARP/routing.
   4. Processos em execução.
   5. Arquivos temporários.
   6. Disco.
   7. Logs remotos, backups.
   8. Configuração física / arquitetura.

2. **Documentação contemporânea**: anotar ações, timestamps, ferramentas, versões — enquanto faz, não depois.

3. **Integridade por hash**: SHA-256 (mínimo) de cada imagem/arquivo coletado; hash duplicado em fonte separada; qualquer alteração é detectada.

4. **Cópia bit-a-bit**: trabalhe em cópias; nunca no original. **Write blockers** (hardware ou software) impedem alteração acidental.

5. **Cadeia de custódia**: quem teve posse, quando, para quê. Documentada em formulário assinado.

## Categorias

- **Host forensics** (endpoint): disco, memória, logs, artefatos.
- **Network forensics**: captura de tráfego, NetFlow, metadados.
- **Mobile forensics**: iOS, Android — desafios de criptografia, extração física vs lógica.
- **Cloud forensics**: IaaS (snapshots), SaaS (logs), desafios de jurisdição e acesso.
- **Memory forensics**: dump de RAM, análise offline.
- **Malware forensics**: intersecção com `MALWARE_ANALYSIS.md`.
- **eDiscovery**: descoberta eletrônica em litígio civil; volume gigante, foco em conteúdo.

## Aquisição

### Disco

- **Imagem "raw"** (`dd`, `dcfldd`, `ewfacquire`): bit-a-bit. Formatos: `dd` puro, EWF/E01 (comprimido com metadata e hash), AFF4.
- **Write blocker** entre disco-fonte e máquina forense — hardware (Tableau, WiebeTech) ou software (linux `blockdev --setro`, Windows registry flag).
- **Disco criptografado**: precisa da chave. BitLocker com TPM → precisa acesso autenticado ao sistema. FileVault, LUKS idem. Sem chave → acquisition ainda preserva, mas análise é limitada.
- **Live vs dead acquisition**: dead (máquina desligada) é mais limpo; live pode ser necessária (sistema criptografado ligado, servidor que não pode parar).

### Memória

**Ordem crítica** — RAM some ao desligar.

Ferramentas:
- **Magnet RAM Capture, FTK Imager, winpmem** (Windows).
- **LiME, AVML** (Linux).
- **OSXPMem, MacQuisition** (macOS).

Alguns ativos anti-malware bloqueiam dump — considerar antes.

### Cloud

- **Snapshot de VM** — quando possível e apropriado com dono da conta.
- **Export de logs** — CloudTrail (AWS), Audit Logs (GCP), Activity Logs (Azure).
- **API-based collection** — Graph API (M365), admin SDK (Google Workspace).
- Ferramentas: AWS IR tooling, **Prowler** (para assessment/collection de config), KAPE com plugins cloud.

### Mobile

- **Lógico**: via backup (iTunes/iCloud, Android backup). Limitado.
- **Físico**: imagem do storage direto. Exige bypass de encriptação — Cellebrite, GrayKey (law enforcement). Em Android, *Bootloader Unlock* + chip-off em casos extremos.
- **Artefatos chave**: SQLite databases (mensagens, contatos), plist, keychain (iOS), logs, WhatsApp db, localização.

## Artefatos Importantes

### Windows

- **Registry hives**: SYSTEM, SOFTWARE, SAM, SECURITY, NTUSER.DAT, UsrClass.dat.
  - `Run`/`RunOnce` — persistência.
  - `Shimcache`/`AppCompatCache` — executables que rodaram.
  - `Amcache` — similar, mais granular.
  - `UserAssist` — GUI-launched programs.
  - Evidence of USB connections (`USBSTOR`).
- **Prefetch** (`C:\Windows\Prefetch\*.pf`): programas executados, frequência, último tempo.
- **LNK files** (Recent, Desktop) — arquivos acessados.
- **Jump Lists** — similar.
- **Browser artifacts**: history, cache, cookies, downloads (SQLite).
- **Event Logs** (`.evtx`): Security, System, Application, PowerShell, Sysmon.
- **$MFT** (NTFS Master File Table): timestamps $SI e $FN, presença mesmo após delete.
- **$USNJrnl, $LogFile**: journal de mudanças.
- **Shadow Copies** (VSS): snapshots históricos.
- **Hibernation file** (`hiberfil.sys`) — contém memória do último hibernate.
- **Pagefile** (`pagefile.sys`) — restos de RAM.
- **Windows Timeline (ActivitiesCache.db)**.
- **RDP Cache** — bitmaps de sessões remotas.

### Linux

- **Logs**: `/var/log/` — auth.log, syslog, audit.log (auditd), journalctl.
- **Bash history**: `~/.bash_history`, `.zsh_history`.
- **Cron**: `/etc/crontab`, `/etc/cron.d/`, `crontab -l` por usuário.
- **Systemd units**: `/etc/systemd/`, `~/.config/systemd/user/`.
- **SSH**: `~/.ssh/known_hosts`, `authorized_keys`, `/var/log/auth.log`.
- **utmp/wtmp/btmp**: logins ativos/historico/falhos.
- **/tmp, /dev/shm** — staging comum para atacantes.
- **Package manager logs** (`/var/log/dpkg.log`, `yum.log`).
- **Inodes**: número, timestamps (mtime, atime, ctime, **crtime** em ext4).

### macOS

- **Unified logs** (`log show`): consolidação moderna.
- **FSEvents**: mudanças em filesystem.
- **Spotlight metadata**.
- **QuarantineEventsV2** (`~/Library/Preferences/com.apple.LaunchServices.QuarantineEventsV2`): arquivos baixados e de onde.
- **KnowledgeC.db**: atividade do sistema (uso de apps).

## Timelines

Uma das técnicas mais poderosas — reconstruir cronologia de eventos:

- **Plaso / log2timeline**: ingere múltiplas fontes, produz super-timeline em CSV/ElasticSearch.
- **Timesketch**: frontend web para Plaso.
- **Timelines manuais** em casos pequenos — planilha com coluna de fonte.

Desafios: fusos horários, **timestomp** (atacante muda timestamps), relógios dessincronizados. Sempre correlacionar múltiplas fontes.

## Memory Forensics

Memória contém o que disco nunca teve: processos em execução, injeção, chaves, conexões ativas.

### Volatility (2 e 3)

Framework canônico. Plugins por tipo de investigação:

- `pslist`, `pstree`, `psscan` — processos (via lista, árvore, scan em pool).
- `cmdline`, `cmdscan`, `consoles` — linhas de comando.
- `netstat`, `netscan` — conexões.
- `dlllist`, `ldrmodules` — módulos carregados (ldrmodules detecta DLL injection unlinked).
- `malfind` — heurística para código injetado (PE em região RWX sem arquivo mapeado).
- `apihooks`, `ssdt` — hooks de API/SSDT (rootkit).
- `filescan`, `dumpfiles` — arquivos em memória.
- `hashdump`, `lsadump`, `cachedump` — credenciais (hashes NT).
- `yarascan` — regras YARA sobre RAM.

### Rekall

Concorrente, mais moderno; menos usado hoje.

### MemProcFS

Expõe memória como sistema de arquivos — abordagem única para exploração.

## Disk Forensics

### The Sleuth Kit + Autopsy

- `mmls`, `fsstat`: partições, filesystem.
- `fls`: listing recursivo.
- `icat`, `istat`: extração e metadata por inode.
- **Autopsy**: GUI cross-platform para Sleuth Kit + módulos.

### Carving

Recuperar arquivos apagados ou de espaço não alocado:

- **PhotoRec, Foremost, Scalpel**: detectam headers/footers.
- Limita-se a formatos conhecidos e arquivos não fragmentados.

### Slack space

Espaço entre fim do arquivo e fim do último cluster. Pode conter fragmentos de arquivos anteriores.

### Deleted files

NTFS: entrada MFT marcada como não-utilizada mas presente. Ext4: menos recuperável (metadata zerada). Recovery depende do tempo desde delete e ocupação do disco.

## Network Forensics

- **Full packet capture (FPC)**: ideal mas caro em storage.
- **NetFlow / IPFIX**: metadados (IPs, portas, volume); muito barato, muito útil.
- **DNS logs**: zona de ouro — IOC, DGA, tunneling.
- **Proxy logs**: URL, user-agent, size.
- **Zeek (antigo Bro)**: transforma tráfego em logs ricos.
- **Suricata / Snort**: IDS que também loga. Extrai arquivos de fluxos.
- **Moloch / Arkime**: FPC searchable em escala.
- **Wireshark**: análise manual.

## Ferramentas de Coleta Live

- **KAPE** (Kroll Artifact Parser and Extractor): Windows, muito rápido, alvos pré-definidos.
- **Velociraptor**: client-server, agentless-ish, VQL para queries em escala de frota.
- **GRR Rapid Response**: Google, enterprise scale.
- **TheHive + Cortex**: case management.

## Artefatos Anti-Forense

Atacantes apagam rastros (ver `ANTI_FORENSICS.md`):

- **Log wiping** (Event Logs, syslog).
- **Timestomping** — `touch -t`, tools como `timestomp` da Metasploit.
- **Secure delete** (`sdelete`, `shred`, `srm`) — rewrite overwrites.
- **Fileless malware** — tudo em memória; nada no disco.
- **LOLBins** (Living Off the Land Binaries) — usar binários legítimos (powershell, wmic, certutil, bitsadmin) para reduzir IOCs.

Detecção contra: multiple sources, logs centralizados imutáveis, memory forensics, EDR com telemetria rica, Sysmon bem configurado.

## Hashing e Integridade

- **MD5**: legível histórico; hoje rompido para colisão, mas ainda usado em forense (baixo risco em contexto).
- **SHA-1**: similar.
- **SHA-256**: padrão atual.
- **SHA-3**: alternativa moderna; raramente necessária.
- **Fuzzy hashing** (ssdeep, TLSH, sdhash): detecta similares, não idênticos. Malware, documentos.

## Admissibilidade em Tribunal

Varia por jurisdição; princípios gerais:

- **Autenticidade**: prova de que é o que se diz que é.
- **Integridade**: hash + cadeia de custódia.
- **Relevância**: relação com o caso.
- **Reliability**: método aceito, reproduzível.
- **Best evidence rule**: original quando possível; cópia com documentação de por quê.

Analista forense muitas vezes atua como **perito** — testemunho sob juramento, sujeito a contraditório.

## Certificações e Formação

- **GCFA, GCFE, GNFA, GREM** (SANS/GIAC) — padrão ouro, caras.
- **EnCE** (Guidance/OpenText EnCase).
- **ACE** (AccessData).
- **CHFI** (EC-Council).
- Academic: forensics em cursos de pós em ciência forense.

## Recursos

- **"File System Forensic Analysis"** — Brian Carrier. Canônico para filesystems.
- **"The Art of Memory Forensics"** — Ligh et al.
- **"Windows Forensic Analysis"** — Harlan Carvey.
- **SANS FOR500, FOR508, FOR572** — cursos marcantes.
- **Blogs**: SANS DFIR, dfir.training, Andrea Fortuna.

## Princípios Operacionais

1. **Documente em tempo real**. Memória humana é adversária da cadeia de custódia.
2. **Trabalhe em cópias**. Original intocado.
3. **Validação por hash** em cada etapa.
4. **Não confie em timestamps isolados**. Correlacione sempre.
5. **Respeite a volatilidade**. Memória antes de disco; disco antes de cloud; cloud antes do que amanhã pode ter sido deletado.
6. **Escopo**. Forense completo em cada incidente é caro; focar no que responde à pergunta.
