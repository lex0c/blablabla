# ANONYMITY NETWORKS — I2P, Freenet, Mixnets, e afins

"Rede anônima" é um termo guarda-chuva. Cada sistema resolve um problema distinto sob um threat model distinto, e as diferenças arquiteturais são o que determina quando um serve e outro não. Este arquivo cobre **alternativas e complementos a Tor** — para Tor em si, ver `TOR.md`.

Principais sistemas em uso ou estudados:

- **Tor** — low-latency onion routing para tráfego interativo (coberto em `TOR.md`).
- **I2P** — low-latency garlic routing, rede **interna** (não é feita para sair para clearnet).
- **Freenet / Hyphanet** — datastore P2P resistente à censura, não é para tráfego interativo.
- **Mixnets modernas** (Loopix, Nym, Katzenpost) — high-latency, resistentes a GPA, inviáveis para web mas viáveis para messaging/metadata.
- **Sistemas acadêmicos** — DC-nets, Riffle, Vuvuzela, Atom, Karaoke — exploram o limite teórico.

O eixo central: **low-latency resolve anonimato contra adversário local**; **high-latency resolve contra adversário global**. Nenhum faz os dois.

## I2P (Invisible Internet Project)

Rede overlay P2P, roteamento em camadas, **foco em serviços internos** (eepsites `.i2p`). Diferente de Tor em quase toda decisão.

### Garlic routing

Extensão de onion routing: múltiplas mensagens (cloves) são empacotadas no mesmo "bulbo" cifrado. Hop de saída vê um bulbo, descasca, descobre múltiplas mensagens com destinos diferentes e tempos diferentes (incluindo delays opcionais).

Benefícios frente a onion routing puro:

- **Confunde análise** por volume — um bulbo carrega mais contexto que uma cell Tor.
- **Cover traffic barato** — cloves falsas são gratuitas do ponto de vista do protocolo.
- **Batching parcial** natural.

### Tunnels unidirecionais

Diferença arquitetural crítica vs Tor:

- Cada peer mantém **inbound tunnels** (por onde outros peers enviam mensagens para você) e **outbound tunnels** (por onde você envia).
- Inbound ≠ outbound — caminhos diferentes, peers diferentes.
- Uma "conversa" A↔B usa 4 caminhos distintos: outbound A, inbound B, outbound B, inbound A.

Em Tor, circuito é bidirecional: mesmo conjunto de 3 relays carrega request e response. Em I2P, adversário precisa correlacionar **dois tunnels independentes** (outbound do emissor + inbound do receptor) para fazer end-to-end correlation. Janela de ataque é mais difícil de explorar.

Custo: overhead maior, setup mais complexo, mais peers por "conexão".

### NetDB

Diretório distribuído em **floodfill routers** (Kademlia-like). Dois tipos de entradas:

- **RouterInfo**: info sobre cada router (endereço, capabilities, chaves).
- **LeaseSet**: conjunto de tunnels atualmente válidos para um destination (~equivalente ao HS descriptor em Tor).

Publicações são assinadas e replicadas em floodfills próximos ao hash do destino. **LeaseSets têm TTL curto** (minutos), por isso forçar um peer offline já derruba a acessibilidade.

Centralização muito menor que Tor: qualquer router pode se tornar floodfill se atender critérios. Contraparte: mais vetores para Sybil no floodfill set.

### Endereços

- **Destination**: chave pública longa (~500 chars base64). Inutilizável humanamente.
- **b32 address**: hash SHA256 base32 de destination → `abc...xyz.b32.i2p` (52 chars).
- **Hosts.txt / jump services**: mapeamento nome→destination, estilo DNS alternativo. Sem raíz centralizada; cada user confia em conjunto de "address books".

### Protocolo

Stack (de baixo pra cima):

- **NTCP2 / SSU2**: transporte cifrado entre routers. NTCP2 usa Noise framework (baseado em Curve25519, ChaCha20-Poly1305). SSU2 é UDP equivalente.
- **I2NP** (Invisible Internet Network Protocol): mensagens entre routers (tunnel build, data, garlic).
- **I2CP** (Client Protocol): app ↔ router local. Apps não falam com a rede direto.
- **Streaming library**: TCP-like em cima de I2NP (ordenação, retransmissão).

