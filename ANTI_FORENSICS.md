# Anti-Forense

**Anti-forense** são técnicas que atacantes (ou pessoas legítimamente evitando vigilância) usam para dificultar **análise forense** (ver `FORENSICS.md`). Este arquivo é escrito para **defensor** — entender o que atacantes fazem é essencial para detectar, preservar evidência apesar das tentativas e projetar sistemas mais resilientes.

Ver também `FORENSICS.md`, `MALWARE_ANALYSIS.md`, `OPSEC.md`.

## Categorias (Harlan Carvey / Marcus Rogers)

1. **Data hiding**: mascarar onde os dados estão.
2. **Artifact wiping**: apagar artefatos deixados.
3. **Trail obfuscation**: confundir linha temporal.
4. **Attack against forensic tools**: explorar bugs nas ferramentas forenses.

## Data Hiding

Esconder em vez de apagar. Dificulta descoberta; preservação é possível se encontrado.

### Esteganografia

Ver `STEGANOGRAPHY.md`. Dados em imagens, áudio, vídeo. Cover traffic em protocolos.

### Volumes Cifrados Com Negação Plausível

- **VeraCrypt hidden volume**: dois passwords, dois volumes. Um password revela "inócuo"; outro revela o real. Sem como provar que o segundo existe.
- **Rubberhose filesystem**: concept anterior.

### Slack Space e Metadata

- **Slack space**: espaço entre fim do arquivo e fim do cluster. `bmap`, `slacker` escrevem lá — tools de alto nível não vêem.
- **ADS (Alternate Data Streams)** em NTFS: `file.txt:hidden.exe` — stream não aparece em `dir` tradicional.
- **Meta fields em EXIF/ID3**: esconder texto em tags raras.

### Filesystems "exóticos"

Container de arquivo em formato raro ou não-montável pelo analista. Força decisão entre tempo em RE de filesystem vs deixar passar.

### Disk encryption

FDE (LUKS, BitLocker, FileVault, VeraCrypt). Se dispositivo apreendido desligado com chave não disponível, dados efetivamente inacessíveis.

**Cuidados para atacante**: tudo que não está cifrado at rest (hibernation file, swap com chaves residuais, cold boot).

## Artifact Wiping

Apagar evidência. Estratégias:

### Secure delete

- **shred** (Linux), **sdelete** (Sysinternals), **srm**: sobreescrevem múltiplas vezes. Problema moderno: wear leveling em SSDs torna overwrite não-determinístico.
- **SSD**: melhor usar **ATA Secure Erase** / **NVMe Format**. Mesmo assim, blocos pode ficar em GC. Com FDE + descartar chave, efetivamente apaga.
- **HDD magnético**: wipe é mais confiável. Um pass de zeros geralmente basta (DoD 5220.22-M é overengineered pelo overhead).

### Log wiping

- **Windows**: `wevtutil cl Security`, `Clear-EventLog`, via PowerShell/WMI.
- **Linux**: `> /var/log/auth.log`, truncar syslog.
- **Artifacts residuais**: mesmo após clear, entradas podem permanecer (Windows Event ID 1102 é gerado quando log é limpo — ironicamente deixa evidência).
- **Anti-defesa**: logs remotos **imutáveis** (Graylog/Elastic dedicado com WORM) são não-limpáveis por atacante em host.

### Prefetch e Recent Files

Atacantes deletam `C:\Windows\Prefetch\*.pf`, `RecentDocs` registry keys, `UserAssist`, jump lists.

### Browser history e caches

Privacy mode + clear history. Dados residuais em `hiberfil.sys`, `pagefile.sys`.

### USB connection artifacts

`USBSTOR` registry, setupapi.dev.log — evidências de pendrives conectados. Atacante que quer esconder remove.

### Memory residual

Após uso, memória pode ficar com chaves/payloads. Explicitly zero-out em código sério. Porém, em runtime comprometido, atacante não controla.

## Trail Obfuscation

Confundir análise — frequentemente mais efetivo que apagar, pois nada "sumir" é suspeito.

### Timestomping

Manipular timestamps de arquivos (MAC times em NTFS; mtime/ctime/atime em *nix).

- **Metasploit Meterpreter `timestomp`**, **SetMACE**, **PowerShell**.
- Modificar para **antes** da data da invasão — arquivo parece antigo.
- Modificar para **exatamente igual** a arquivo ao lado — indistinguível.

**Contramedida**:
- **$MFT $SI vs $FN**: Windows NTFS tem timestamps em duas estruturas. `timestomp` tradicional modifica só $SI, deixando $FN original. Ferramentas forenses comparam. Modernos atacantes tocam ambos.
- **USN Journal, $LogFile**: registram mudanças incluindo timestamps anteriores.
- **Logs de backup / replicação** fora do host.
- **Correlacionar** com outros eventos temporais (logs externos).

### False flags

Plantar artefatos que apontam para outro atacante/grupo. Strings em outro idioma, PDB paths de outro grupo, TTPs imitadas.

### Process hollowing / masquerading

Malware renomeia executável para `svchost.exe`, roda de `C:\Windows\System32\...`; procurador rápido confia.

- Detecção: assinatura digital, path exato esperado, hash conhecido.

### LOLBins (Living Off the Land Binaries)

Usar binários legítimos pré-existentes (certutil, bitsadmin, powershell, wmic, msbuild, regsvr32, mshta, rundll32) em vez de malware dedicado.

- **lolbas-project.github.io** cataloga.
- Baixa IOC; alta dificuldade em detectar sem baseline de uso normal.
- Detecção comportamental (DE) é a melhor resposta.

