# OpSec — Operational Security

**OpSec** é a disciplina de proteger **informação sensível** de observação adversária durante operações — sejam elas ofensivas (red team, pesquisa), defensivas (investigação, IR), ou pessoais (jornalistas, ativistas em ambientes hostis). A origem do termo é militar (Vietnã, EUA anos 60), mas as práticas são universais em qualquer contexto onde o adversário observa.

Não é paranoia generalizada — é **análise de ameaça + disciplina consistente**.

## Processo OpSec (5 Passos, CIA/US Military)

1. **Identificar informação crítica**: o que, se vazado, prejudica a operação?
2. **Analisar ameaças**: quem pode estar interessado? capacidades?
3. **Analisar vulnerabilidades**: como essa informação pode vazar?
4. **Avaliar risco**: impacto × probabilidade.
5. **Aplicar contramedidas**: focar onde o ROI é maior; nem tudo se protege.

Esse ciclo é repetido; não é evento único.

## Modelo de Ameaça Individual

Antes de escolher ferramentas, responda:

- **Quem é o adversário?** Estado, crime organizado, concorrente, ex-parceiro, troll, nenhum em particular?
- **Quais capacidades?** Legal (subpoena), técnica (NSA-like), social (engenharia social)?
- **O que quer proteger?** Identidade, localização, comunicações, finances, dados profissionais?
- **Consequência de falha?** Constrangimento, multa, prisão, morte?

A resposta muda tudo. "Seu vizinho curioso" e "estado autoritário" exigem controles em ordens de grandeza distintas.

## Compartimentalização

Princípio central: separar identidades/contextos/dados para que comprometimento em um não propague.

### Identidade

- **Email separado por propósito**. Pessoal ≠ profissional ≠ pesquisa ≠ atividade sensível.
- **Usernames distintos, não correlacionáveis**. Ferramentas de OSINT (ver `OSINT.md`) procuram reutilização.
- **Contas de pesquisa** descartáveis para recon.
- **Autenticação separada** — MFA ≠ entre contas. YubiKey dedicada por identidade, ou passkeys por domínio.

### Dispositivos

- **VM dedicada** para pesquisa em malware, darkweb, investigation.
- **Laptop separado** para contextos muito sensíveis.
- **Air-gapped** para dado crítico (chave GPG, cold wallet cripto).

### Rede

- **VPN diferente por persona**. Cuidado com padrões — VPN sempre mesmo exit = correlação.
- **Tor** para navegação anônima; Tor Browser isolado.
- **Tails OS**: distro live; sem persistência; Tor-first. Para uso sensível pontual.
- **Qubes OS**: isolamento por Xen VMs, cada tipo de atividade em VM.

### Dado

- **Criptografia em repouso**: LUKS (Linux), FileVault (Mac), BitLocker (Windows), VeraCrypt.
- **Passwords managers separados** para work/personal.
- **Backup criptografado offline** (restic/borg + chave em keep-safe).

## Comunicações

### Mensageria

- **Signal**: padrão de ouro para uso civil. E2EE, metadados mínimos, disappearing messages.
- **SimpleX Chat, Session**: alternativas sem identificador persistente (sem número de telefone).
- **Threema**: suíço, paid, permite uso sem phone.
- **WhatsApp**: E2EE (Signal protocol), mas metadata e backup em cloud são vetores.
- **Telegram**: **não é E2EE por default**. Apenas "Secret Chats" são. Não confie em Telegram para sensível.

### Email

- **PGP**: funciona, é feio, metadados expostos (subject, from, to). Proton, Tutanota integram.
- **Tutanota, ProtonMail**: criptografia de conteúdo; metadados ainda disponíveis ao provider.
- **Email efêmero** (mailinator, 10minutemail, Guerrilla): signups descartáveis, mas cuidado — conteúdos são públicos em alguns.

### Reuniões

- **Signal** (chamadas), **Jitsi Meet** (web, self-hostable), **Element/Matrix** (federado, E2EE opcional).
- **Zoom, Teams, Meet**: não são E2EE por default (Zoom tem modo E2EE explícito, desativa recurso).

## Metadados

**O conteúdo pode estar cifrado; metadado é frequentemente onde o adversário lê a história.**

- **Email**: remetente, destinatário, assunto, timestamps — não protegidos por PGP.
- **Foto**: EXIF com GPS, modelo do device, timestamp. `exiftool -all= file.jpg` remove.
- **Documento Office / PDF**: autor, última edição, caminhos internos, versões anteriores (track changes). **mat2** limpa.
- **HTTP**: IP, User-Agent, Referer.
- **Chamadas/DNS**: padrão de comunicação revela relações mesmo com conteúdo E2EE. Signal introduziu **Sealed Sender** e **Private Contact Discovery** para mitigar.

## Fingerprinting

Adversário identifica você não por uma coisa, mas por combinação:

### Browser

- **User agent, screen size, fonts, canvas fingerprint, WebGL**: dezenas de bits → identificação única.
- **Tor Browser**: forçadamente padronizado — todos parecem iguais.
- **amiunique.org** demonstra.
- Extensões **úteis + raras** te identificam paradoxalmente; todo mundo instala o mesmo ad-blocker, mas combinação atípica é sinal.

### Estilo

- **Writing style (stylometry)**: palavras, pontuação, idiossincrasias. Ferramentas de linguistics forenses identificam. Para anonimato total, edite próprio texto radicalmente ou use intermediário (tradução ida-e-volta, LLM para paraphrase).

### Timing