### Criptografia

Transição histórica:

- Legado: ElGamal 2048 + AES256-CBC + DSA 1024. Lento e quebrado pelos padrões atuais.
- Atual: **ECIES-X25519 + AEAD ChaCha20-Poly1305** (rollout em 2020+). Ed25519 para assinaturas.

### Threat model e diferenças vs Tor

| Aspecto | Tor | I2P |
|---|---|---|
| Alvo principal | Acesso anônimo à clearnet | Serviços internos anônimos |
| Diretório | DirAuths centralizados | Floodfill distribuído |
| Tunnels | Bidirecionais, 3 hops | Unidirecionais, 2-3 hops por direção |
| Exit | Exits dedicados | "Outproxies" raros e não são o foco |
| Tamanho da rede | ~7-8k relays, milhões de users | ~50-80k routers, base menor |
| Cliente é relay? | Não por default | **Sim por default** — cada user roteia para outros |
| Threat de Sybil | DirAuths + bwauths resistem parcialmente | Floodfill é onde o ataque acontece (historicamente reportado) |

Usar I2P por quê: serviços internos (torrents anônimos com i2psnark, mail via I2P-Bote, fóruns) onde **não precisar sair para clearnet** é vantagem, pois cada hop é também consumidor da rede → anonimato cresce com uso.

Não usar I2P por quê: acessar web "normal". O modelo de outproxy não foi pensado para isso e é mais frágil/lento que Tor exit.

## Freenet / Hyphanet

Freenet é um animal diferente: **não é rede de transporte**, é **datastore P2P censorship-resistant**. Você não "visita" um site Freenet em tempo real — você publica conteúdo que é **replicado e persistido** em caches de outros peers, acessível depois.

Rebatizado **Hyphanet** em 2023 após fork (Freenet.org pegou o nome para um projeto diferente).

### Modelo: content-addressable datastore

Toda inserção produz uma chave que é o **endereço lógico** do conteúdo. Tipos:

- **CHK (Content Hash Key)**: chave = hash do conteúdo cifrado. Imutável. Qualquer um que saiba a chave pode recuperar; sem a chave, dado é indistinguível de ruído.
- **SSK (Signed Subspace Key)**: namespace assinado por chave pública. Permite publicar "sua" pasta. Mutável só para quem tem a private key.
- **USK (Updatable Subspace Key)**: açúcar sobre SSK com versionamento.
- **KSK (Keyword Signed Key)**: chave derivada de string humana. Cômodo mas inseguro (qualquer um com a string pode sobrescrever).

Um "site" Freenet é um conjunto de arquivos sob uma USK, com HTML estático. Lê-se via "freesite" no `fproxy` local.

### Roteamento

Cada peer tem **location** (float em [0,1)). Conteúdo com hash H é roteado para peers cuja location está próxima de H (metric = distância no círculo). Diferente de DHTs estilo Kademlia: routing é **small-world** e **aprendido** — peers ajustam seus vizinhos ao longo do tempo.

**Requests** propagam com TTL decrescente; peers respondem do cache local se tiverem ou encaminham. **Respostas populares são cacheadas** no caminho de volta → popularidade aumenta disponibilidade. Contrário: conteúdo não acessado é eventualmente expurgado. Freenet **não** garante persistência de nada.

### Opennet vs Darknet mode

- **Opennet**: nó conecta a peers anônimos do pool global. Fácil de usar, mais atacável por Sybil.
- **Darknet (F2F)**: você só conecta a peers que conhece pessoalmente (friend-to-friend). Muito mais resistente à censura estatal — atacante nem sabe que você participa da rede, a menos que já vigie seus amigos. Em troca: difícil de bootstrap, rede pequena, latência alta.

F2F é o **diferencial real** de Freenet/Hyphanet. Nenhum outro sistema em uso sério oferece darknet mode puro (Briar e Retroshare são F2F mas não são datastores).

### Trade-offs