### Fileless

Malware inteiramente em memória. Registry, WMI, in-memory PowerShell. Nada no disco → forense estática quase vazia; memory forensics é essencial.

## Anti-Análise Dinâmica

Mais coberto em `REVERSE_ENGINEERING.md` e `MALWARE_ANALYSIS.md`:

- Anti-debug, anti-VM, anti-sandbox.
- Detection de Frida, strace, tcpdump.
- Sleep long.
- Trigger-based (só ativa em condição específica).

Dificultam RE; parceiros de anti-forense pela mesma família.

## Attacks Against Forensic Tools

Explorar bugs ou assumptions das ferramentas:

- **Zip bomb, PDF bomb**: expande a TB em análise automática. Anti-scanner.
- **Malformed PE** que crasheia Ghidra ou IDA.
- **File systems corrompidos** que causam OOM em `fls`/Sleuthkit.
- **Encoding quirks** (path com unicode confuso).

Pesquisa: aggregated em papers / DEF CON talks. Defensor deve usar múltiplas tools em cross-check.

## Anti-Recovery

Depois de wipe, impedir recovery:

- **Overwrite múltiplo** em HDD (pouco necessário hoje).
- **Physical destruction**: furar, degaussar, moer, incinerar. Padrão em organizações com dado classificado. NIST 800-88 Rev 1 é a referência.
- **Crypto-erase**: sistema onde **chave** é a key — apaga chave, dados efetivamente sumiram (self-encrypting drives, FDE).

## Anti-Forense em Cloud

Atacantes em cloud têm trilha em CloudTrail/Audit Logs — difícil apagar como acesso normal.

- **Disable logging** (via chamada API que é ela mesma logada — catch-22).
- **Delete resources** depois de abuso.
- **Abuse de assumerole cross-account** obfusca quem realmente executou.

Defensivo: **centralizar logs em conta segregada** com permissões apenas de write. Atacante em conta exposta não consegue deletar.

## Anti-Forense em Memória

Memory forensics (Volatility) pode extrair:

- Malware não escondeu presença.
- Unlinked DLLs (DKOM).
- Hooks SSDT.
- Strings de RAM.

Atacantes respondem com:

- **Rootkits kernel** que fazem memory fooling (DKOM, SSDT hooks antigos, hypervisor-based).
- **Self-deletion** na primeira execução — só fica em memória.
- **Hollow containers** em processos legítimos.

## Detecção de Anti-Forense

Paradoxalmente, anti-forense deixa suas próprias pegadas:

- **Event Log 1102**: log foi limpo.
- **Gaps na timeline**: falta de logs inusitada.
- **$MFT entries sem $FN timestamp**: timestomping $SI-only.
- **Executables em local atípico**: `C:\Users\...\AppData\Roaming\svchost.exe`.
- **Conexões out quando processo não deveria** (host-based firewall rules violadas).
- **Processos sem parent legítimo**.
- **Usage incomum de LOLBins**.

Detection engineering (ver `DETECTION_ENGINEERING.md`) moderno identifica **tentativas** de anti-forense como sinal de atividade maliciosa.

## Defesa por Design

Para preservar capacidade forense apesar de atacante competente:

1. **Log centralizado imutável**. WORM storage ou cloud com write-only role.
2. **EDR** com telemetria rica que não é apagável por admin local.
3. **Sysmon** bem configurado (Event ID para process, network, image load).
4. **Backup de logs fora do host** em tempo real.
5. **Baselining** de comportamento — desvios (ex.: wipe mass de Event Logs) alertam.
6. **MFA em ações destrutivas** (delete bucket, disable logging).
7. **Alertas em ações de anti-forense**: "log foi limpo" = P1.
8. **Retenção**: longa o suficiente para descobrir incidentes semi-silenciosos.
9. **File integrity monitoring** (Tripwire, Wazuh).
10. **Timestamp externo confiável**: NTP e correlação com logs externos.

## Aspectos Éticos e Legais

Anti-forense tem usos legítimos:

- **Jornalistas, dissidentes** em regimes hostis: proteger fontes.
- **Proteção de privacidade pessoal**.
- **Destruição de dados conforme compliance** (LGPD/GDPR right to erasure).

E ilegítimos:

- **Obstrução de justiça** — wipe em dispositivo sob investigação é crime em várias jurisdições.
- **Concealment of evidence** em investigação corporativa.

Análise forense legítima opera sob **cadeia de custódia** (ver `FORENSICS.md`). Quebra de custódia ou tampering invalida evidência em tribunal, com consequência penal para quem tenta.

## Recursos

- **Harlan Carvey — "Windows Forensic Analysis"** (tem capítulos sobre anti-forensics).
- **"The Art of Memory Forensics"**.
- **"Counter-Forensics: Attacking the Hard-Boiled Detective"** — papers clássicos.
- **NIST 800-88 Rev 1** — media sanitization.
- **DEF CON, FIRST** — talks sobre estado da arte em AF/AF-detection.

## Princípios (Defensor)

1. **Assuma anti-forense**. Sua infra e procedure precisam resistir.
2. **Redundância** em telemetria. Log em host + cloud + SIEM.
3. **Imutabilidade** em storage de evidência.
4. **Response time < tempo de anti-forense eficaz**. Atacantes precisam de minutos a horas; detectar antes.
5. **Preserve rápido**. Em incidente, imagem antes de tudo.
6. **Múltiplas ferramentas cross-check**. Bug em uma, outra pega.
7. **Anti-forense é IOC**. Tentativa detectável é sinal de adversário.
