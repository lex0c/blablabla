# Deepfakes

**Deepfake** é mídia sintética (vídeo, áudio, imagem, texto) gerada ou manipulada por IA de forma a **enganar sobre autoria ou veracidade**. O termo nasceu em 2017 de um subreddit; a tecnologia evoluiu de GANs para modelos de difusão e VLMs capazes de gerar conteúdo quase indistinguível de real. Em 2026, deepfakes deixaram de ser curiosidade para virar vetor central de fraude, influence operations, reputational harm e litígio.

## Categorias

### Face swap

Troca rosto de uma pessoa por outra em vídeo. Técnica clássica (DeepFaceLab, FaceSwap), com variantes modernas via difusão (Roop, SimSwap). Usado em memes, pornografia não-consensual, fraude.

### Face reenactment

Mantém identidade mas transfere **expressão/movimento** de outra pessoa. Puppet-style. Ferramentas: First Order Motion, SadTalker, LivePortrait. Fonte comum de vídeos com "personalidade X dizendo Y".

### Full-body / lip sync

Sincronização labial + gestos sincronizados a áudio. Rendering realista. Sistemas comerciais (Synthesia, HeyGen, D-ID) produzem speakers "falantes" credíveis a partir de foto + script.

### Voice clone

Clonar voz com segundos a minutos de áudio. ElevenLabs, Resemble, OpenAI Voice Engine, Coqui. Zero-shot com amostra mínima (~3-10 segundos).

### Text-to-video

Gera vídeo a partir de texto. Sora (OpenAI), Veo (Google), Kling, Runway Gen-3, Pika. Qualidade surpreendente em 2024-2026; limitação em consistência temporal longa e física realista.

### Text-to-image

DALL-E, Midjourney, Flux, Imagen, Stable Diffusion. Commodity desde 2022; realismo foto-quality comum.

### Synthetic text

Textos por LLM — não rotineiramente rotulados como "deepfake" mas funcionalmente equivalentes em influence operations.

## Tecnologia

### Geradores (brevíssimo)

- **GANs** (2014-2022): generator vs discriminator em jogo adversarial (ver `GAME_THEORY.md`). Picos StyleGAN3, difícil treinar.
- **Autoencoders variacionais / flow models**: modelos anteriores.
- **Modelos de difusão** (2020+): Denoising Diffusion Probabilistic Models; training estável, qualidade superior; dominantes em imagem/vídeo.
- **Latent diffusion** (Stable Diffusion): operar em latent space, muito mais eficiente.
- **Transformers de vídeo / DiT**: spatio-temporal; base de Sora, Veo.
- **Flow matching, rectified flow**: refinamentos recentes.

### Pipelines de produção

- **Face swap clássico**: detectar landmarks → treinar encoder-decoder paired (fonte ↔ alvo) → blend.
- **Moderno**: few-shot com modelo pretreinado; LoRA/Dreambooth para identidade; ControlNet para pose/expressão.
- **Voice**: reference embedding + vocoder (HiFiGAN, BigVGAN) + modelo TTS.

## Usos Ilegítimos

### Fraude

- **CEO fraud / BEC com voz**: CFO recebe "ligação do CEO" autorizando transferência. Documented: £200k UK 2019; US$25M Hong Kong 2024 (videoconferência inteira deepfake).
- **Phishing com voz de familiar**: "mãe, estou sequestrado, mande PIX".
- **Deepfake em entrevista de emprego** (FBI warned 2022+): candidato usa identidade falsa para se contratar remotamente — especialmente em funções com acesso sensível, frequentemente por estrangeiros sancionados (DPRK IT workers).
- **Verification bypass**: KYC selfie video spoofed.

### Political influence / disinformation

- Vídeos de líderes dizendo coisas nunca ditas.
- Áudio de candidato (robocall deepfake de Biden em New Hampshire, jan 2024).
- Fake imagens de eventos (guerra, catástrofe).
- Timing: véspera de eleição maximiza dano — difícil desmentir em tempo.

### Non-consensual intimate imagery (NCII)

Uso mais prevalente em volume. Maioria das vítimas são mulheres. Plataformas produzidas especificamente (undress sites) proliferaram 2023-2024.

Legal: leis criminalizando variam (UK 2023, CA/NY, Brasil vagas, UE AI Act). Takedown ainda fraco; regulação em avanço.

### Reputation attack

Falsificar declaração pública ou comportamento comprometedor de alvo. Chantagem ou discrediting.

### Harassment e bullying

Adolescente com colegas fabricando imagens. Crise em escolas.

### Market manipulation

Fake declaração de CEO afetando preço de ação; fake imagem de explosão em refinaria afetando commodity. Documentados em 2023+.

### Evidence tampering

Mídia fabricada submetida em tribunal. **Liar's dividend** — todo mundo pode negar mídia real como "deepfake", dissolvendo valor probatório.

