# TOR — The Onion Router

Tor é uma rede overlay de **low-latency anonymous communication**: roteia TCP através de circuitos de 3 relays com criptografia em camadas, quebrando a ligação entre origem e destino para observadores que não controlam simultaneamente os dois extremos. Não é "proxy com criptografia" nem "VPN distribuída" — é um sistema com threat model específico, garantias precisas e falhas previsíveis.

Este arquivo foca em **arquitetura, protocolo e ataques**. OPSEC de usuário está em `OPSEC.md`. Fundamentos de tunneling em `TUNNELING.md`. Outras redes anônimas (I2P, Freenet, mixnets) em `ANONYMITY_NETWORKS.md`.

## Taxonomia: surface, deep, dark

Confusão terminológica comum:

- **Surface web**: indexável por mecanismos públicos (Google, Bing). O que a maioria chama "internet".
- **Deep web**: tudo não-indexável — intranets, paywalls, dashboards autenticados, APIs, DBs privados, e-mails. **A maior parte do tráfego web está aqui** e é perfeitamente legítima. "Deep web" ≠ crime.
- **Dark web**: subconjunto da deep web acessível apenas via overlay networks (Tor, I2P, Freenet). Endereços `.onion`, `.i2p`, chaves Freenet.

Mídia popular funde os três. Tecnicamente são coisas distintas. Muito do que é chamado "dark web" é mirror legítimo: NYT, BBC, ProPublica, DuckDuckGo, Facebook, SecureDrop, Debian, arquivos acadêmicos.

## Threat model e o que Tor **não** faz

Tor protege contra:

- **Observador local** (ISP, Wi-Fi público, rede corporativa) que vê quais sites você visita.
- **Site de destino** correlacionando seu IP real com seu comportamento.
- **Censura** baseada em bloqueio de domínios/IPs (parcialmente, via bridges).
- **Adversário que controla um único relay** do seu circuito.

Tor **não** protege contra:

- **Adversário global passivo (GPA)** que observa grande fração da internet. Correlação de timing entre entrada e saída deanonimiza. Assumido irresoluvel para low-latency por design.
- **Adversário controlando guard + exit simultaneamente** (ou guard + HSDir para onion services).
- **Vazamentos de aplicação**: JavaScript, WebRTC, DNS fora do Tor, metadata em arquivos, plugins.
- **Deanonimização por conteúdo**: você logar na sua conta real em um .onion.
- **Malware no endpoint**.
- **Exploits no browser** (caso clássico: FBI NIT em 2013/2015 explorando Firefox ESR do Tor Browser para enviar IP/MAC real).
- **Estilometria, horários, fuso, idioma** — correlação comportamental.

Formulação curta: Tor esconde **com quem você fala**, não **quem você é**. Se você se auto-identifica dentro da sessão, o anonimato de rede é irrelevante.

## Onion routing — base teórica

Descendente das **mixes de Chaum** (1981): sequência de relays, cada um remove uma camada de cifragem e encaminha. Observador em um relay vê só o próximo salto; destino não vê origem; origem não vê destino.

**Diferenças de Tor vs Chaum mix puro**:

- Tor é low-latency: **não** introduz delays nem cover traffic. Trade direto: usável para web em troca de vulnerabilidade a correlação de tráfego.
- Circuitos são **telescópicos**: cliente negocia chaves com relay 1, depois estende a relay 2 **via** relay 1, depois a relay 3 via 2. Nenhum relay aprende mais que seu vizinho imediato.
- Circuito reusado para múltiplos streams (~10 min default) para amortizar custo do setup.

**Forward secrecy** do circuito: chaves efêmeras por hop, descartadas ao destruir o circuito. Compromisso posterior de chave de longo prazo do relay **não** decifra tráfego passado.

## Arquitetura

### Componentes

- **OP (Onion Proxy / client)**: o Tor que roda na sua máquina. Constrói circuitos, faz SOCKS para apps.
- **OR (Onion Router / relay)**: nó da rede. Subtipos:
  - **Guard**: primeiro hop. Mudança lenta (meses) por user.
  - **Middle**: segundo hop. Rotaciona por circuito.
  - **Exit**: último hop, faz a conexão TCP ao destino final. Vê tráfego em claro (se não for HTTPS).
  - **Bridge**: relay não listado no consensus público, usado como guard por quem está em rede censurada.
- **Directory Authority (DirAuth)**: ~9 operadores de confiança que assinam o consensus horário da rede. Ponto mais centralizado de Tor. Compromisso da maioria = game over.
- **HSDir**: relays com flag para hospedar descritores de onion services.
- **BridgeAuthority**: DirAuth especial que lista bridges.

