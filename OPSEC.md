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
- **Whonix**: duas VMs — Gateway (roda Tor) + Workstation (roda apps). App comprometido na Workstation **não vê o IP real** porque só tem rota pela Gateway. Diferente de Tails (live, amnésico) — Whonix é persistente e roda em cima de hypervisor (KVM/VirtualBox/Qubes). Use Whonix quando precisa persistência + isolamento forte; Tails quando precisa não deixar rastro no hardware.
- **Tails**: live USB amnésico, tudo via Tor por default, MAC spoofing ligado, `mat2` integrado. Volume persistente opcional em LUKS — mas persistência **contradiz** a ideia de amnésia, use com intenção.
- **Qubes OS**: compartimentalização por Xen VMs (qubes) por propósito — work, personal, vault, dvm (disposable), sys-net, sys-firewall. Combinável com Whonix (`sys-whonix`) para Tor sistêmico. Cost de aprendizado alto; paga em threat models sérios.

### Rede

- **VPN diferente por persona**. Cuidado com padrões — VPN sempre mesmo exit = correlação.
- **Política de logs** do VPN é marketing até ser auditada ou judicialmente testada (Mullvad, IVPN têm histórico mais defensável que providers grandes).
- **Multi-hop** (VPN→VPN em jurisdições distintas) reduz risco de single-point, **não** resolve correlation se ambos provedores estão sob mesma pressão.
- **Killswitch** é obrigatório — sem ele, reconexão vaza IP real.
- **Tor** para navegação anônima — profundidade em `TOR.md`.

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

## Autenticação

MFA nem todo é igual. Ordem de resistência a phishing (melhor → pior):

1. **Hardware keys FIDO2/WebAuthn** (YubiKey, SoloKey, passkey em secure enclave). Autentica o **domínio** criptograficamente; phishing impossível por design. Fallback obrigatório — tenha 2 chaves, uma offline.
2. **Passkeys** sincronizadas (Apple, Google, 1Password). Phishing-resistant, mas herdam confiança do provider de sync.
3. **TOTP** (Aegis, 2FAS, KeePassXC). Vulnerável a phishing em tempo real (atacante repassa código em <30s). Backup do seed é crítico — exporte e guarde em cofre.
4. **Push notification** (Duo, Microsoft Authenticator). Vulnerável a MFA fatigue attacks — atacante dispara pushes até usuário aceitar por erro.
5. **SMS / voz**. **Hostil**. SIM swap (atacante convence operadora a portar seu número) deanonimiza e toma contas. Nunca use SMS para conta de alto valor; se obrigado, isole o número num eSIM dedicado e desative portabilidade com a operadora.

Outros pontos:

- **Recovery codes** são segredo de mesmo nível que a senha. Cofre cripto, não screenshot em Google Drive.
- **Recovery phrases (BIP39)** de wallets: papel ou metal, nunca digital, nunca foto. **Shamir Secret Sharing** (SLIP39, ou `ssss`) distribui em N-de-M shares para herança/resiliência sem ponto único.
- **Gerenciadores de senhas**: Bitwarden, 1Password, KeePassXC. Master password longa, MFA no próprio manager. Não use navegador como gerenciador primário.
- **Breach check** recorrente — HIBP (haveibeenpwned.com), integrado em Bitwarden/1Password. Senhas em breach rotacionam imediatamente.
- **Passkeys > senhas** onde suportado. A transição é lenta porque vendors divergem, mas passkey phishing-resistant é ganho real.

## Dispositivo e boot chain

Criptografia de disco (LUKS/FileVault/BitLocker) só protege **com a máquina desligada**. Em uso, ataques possíveis:

- **Evil maid**: adversário com acesso físico breve modifica bootloader/firmware para capturar senha no próximo boot. Mitigações: **Heads** (coreboot-based firmware com attestation via TPM), **PureBoot** (System76/Purism), Secure Boot com próprias chaves, tamper-evident seals físicos.
- **Cold boot**: RAM mantém chaves por segundos após desligar; adversário congela DIMM e lê. Mitigação: fechamento com poweroff (não suspend), memory scrubbing em shutdown.
- **DMA attacks**: Thunderbolt/FireWire/PCIe externo pode ler RAM direto sem CPU. Mitigação: **IOMMU** (VT-d/AMD-Vi) + desativar Thunderbolt em BIOS ou kernel, `kernel_lockdown`.
- **Firmware/UEFI implants**: LoJax, MoonBounce. Persistem após reinstall. Mitigação: attestation de firmware, boot measurement no TPM, Heads.
- **TPM sealing**: chave de disco selada a medidas PCR do boot; modificação do bootloader quebra sealing e disco fica inacessível. Requer config deliberada (`systemd-cryptenroll --tpm2-device=`).

Suspend vs hibernate vs poweroff:

- **Suspend**: chaves em RAM, disco destravado. Laptop fechado ≠ seguro.
- **Hibernate**: estado pra disco cifrado se swap está em LUKS. Melhor que suspend em viagem.
- **Poweroff**: único estado verdadeiramente trancado.

Em travessia de fronteira, **sempre poweroff**. Suspend é prenda entregue.

## Metadados

**O conteúdo pode estar cifrado; metadado é frequentemente onde o adversário lê a história.**

- **Email**: remetente, destinatário, assunto, timestamps — não protegidos por PGP.
- **Foto**: EXIF com GPS, modelo do device, timestamp. `exiftool -all= file.jpg` remove.
- **Documento Office / PDF**: autor, última edição, caminhos internos, versões anteriores (track changes). **mat2** limpa.
- **HTTP**: IP, User-Agent, Referer.
- **Chamadas/DNS**: padrão de comunicação revela relações mesmo com conteúdo E2EE. Signal introduziu **Sealed Sender** e **Private Contact Discovery** para mitigar.

### Documentos vazados

Impressoras a cores imprimem **Machine Identification Code** (MIC, yellow dots) codificando número de série + timestamp em padrão quase invisível. Caso **Reality Winner** (2017): documento vazado ao *The Intercept* foi escaneado com dots visíveis → NSA identificou impressora → funcionária presa em 3 dias. Contramedidas: imprimir em preto-e-branco com toner sem dots, OCR + reimpressão em outro device, ou passar fonte por conversão radical (foto de tela → OCR).

Para leaks em geral:

- **Reescreva em canal seu**; não repasse o PDF original — ele tem autor, revisões, MIC dots, UUIDs em fontes.
- **Screenshots têm metadata da tool** (Snipping Tool, macOS screencapture) e às vezes timestamps de compilação do SO.
- **Track changes e comentários** em `.docx` e `.pdf` sobrevivem a "salvar como PDF". Abra no LibreOffice e valide.
- **Watermarking invisível** (grammar-level, spacing-level) é usado por orgs que distribuem docs para identificar quem vazou. Assuma que existe.

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

### Geolocalização por conteúdo

Estilo Bellingcat / GeoGuessr — adversário identifica lugar em foto/vídeo sem EXIF:

- **Sombras** → latitude + hora. Ângulo solar + direção = triangulação.
- **Arquitetura, tomadas, placas, faixas de pedestre, vegetação, céu** → país/região.
- **Sons ambiente** em vídeo → idioma, chamado à oração, buzinas regionais, cigarras (espécie → biogeografia).
- **Reflexos** em janelas, olhos, objetos metálicos — revelam o que está fora do enquadramento.
- **Background ao vivo** em videoconferência: janela mostra prédio reconhecível, livro em língua específica, fuso horário implícito em iluminação.
- **Drone/satelitário cruzado**: foto de rua + imagem de satélite (Google Earth) casam em minutos para analistas treinados.

