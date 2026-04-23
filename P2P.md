# P2P — Redes Peer-to-Peer

Peer-to-peer é o modelo onde **cada nó é simultaneamente cliente e servidor**, sem autoridade central obrigatória. Não é "moderno" nem "antigo" — é uma escolha de topologia com trade-offs severos e diferentes do cliente-servidor. BitTorrent, Bitcoin, IPFS, Tor, Secure Scuttlebutt, WebRTC, libp2p: todos diferentes instâncias do mesmo problema.

Este arquivo foca em **arquitetura**, não em um protocolo específico. Fundamentos de sistemas distribuídos em `DISTRIBUTED_SYSTEMS.md`. Criptografia primitiva em `CRYPTOGRAPHY.md`.

## Quando P2P faz sentido (e quando não)

**Faz sentido quando**:
- **Resistência à censura / ponto único de falha** é requisito (ativismo, samizdat digital, messaging).
- **Escalabilidade horizontal espontânea** — banda/storage do sistema cresce com número de users (BitTorrent: mais seeders = mais rápido).
- **Conteúdo intrinsecamente compartilhado** — arquivos, logs públicos, blockchain.
- **Custo de infraestrutura central é proibitivo** ou politicamente inviável.
- **Latência end-to-end importa** — dois users na mesma LAN não deveriam passar por servidor em outro continente (voice, video).

**Não faz sentido quando**:
- Há um provedor claro com interesse em centralizar (Netflix, banco, SaaS B2B).
- Latência de bootstrap é crítica (user chega, quer resposta em <200ms).
- Confiança centralizada resolve melhor (healthcare, sistema financeiro regulado).
- Escala pequena (<100 nós) — custo de complexidade não compensa.
- Estado muda rápido e precisa linearizability forte — CAP te pega.

P2P costuma ser **vendido como ideologia** (descentralização!). Tecnicamente é **ferramenta** — use quando o problema casa.

## Topologias

### Unstructured

Nós se conectam de forma ad-hoc; descoberta via gossip ou flooding. Baratos de manter, caros pra buscar coisas específicas (lookup é O(N) no pior caso).

Exemplos: Gnutella (legado), gossip em Bitcoin/Ethereum, Secure Scuttlebutt.

Melhor pra: broadcast, replicação oportunística, mensagens sociais.

### Structured (DHT)

Nós organizados por ID, com regras determinísticas de roteamento. Lookup em O(log N). Custa manutenção (tabelas de roteamento, handle churn).

Algoritmos clássicos:

- **Kademlia** (Maymounkov & Mazières, 2002) — distância XOR entre IDs. Cada nó mantém k-buckets por prefixo de distância. Dominante na prática (IPFS, BitTorrent DHT, libp2p, Ethereum devp2p).
- **Chord** (Stoica et al.) — anel com finger table. Elegante, menos robusto que Kademlia a churn.
- **Pastry, Tapestry** — prefix routing. Influenciaram design de overlays.
- **CAN** (Content-Addressable Network) — espaço d-dimensional, cada nó dono de uma zona.

Melhor pra: lookup de chave específica, content addressing, peer discovery em escala.

### Hybrid / super-peer

Alguns nós com mais responsabilidade (relays, trackers, signaling). Mais eficiente que puro, menos resistente a censura.

Exemplos: BitTorrent com trackers (pré-DHT), Skype original (supernodes), Matrix federado (homeservers são super-peers), cluster WebRTC com SFU.

### Federado ≠ P2P

**Federado** (Matrix, Mastodon, email): múltiplos servidores, cada user tem servidor-lar. Servidores falam entre si, users falam com seu servidor.

**P2P puro**: sem servidor; cada user é peer.

Federado é muito mais fácil de fazer funcionar; P2P puro é mais resistente mas sofre com mobile (bateria, NAT, conexão intermitente). **Friend-to-friend (F2F)** é P2P só entre peers que se conhecem (Briar, Retroshare).

## Descoberta (bootstrap e peer discovery)