### Consensus

A cada hora, DirAuths votam e publicam o **consensus document**: lista de relays ativos, suas chaves, flags (Guard, Exit, Stable, Fast, HSDir, BadExit, etc.), peso de banda.

- **Bandwidth Authorities (bwauth)**: medem relays ativamente e atribuem peso — relay com 10× mais banda é escolhido 10× mais frequentemente.
- **Flags**: `Guard` (estável o bastante pra ser guard), `Exit` (tem exit policy permissiva), `Stable`/`Fast` (uptime/banda), `HSDir` (pode hospedar descritores), `BadExit` (comportamento suspeito, excluído da seleção como exit).
- **Consensus é assinado** por threshold dos DirAuths. Cliente rejeita consensus sem maioria válida.

Centralização dos DirAuths é **o** trade-off arquitetural. Alternativas descentralizadas (Walking Onions, proposal 323) em desenvolvimento.

## Cells e transporte

Unidade de transmissão entre relays: **cell fixa de 514 bytes** (antigamente 512 + 2 para circ_id v4). Tamanho fixo frustra análise por tamanho. Cifragem link-layer TLS entre ORs + criptografia em camadas sobre isso.

Tipos principais:

- **CREATE/CREATED, CREATE2/CREATED2**: inicia circuito (handshake).
- **EXTEND/EXTENDED**: estende circuito mais um hop.
- **RELAY**: carrega dados aplicacionais dentro do circuito (sub-tipos: RELAY_DATA, RELAY_BEGIN, RELAY_END, RELAY_CONNECTED, RELAY_RESOLVE).
- **DESTROY**: encerra circuito.
- **PADDING, PADDING_NEGOTIATE**: padding contra fingerprinting (proposta 251).

RELAY cells são cifradas em camadas no sentido cliente→relay; cada relay remove uma camada. Resposta é cifrada no sentido inverso.

## Handshake: ntor e ntor-v3

**TAP** (legado, baseado em DH + RSA-1024): descontinuado, vulnerável.

**ntor** (proposal 216, Goldberg/Stebila/Ustaoglu 2013): handshake de uma via sobre Curve25519. Para estender ao relay R:

1. Cliente gera par efêmero `(x, X)`. Envia `(ID_R, B_R, X)` ao relay, onde `B_R` é a public key estática de R.
2. Relay gera efêmero `(y, Y)`. Computa `secret_input = EXP(X, y) || EXP(X, b) || ID || B || X || Y || "ntor-curve25519-sha256-1"`.
3. KDF produz chave de sessão + MAC de autenticação.
4. Relay devolve `(Y, auth)`. Cliente verifica auth; se ok, chave compartilhada é derivada.

Autentica o relay (precisa conhecer `b`) e estabelece forward secrecy (`x`, `y` descartados).

**ntor-v3** (proposal 332): estende ntor com campos extras de negociação (congestion control, features opcionais) e hybrid post-quantum está sendo integrado (ML-KEM combinado com Curve25519). PQ urgente por conta de "harvest now, decrypt later" — tráfego capturado hoje pode ser revertido quando computadores quânticos existirem.

## Circuit construction

Processo telescópico:

```
Client -> Guard:   CREATE2 (ntor para Guard)
Guard  -> Client:  CREATED2 (chave K1 estabelecida)

Client -> Guard:   RELAY_EXTEND2 (ntor para Middle, cifrado com K1)
Guard  -> Middle:  CREATE2
Middle -> Guard:   CREATED2
Guard  -> Client:  RELAY_EXTENDED2 (K2 estabelecida)

Client -> Guard:   RELAY_EXTEND2 (ntor para Exit, cifrado com K1, depois K2)
... análogo para o terceiro hop ...
```

Depois do setup:

```
Client: AppData  --enc K1--enc K2--enc K3-->  Guard
Guard:  --enc K2--enc K3-->                    Middle
Middle: --enc K3-->                            Exit
Exit:   AppData em claro                       Destination
```

Cliente envia **stream** pelo circuito via RELAY_BEGIN (host, porta). Exit abre TCP até o destino. RELAY_DATA carrega bytes da aplicação.

**Stream isolation**: múltiplos streams podem compartilhar circuito, mas são isolados por destino/porta/SOCKS auth. Tor Browser usa `IsolateDestAddr` para isolar circuitos por first-party domain — evita que adversário que controle circuito A correlacione com circuito B do mesmo usuário.

## Path selection

Regras simultâneas:

- **Weighted by bandwidth** (do consensus). Relay de 100 MB/s é escolhido ~100× mais que um de 1 MB/s.
- **No same /16** entre hops (evitar adversário que controla um AS).
- **No same family** — operadores declaram famílias (`MyFamily`) para que relays do mesmo operador não sejam usados juntos; rejeitado se qualquer par no circuito está na mesma família.
- **Guard só como guard, Exit só como exit**, middle qualquer relay com banda suficiente.
- **Guard rotacionado** lentamente (prop 271: um guard primary por meses). Middle rotaciona por circuito. Exit escolhido para casar com exit policy da porta de destino.

### Por que guards existem

Sem guards, cada novo circuito escolhe hop 1 aleatoriamente. Dado tempo suficiente, adversário com fração `f` da rede eventualmente casa-se em **algum** circuito como primeiro hop. Com guards fixos, ou seu guard é malicioso (prob. `f`) ou nunca é (prob. `1-f`). Reduz deanonimização progressiva ao custo de concentração de risco.

**Vanguards** (antes: vanguards-lite, agora proposal 292 integrada): guards extras em camada 2 e 3 para onion services, com rotação escalonada. Mitiga guard discovery contra HS (adversário rodando muitos middles tentando virar vizinho do guard de um HS).

## Onion services (hidden services)

Serviços acessíveis só via Tor, onde tanto cliente quanto servidor são anônimos. Endereços `.onion`.

### v2 — descontinuado

Descontinuado em outubro de 2021. Motivos:

- Endereços de 80 bits (16 caracteres base32). Vulneráveis a collision/enumeration.
- SHA-1 no design.
- **HSDir enumeration**: adversário rodando HSDirs conseguia listar todos onion services existentes e monitorar popularidade.
- Criptografia legada.

### v3 — atual (proposal 224)

- Endereços de 56 caracteres base32, derivados de chave **ed25519** pública (pubkey + checksum + version). Exemplo: `facebookwkhpilnemxj7asaniu7vnjjbiltxjqhye3mhbshg7kx5tfyd.onion`.
- **Blinded keys**: chave pública é "cegada" por período (24h default), de forma que HSDirs responsáveis por hospedar descritor **não aprendem** a identidade real do onion service nem podem enumerar serviços.
- **Shared Random Value (SRV)**: DirAuths publicam no consensus um valor aleatório consensual por dia. Define quais HSDirs hospedam descritor de qual serviço — impede adversário de "posicionar" HSDirs malicioso para serviço-alvo.
- **Client authorization**: operador pode restringir acesso a clients com chave x25519 autorizada.

### Fluxo de acesso a `.onion` v3

1. **HS sobe**: escolhe 3 **introduction points** (IPs, circuitos até relays Intro). Publica descriptor (lista de IPs, blinded key, etc.) em HSDirs responsáveis no período.
2. **Cliente resolve endereço**: deriva blinded key, pergunta aos HSDirs pelo descriptor, decifra.
3. **Cliente escolhe rendezvous point (RP)**: relay qualquer. Conecta via circuito próprio.
4. **Cliente envia INTRODUCE1** a um intro point do HS, contendo RP + cookie + chave efêmera.
5. **HS recebe INTRODUCE2**, conecta a RP via circuito próprio, envia RENDEZVOUS1.
6. **RP junta os dois circuitos**. Cliente e HS falam ponta-a-ponta através de **6 hops** (3 do cliente + 3 do HS, com RP no meio).

Latência é maior que Tor normal. Anonimato protege os dois lados.

### Variantes

- **Single onion services**: HS não usa 3 hops próprios (faz só 1 ou 0 até o RP). Perde anonimato do servidor em troca de latência menor. Uso: sites grandes (Facebook, Cloudflare) que **querem** o endereço .onion mas não escondem localização.
- **Onionbalance**: HA para onion services — múltiplas instâncias publicando o mesmo descritor.
- **Vanity addresses** (`mkp224o`, `scallion`): brute-force de prefixo bonito. Custo cresce exponencial no comprimento. `facebookwkh...` levou milhões de core-hours.

## Bridges e pluggable transports

Tor vanilla é detectável: handshake TLS tem assinatura, IPs de relays estão no consensus público (baixe a lista e bloqueie). Para usar Tor sob censura:

### Bridges

Relays **não** listados no consensus público. Usuário pede bridges ao BridgeDB (via email, web com CAPTCHA, Telegram bot, Moat) e configura manualmente. Efetivos contra bloqueio por lista de IPs, inúteis contra DPI que identifica o protocolo Tor.

