# Criptografia Avançada

Complementa `CRYPTOGRAPHY.md` (fundamentos). Este arquivo trata de protocolos modernos, misuse patterns, side channels e o horizonte pós-quântico — temas onde grande parte das vulnerabilidades criptográficas reais acontece hoje.

> Regra zero: **nunca implemente primitivas criptográficas do zero**. Use libs maduras (libsodium, BoringSSL, OpenSSL, Tink, age, `ring`). Crypto implementada "para aprender" é insegura em 100% dos casos não-acadêmicos.

## AEAD — O Que Usar Hoje

**AEAD** (Authenticated Encryption with Associated Data) combina confidencialidade + integridade. Substitui encryption-then-MAC ou MAC-then-encryption (ambos propensos a erro).

Opções seguras em 2026:

1. **ChaCha20-Poly1305**: não exige AES-NI em hardware; constante em tempo por design. Padrão em TLS 1.3, WireGuard, libsodium.
2. **AES-GCM**: padrão em hardware moderno (AES-NI, ARMv8 crypto). Cuidado: nunca reutilizar nonce com mesma chave — catastrófico.
3. **AES-GCM-SIV**: deterministic-authenticated; tolerante a nonce reuse em certo grau. Bom quando nonce random é difícil.
4. **XChaCha20-Poly1305**: nonce de 192 bits; random seguro.

**Evitar**: AES-CBC (padding oracle), AES-ECB (nunca), RC4 (quebrado), DES/3DES (depreciado), MD5/SHA-1 (colisão prática).

## TLS 1.3

Reforma substancial do protocolo. Diferenças principais vs 1.2:

- **Handshake em 1 RTT** (vs 2 em 1.2). `0-RTT` opcional — veja caveats.
- **Só ciphersuites AEAD** — elimina CBC, RC4, MD5.
- **Ephemeral-only key exchange** — forward secrecy obrigatório. Removido RSA key transport.
- **Handshake criptografado** exceto `ClientHello` (mesmo assim, extension **Encrypted Client Hello — ECH** protege SNI).
- **HKDF** (HMAC-based key derivation) consistente ao longo do protocolo.
- **Menos opções** → menos erros de configuração.

### 0-RTT

Cliente envia dados no primeiro pacote, usando chave pré-compartilhada de sessão anterior. Trade-off: **replayable**. Não usar para operações não-idempotentes (POSTs mutativos). Muitos perfis desabilitam.

### Certificates

TLS depende de PKI. Em 2026, ecossistema está em:

- **ACME** (Let's Encrypt, ZeroSSL) para emissão automatizada.
- **Certificate Transparency** obrigatório para CAs no Chrome root store. Monitorar CT logs (`crt.sh`) para emissões inesperadas no seu domínio.
- **CAA record** no DNS restringe quais CAs podem emitir.
- **OCSP Stapling** + **Must-Staple**: revogação checada no handshake.
- **Short-lived certs** emergindo (7-90 dias) — rotação > revogação.

### Ataques Históricos (TLS 1.0/1.1/1.2)

Saber o nome porque ainda aparecem em scans:

- **POODLE**: SSLv3, CBC padding.
- **BEAST**: TLS 1.0, IV previsível em CBC.
- **CRIME**: compressão + cookie leak.
- **BREACH**: compressão em HTTP response + CSRF token.
- **Lucky13**: timing em CBC padding.
- **Heartbleed**: bug em OpenSSL (não no protocolo).
- **FREAK, Logjam**: downgrade para export crypto.
- **DROWN**: SSLv2 em servidor afeta chave compartilhada.

**Mitigação geral**: ficar em TLS 1.2 com ciphers modernos OU TLS 1.3; desabilitar tudo mais velho.

## Noise Protocol

Framework para construir protocolos de handshake; base de **WireGuard, Signal, Lightning Network**. Mensagens de handshake são compostas de tokens (`e`, `s`, `ee`, `es`, etc.) que combinam Diffie-Hellman ephemeral/static. Resultado: chave compartilhada com propriedades desejadas (forward secrecy, identity hiding).

Padrões: `Noise_XX`, `Noise_IK`, etc. — cada letra descreve qual parte envia qual chave, quando.

Para casos onde TLS é complexo demais (UDP, RPC customizada), Noise é o caminho moderno.

## Key Derivation

Transformar um valor (senha, segredo compartilhado) em chaves criptográficas.

- **HKDF** (HMAC-based): padrão para derivar de um já-aleatório bom (master secret de handshake). Extract-then-expand.
- **Argon2id**: de senhas. Resistente a GPU/ASIC. Ver `AUTHENTICATION.md`.
- **scrypt, bcrypt, PBKDF2**: alternativas; Argon2id é state-of-the-art para novos sistemas.

Erro comum: usar SHA-256 direto como "KDF" de senha. Velocidade é inimiga do defensor aqui.

## Nonces e IVs

Muitos modos exigem **nonce** (number used once). Três fontes:

- **Random**: sortear a cada mensagem. Exige espaço grande (96 bits no AES-GCM é marginal; 192 do XChaCha20 é seguro).
- **Counter**: manter contador monotônico por chave. Preciso; vazamento de estado é catástrofe.
- **Derivado**: de contexto (session ID + counter).

**Reutilizar nonce com mesma chave** em GCM: o atacante recupera XOR de dois plaintexts + forja MAC arbitrariamente. Catastrófico. Nunca.

## RNG — Números Aleatórios

Tudo descamba sem RNG bom.

- **`/dev/urandom`** (Linux moderno): seguro desde boot (após pool ready). Use direto.
- **`getrandom(2)`**: syscall que bloqueia se pool não pronto.
- **`CryptGenRandom` / `BCryptGenRandom`** (Windows).
- **Libs**: `secrets.token_bytes()` (Python), `crypto/rand` (Go), `ring::rand` (Rust). Sempre essas sobre `random`/`math.random`.

**Erros históricos**:

- **Debian OpenSSL 2006-2008**: patch errado reduziu entropia a 15 bits. Chaves SSH/SSL geradas em Debian nessa janela eram enumeráveis.
- **Dual_EC_DRBG**: backdoor NSA via curvas NIST. Depreciado.
- **Javascript `Math.random()`**: NÃO criptográfico. Use `crypto.getRandomValues`.

## Curvas Elípticas

- **NIST P-256, P-384, P-521**: amplamente usadas; implementações podem ter issues de timing. OK com libs sérias.
- **Curve25519** (X25519 para KEX, Ed25519 para assinatura): desenhada por djb, resistente a implementação ruim, rápida. Padrão em sistemas modernos (SSH, Signal, WireGuard).
- **secp256k1**: Bitcoin, Ethereum. Não otimizada para general-purpose; use apenas onde exigido.

## Signatures

- **RSA-PSS**: se precisar RSA, use PSS padding (não PKCS#1 v1.5).
- **Ed25519**: simples, rápido, resistente a RNG ruim na assinatura.
- **ECDSA**: requer RNG bom a cada assinatura — falhas de RNG vazam chave privada (ex.: PS3, early Android wallets). **Deterministic ECDSA** (RFC 6979) resolve.

## Zero-Knowledge Proofs

Prova que uma afirmação é verdadeira sem revelar informação além da validade.

- **Snarks** (zk-SNARKs): sucintos, não-interativos. Trusted setup em algumas variantes.
- **Starks**: sem trusted setup, maiores.
- **Bulletproofs**: range proofs eficientes.

Aplicações: criptomoedas de privacidade (Zcash), rollups Ethereum, credentials (BBS+).

## Homomorphic Encryption

Operações sobre ciphertext sem decifrar.

- **Partially** (PHE): uma operação (adição OU multiplicação). Paillier.
- **Somewhat** (SHE): limitado em depth.
- **Fully** (FHE): qualquer operação. BGV, BFV, CKKS, TFHE. Lento; prático só em casos específicos (privacy-preserving ML em ciphertext).

Frameworks: **SEAL, OpenFHE, TFHE-rs**.

## MPC — Multi-Party Computation

N partes computam sobre suas entradas sem revelá-las. SPDZ, GMW protocols. Threshold signatures (TSS) para wallets: chave privada nunca existe inteira.

## Post-Quantum Cryptography

Algoritmos de Shor quebram RSA, DH, ECC em computador quântico suficientemente grande. Ainda não existe. **Mas**: dados encriptados hoje podem ser armazenados e decriptados quando chegar ("harvest now, decrypt later"). Sistemas de longo prazo precisam migrar.

NIST PQC process (2016-2024) selecionou:

- **ML-KEM** (Kyber): key encapsulation.
- **ML-DSA** (Dilithium): assinatura.
- **SLH-DSA** (SPHINCS+): hash-based signature, backup conservador.
- **FN-DSA** (Falcon): assinatura, menor tamanho que Dilithium, implementação mais difícil.

**Migração**: modo **híbrido** — combinar clássico + PQ (ex.: X25519 + Kyber). Protege se qualquer dos dois falhar. Browsers Chrome/Firefox já suportam híbrido em TLS 1.3.

Linha temporal: sistemas de longo prazo (chip embarcado, infra crítica) devem migrar agora. Aplicações web rotativas podem esperar maturação das libs. CNSA 2.0 (NSA) exige PQ em sistemas government US até 2030-2035 dependendo da categoria.

## Side Channels

**Ataques em informação não-intencional** — timing, cache, energia, eletromagnetismo.

### Timing

Operação que varia em tempo com o segredo vaza o segredo.

- **String compare** em autenticação: `==` curto-circuita no primeiro byte diferente. Atacante mede timing e descobre byte a byte. Use **constant-time compare** (`hmac.compare_digest` em Python, `subtle.ConstantTimeCompare` em Go).
- **Modular exp** em RSA: sem blinding, tempo depende de bits da chave.
- **Branching em chave**: `if key[i] == 0 then X else Y`.

### Cache

Atacante mede tempo de acesso a memória compartilhada (cache hit vs miss) — deduz o que a vítima acessou.

- **Flush+Reload, Prime+Probe**: técnicas clássicas.
- **AES software** implementações com lookup tables vazam via cache. Use AES-NI ou bitsliced.

### Spectre / Meltdown (2018)

Execução especulativa da CPU deixa resíduos no cache, legíveis via side channel.

- **Meltdown** (CVE-2017-5754): ler memória do kernel de user mode.
- **Spectre v1** (bounds check bypass), **v2** (branch target injection), variantes novas ao longo dos anos.
- **Mitigações**: microcode updates, KPTI (isolação de page tables), `retpolines`, LFENCE em pontos sensíveis. Perf hit em 5-30%.

Lições: microarquitetura é superfície de ataque; pureza de abstração de ISA é uma ficção.

### Rowhammer

Memória DRAM: acessar uma linha repetidamente causa bitflips em linhas vizinhas. Escrita sem permissão via side effect. Atinge até DDR4; ECC reduz mas não elimina.

### Fault Injection

Indução de erro em hardware (glitch de voltagem/clock/EM) causa instrução errada — frequentemente pula verificação ou escreve byte errado. Ataque a smartcards, HSMs. Ver `HARDWARE_ATTACKS.md`.

### Acoustic, EM, Power

- **Chaves RSA de 4096 bits extraídas** observando ruído acústico de laptop (Genkin et al.).
- **Power analysis** (SPA/DPA): análise de consumo em operação criptográfica. Relevante em embarcados.

## Misuse Patterns Frequentes

Onde desenvolvedores se machucam:

1. **ECB mode**: imagem do pinguim mostra padrão através de blocos iguais. Nunca.
2. **IV/nonce reutilizado**.
3. **Padding oracle**: servidor revela se padding é válido → decrypt (CBC).
4. **Wrong hash para senhas**: SHA-256, MD5 em vez de Argon2id.
5. **RSA < 2048 bits**. Hoje 3072+ recomendado; ECC (Curve25519) em novos sistemas.
6. **Sem authentication**: encriptar sem MAC/AEAD. Malleable.
7. **Random fraco**: `Math.random()`, `rand()` C, MT19937 — não para crypto.
8. **Chave derivada só de timestamp / username**: atacante recomputa.
9. **Certificado pinning rígido**: útil, mas quebra quando precisar rotacionar. Pinar ao **SPKI da CA intermediária** ou múltiplos pins com fallback.
10. **Homebrew crypto**: sempre ruim.

## Chaves e Gestão

- **KMS** (AWS, GCP, Azure) para chaves em repouso.
- **HSM** (CloudHSM, AWS CloudHSM, Thales, YubiHSM) para proteger chave privada.
- **Envelope encryption**: KEK (no KMS) encripta DEK (por registro). Rotação de KEK sem reencriptar tudo.
- **Rotation** explícita; versionamento de chave.
- **Key custody**: quem pode assinar com a chave? Audit log obrigatório. Separação de duties.

## Certificados Intermediários e HSM

CA root **offline** (em HSM, sala cofre, airgap). Intermediárias online assinam certs finais. Se intermediária comprometida → revogação limitada a intermediária, não root. Minimização de impacto por design.

## Protocolo Signal

Messaging com forward secrecy + post-compromise security. **Double Ratchet**: combina ratchet Diffie-Hellman (por mensagem exchange) + symmetric key ratchet (por mensagem). Comprometimento de chave no tempo T não expõe passado nem futuro (após alguma interação).

Usado em Signal, WhatsApp, Facebook Messenger (conversas secretas), Google Messages (em RCS E2EE).

## age e Moderno Tooling

**age** (Filippo Valsorda): file encryption moderna, simples, sem opções ruins. Substitui GPG para uso casual. Compatível com chaves SSH.

## Recursos

- **"Cryptography Engineering"** — Ferguson, Schneier, Kohno.
- **"Serious Cryptography"** — Jean-Philippe Aumasson.
- **"A Graduate Course in Applied Cryptography"** — Boneh & Shoup (free online).
- **Real World Cryptography** — David Wong.
- **Cryptopals challenges** — aprender quebrando.
- **latacora.com/blog/2018/04/03/cryptographic-right-answers** — quick reference perene.

## Princípios

1. **Use AEAD moderno**. ChaCha20-Poly1305 ou AES-GCM. Esqueça CBC+MAC.
2. **Ephemeral key exchange** sempre. Forward secrecy é não-negociável.
3. **Nonce nunca se repete** com mesma chave. Cuide da fonte.
4. **Senhas com Argon2id**. Sem exceções.
5. **Libs maduras, não roll your own**.
6. **Constant-time** em operações sobre segredo.
7. **Post-quantum planejamento**: sistemas de longa vida devem migrar para híbrido já.
8. **Gestão de chaves é 80% do trabalho**. Algorithm é o fácil; escopo, rotação, custody é o difícil.