O problema fundacional: "como peer A acha peer B sem servidor central?"

### Bootstrap

- **Seed nodes hardcoded**: endereços IP/DNS conhecidos no código/config. Bitcoin faz isso (`seed.bitcoin.sipa.be` etc). Menos "puro" mas pragmático.
- **DNS seeds**: consulta DNS retorna lista de peers ativos. Mais flexível que hardcode.
- **Rendezvous server**: pequeno servidor só para apresentar peers (não carrega conteúdo). Minimiza dependência central mas não elimina.
- **Multicast/mDNS**: descoberta em LAN (Bonjour/Avahi). Zero-conf local.
- **Pré-compartilhamento out-of-band**: QR code, convite por link (Signal safety numbers, Briar).

Depois do primeiro peer: **crawl**. Peça lista de peers ao bootstrap, conecte, peça mais, expanda.

### DHT-based discovery

Com DHT rodando, descoberta vira lookup de chave: "quem tem conteúdo com hash X?" → DHT devolve peers. BitTorrent DHT (Mainline), IPFS Amino DHT, Ethereum Discovery.

### Gossip-based discovery

**HyParView + Plumtree** (Leitão et al.): membership via HyParView (mantém partial view ativa + passiva), broadcast via Plumtree (árvore otimizada dentro do gossip).

**SWIM** (Das et al.): gossip-style failure detection + membership. Usado em Serf/Consul, Hashicorp stack. Escala bem para milhares de nós.

**Epidemic (anti-entropy)**: cada nó compara estado com peer aleatório periodicamente. Convergência probabilística. Cassandra, DynamoDB usam.

## NAT traversal

95% dos users está atrás de NAT (home router, carrier-grade NAT móvel). Dois peers atrás de NATs diferentes não podem simplesmente abrir TCP um pro outro.

### Técnicas

- **STUN** (Session Traversal Utilities for NAT): servidor público ajuda peer a descobrir seu IP:port externo. Simples, barato, mas **falha em NAT simétrico** (porta muda por destination).
- **TURN** (Traversal Using Relays around NAT): servidor faz relay do tráfego. Funciona sempre mas não é mais P2P (tráfego passa por servidor, custa banda). Usado como fallback.
- **ICE** (Interactive Connectivity Establishment): framework que tenta STUN primeiro, TURN depois, combinando candidates. Padrão em WebRTC.
- **UDP hole punching**: ambos peers enviam pacote simultaneamente ao outro; NAT pensa que é resposta e abre caminho. Funciona em NAT cone (full, restricted, port-restricted), falha em simétrico.
- **TCP hole punching**: mais difícil, funciona com ajuda de STUN TCP.
- **UPnP / NAT-PMP / PCP**: pede ao router pra abrir porta. Dependente de router, muitas vezes desabilitado por segurança.
- **Relayed via super-peer**: peer confiável com IP público faz relay. Solução de fallback universal.
- **IPv6**: elimina NAT — cada device tem IP público. Adoção crescendo mas não universal.

### Tipos de NAT (RFC 3489 / 4787)

- **Full cone**: mapeia interno → externo, aceita pacote de qualquer IP externo. Mais permissivo.
- **Restricted cone**: aceita de IPs externos que interno já contatou.
- **Port-restricted cone**: idem + porta específica.
- **Symmetric**: porta externa muda por destino. **Pior caso**; hole punching falha.

Carrier-grade NAT (CGNAT, operadoras móveis) frequentemente é simétrico. Por isso TURN é quase sempre necessário como fallback em apps mobile.

## Transportes

- **TCP**: confiável, flow control, head-of-line blocking. Bom pra transferência bulk.
- **UDP**: não-confiável, baixo overhead, base de QUIC/WebRTC. Melhor pra real-time e custom congestion control.
- **QUIC** (sobre UDP): multiplexação sem head-of-line, 0-RTT handshake, migração de conexão. Base do HTTP/3. libp2p tem transporte QUIC.
- **WebRTC Data Channel**: P2P no browser. DTLS + SCTP sobre UDP (ou TCP fallback). Ordered ou unordered, reliable ou não. Interface natural para P2P browser-side.
- **WebSocket**: não é P2P mas usado pra signaling e fallback de browser.
- **Tor hidden service / I2P / Yggdrasil**: camadas de overlay anônimas. P2P sobre redes privacy-preserving.