Caso **Shamima Begum** (IS, 2015): identificada em entrevistas parcialmente pelo background. Caso recorrente em RUS-UKR: soldados postando selfies ranqueados por background → posição de artilharia. Para ativistas/jornalistas em campo: sanitize background **antes** de gravar; filme paredes lisas ou chroma key.

## Mobile

Celular é o dispositivo mais hostil a OPSEC: sempre ligado, sempre conectado, cheio de SDKs que fazem tracking silencioso, sensor-rich (GPS, mic, câmera, acelerômetro), e integrado à vida civil (banco, 2FA, mapas) ao ponto de não poder ser simplesmente "desligado".

### Android

- **GrapheneOS** (Pixel only): hardening agressivo, permissões granulares (Storage/Network per-app), Vanadium browser, sandbox Google Play opcional, profiles completos para isolar vida pessoal/work/research. Padrão atual para Android security-conscious.
- **CalyxOS**: similar em objetivo, menos aggressive hardening, MicroG pré-integrado (API Google sem tracking pesado).
- **Stock Android**: assumir tracking ubíquo. Mesmo sem conta Google, SDKs em apps (Facebook SDK, Firebase, AppsFlyer, Branch) enviam device ID + comportamento.
- **Permissões a revisar sempre**: Localização, Microfone, Câmera, Contatos, SMS, Acessibilidade (acessibilidade = root de app, usado por malware).

### iOS

- **Lockdown Mode** (iOS 16+): desliga superfície de ataque pesada (JIT JavaScript, tipos de anexos, FaceTime de desconhecidos, MDM). Foi projetado contra mercenary spyware (NSO Pegasus). Ativar se você é alvo plausível (jornalista, ativista, dissidente, político).
- **Advanced Data Protection**: iCloud E2EE real (ativa explicitamente — não é default em todos países).
- Apple coopera com subpoenas; E2EE real está só onde ADP cobre. Metadata de iMessage (quem falou com quem, quando) **é** coletada.
- Jailbreak = caminho para instrumentação livre, mas quebra integridade e atrai spyware comercial.

### SIM, eSIM, número

- **SIM swap attack**: atacante engenharia-social a operadora, porta seu número. Em minutos perde-se SMS 2FA e contas derivadas. Mitigação: PIN/senha na operadora, bloqueio de portabilidade, **não use SMS como 2FA** para nada importante.
- **eSIM** facilita troca e compra anônima em alguns países; em outros exige KYC estrito. Conheça a regra local.
- **Burner**: número dedicado, operadora diferente, nunca cruzado com identidade real. Mesmo com burner, celular **liga na mesma torre** que seu número real → correlação trivial.

### SDK tracking em apps legítimos

Apps comuns enviam:

- **Device advertising ID** (IDFA/AAID).
- **Instalações e eventos** para MMPs (AppsFlyer, Adjust, Branch).
- **Fingerprint de hardware** (CPU, RAM, modelo, build, idioma, fuso).
- **Localização aproximada** mesmo sem GPS (Wi-Fi, celular).

Agregadores de dados (Gravy, X-Mode, Anomaly Six, LocationSmart) compram/revendem esses streams. Dados vão para governo sem warrant em alguns regimes. Limite superfície: apps só no perfil dedicado, `Advertising ID` zerado, permissões mínimas.

### Burner phone

- Aparelho separado, nunca ligado perto do telefone pessoal (correlação por torre + acelerômetro).
- Compra em cash, ativação longe da sua área normal.
- Se precisa de nome, não seja o seu.
- **Nunca** logue serviço pessoal no burner — isso o despersonaliza em minutos.

## OPSEC financeira

Dinheiro é metadado concentrado. Cadeia completa: identidade → conta bancária → exchange KYC → endereço blockchain → destino. Qualquer elo quebra anonimato do todo.

### Bitcoin não é anônimo