### Pluggable transports (PT)

Camada que ofusca o tráfego Tor para parecer outra coisa.

- **obfs4** (padrão hoje): randomiza bytes do handshake — tráfego parece "random noise". IAT (inter-arrival time) opcional para anti-timing fingerprint. Eficaz contra China/Irã **por muito tempo**, mas detectável por ML/entropia alta num mundo onde tráfego real tem estrutura.
- **meek**: domain fronting via CDN. SNI/Host apontam para domínio de alta reputação (Azure, Amazon, Google); request HTTP real vai para bridge Tor dentro da CDN. Google/Amazon/Azure removeram domain fronting ~2018–2020; status atual é marginal.
- **Snowflake**: usa **WebRTC**. Voluntários rodam proxy Snowflake no browser (extensão ou página web); cliente censurado conecta-se a um proxy ephemeral via broker, e o proxy relay para uma bridge Tor. Tráfego parece videoconferência. Altamente efetivo contra bloqueio em massa; proxies churn rápido.
- **WebTunnel**: bridge atrás de HTTPS comum — servidor é indistinguível de website normal para o censor. Reescrita moderna da ideia de "pareça HTTPS real, não Tor".
- **Conjure, TapDance, refraction networking**: requerem cooperação de ISPs que redirecionam tráfego aparentemente para site X para a rede Tor. Deploy difícil; experimentais.

## Ataques reais e acadêmicos

### End-to-end correlation

Adversário que vê **entrada** (cliente↔guard) e **saída** (exit↔destino, ou HS↔RP) consegue correlacionar por timing e volume com altíssima precisão. Low-latency é fundamentalmente incompatível com resistência a GPA. Tor **assume** isso como limite.

### Website fingerprinting

Mesmo sem ver o conteúdo, adversário observando só cliente↔guard pode treinar ML nos padrões de tamanho/timing de cells para identificar qual site você visita, entre um conjunto fechado. Accuracy publicada varia 70-95% em configurações open-world realistas. Defesas:

- **WTF-PAD** (Walkie-Talkie, proposta 254) — padding adaptativo baseado em distribuição.
- **Circuit padding** integrado ao Tor modern (proposta 304).
- Netflow-level defenses ainda abertas.

### Sybil attacks

Adversário roda muitos relays. Se >50% da banda, compromete a rede. Mesmo <10% já é significativo. Caso real:

- **KAX17** (dez/2021): pesquisador "nusenu" revelou operador mantendo centenas a milhares de relays com padrões comuns (email, CAs, timing de subida) por anos. Picos próximos de 10% de banda guard. DirAuths removeram em rodadas. Motivação nunca confirmada; suspeita de deanon state-level.

### Tagging / relay early (CERT/CMU, 2014)

Pesquisadores da CMU (contratados pelo FBI segundo tribunal) operaram ~100 relays com flags Guard e HSDir. Modificavam RELAY_EARLY cells para marcar tráfego, propagando marca entre hops — correlacionava cliente e onion service pela marca. Usado para deanonimizar Silk Road 2.0, Lorax e outros. Tor respondeu com patches (limite a RELAY_EARLY, auditoria de cell tipo).

### Murdoch-Danezis congestion attack

Adversário induz carga em relay X e mede se latência de uma conexão-alvo aumenta. Se sim, conexão passa por X. Permitia mapear circuitos. Mitigado com mudanças em scheduling e reação a carga, mas classe geral de congestion attacks permanece.

### Guard discovery contra HS

Adversário quer achar o guard de um HS específico (para então subpoena-lo ou comprometê-lo). Roda muitos middles, conecta-se ao HS repetidamente, mede quem é vizinho do guard. Vanguards (L2/L3) mitigam aumentando o anonymity set. Descoberto o guard, **adversário pode forçar cooperação legal** e acabou.

### AS-level e BGP

Um único AS pode ter visibilidade do tráfego cliente↔guard **e** exit↔destino. **Raptor attacks** (2015) mostram BGP hijacks para forçar tráfego por AS controlado. Tor tem contramedidas limitadas (path selection consciente de AS, proposta 247, implementação parcial).

### Browser e aplicação

- **FBI NIT (Network Investigative Technique)**: Firefox exploits em sites `.onion` tomados pelo FBI (Playpen, Freedom Hosting) enviavam payload que chamava casa **fora do Tor**, revelando IP real. Tor Browser agora roda em sandbox mais restrito; NoScript block default.
- **WebRTC** vaza IP local via STUN — desativado no Tor Browser.
- **DNS vazando** se SOCKS não estiver configurado para resolver remotamente (`socks5h://` e não `socks5://`).
- **Timezone, Accept-Language** — `letterboxing` + `resist-fingerprinting` mitigam.