## Content addressing

Ideia central: endereço do dado é **hash do dado**, não URL mutável. Mesmo conteúdo = mesmo endereço, onde quer que esteja.

- **CID** (Content Identifier) em IPFS: multihash + multicodec + multibase. `QmXoypizjW...` em base58 (v0) ou `bafybei...` em base32 (v1).
- **Merkle DAG**: estrutura acíclica onde nós apontam para filhos por hash. Mudança em folha propaga hashes acima (cada nível inclui hash dos filhos).
- **Verificação barata**: download o bloco, hash, compara com CID. Qualquer peer pode servir; integridade não depende de confiar em quem enviou.
- **Deduplicação natural**: dois arquivos idênticos têm mesmo CID. IPFS não armazena duplicado.

### Chunking

Arquivo grande dividido em blocos (típico 256KB-1MB). Cada bloco tem CID; árvore Merkle sobre blocos dá CID do arquivo todo. Permite download paralelo, resume, dedup parcial.

**Rolling hash (Rabin, FastCDC)** pra content-defined chunking: limites de bloco dependem do conteúdo, não da posição. Mudança em começo do arquivo não invalida resto da árvore (rsync-style).

### Limitações

- **Imutável** por design. Atualizar = novo CID. Precisa camada extra pra "versão atual" (IPNS, DNSLink, DIDs).
- **Popularidade importa**: bloco que ninguém pediu some da rede. IPFS não garante persistência — precisa *pinning* explícito (pinata, web3.storage, self-host).
- **Privacidade**: hash é determinístico — dois nós com mesmo conteúdo revelam que têm. Precisa encriptar antes se privacidade importa.

## Gossip protocols

Epidemic protocols: informação se espalha como vírus. Cada rodada, nó contata K peers aleatórios, troca info. Convergência em O(log N) rodadas com alta probabilidade.

### Tipos

- **Push**: nó ativo empurra update.
- **Pull**: nó pergunta "tem novidade?".
- **Push-pull**: ambos. Convergência mais rápida.
- **Anti-entropy**: periódico, reconcilia estado completo. Pesado mas completo.
- **Rumor mongering**: espalha só updates novos, para após N repetições sem info nova. Barato mas pode perder updates.

### Plumtree

Gossip constrói **árvore de broadcast eager** sobrepondo rede, com laterals lazy pra recuperação. Traffic no steady-state é O(N) (árvore), não O(N²) (gossip full).

### Aplicações

- **Cluster membership** (SWIM, Serf, Consul, Cassandra).
- **State reconciliation** (Amazon Dynamo, Riak).
- **Blockchain tx / block propagation** (Ethereum, Bitcoin).
- **CRDT sync** (Automerge, Yjs em modo gossip).

### Parâmetros

- **Fanout (K)**: peers contatados por rodada. Típico 3-6.
- **Rodada**: frequência (ex: 1s). Latência vs bandwidth trade-off.
- **TTL**: limite de hops — previne loop em push.

## Sincronização e replicação

Problema: N peers têm cópias divergentes de dados; como convergir sem coordenador?

### CRDT (Conflict-free Replicated Data Types)

Tipos de dado cujas operações são **comutativas, associativas, idempotentes** — ordem de aplicação não importa. Merge é determinístico.

Tipos:

- **Counter** (G-Counter, PN-Counter).
- **Set** (G-Set, 2P-Set, OR-Set, LWW-Set).
- **Register** (LWW-Register, MV-Register).
- **Sequence** (RGA, LSEQ, Logoot, YATA, Peritext) — colaboração em texto.
- **Map, List, Tree** — compositions.