Bitcoin é **pseudônimo e transparente** — todas as transações são públicas. Empresas de **chain analysis** (Chainalysis, Elliptic, TRM Labs, CipherTrace) clusterizam endereços por heurística (common-input, change-detection), cruzam com dados de exchanges KYC, e reconstroem fluxo. Governos compram esses serviços.

Se você:

1. Comprou BTC em exchange KYC,
2. Transferiu para endereço próprio,
3. Pagou em destinatário qualquer,

— um analista pluga endereço final → reverse-path → exchange → seu nome. Questão de tempo, não de capacidade.

### Mitigações parciais

- **Endereços novos por transação** (wallets HD fazem por default). Limita agregação, não resolve.
- **CoinJoin** (Wasabi 2.0, Samourai Whirlpool — status varia): múltiplos inputs/outputs coalescidos, quebra heurísticas. Eficácia depende do anonymity set. Alguns exchanges sinalizam coins pós-CoinJoin como "tainted" e bloqueiam.
- **Lightning Network**: off-chain, melhor privacy de rota mas extremos (open/close channel) vazam on-chain.
- **Non-KYC onramp**: ATMs Bitcoin (com limites), P2P (LocalBitcoins morreu; Hodl Hodl, Bisq, RoboSats sobrevivem), mining. Cada um tem trade-off próprio.

### Monero

Privacy-by-default: **ring signatures** (assinante é um de N, não identificável), **stealth addresses** (destinatário único por tx), **RingCT** (valores escondidos). Chain analysis contra Monero é qualitativamente mais difícil que contra Bitcoin — capacidades reais de governo em Monero são **classified**; pesquisa acadêmica mostrou vazamentos menores corrigidos.

Trade-offs: liquidez menor, delisted em várias exchanges reguladas (Kraken UE, Binance em várias jurisdições), fungibilidade às vezes contestada.

Para uso prático anônimo: **XMR como camada de mixing** entre BTC compra KYC → XMR → BTC de outra wallet é padrão conhecido (e monitorado).

### Mixers centralizados

**Tornado Cash** (Ethereum): sancionado pelo OFAC (EUA) em 2022; desenvolvedor preso na Holanda. Uso técnico é legal em várias jurisdições, mas **consequência social/bancária** é real — fundos passados por Tornado são flaggados indefinidamente.

Mixers centralizados custodians em BTC foram desligados pelo DoJ (Bitcoin Fog, Helix). Assuma que mixer com custodian é vulnerável a seizure + disclosure.

### Cash, pre-paid, gift cards

- **Cash** ainda é o único anônimo puro em transação face-a-face. Saques grandes atraem CTR (currency transaction reports acima de ~10k USD equivalente).
- **Pre-paid cards** (Visa/Mastercard gift): KYC frequentemente não exigido para valor baixo, mas vendedor registra compra. Usável para signup online não-bancário.
- **Gift cards de vendor** (Amazon, Steam): trocáveis por cripto em plataformas P2P. Cadeia longa mas viável.

### Regra prática

Identifique o **elo mais fraco**. Cripto privacy adianta pouco se sua exchange fiat→cripto tem seu passaporte e envia relatórios. Na outra ponta, adianta pouco se você paga `casa.com.br` em endereço com seu nome no cadastro.

## Deniability

Nem todo threat model permite assumir "adversário respeita limites". Em alguns (coerção física, apreensão com decriptação forçada), precisa-se de segredos **que você pode plausivelmente negar**.

