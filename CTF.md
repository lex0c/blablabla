# CTF — Capture the Flag

**CTF** é uma competição de segurança ofensiva onde participantes resolvem desafios técnicos para encontrar **flags** (tokens como `flag{...}`). É onde muitos profissionais de segurança aprendem, praticam e se mantêm em forma: ambiente legal, contraditório, com feedback imediato.

Complementa e concretiza praticamente: `BINARY_EXPLOITATION.md`, `REVERSE_ENGINEERING.md`, `WEB_SECURITY.md`, `CRYPTO_ADVANCED.md`, `FORENSICS.md`, `STEGANOGRAPHY.md`, `OSINT.md`.

## Formatos

### Jeopardy

Formato mais comum. Desafios independentes em categorias; pontos por solução (dificuldade). Duração: 24-48h. Primeiro solver ganha first-blood bonus em alguns.

### Attack-Defense (A/D)

Equipes têm **mesma infraestrutura vulnerável**. Pontuam por: (1) manter serviços up, (2) patchar vulns contra outros, (3) roubar flags alheias. Realístico, tenso. DEF CON CTF é o padrão ouro.

### King of the Hill

Equipes competem por controlar um servidor. Mantendo controle → pontos por intervalo.

### Mixed / Boot2Root / Pentest-like

Menos formal — HackTheBox, TryHackMe, Proving Grounds, VulnHub. Uma máquina a comprometer da foot a root. Não é CTF tradicional mas mesma natureza.

## Categorias Padrão

### Web

Desafios em aplicações web vulneráveis. Vetores típicos (ver `WEB_SECURITY.md`):

- **SQLi** com filter evasion.
- **XSS** para roubar cookie/admin session.
- **SSRF** → cloud metadata → RCE.
- **SSTI** → RCE via template.
- **JWT** manipulation (`alg:none`, key confusion).
- **Prototype pollution** (JS/Node).
- **HTTP smuggling, request smuggling**.
- **Business logic**: race conditions, sequence bugs.

Ferramentas: Burp Suite, curl, Python requests, ffuf, sqlmap.

### Pwn / Binary Exploitation

Exploração de binário (geralmente Linux x86-64). Vetores (ver `BINARY_EXPLOITATION.md`):

- Stack buffer overflow → ret2libc / ROP.
- Format string.
- Heap: UAF, House of ..., tcache poisoning.
- Integer overflow.
- Kernel exploitation em dificuldades avançadas.

Ferramentas: **pwntools** (obrigatória), gdb + pwndbg/gef, Ghidra, ROPgadget.

### Reverse Engineering

Entender binário para achar flag. Crackmes, malware-like challenges.

- Estática: Ghidra/IDA.
- Dinâmica: gdb, strace.
- Simbólica: **angr** para desafios com branches complexos.
- Decompilar bytecode (Java, .NET, Python). Ver `REVERSE_ENGINEERING.md`.

### Crypto

Errors em aplicação de primitivas:

- **XOR** com chave curta (padrão de chars frequente revela).
- **RSA com e pequeno** (cube root), **RSA common modulus**, **Wiener attack**, **Håstad broadcast**.
- **ECDSA com nonce reusado** → extrair chave.
- **AES-ECB** revelando estrutura.
- **Padding oracle**.
- **Length extension** em MD5/SHA-1.
- **CBC bit flipping**.
- **LLL / Coppersmith** em problemas de lattice.

Ferramentas: Python (`gmpy2`, `pycryptodome`), SageMath (algebra computacional; indispensável em crypto avançado), RsaCtfTool, alpertron calculators.

### Forensics

Artefatos a analisar — imagens de memória, PCAPs, filesystems, arquivos corrompidos.

- Wireshark filtering de PCAP.
- **Volatility** em dumps de memória.
- `binwalk`, `foremost` em imagens.
- Reconstrução de arquivos de chunks/streams.
- Ver `FORENSICS.md`.

### Stego

Ver `STEGANOGRAPHY.md`. Ferramentas: `stegsolve`, `zsteg`, `stegseek`, `exiftool`, `binwalk`.

### OSINT

Recon público. Fotos de satélite, geoguessr, username search, reverse image. Trace Labs CTFs usam missing persons reais (impacto concreto).

### Misc

Categoria abrigando o resto — programação, trivia, escape rooms, hardware, redes sociais.

### Hardware / IoT / Radio

Em CTFs maiores (DEF CON): firmware, UART, JTAG, RF com SDR. Ver `HARDWARE_ATTACKS.md`, `IOT_FIRMWARE_SECURITY.md`.

## Fluxo de Resolução

1. **Leia o desafio inteiro**. Cada palavra pode ser pista.
2. **Analise o artefato** fornecido. Tipo, tamanho, entropia, conteúdo óbvio.
3. **Faça uma hipótese**. "É SQLi?" "É XOR?" "É LFI?"
4. **Teste rápido**. Input pequeno; observe comportamento.
5. **Iterate**.
6. **Pulo para outro** quando preso — CTFs são timed; custo de afundar é alto.

## Técnicas de Mindset