**State-based (CvRDT)**: nós trocam estado completo (ou diffs); merge via join (lattice). Simples, mas pesado.

**Operation-based (CmRDT)**: trocam ops; exige delivery exactly-once em ordem causal. Broadcast confiável obrigatório.

**Delta-state**: meio-termo; diffs pequenos em vez de estado completo.

Libs: **Automerge** (JS/Rust, document-oriented), **Yjs** (JS, text-heavy, muito usado), **Redis CRDTs**, **AntidoteDB**, **Riak**.

Trade-off: não há linearizability (qualquer op convergir). Ok pra muitos casos (colab em texto, presença, contadores), não pra todos (saldo bancário).

### OT (Operational Transformation)

Predecessor dos CRDTs. Ops são transformadas contra ops concorrentes antes de aplicar (ex: insert em posição i ajustado se alguém inseriu em i-k). Histórico famoso em Google Docs. Complicado de implementar corretamente — CRDTs ganharam mindshare.

### Merkle sync / diff

Árvore Merkle sobre estado. Peers comparam root hash: iguais → sync. Diferentes → comparam children, desce recursivamente até achar blocos divergentes. Custo O(log N) quando mudanças são localizadas.

Usado em: Cassandra (anti-entropy via Merkle), Dynamo, Scuttlebutt, Git (árvore de objetos).

### Vector clocks / version vectors

Mapa peer → contador. Cada op incrementa contador do autor. Compara dois vetores: um domina outro (causally after), iguais, ou **concorrentes** (conflito). Base de causality tracking em distributed systems.

**Hybrid Logical Clocks (HLC)**: combina physical time + logical counter. Usado em CockroachDB, MongoDB.

### Dat/Hypercore (append-only logs)

Cada peer tem **log assinado** (append-only, imutável), identificado por chave pública. Outros peers replicam. Merkle tree sobre log dá verificação de slice arbitrário.

Aplicações: Dat (arquivos), Scuttlebutt (social), Cabal (chat), Earthstar. Naturalmente distribuído, resistente a revogação, mas *esquecer* é difícil (right to erasure vs append-only).

## Integridade

### Hashing e signatures

- **Hash** do conteúdo (SHA-256, BLAKE3) verifica integridade contra corrupção.
- **Signature** do hash (Ed25519 típico) autentica autor.
- **Content addressing** elimina necessidade de signature pra integridade (hash é endereço).
- **Signed log** (hypercore-style): cada append inclui signature do log todo até aquele ponto → peer não pode reescrever histórico sem detectar.

### Merkle proofs

Provar que bloco X pertence a árvore com root R em O(log N) hashes (irmãos no path). Base de:
- Certificate Transparency
- Light clients em blockchain
- Verified BitTorrent pieces
- Sparse updates em IPLD

### Verifiable computation

Peer executou computação e pode provar o resultado:
- **zk-SNARKs / zk-STARKs**: prova de que execução está correta sem revelar inputs. Caro pra produzir; barato pra verificar.
- **Optimistic rollup / fraud proofs**: assume correto, challenge em janela de disputa.

Aplicações crescentes em blockchains (L2s), e começando em P2P para computação descentralizada genuína.

## Segurança — ataques e defesas

P2P é **adversarial by default**. Qualquer um pode entrar.

### Ataques