- **VeraCrypt hidden volumes**: um volume cripto contém um segundo volume escondido no espaço "livre". Senha 1 → volume decoy inócuo. Senha 2 → volume real. Adversário não consegue provar que o segundo existe (espaço livre em FDE parece aleatório de qualquer forma). Limitações: metadata de acesso, inconsistências se montado errado, suspeita se nada útil aparece no decoy.
- **Duress password**: senha alternativa que destrói dados ou abre espaço limpo em vez da conta real. Suportado em alguns setups (Veracrypt, KeePassXC via plugins, sistemas de full-disk custom).
- **Plausibly deniable messaging**: OTR original tinha **deniable authentication** (você prova identidade no momento, mas não pode *provar a terceiros* depois). Signal **sacrificou** isso por simplicidade UX — mensagens Signal são forwardable como print/share.
- **Friend-to-friend darknet** (Freenet darknet mode, Briar, Retroshare): participa-se sem que adversário externo saiba que você participa. Ver `ANONYMITY_NETWORKS.md`.
- **Steganography**: esconder dados dentro de imagem/áudio/vídeo normal. Útil contra varredura automatizada, não contra análise forense dedicada (chi-square, análise estatística detecta payloads conhecidos).

Deniability **não** é força de criptografia — é uma propriedade de modelo de ataque. Funciona quando adversário tem poder limitado (não pode comparar com backups, não pode fazer análise de timing indefinida). Em custódia real, chaveiro físico + beating supera criptografia; XKCD 538 é a referência canônica.

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

### Self-OSINT audit

Antes de qualquer operação séria (ou periodicamente como saúde OPSEC), rode OSINT **contra si mesmo**. O adversário vai — você deveria ir primeiro.

- **Busca por seus usernames** em Google, sherlock, WhatsMyName, epieos. Reuso revela persona cruzada.
- **Busca de email** em HIBP, Dehashed, IntelX — o que vazou, onde, com que senha.
- **Imagem reversa** de seu avatar e fotos públicas (TinEye, Google, Yandex — Yandex é superior para rostos).
- **Telefone** em truecaller / PhoneInfoga — quem te tem agendado.
- **Grafo social**: tags em fotos de amigos, "friends of friends" no Facebook, "people you may know" que o LinkedIn te oferece revelam conexões que você nunca declarou.
- **Dados de registro** (CPF, CNPJ, imóvel, processo) em buscadores públicos locais.
- **Documente**: o que achou, o adversário também acha. Planeje assumindo isso.

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

## Tor — Uso prático

Arquitetura, ataques e onion services cobertos em `TOR.md`. Do ponto de vista de OPSEC do usuário, o que importa:

- **Use Tor Browser** ou Tails/Whonix — não Tor bruto com navegador qualquer. Fingerprint do browser é o que denuncia.
- **Mantenha NoScript/JS default**; JS ligado foi o vetor de quase todas as deanonimizações públicas.
- **Sempre HTTPS**. Exit node malicioso faz MITM em HTTP.
- **Não redimensione a janela** — letterboxing depende disso.
- **Um contexto por sessão**. Novo circuito para nova identidade (`New Identity` no Tor Browser).
- **Não logue em conta real** via Tor e depois via clearnet da mesma máquina; correlação trivial.
- **Bridges + PT** (obfs4, Snowflake, WebTunnel) se Tor bruto é bloqueado ou você não quer que ISP veja Tor.
- **Onion services v3** quando disponível para o destino (NYT, DuckDuckGo, ProPublica, Signal): tráfego nunca sai da rede, sem risco de exit MITM.

## Operações de Field

- **Viagens** para locais hostis: **burner phone + burner laptop**, dados essenciais sincronizados só no necessário. Na volta, reimagem.
- **Alfândega pode forçar unlock de devices** — esperar isso; não leve o que não pode revelar.
- **Power off** antes de fronteira. Suspend = disco destravado = chaves em RAM = lido trivialmente com forense.
- **Cloud-only durante viagem**: zero dado local; baixe o necessário no destino por canal E2EE, delete ao sair. Mais defensável em apreensão ("não tenho arquivos sensíveis") e verdade literal.
- **Jurisdições**: países 5-Eyes / 9-Eyes / 14-Eyes compartilham inteligência. Trânsito por hub 5-Eyes (Londres, NYC) implica possível exposição mesmo em voo conectando fora.
- **Direito ao silêncio em fronteira varia**: EUA permite recusar senha com custo (entrada negada/dispositivo apreendido); cidadão é menos pressionável que não-cidadão. Reino Unido (RIPA s.49) pune recusa com prisão. Pesquise **antes** de viajar.
- **Câmeras onipresentes**: FR pode estar em uso; fotos de passaporte em checagem; padrões de deslocamento em CFTV conectado.
- **Pay com cash, carros alugados em nome diferente** (não sempre possível/legal).

