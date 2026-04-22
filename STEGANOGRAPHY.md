# Esteganografia

Diferente da criptografia, que **torna o conteúdo ilegível** para quem intercepta, a esteganografia **esconde a existência** da mensagem. O ideal estereográfico: um observador não desconfia sequer que há comunicação — só vê uma foto de gato comum.

Ambas são complementares: cifrar primeiro, esconder depois.

## Conceitos

- **Cover medium / carrier**: o arquivo portador (imagem, áudio, vídeo, texto, protocolo).
- **Payload / stego-object**: mensagem escondida.
- **Chave stego**: semente/algoritmo que determina onde o payload está no carrier.
- **Capacity**: quantos bits cabem num carrier de tamanho X sem degradar perceptivelmente.
- **Detectability**: quão distinguível o carrier com payload é de um carrier limpo.

Trade-off fundamental: **capacidade × indetectabilidade**. Mais payload → mais mudança → mais suspeita.

## Técnicas por Mídia

### Imagens

**LSB (Least Significant Bit)**

Substituir bits menos significativos de pixels pelos bits da mensagem. Em RGB 8-bit, mudar o LSB de cada canal altera cor em 1/256 — invisível ao olho humano.

- **Capacidade**: 3 bits/pixel em RGB (1 por canal); 25% da imagem.
- **Detecção**: análise estatística (chi-squared, RS analysis) detecta — distribuição de LSB vira demasiado uniforme. StegExpose, Zsteg.
- **Suscetível a**: recompressão (JPEG), filtros, resize.

**DCT em JPEG**

Modificar coeficientes DCT durante compressão. **F5, JSteg, Outguess, OpenStego**. Mais robusto a recompressão.

**Adaptive steganography**

Escolhe regiões de alta textura/ruído para esconder — mais difíceis de detectar estatisticamente. HUGO, WOW, UNIWARD (state-of-art). Usados em pesquisa acadêmica.

**Metadados**

Campos EXIF, IPTC, XMP — rápido mas qualquer steg detector óbvio olha lá.

### Áudio

- **LSB** em samples PCM (WAV).
- **Phase coding**: modifica fase; imperceptível, robusto.
- **Spread spectrum**: dispersa payload ao longo do espectro.
- **Echo hiding**: introduz ecos sutis que codificam bits.
- Em MP3: coeficientes MDCT analogamente ao DCT em JPEG.

### Vídeo

- Combina técnicas de imagem (frames) e áudio (track).
- Alta capacidade dada duração.
- Recompressão em plataformas (YouTube, Instagram) destrói a maior parte dos esquemas.

### Texto

Mais difícil; pouco ruído natural para mascarar.

- **Typographic**: espaços duplos, tabs no fim de linha.
- **Linguistic**: sinônimos escolhidos para codificar bits. `Stegosaurus` (Python) usa.
- **Unicode look-alikes**: caracteres homoglyph (`а` cirílico vs `a` latino) codificam bits — difícil de detectar visualmente.
- **Zero-width characters**: `U+200B` (ZWSP), `U+200C` (ZWNJ). Popular em "marcar" textos colados. Detectável por scanner.

### Protocolos de Rede

**Network steganography** — covert channels:

- **TCP headers**: campos como initial sequence number, timestamps, options — cabem bits.
- **ICMP payload**: dado em echo request/reply.
- **DNS queries**: subdomínios codificam dados. Detecção: comprimento anormal, volume alto.
- **HTTP headers**: User-Agent, cookies.
- **TLS SNI**: vazava até ECH.

Tunneling é parente técnico — ver `TUNNELING.md`.

## Esteganografia vs Watermarking vs Fingerprinting

- **Steg**: esconder *comunicação secreta*; imperceptibilidade máxima.
- **Watermarking**: marca de identidade/origem em mídia; visível ou invisível, mas **robusta** contra remoção. Ex.: "Propriedade da Sony".
- **Fingerprinting**: identifica distribuição para rastrear quem vazou. Ex.: screener de filme com fingerprint único por destinatário.

Mesmo ferramental técnico, objetivos diferentes.