- **Horário de postagem/login** revela timezone; hábitos revelam rotina.

### Hardware

- **MAC address** em Wi-Fi: MAC randomization é comum mas pode ser revertida.
- **Device ID** em apps: persistência mesmo apagando app.
- **Typing biometrics**: cadência, pressão.

## Erros Recorrentes em OpSec

Atacantes (e defensores) caem nos mesmos padrões:

1. **Reuso de username**. Pesquisa OSINT adora.
2. **Compartilhamento entre personas** — logar em conta errada pelo hábito.
3. **Ligação via metadado**: mesma foto de perfil em duas contas; timestamp de criação correlacionado; mesma resposta a pergunta de segurança.
4. **Burner que vira regular**. Telefone/email "só para isso" vira habitual.
5. **Revelar localização em background**: janela com documento em português, ventilador em 60Hz (= 110V, Americas), sombras implicando latitude, sotaque em voz.
6. **Dizer mais do que deve**: flexing em foruns leva a atribuição. Muitos criminosos caíram por vaidade.
7. **Ignorar plataformas que já te conheciam**: Google "anonymous" é Google logado.
8. **Ferramentas de proteção sem prática**: PGP configurado errado, Signal com backup em cloud sem criptografar, VPN com DNS leak.

## OpSec Técnica

### Checagem pré-operação

- **DNS leak test** com VPN ativo.
- **WebRTC leak**: JS revela IP real mesmo com VPN. Desabilitar em browser ou confiar em Tor.
- **IPv6 leaks**: VPN só para IPv4; IPv6 sai direto.
- **Killswitch** no VPN: se cair, tráfego não vaza.

### Durante

- **Um contexto por sessão**. Não misture VMs/browsers.
- **Logs locais**: clipboard, histórico de shell, caches. Desabilitar history em shell para tasks sensíveis (`unset HISTFILE`, `HISTCONTROL=ignorespace`).
- **No relato em tempo real**. Post no Twitter enquanto opera = IOC.

### Pós

- **Sanitizar** artefatos (snapshot revert, shred files).
- **Rotacionar credenciais** usadas na op.
- **Destruir dispositivo** se queima total (operações de estado/crime).

## Signal — Detalhes Úteis

- **Phone number → username** (Signal 2024+): abstrai. Ainda assim, o número vinculado existe.
- **Disappearing messages** por thread.
- **Screen security** no iOS (desabilita screenshot).
- **Verify safety numbers** — confere chave out-of-band para evitar MITM.
- **Molly** (fork) com hardening extra em Android.

## Tor — Detalhes Úteis

- **Entry + middle + exit** — três nós. ISP vê que você usa Tor, não o destino. Exit node vê destino, não você.
- **Exit nodes maliciosos** fazem MITM em HTTP (não HTTPS). Sempre HTTPS mesmo via Tor.
- **Hidden services (.onion)**: tráfego nunca sai da rede Tor. Sem exit.
- **Bridges e pluggable transports** (obfs4, snowflake): ofuscam que é Tor; passam por DPI de países repressivos.
- **Ataque de correlação por timing** é o calcanhar-de-aquiles; entidades observando grandes partes da Internet podem correlacionar.

## Operações de Field

- **Viagens** para locais hostis: **burner phone + burner laptop**, dados essenciais sincronizados só no necessário. Na volta, reimagem.
- **Alfândega pode forçar unlock de devices** — esperar isso; não leve o que não pode revelar.
- **Câmeras onipresentes**: FR pode estar em uso; fotos de passaporte em checagem; padrões de deslocamento em CFTV conectado.
- **Pay com cash, carros alugados em nome diferente** (não sempre possível/legal).

## Estudos de Caso

- **Silk Road / Ross Ulbricht**: username `altoid` usado em forums de maconha e em uma questão técnica em Stack Overflow. Pivot OSINT → identidade real.
- **LulzSec / Sabu**: logou em IRC sem Tor uma vez.
- **Conti leaks**: vazamento interno de chats após stance pró-Rússia; OpSec interna inexistente.
- **Hutchins / MalwareTech**: pesquisador de cyber preso em viagem a DEF CON — OpSec legal (evitar fronteira hostil) é parte.

Lição uniforme: um **único** descuido frequentemente é suficiente.

## Ferramentas e Recursos

- **Tor, Tails, Qubes, Whonix**.
- **Signal, SimpleX, Session**.
- **Veracrypt, Cryptomator** para volumes/backup.
- **Bitwarden, 1Password, KeePassXC**.
- **mat2** (metadata removal).
- **Exiftool** (leitura e strip).
- **Syncthing criptografado** para sync sem cloud central.
- **Molly, Element (Matrix)**.

Leituras:

- **EFF Surveillance Self-Defense** (ssd.eff.org).
- **The Tails Handbook**.
- **Security Planner** (Citizen Lab).
- **micahflee.com** — posts de jornalismo investigativo e OpSec.

## Princípios

1. **Threat model primeiro**. Sem saber contra quem, toda proteção é teatro ou exagero.
2. **Compartimentalize**. Uma conta, um device, uma rede, um propósito.
3. **Pratique sob estresse**. OpSec lembrada só em calma falha na pressão. Rotinas automatizadas.
4. **Menos é mais**. Menos metadata, menos identidades, menos contatos, menos risco.
5. **Reavalie**. Contexto muda; ameaça muda. Revise trimestralmente.
6. **Aceite trade-offs**. Perfeição = paralisia. 80% de OpSec consistente > 100% intermitente.
7. **Documente em canal separado**. Your plan itself é informação a proteger.