### Protesto / crowd

- **Celular desligado ≠ rastreável**: celular em bateria ainda emite IMSI periodicamente. Remoção de bateria (quando possível) ou **Faraday bag** é o único desligamento real.
- **Burner dedicado** se for documentar; não leve o pessoal. Pareamento de dois celulares com mesma rota/torre em mesmo evento correlaciona.
- **Reconhecimento facial** em multidão: mais presente do que o público imagina. Máscara + óculos + boné quebra alguns sistemas, não todos. CV Dazzle (padrões faciais) funciona contra alguns classificadores legados; contra modelos modernos, não. Cabeça baixa + distância do canto da câmera é mais efetivo que maquiagem.
- **Metadata de foto publicada**: sanitize antes de postar. Nunca publique em tempo real — timestamp de upload é IOC.
- **Bluetooth/Wi-Fi ligado** em multidão faz MAC probing; adversário com tracker Wi-Fi mapeia quem esteve onde. Airplane mode + desligar Wi-Fi/BT explicitamente (airplane ainda deixa BT em alguns devices).

## Estudos de Caso

- **Silk Road / Ross Ulbricht**: username `altoid` usado em forums de maconha e em uma questão técnica em Stack Overflow. Pivot OSINT → identidade real.
- **LulzSec / Sabu**: logou em IRC sem Tor uma vez.
- **Conti leaks**: vazamento interno de chats após stance pró-Rússia; OpSec interna inexistente.
- **Hutchins / MalwareTech**: pesquisador de cyber preso em viagem a DEF CON — OpSec legal (evitar fronteira hostil) é parte.
- **Reality Winner (2017)**: leak impresso ao *The Intercept* foi escaneado com MIC dots visíveis. NSA reverseou dots → número de série da impressora → registro de acesso ao documento original. Tempo entre publicação e prisão: 3 dias.
- **AlphaBay / Alexandre Cazes (2017)**: email de recovery `pimp_alex_91@hotmail.com` em headers de welcome messages do market. Mesmo email em fóruns públicos anos antes. Prisão por metadado que ele mesmo incluiu automaticamente.
- **Hansa (2017)**: apreendido silenciosamente pela polícia holandesa, operado por ~1 mês. Usuários que migraram de AlphaBay pós-takedown foram para uma honeypot em operação. OPSEC individual correta não protege contra "refúgio" comprometido.
- **Shamima Begum (2015)**: background em entrevistas parcialmente revelador de localização. Exemplo de fingerprinting por conteúdo, não por rede.

Lição uniforme: um **único** descuido frequentemente é suficiente.

## Ferramentas e Recursos

- **Tor, Tails, Qubes, Whonix**.
- **Signal, SimpleX, Session, Briar, Cwtch**.
- **Molly, Element (Matrix)**.
- **GrapheneOS, CalyxOS** (Android hardening).
- **iOS Lockdown Mode**.
- **YubiKey, SoloKey, passkeys** (FIDO2/WebAuthn).
- **Aegis, 2FAS, KeePassXC** (TOTP).
- **Bitwarden, 1Password, KeePassXC** (senhas).
- **Veracrypt, Cryptomator, age, gocryptfs** (volumes/backup cripto).
- **Heads, PureBoot** (firmware anti-evil-maid).
- **mat2** (metadata removal), **Exiftool** (leitura e strip).
- **Syncthing criptografado** para sync sem cloud central.
- **Sherlock, WhatsMyName, epieos, HIBP** (self-OSINT).
- **Monero CLI, Feather Wallet** (cripto privada).

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