- **Sybil attack**: adversário cria múltiplas identidades falsas pra ganhar influência desproporcional. Sem custo de identidade, é ilimitado. Contramedidas: PoW (Bitcoin), PoS (Ethereum), rate-limiting por IP (fraco), proof of personhood (Worldcoin, BrightID; controverso), web of trust (PGP, SSB).
- **Eclipse attack**: adversário cerca um peer com nós maliciosos, controlando toda a visão dele da rede. Mitigações: peer selection diversificado (random + low-latency + prefix-diverse), persistência de peers confiáveis entre restarts, múltiplas DHT lookups independentes.
- **Routing attack / black hole**: nó malicioso aceita requests e dropa. DHT Kademlia mitiga parcialmente com lookup paralelo a múltiplos peers (α, geralmente 3).
- **Partition attack**: split da rede em subgrafos que não se falam. Grave em blockchain (double-spend possível em partição).
- **Poisoning attack**: injetar dados ruins associados a chave legítima. Content addressing elimina (hash não bate); em sistemas mutáveis, exige signature.
- **DoS / amplification**: floodar peer com requests. Rate limiting, proof-of-work puzzle, reputação.
- **Man-in-the-middle**: interceptar comunicação. TLS / Noise Protocol Framework / libp2p secure channels (TLS 1.3, Noise).
- **Deanonymization**: correlação de tráfego, timing analysis, IP leakage. Tor/I2P tentam mitigar via onion/garlic routing; imperfeitos contra adversário global.
- **Bitswap starvation / free-rider**: peer baixa sem compartilhar. BitTorrent resolve com *tit-for-tat* (choke/unchoke baseado em reciprocity). IPFS Bitswap tem reputação leve.

### Sybil resistance — teorema de Douceur (2002)

Sem custo pra criar identidades e sem autoridade central, Sybil é intratável. Quatro rotas práticas:

1. **Custo computacional** (PoW).
2. **Custo econômico** (PoS, stake).
3. **Autoridade centralizada** (DIDs, CAs) — compromete P2P puro.
4. **Social graph / web of trust** — caro pra atacante replicar, mas frágil.

Projetos P2P "puros" sem Sybil resistance viável são inerentemente vulneráveis. Admita ou mitigue.

### Defesas arquiteturais

- **Encrypt-by-default**: payloads sempre criptografados end-to-end. Nunca trust no transport.
- **Signed messages**: autoria verificável. Rejeite qualquer coisa não assinada.
- **Verify before relay**: nó não propaga mensagens mal formadas ou inválidas (spam mitigation).
- **Allow-listing**: em F2F, só peers convidados.
- **Audit logs**: cada peer mantém log assinado do que recebeu — detectar fork/rewriting retroativo.
- **Reputation**: local (meu peer lembra quem me serviu bem) ou rede (gossip de scores — caro, atacável).

## Reputação e trust

Problema: num grupo de estranhos, em quem confiar?

### Modelos

- **Web of Trust** (PGP): confiança transitiva por assinaturas. Ainda útil pra keybase-style, muito fricção pra users.
- **EigenTrust** (Kamvar et al.): PageRank sobre grafo de feedback. Iterativo, converge. Vulnerável a conluio se atacantes têm rating mútuo alto.
- **Tit-for-tat** (BitTorrent): reciprocidade direta. Simples, eficaz em file sharing.
- **Proof-of-behavior**: stake que pode ser "slashed" se mau comportamento (PoS em blockchains).
- **Verifiable credentials / DIDs (W3C)**: identidade descentralizada com atributos assinados por issuers confiáveis.
- **Social graph delegation**: confio em amigos; confio parcialmente em amigos de amigos (Scuttlebutt usa profundidade limitada).

### Limite

Sem custo de identidade ou ancoragem em algo caro (compute, stake, social capital), reputação colapsa em adversarial setting. Accepte ou adicione ancoragem.

## Processamento distribuído (P2P compute)

Além de dados: **dividir trabalho entre peers**.

### Sistemas clássicos

- **SETI@home** (1999-2020): clientes baixavam pedaços de sinal de radiotelescópio, processavam, subiam resultado. Validação via redundância (mesma tarefa pra 3 clientes, maioria vence). Não é P2P puro — servidor central distribuía trabalho.
- **BOINC** (plataforma generalizada de SETI@home): Folding@home, Einstein@home, Rosetta@home, etc.
- **Folding@home**: protein folding; >1 exaFLOP em 2020 durante COVID-19.

Padrão: **coordinator + workers**. Fácil de escalar, nada resistente a censura.

### P2P compute puro