Freenet optimiza **censura-resistência e deniability** — não latência, não anonimato interativo, não volume.

- Publicar algo no Freenet e sumir: o conteúdo pode viver anos se demandado. Apagar é impossível se popular. Esse é o ponto.
- **Plausible deniability**: seu datastore contém fragmentos cifrados de coisas que você nunca escolheu hospedar. Legalmente/politicamente útil em alguns contextos.
- **Latência alta** (segundos a minutos para fetch).
- Histórico de problemas com Sybil em opennet e com inserção de conteúdo abusivo (dilema inerente a censorship-resistant storage).

## Mixnets modernas

Mixnets vêm de Chaum (1981): relay batcha mensagens, embaralha e reenvia com delay. Cover traffic mascara quem fala com quem. Historicamente sofreram com **latência proibitiva** e **difícil adoção**. Pesquisa recente foca em mixnets **aplicáveis** a messaging (não web).

### Loopix (Piotrowska et al., 2017)

Base teórica de várias mixnets atuais. Ideias centrais:

- **Stratified topology**: N camadas de mixes; mensagem atravessa uma por camada.
- **Poisson mixing**: cada mensagem recebe delay exponencial independente em cada hop (não batch clássico). Estatisticamente equivalente a embaralhar sem esperar batch cheio.
- **Cover loops**: cada cliente envia mensagens fantasma periodicamente, rotuladas como cover, e a si mesmo via percurso aleatório. Camuflagem de tráfego sem desperdício coordenado.
- **Drop cover**: similar mas rotas terminam em "drop" num mix, não retornam.

Resultado: anonimato contra GPA com latência média em torno de 1-5s (ajustável) e banda dominada por cover (alto, mas previsível).

### Nym

Mixnet com **incentivo econômico** via token (NYM). Mixnodes recebem recompensa proporcional a qualidade de serviço medida por "mixnodes measurement". Threat model: assume que incentivo monetário atrai operadores suficientes para um anonymity set grande.

Arquitetura é Loopix-like: Sphinx packet format (payload size-padded, hop-cifrado, indistinguível entre mensagens genuínas e cover). Roteamento em 3 camadas estratificadas.

Usos propostos: messaging, transações privadas, APIs que precisam proteger quem-fala-com-quem (metadata protection). Web continua inviável (latência).

### Katzenpost

Implementação open-source de Loopix, foco acadêmico/produção para messaging e voting. Usado como base de pesquisa (post-quantum mix design, decoy routing).

### Trade-offs realistas de mixnets

- **Anti-GPA**: mixnets são a única classe conhecida de solução para adversário global passivo em comunicação em rede.
- **Latência 1s+**: não serve para web interativa.
- **Cover traffic caro**: banda constante mesmo sem uso, ou anonimato degrada.
- **Adoção**: rede pequena = anonymity set pequeno. Problema de arranque típico de sistemas anônimos.

Se seu threat model inclui "um estado observa backbone inteiro e captura todos os cabos" (five-eyes-level), Tor não salva; mixnet pode.

## Sistemas acadêmicos relevantes

Não em produção, mas delimitam o espaço de design:

### DC-nets (Dining Cryptographers, Chaum 1988)

Grupo de N participantes pode publicar **uma** mensagem anônima por rodada, com anonimato **teoricamente perfeito** contra qualquer adversário (inclusive os outros participantes, até N-2 colluders). Custo: cada rodada requer XOR coordenado entre todos os participantes e permite apenas um emissor (colisão se dois tentam).

Escalar DC-nets é o problema. **Herbivore, Dissent**: tentam estruturar hierarquicamente. **Riffle** mistura DC-nets com mixnets para throughput maior.

### Riffle (Kwon et al., 2016)

Messaging anônimo com **verifiable shuffle** (mixnet cujas mixes provam que embaralharam corretamente via zero-knowledge). Resistente a mix ativo malicioso. Alto custo computacional.

### Vuvuzela / Alpenhorn (van den Hooff et al., 2015)

Messaging que esconde **metadata de quem-fala-com-quem** via broadcast com dead drops e cover traffic massivo. Latência de 30-60s por mensagem. Paper seminal pra "metadata-private messaging".