- **Nada é acidente** no desafio. Arquivo com nome estranho, parâmetro extra, string aparentemente aleatória — tudo é projetado.
- **Leia write-ups** depois. É onde se aprende — como outros pensaram.
- **Automatize o repetitivo**. Scripts que já usou.
- **Template de scripts** (pwntools boilerplate, request Burp template) reduz fricção.
- **Notas compartilhadas em equipe** durante competição — evitar duplicação.
- **Don't rabbit-hole**. Se 2h sem progresso, outro desafio.

## Stack do Competidor

### Ambiente

- **Kali/Parrot Linux** em VM ou host dedicado.
- **VS Code + Python + tmux + zsh + Ghidra** — setup comum.
- **Burp Community/Pro**.
- **pwntools** sempre.
- **Scripts pessoais** em dotfiles.

### Ferramentas por categoria

| Categoria | Ferramentas |
|---|---|
| Web | Burp, ffuf, sqlmap, XSStrike, gobuster |
| Pwn | pwntools, gdb+pwndbg, ROPgadget, one_gadget, angr |
| Rev | Ghidra, IDA Free, radare2, jadx, dnSpy |
| Crypto | SageMath, pycryptodome, RsaCtfTool, CyberChef |
| Forensics | Volatility, binwalk, Wireshark, exiftool, aperi'solve |
| Stego | stegsolve, zsteg, stegseek, exiftool, aperi'solve |
| OSINT | sherlock, reverse image, theHarvester, Maltego |
| Misc | CyberChef (swiss army knife para encoding) |

## Plataformas

### Online contínuas (aprendizado)

- **HackTheBox**: máquinas + academy + CTFs.
- **TryHackMe**: didático, gradação em rooms.
- **PortSwigger Web Security Academy**: referência em web; gratuito.
- **pwn.college**: pwn, rev, crypto. Curso da ASU Prof. Yan Shoshitaishvili.
- **Root-Me**: francês, vasto.
- **CryptoHack**: crypto exclusivamente, excelente.
- **PicoCTF** (Carnegie Mellon): entrada, kids-friendly mas excelente.
- **OverTheWire**: wargames clássicos (Bandit para iniciantes).
- **RingZer0**, **VulnHub** (VMs para baixar).
- **crackmes.one**: reverse engineering dedicado.

### Eventos ao vivo

- **DEF CON CTF** (EUA, ápice): qualifier + finals em Las Vegas.
- **PlaidCTF** (CMU).
- **Google CTF**, **Meta CTF**, **BSides CTFs**.
- **Flare-On** (Mandiant): anual de reverse engineering, gratuito, cert no fim.
- **TyphoonCon, HITB, InsomniHack**: internacionais.
- **BR**: **H2HC CTF, Pwn2Win CTF** (Brasil Top Crew / GoN).
- **ECSC** (European Cyber Security Challenge) — por país.

### Rankings

- **CTFtime.org**: calendário, ranking global de times, writeups.

## Community

- **CTFtime writeups**: ler writeups pós-evento é onde 50% do aprendizado acontece.
- **Discord/Matrix** de times: PPP, Dragon Sector, ID-10-T, TWN48 (top globais).
- **BR**: discord brasileiros (FireShell, Galoinhas, RATdroid), comunidade H2HC.

## Monte Carlo da Construção de Time

- **Diversidade**: pwn, web, crypto, rev, forensics — cada um precisa de especialista.
- **Coordenação**: Discord + Google Doc + Git para artifacts.
- **Assinatura por canal** por categoria: "estou no pwn #3" evita conflito.
- **Revezamento**: não existe CTF de 48h sem dormir; divida turnos.

## Ética e Aspectos Legais

CTF **é legal por design** — targets autorizados, controlados.

**Cuidados**:

- **Não saia do escopo**. Infra do organizador ≠ alvo.
- **Não ataque outros times** em Jeopardy (é AD que tem ataque mútuo; diferente modelo).
- **Não exfiltrar** dados de máquinas que não deveriam (exceto AD com regra explícita).
- **Respeite rate limits**. Muito denial of service prejudica todos.
- **Não use conhecimento fora do CTF** em alvos sem autorização.

Frequentemente CTFs têm ToS explícitos — ler antes.

## Carreira

CTF é **credencial informal** respeitada. Writeups públicos + participação em time conhecido = portfólio forte para vagas em offensive security. Pode substituir parcialmente certificações em interviews técnicos bons.

## Princípios

1. **Leia tudo**. Cada desafio é desenhado; todo detalhe importa.
2. **Papel e caneta funcionam**. Crypto não se resolve só no teclado.
3. **Automatize o que vai repetir**. Script hoje, economia em futuros.
4. **Write-ups pós**: escrever consolida aprendizado.
5. **Falhe público, pergunte em comunidade**. Ego é freio.
6. **CTF treina mas não substitui**. Realidade tem escopo cinza, stakeholder difícil, dinheiro real — CTF remove essas frictions deliberadamente.
7. **Divirta-se**. Quando vira obrigação, rende menos. CTF é esporte de mente.