Mais difícil. Problemas:
- **Verificar resultado sem refazer**: exige prova (zk-proof, redundância majoritária, verifiable computation).
- **Incentivo**: por que peer gastaria CPU? Token (mining), reputação, altruísmo.
- **Alocação**: quem executa qual task? DHT, market-based matching.

Tentativas modernas: **Golem** (GPU market), **Akash** (kubernetes descentralizado), **Bacalhau** (compute over IPFS data), **Gensyn** (ML training), **Bittensor** (ML inference). Sérias em nicho; ainda distantes de substituir cloud generalizada.

### Edge / fog computing

Adjacente: deslocar compute pra próximo do user (CDN-like). Cloudflare Workers, AWS Lambda@Edge. **Não é P2P** estritamente — infraestrutura ainda é centralizada, só distribuída geograficamente. Mas padrões de design (stateless, CRDT-friendly) convergem.

### Federated learning

Treino de ML distribuído onde dados não saem do device. Server coordena; devices treinam localmente, sobem **gradientes** (não dados), server agrega.

**Flavors**:

- **Centralized FL** (Google Gboard): server coordinator. Não é P2P.
- **Decentralized FL / swarm learning** (Hewlett Packard Enterprise): peers trocam gradientes via blockchain/gossip, sem server. P2P verdadeiro.
- **Split learning**: modelo dividido; device roda camadas baixas, server camadas altas.

Problemas reais: privacy (gradientes vazam info — **differential privacy** + **secure aggregation**), heterogeneidade de devices, non-IID data, Byzantine attacks (nó malicioso envia grad envenenado — defesas: Krum, trimmed mean, Bulyan).

## Frameworks e bibliotecas

### libp2p

Biblioteca modular extraída do IPFS (Protocol Labs). Implementações em Go, Rust, JS, Nim, C++. Abstrai:
- Transports (TCP, QUIC, WebSocket, WebRTC, WebTransport)
- Secure channels (TLS 1.3, Noise)
- Multiplexing (yamux, mplex)
- Peer IDs (hash de public key)
- Discovery (mDNS, DHT, rendezvous, bootstrap)
- Pubsub (GossipSub — gossip otimizado com meshes, usado em Ethereum consensus layer, Filecoin)

Base de: IPFS, Filecoin, Polkadot, Ethereum 2 consensus, muitos outros.

### Hyperswarm / Hypercore protocol

Ecossistema do projeto Dat/Holepunch. Hypercore (append-only log assinado), Hyperbee (kv sobre hypercore), Hyperdrive (fs). Hyperswarm pra peer discovery via DHT. Forte em JS/Node, integração browser OK.

### WebRTC stack

`simple-peer`, `PeerJS`, `trystero` (multi-backend via torrent trackers/IPFS/Nostr). Browser-first. Use pra apps client-side verdadeiramente P2P.

### GunDB

DB graph real-time sincronizado via CRDT-like convergence. Controverso — reivindicações de P2P puro ambíguas (precisa relay peers pra funcionar bem), garantias de consistência fracas. Bom pra protótipo, cuidado em produção.

### OrbitDB

DB distribuído sobre IPFS. Eventos em hypercore-like log, CRDT merges. Mais honesto sobre garantias que GunDB, menos maduro.

### Earthstar

Local-first, user-controlled workspaces. Pequeno, opinativo. Focado em small groups (classes, famílias, clubes).

### Automerge / Yjs

Focam em CRDT para colaboração. Transporte agnóstico (WebRTC, WebSocket, libp2p, whatever). Yjs é o padrão de facto em edição colaborativa em 2024-2026 (Notion-like, Tiptap, BlockNote).

### Nostr

Protocolo simples de event publishing assinado. "Clients post signed JSON events to relays". Relays são servidores dumb; resiliência vem de múltiplos relays. **Federado-ish**, não P2P puro, mas muito mais simples que alternativas. Adoção real via apps (Damus, Amethyst).

### Scuttlebutt (SSB)

Replicação append-only + social graph. "Gossip + crypto = social". Offline-first sólido. Adoção modesta mas fiel.