## Casos Documentados (sample)

- **2019**: áudio CEO UK fraudado, £200k desviados.
- **2022**: Zelensky deepfake pedindo Ucrânia render-se. Baixa qualidade, rapidamente desmentido.
- **2023**: falsa imagem do Pentagon em chamas move mercado brevemente.
- **Jan 2024**: robocall de "Biden" em New Hampshire desencorajando voto; voz clonada via ElevenLabs.
- **Feb 2024**: fraudador usa deepfake em videoconferência para levar US$25M de filial Hong Kong.
- **Ongoing**: Taylor Swift NCII (jan 2024) forçou X a bloquear buscas temporariamente; caso público de NCII.
- **Ongoing**: DPRK workers usando face-swap ao vivo em entrevistas Zoom para empregos remotos em empresas ocidentais.

## Detecção

### Sinais clássicos (cada vez menos confiáveis)

- Piscar anormal (corrigido em modelos modernos).
- Reflexos em olhos inconsistentes.
- Bordas de rosto.
- Pescoço/ombros com descontinuidade.
- Textura de pele muito lisa.
- Sincronização audio-labial imperfeita.
- Sombras inconsistentes.
- Hands (persistente debilidade em text-to-image; melhor 2024+).
- Contagem de dedos.

**Limitações**: modelos de última geração eliminaram maioria das pistas visíveis. Detecção humana caiu abaixo de 50% em experimentos recentes para áudio e imagem de ponta.

### Detecção automatizada

- **Classifier-based**: CNN treinado em real vs fake.
- **Artefato-based**: buscam artefatos de upsampling, frequency-domain signatures (GAN fingerprints), JPEG inconsistency.
- **Physiological**: pulso via mudança sutil de cor de pele (rPPG); deepfakes antigos falham aqui.
- **Temporal consistency**: inconsistências entre frames.
- **Audio**: prosody, breathing patterns, harmonic structure.
- **Plataforma-embedded**: Intel FakeCatcher, Microsoft Video Authenticator, TruePic.

**Problema fundamental**: detectores envelhecem rapidamente. Nova arquitetura de geração → detector falha. Arms race perpétua.

### C2PA / Content Credentials

Coalition for Content Provenance and Authenticity. Padrão aberto (Adobe, Microsoft, Nikon, Leica, BBC, Canon) para **assinar** mídia autêntica com metadata de origem (device, software, edits).

- **Sinaliza autêntico**, não detecta fake.
- Adoção crescente em câmeras (Leica M11-P, Sony α), em plataformas (TikTok, LinkedIn), em IA (OpenAI, Adobe Firefly).
- Cadeia: cam → edit → platform mantém chain-of-custody.

Limitação: vídeos sem credential ficam ambíguos; não é bala de prata.

### Watermarking

Embutir sinal invisível detectável.

- **SynthID** (Google): texto, imagem, audio.
- **Stable Signature** (Meta).
- **C2PA** (metadata, robusta a edição).
- **Invisible watermarks** em modelo de difusão.

Trade-off: robustez a transformações (compressão, crop, re-render) vs imperceptibilidade.

### Provenance tools

- **Amber, TruePic**: câmera de captura assinada.
- **OriginStamp, Numbers Protocol**: blockchain-based.

### Coalizão

- **Content Authenticity Initiative (Adobe)**.
- **Partnership on AI**.
- **Project Origin** (BBC, CBC, Microsoft, NYTimes).

## Mitigação em Produtos

### Provedores de serviço

- **Voice auth / biometric**: combinar múltiplos fatores, liveness, challenge-response.
- **Transação crítica**: exige callback, secondary channel, code word.
- **KYC**: liveness check, document + selfie correlation, passive anti-spoofing.
- **Endpoint em comunicação corporativa**: treinar funcionários para phishing com voz e vídeo; políticas para autorizações.

### Plataformas

- **Labels** (Meta, X, TikTok): "conteúdo manipulado". Enforcement inconsistente.
- **Bans de conteúdo** (NCII, eleição): variam por jurisdição.
- **Detecção em upload**.
- **Synthetic media policies**: em evolução.

### Governo / regulação

- **EU AI Act (2024+)**: deepfake disclosure obrigatório; sistemas de alto risco categorizados.
- **UK Online Safety Act (2023)**: criminalizou NCII deepfake.
- **US state laws** (CA, TX, VA, NY): criminalizam NCII e deepfake eleitoral.
- **Brasil**: LGPD aplica parcialmente; projetos de lei em tramitação; CGJ tem resolução sobre uso em eleição.
- **China**: regulações 2023 exigindo marcação de conteúdo gerado.
- **DEEPFAKES Accountability Act** (US, proposed): labeling + watermarking federal.

## Ferramentas

### Geração (exemplos públicos)