### Protocolo e metadata

- **TLS fingerprint**: handshake do Tor tem JA3/JA4 específicos. Mudam entre versões; detecção é arms race.
- **Clock skew**: relógios dessincronizados no servidor permitem fingerprinting de máquina (Murdoch, 2006).
- **CircuitID reuse**, **padding patterns**: fontes menores de fingerprinting.

## Tor vs VPN vs proxy

| Aspecto | Proxy simples | VPN | Tor |
|---|---|---|---|
| Confiança | 1 parte | 1 parte | Distribuída (3 relays + DirAuths) |
| Esconde tráfego do ISP | Parcial (SNI vaza) | Sim | Sim |
| Esconde IP do destino | Sim | Sim | Sim |
| Resiste à subpoena | Não | Depende (logs?) | Parcialmente (precisaria comprometer múltiplos relays em jurisdições diferentes) |
| Anonimato de rede | Baixo | Baixo-médio | Alto (sob threat model) |
| Latência | Baixa | Baixa | Alta (3 hops, geografia aleatória) |
| Banda | Alta | Alta | Baixa |

VPN e Tor resolvem problemas diferentes. VPN é "confio neste provedor mais que no meu ISP"; Tor é "não confio em ninguém individualmente mas no agregado". Combinações (Tor-over-VPN, VPN-over-Tor) têm trade-offs próprios — geralmente pioram o anonimato, não melhoram.

## Implementações

- **C Tor** (`tor`): implementação histórica, C, single process async. Em manutenção; features novas vão prioritariamente para Arti.
- **Arti**: reescrita em Rust pela Tor Project. Objetivos: memory safety, modularidade, API embarcável em apps. Client Arti usável em produção; suporte a relay e onion services maturando. Longo prazo: substituir C Tor.
- **Onionbalance, stem (Python control lib), nyx (monitor)**: ferramentas satélite.

## Operacional e legal

### Rodar relays

- **Middle**: baixo risco legal, bom para contribuir. Banda é o custo.
- **Guard**: idem middle mas com estabilidade requerida.
- **Exit**: **alto risco** — seu IP aparece em logs de abuso (torrent, scan, crime). Cartas de abuse providers, subpoenas, possível busca. Rodar via entidade jurídica (TorServers.net, EFF guidelines) com exit policy reduzida e **ISP aliado**. Não rode exit em casa.
- **Bridge**: médio risco — não é listado publicamente mas pode atrair atenção em regimes que querem bloquear Tor.

### Operações legais contra Tor

- **Operation Onymous** (2014): coordenada Europol/FBI, tomou Silk Road 2.0 e dezenas de markets. Vetor nunca totalmente revelado; especulação entre CMU tagging e erros operacionais dos alvos.
- **Operation Bayonet** (2017): AlphaBay e Hansa. AlphaBay caiu por erro operacional de Cazes (email de recovery pessoal num header). Hansa foi **apreendido silenciosamente** e operado pela polícia holandesa por ~1 mês para coletar dados de users que migraram.
- **Padrão histórico**: raramente Tor é quebrado criptograficamente. Quase sempre o alvo se auto-identifica ou é pego por browser exploit / OPSEC ruim.

## Futuro

- **Pós-quântico**: integração de ML-KEM hybrid no ntor-v3 (harvest-now-decrypt-later é o risco concreto).
- **Walking Onions** (proposal 323): descentralizar DirAuths, reduzindo o único ponto de centralização estrutural.
- **Arti** substitui C Tor a médio prazo.
- **Melhor resistência a correlation attacks**: research em circuit padding, batching parcial (mas sempre em tensão com usabilidade web).
- **Censorship arms race**: próximos PTs (WebTunnel, Conjure) versus DPI ML.

## Referências do repositório

- OPSEC de usuário (Tails, Whonix, erros comuns): `OPSEC.md`
- Tunneling geral e SSH: `TUNNELING.md`
- Redes anônimas comparadas (I2P, Freenet, mixnets): `ANONYMITY_NETWORKS.md`
- OSINT em dark web (enumeração, Ahmia, monitoring): `OSINT.md`
- Primitivas cripto (Curve25519, ed25519, HKDF): `CRYPTOGRAPHY.md`
- P2P e DHT (contexto para onion service descriptor distribution): `P2P.md`
- Firewall / DPI e detecção de Tor: `FIREWALL.md`