### Matrix

**Federado** (não P2P puro). Homeservers são peers federados; users têm conta em um home server. P2P experimental (Pinecone) em desenvolvimento.

### Tor / I2P

Overlay de privacidade. Tor: onion routing; acesso a serviços `.onion` (hidden services) é P2P anônimo. I2P: mais puro, focado em serviços internos; Garlic routing.

## Casos reais (como cada um resolveu o quê)

### BitTorrent

- **Tracker** (central, legado) ou **DHT Mainline** (Kademlia) pra descoberta.
- **Peer exchange (PEX)**: peers trocam listas de outros peers.
- **Pieces** (~256KB-4MB) com Merkle tree (BT v2) ou flat hash list (v1).
- **Tit-for-tat**: reciprocidade com choke/unchoke.
- **Rarest first**: prioridade pra pieces raros → distribuição uniforme.

Resiliente, eficiente, mas sem anonimato nem autenticação de conteúdo (info_hash é o que garante integridade).

### IPFS

- **Content addressing** (CID).
- **libp2p** para transporte/discovery.
- **Bitswap** para troca de blocos (pedido-resposta, tit-for-tat leve).
- **DHT Kademlia** (Amino) para achar peers com bloco.
- **MFS** + **IPNS** / **IPLD** para mutabilidade em cima do imutável.

Forte em imutabilidade e content routing; fraco em persistência sem pinning e em latência de lookup (2-10s comuns).

### Bitcoin / Ethereum

- **Gossip** flat para tx e block propagation (Bitcoin usa compact block relay, BIP152; Ethereum GossipSub).
- **PoW (Bitcoin) / PoS (Ethereum)** resolve Sybil + consensus.
- **Blockchain + Merkle tree** para auditability.
- **Sybil cost** = hardware ou stake.

Caro e lento (por design), mas byzantine-tolerante a nível que nenhum sistema P2P "clássico" alcança.

### Tor

- **Onion routing** em 3 hops.
- **Directory authorities**: lista de relays assinada (ponto semi-centralizado).
- **Hidden services**: endereço `.onion` é hash de public key; rendezvous via DHT.

Trade-off: latência alta, privacidade forte. Não é P2P puro (authorities), mas arquitetura influencia muitos projetos.

### Briar

F2F puro. Contacts só via QR code / link. Sem servidor central. Tor para transport. Store-and-forward entre contacts quando online. Ideal pra contextos hostis (ativismo, guerra).

## Browser P2P — peculiaridades

- **Restrições**: sem raw sockets (precisa WebSocket/WebRTC/WebTransport), sem serving porta, CORS, limites de storage.
- **WebRTC Data Channel** é o transporte viável; exige signaling (server, mesmo que pequeno).
- **Fechou aba = saiu da rede**. Persistência limitada (Service Worker só roda em background com event trigger).
- **Libs**: `libp2p-js` (mais completo), `simple-peer`, `Hyperswarm Web`, `js-ipfs`/`helia`.
- **WebTransport (HTTP/3)**: interface moderna, ainda em adoção.
- **NAT traversal via STUN público**: Google STUN (`stun.l.google.com:19302`) etc. TURN exige self-host ou pago.

Realidade: "100% browser-only P2P" é viável mas limitado; apps sérios usam extension ou desktop companion.

## Métricas e observabilidade

O que medir em P2P saudável:

- **Peer count (total, ativos)**: baseline de saúde.
- **Churn rate**: peers entrando/saindo por min. Alto churn = rede instável.
- **Routing table completeness**: k-buckets cheios?
- **Lookup latency**: tempo médio pra achar CID/key.
- **Hop count**: médio e p99 em routing.
- **Bandwidth in/out**: por peer, por tipo de mensagem.
- **Bitswap duplicate blocks**: bloco baixado duas vezes = ineficiência.
- **Failed dials / connection success rate**: saúde de NAT traversal.
- **Block propagation time**: em blockchain/pubsub.
- **Partition detection**: peers que param de ouvir o grupo.