| Categoria | Ferramentas |
|---|---|
| Image | Midjourney, DALL-E, Stable Diffusion, Flux, Imagen |
| Video | Runway Gen-3, Sora, Veo, Kling, Pika |
| Voice | ElevenLabs, Resemble, OpenAI Voice Engine, Coqui, Bark |
| Face swap (legacy) | DeepFaceLab, FaceSwap, SimSwap |
| Avatar / lip-sync | Synthesia, HeyGen, D-ID, SadTalker, LivePortrait |

### Detecção

| Categoria | Ferramentas |
|---|---|
| Generic | Hive Moderation, Sensity AI, Reality Defender, Intel FakeCatcher |
| Voice | Pindrop, Resemble Detect, AI Voice Detector |
| Provenance | C2PA-compatible (Adobe, Leica, TruePic) |
| Open source | Deepware Scanner, FaceForensics++, deepfake-detection models |

### Análise forense

- **InVID**: verificação de vídeo para jornalistas.
- **FotoForensics**: análise de imagem.
- **Reverse image search**: Google Images, Yandex, TinEye, PimEyes.
- **Forensic framework**: Amped FIVE, DVR Examiner (enforcement).

## Para Consumidores / Cidadãos

Verificação rápida:

1. **Origem**: quem postou? Confiável? Primeira divulgação em conta duvidosa é red flag.
2. **Corroboração**: mídia grande credível corrobora?
3. **Reverse search**: imagem/frame em TinEye/Yandex.
4. **Metadata** (quando presente).
5. **Consulte fact-checkers**: AFP, Reuters Fact Check, Lupa (BR), Aos Fatos (BR), Comprova (BR).
6. **Espere**: nova mídia bombástica horas antes de evento → ceticismo elevado.
7. **Padrão cognitivo**: se conteúdo reforça seu viés muito convenientemente, desconfiar mais.

## Para Desenvolvedores

Se você constrói aplicação que gera ou exibe mídia:

- **Watermark** output (SynthID ou similar).
- **Metadata C2PA** quando aplicável.
- **Label** conteúdo gerado.
- **Políticas de uso** claras.
- **Abuse detection**: filtrar geração de rostos reais, figuras públicas, minors, NCII.
- **Consent** em caso de voz/rosto de terceiros.
- **Log** + audit trail.
- **Reporting channel** para vítimas.

## Ética de Pesquisa

- **Dual-use**: detectar deepfakes exige estudá-los. Pesquisadores publicam modelos melhores → atacantes usam.
- **Responsible disclosure** em vulnerabilidades de detectores.
- **NCII**: alguns modelos foram retidos (OpenAI Voice Engine limitou release por risco). Outros não (Stable Diffusion liberado irresponsavelmente segundo muitos).
- **Fine-tuning em rostos de pessoas reais sem consent** é eticamente problemático, legal em zona cinzenta.

## Futuro Próximo

1. **Realismo atinge ceiling em 2026-2028**: humano não consegue distinguir em casos comuns. Detecção depende de watermarks + provenance + análise pesada.
2. **Synthetic media as default**: maioria de conteúdo online será parcialmente sintético (edits, upscaling, lip-sync em traduções).
3. **Liar's dividend** cresce: descredito de mídia real por suspeita.
4. **Plataformas obrigadas por lei** a verificar/rotular (EU AI Act, similares).
5. **Identity verification** reforma: multi-factor, biometria passiva, secondary channels.
6. **Legal frameworks** maduram, com diferenças jurisdicionais grandes.
7. **Adoção de C2PA**: câmeras, editores, plataformas assinando cadeia.
8. **Arms race continua**: detector vs gerador.

## Recursos

- **"Deepfakes: The Coming Infocalypse"** — Nina Schick.
- **"Tools and Weapons"** — Brad Smith (Microsoft perspective, broader).
- **Partnership on AI — Media Integrity work**.
- **Witness.org** — advocacy e tooling para defender jornalistas, ativistas.
- **MIT Media Lab Detect Fakes**.
- **C2PA.org** — especificação e tooling.
- **Deeptrace, Sensity AI reports** (estado da arte em volume/uso).
- **DEF CON AI Village**.
- **Lawfare, Just Security**: análise policy.

## Princípios

1. **Não confie em mídia isolada**. Corroboração múltiplo-fonte.
2. **Cadeia de proveniência > detecção pós-fato**. C2PA é defesa proativa.
3. **Assine o autêntico, não só detecte o falso**. Mudança de paradigma.
4. **Múltiplos canais em decisão crítica**. Voice + video + secondary callback + code word.
5. **Treine pessoas**. Cialdini + urgência + voz familiar = humanos erram facilmente. Educação reduz taxa.
6. **Frameworks legais ainda maturam**. Opere acima do mínimo legal em ética.
7. **Se constrói, assuma misuse**. Filtros, watermark, logging não são opcionais.
8. **Nada é perfeito**. Detecção, watermarks, laws — todos parciais. Defesa em profundidade.