### Karaoke, Atom, XRD

Herdeiros de Vuvuzela com throughput maior (Karaoke) ou assumption mais fraca (Atom). Nenhum em uso prático, mas mostram como fronteira teórica evolui.

## Comparação prática

Matriz honesta, sem marketing:

| Sistema | Threat model | Latência | Uso real | Onde brilha | Onde falha |
|---|---|---|---|---|---|
| **Tor** | Adversário local, não global | 0.3-2s | Navegação, onion services | Anonimato interativo usável | GPA, correlation |
| **I2P** | Similar a Tor, rede interna | 0.5-3s | Serviços internos, torrents | Arquitetura P2P "pura" | Exit para clearnet |
| **Freenet** | Censura, deniability | Segundos-minutos | Publicação persistente | F2F darknet | Anything interativo |
| **Mixnets (Nym, Loopix)** | GPA | 1-10s+ | Messaging, metadata | Anti-GPA | Web, adoção |
| **DC-nets / Vuvuzela** | GPA + malicious insiders | 10-60s | Pesquisa | Anonimato ótimo | Escala, throughput |
| **VPN** | ISP local | <50ms | Geobloqueio, privacy rasa | Performance | Um único ponto de confiança |

### Escolha por caso de uso

- **"Quero navegar sem meu ISP/governo local saber o que vejo"** → Tor, eventualmente com bridge+PT.
- **"Quero rodar serviço sem revelar onde está"** → Tor onion service v3.
- **"Quero publicar texto que governo X quer apagar"** → Freenet/Hyphanet (resistência) ou IPFS + mirrors (velocidade, menos deniability).
- **"Quero comunicar em rede small-world sem que atacante saiba que participo"** → Freenet darknet mode, Briar, Retroshare.
- **"Preciso esconder metadata de quem-fala-com-quem contra estado"** → Nym / mixnet-based messaging, Cwtch, ou aplicações sobre Loopix.
- **"Só quero burlar geoblock"** → VPN. Tor é overkill e lento.

### O que **nenhum** deles resolve

- Endpoint comprometido (malware, physical access).
- Auto-identificação (logar na conta real, reusar pseudônimo cross-site).
- Estilometria, análise comportamental, biometria de digitação.
- Adversário que correlaciona pagamentos, horários, cultura, idioma.
- Ordem judicial contra a empresa que hospeda sua identidade pseudônima.

**Anonimato de rede é condição necessária, não suficiente.** Ver `OPSEC.md`.

## Combinações (e por que raramente ajudam)

- **Tor-over-VPN** (VPN primeiro, depois Tor): esconde de ISP que você usa Tor. Acrescenta um provedor confiável. Útil apenas se a ameaça é "ISP sabe que sou Tor user"; não melhora anonimato contra destino/exit.
- **VPN-over-Tor** (Tor primeiro, depois VPN): exit Tor sai para VPN que sai para internet. VPN vê seu tráfego em claro (se não HTTPS) mas não seu IP real. Piora drasticamente se VPN tem KYC ou logs — você acabou de **assinar** seu tráfego Tor.
- **Tor + I2P simultâneos**: possível (ligações diferentes para destinos diferentes). Não "soma anonimato"; só permite acessar os dois espaços.
- **Mixnet + Tor**: interessante teoricamente (mixnet para metadata, Tor para transporte); implementações práticas raras.

Regra geral: **mais camadas ≠ mais anonimato**. Cada camada adiciona superfície de ataque, complexidade e ponto de confiança. Entenda o threat model e escolha **uma** ferramenta adequada.

## Referências do repositório

- Tor em profundidade: `TOR.md`
- OPSEC de usuário (Tails, Whonix, erros): `OPSEC.md`
- P2P, DHT, floodfill, Kademlia: `P2P.md`
- Criptografia (Curve25519, ed25519, Noise, Sphinx): `CRYPTOGRAPHY.md`, `CRYPTO_ADVANCED.md`
- Tunneling geral: `TUNNELING.md`
- OSINT em darknets: `OSINT.md`