Tooling: **libp2p metrics** (Prometheus), **ipfs-inspect**, **GossipSub tracer**, custom. Bastante imaturo comparado a observability centralizada.

## Arquiteturas — padrões de projeto

### Coordinator-free (puro)

Nenhum papel especial. Kademlia-based file sharing, Bitcoin. Max resistência, max complexidade.

### Super-peers

Alguns nós com mais recursos (uptime, banda) carregam DHT, gossip, relay. Pragmático; escala melhor.

### Local-first

Dispositivo é fonte de verdade; sync é opcional/oportunístico. CRDTs + hypercore/Automerge. Filosofia "Local-first software" (Ink & Switch, Kleppmann et al. 2019). Ganhando tração.

### Hub-and-spoke

Peers falam só com "hub" conhecido (não-P2P na essência). Usado quando requisitos de censura são fracos mas descentralização é desejável (Mastodon é federado, não P2P).

### Onion / multi-hop

Cada mensagem passa por N intermediários. Anonimato via desvinculação. Tor, I2P.

### Offline-first / store-and-forward

Mensagem fica em peer intermediário até destino aparecer. Bluetooth mesh (Briar), sneakernet, delay-tolerant networks. Crítico em conectividade ruim.

## Trade-offs fundamentais

1. **Descentralização ↔ performance**: mais decentralizado → mais hops, mais verificação, mais latência.
2. **Privacidade ↔ observabilidade**: debug de P2P privado é dolorosíssimo.
3. **Resistência ↔ usabilidade**: instalação de app nativo, gestão de chaves, bootstrap — users normais abandonam.
4. **Imutabilidade ↔ atualização**: content addressing é lindo mas "como atualizo perfil?" exige mutable pointer, que é ponto central.
5. **Custo de identidade ↔ adoção**: Sybil resistance requer barreira; users não querem pagar/esperar/provar personhood.

Muito projeto P2P morre por subestimar esses. Desenhe sabendo.

## Princípios

1. **Escolha P2P por razão técnica, não ideológica**. "Porque é descentralizado" não é requisito.
2. **Content addressing é trunfo**. Onde couber, use — integridade vira grátis.
3. **Sybil resistance não é opcional**. Tenha um plano.
4. **NAT é realidade**. Se ignorar, metade dos users não conecta. TURN fallback é pragmático.
5. **CRDT > OT em 2026**. Menos código, composable, transport-agnostic.
6. **Reputação local é tratável; reputação global é frágil**. Projete conservador.
7. **Offline-first é feature, não bug**. Users aceitam latência de sync se app funciona offline.
8. **P2P puro em browser é romântico; P2P pragmático usa relay mínimo**. Reconheça.
9. **Federação resolve 80% dos problemas de "centralização" com 20% da complexidade de P2P puro**. Considere antes de partir pro fundo.
10. **Observabilidade é desafio não-trivial**. Planeje desde começo.

## Recursos

- **"Peer-to-Peer Systems and Applications"** — Steinmetz, Wehrle (ed.) — livro acadêmico.
- **Papers clássicos**: Kademlia (Maymounkov 2002), Chord (Stoica 2001), CAN (Ratnasamy 2001), Gnutella, BitTorrent (Cohen 2003), SWIM, Plumtree, HyParView.
- **"Local-first software"** — Kleppmann, Wiggins, van Hardenberg, McGranaghan (2019). Manifesto.
- **libp2p docs** — `docs.libp2p.io`. Moderna e viva.
- **IPFS docs** — `docs.ipfs.tech`.
- **Hypercore protocol docs** — `docs.holepunch.to`.
- **Ink & Switch research** — `inkandswitch.com`. Pesquisa sobre local-first, CRDTs aplicados.
- **"Designing Data-Intensive Applications"** — Kleppmann. Distribuído em geral, relevante aqui.
- **Ethereum devp2p / GossipSub specs** — referência prática moderna.
- **Awesome-p2p, awesome-distributed-systems** — listas curadas.