## Esteganálise — detecção

Ramo inverso. Técnicas:

### Visual / simples

- **Ver strings** em arquivos.
- **Comparar com "clean"** da mesma fonte.
- **Abrir em múltiplos formatos** (às vezes há ZIP concatenado).

### Estatística

- **Histograma de LSBs**: num carrier limpo, LSB quase aleatório; com payload, pode ficar uniforme demais.
- **Chi-squared test** em pares de intensidade.
- **RS analysis** (Regular-Singular): divide em blocos, aplica flip, mede.
- **Calibração**: recomprimir e comparar.

### Machine Learning

- **Rich models** (SRM, CFA): features complexas sobre sub-bandas.
- **CNNs**: detectores modernos classificam imagens "stego vs clean" com alta acurácia em benchmarks.
- **Steganalysis Toolbox (StegExpose, Aletheia, Stegdetect, zsteg)**.

### Kerckhoffs em steg

Se algoritmo é conhecido, o adversário pode aplicar o detector específico. Resistência baseia-se em **chave**, não em obscuridade.

## Ferramentas

| Nível | Ferramenta |
|---|---|
| Simple LSB | steghide, OpenStego, SilentEye |
| F5/DCT | F5 (Java), JSteg, Outguess |
| Unicode/zero-width | `zero-width-steganography`, `stegcloak` |
| Audio | DeepSound, MP3Stego |
| Forense | binwalk, zsteg, StegOnline (web), stegseek |
| Steganalysis | Aletheia, StegExpose, Stegdetect |
| CTF utilitário | stegsolve (Java GUI clássica), exiftool |

## Usos Legítimos e Ilegítimos

**Legítimos**:

- **Fingerprinting de mídia**: rastrear vazamentos internos.
- **Marca d'água** para direitos autorais.
- **Comunicação em regimes repressivos**: ativistas, jornalistas.
- **DRM** (controverso).
- **Pesquisa forense**: ocultar evidência que não deve ser alterada.

**Ilegítimos**:

- **Exfiltração** por malware: dados em imagens uploaded.
- **C2 stealth**: comandos para bot em posts de rede social.
- **Distribuição de payload** em imagens (triggers infosec alerts).
- **CSAM distribution** (triste mas real).

## Stego em Malware

Famílias que documentaram uso:

- **Stegoloader, Zeus, Lurk, DNSChanger**: config ou payload em imagens.
- **Steganography + C2**: imagem em CDN legítima (Imgur, Flickr) como canal.
- **Invisimole, Turla**: grupos APT com routines esteganográficas sofisticadas.

Defensores com DLP + análise de imagens saindo da rede são poucos — vetor relativamente subexplorado.

## Covert Channels — Categorias

Além da steg em mídia:

- **Storage channels**: atacante e conspirator compartilham estado modificável (marcação de arquivo, timestamp).
- **Timing channels**: codificar bits em intervalo entre ações.

Sistema confinado (ex.: VM com sidechannel à cloud) — covert channel pode vazar dado entre entidades sem canal explícito.

## CTF e Prática

Esteganografia é categoria comum em CTFs (ver `CTF.md`).

- **Checklist típico**:
  1. `file`, `strings`, `exiftool`.
  2. `binwalk` — há arquivos embutidos?
  3. LSB visual (stegsolve planes).
  4. Senha óbvia em description + `steghide`.
  5. Cabeçalhos unusual em hex editor.
  6. Zero-width em texto.

- Treinar em: picoCTF, RingZer0, crackmes.

## Princípios

1. **Cifre antes de esconder**. Esteganografia sozinha cai com algoritmo conhecido.
2. **Carrier com ruído natural** é melhor (fotos texturais > gráficos planos).
3. **Capacidade baixa** = maior indetectabilidade. Nem tudo cabe em uma imagem.
4. **Recompressão mata schemas frágeis**. Se o canal tem re-encode, use schemas DCT/frequência.
5. **Steganálise existe**. Ambiente com DLP moderno pode detectar schemas clássicos.
6. **Legalidade varia**. Esteg por si não é crime; propósito e conteúdo é que determinam.
