# Áudio — Análise e Detecção de Padrões

Áudio é **sinal temporal 1D** que, quando decodificado corretamente, carrega informação rica: voz, música, sons ambientais, falhas mecânicas, comunicação animal. Análise de áudio abrange captura, representação, extração de features, classificação, separação, síntese. Evolução recente de pipelines "feature + classificador" (MFCC + SVM) para **redes neurais profundas end-to-end** (Whisper, Wav2Vec) não eliminou os fundamentos — DSP clássico ainda é base para pré-processar, inspecionar, e entender o que o modelo faz.

Complementa `AI.md`, `NEUROSCIENCE_101.md` (sistema auditivo), `DEEPFAKES.md` (voice clone), `MLOPS.md`, `CIRCUIT.md` (microfones, ADC/DAC), `CONDITION_MONITORING.md` (análise acústica industrial), `MECHANICS.md` (acústica).

## Fundamentos

### Som

Onda de pressão em meio (ar, água, sólido). Caracterizada por frequência (Hz), amplitude (Pa, dB SPL), fase.

- **Faixa audível humana**: ~20 Hz - 20 kHz (degrada com idade, especialmente agudos).
- **Infrassom** (<20 Hz), **ultrassom** (>20 kHz).
- **Velocidade no ar**: ~343 m/s a 20°C.
- **dB SPL**: logarítmica; 0 dB ~ limite de audição; 60 dB conversa; 85 dB risco prolongado; 120 dB dor.

### Digitalização

Conversão analógico → digital (ADC):

1. **Amostragem** a taxa `fs`. **Teorema de Nyquist-Shannon**: `fs > 2 · f_max` para reconstrução perfeita. Alias se violar.
2. **Quantização** em bit depth: 8, 16, 24, 32 bits.
3. **Filtro anti-aliasing** antes de amostrar — sem ele, frequências >fs/2 dobram para baixo, irreversível.

### Taxas comuns

| fs | Uso |
|---|---|
| 8 kHz | telefonia (narrowband) |
| 16 kHz | speech ASR |
| 22.05 kHz | speech/podcast low-bandwidth |
| 32-44.1 kHz | música lossy (AAC), CD (44.1) |
| 48 kHz | vídeo/streaming profissional |
| 88.2-96 kHz | estúdio alta resolução |
| 192 kHz | mastering, arquivo |

### Bit depth e faixa dinâmica

- **16-bit**: ~96 dB dynamic range. Suficiente para consumo.
- **24-bit**: ~144 dB. Headroom para edição/mixagem.
- **32-bit float**: overflow impossível — padrão em produção.

### Formatos

- **PCM** (raw): WAV, AIFF, FLAC (lossless comprimido).
- **Lossy**: MP3, AAC, Opus, Vorbis.
- **Containers** vs codecs: MP4/MKV contém codec; distinguir.
- **Metadata**: ID3v2 (MP3), Vorbis comments (FLAC), iXML.

## Representações

### Domínio do tempo

Amplitude vs tempo. Útil para: onset detection grosseiro, waveform visual, envelope (ADSR), zero-crossing rate.

Limitação: frequências ocultas; difícil para análise harmonic.

### Domínio da frequência (FFT)

**Fast Fourier Transform** converte janela temporal em magnitude + fase por frequência.

- **Janela** (Hanning, Hamming, Blackman, Kaiser): reduz spectral leakage.
- **Tamanho da FFT**: `N`. Resolução frequência `fs/N`; resolução temporal é inversa.
- **Zero padding** aumenta visualmente resolução, não real.

### STFT — Short-Time Fourier Transform

Aplica FFT em janelas deslizantes sobrepostas → **espectrograma**: matriz tempo × frequência de magnitude.

- **Hop size** define overlap (75% comum).
- Trade-off: janela longa → boa resolução em frequência, má em tempo; curta → inverso (incerteza de Heisenberg em DSP).

### Mel Spectrogram

Escala **mel** aproxima percepção humana (log em alta freq). Transforma STFT com banco de filtros mel.

- 40-128 bandas mel típicas.
- Log magnitude frequente: `log(mel + ε)`.
- Base de maioria dos modelos de áudio moderno.

### MFCC — Mel-Frequency Cepstral Coefficients

Cepstrum do log-mel spectrogram. 13-40 coeficientes.

- Histórico dominante em speech antes de deep learning.
- Descorrelaciona bandas → bom para GMM/HMM.
- Menos usado hoje: modelos aprendem features direto de mel-spec.

### CQT — Constant-Q Transform

Resolução constante por oitava (log em frequência). Ideal para **música** — notas estão em escala log.

### Wavelets

Decomposição multi-resolução. Útil para transientes, sinais não-estacionários. Menos comum em áudio moderno que STFT.

### Outras

- **Chromagram**: 12 pitches × tempo, enfatiza classe de nota independente de oitava.
- **Tonnetz**: 6D harmonic network.
- **Spectral contrast, flux, centroid, rolloff, bandwidth**: scalar features por frame.
- **Tempogram**: para beat/tempo.

## Features Clássicas

Frame-level (janelas curtas) + resumos (estatísticas):

- **RMS energy**: amplitude média — envelope.
- **Zero-crossing rate (ZCR)**: passagens por zero. Alta em ruído/consoantes não-vozeadas, baixa em vogais.
- **Spectral centroid**: "brilho" do som.
- **Spectral bandwidth, rolloff, flux**.
- **MFCC + delta + delta-delta**: primeira/segunda derivadas temporais — padrão antigo em ASR/MER.
- **Pitch (F0)**: fundamental. Métodos: autocorrelação (YIN), SWIPE, PYIN, CREPE (deep).
- **HNR (Harmonic-to-Noise Ratio)**: qualidade vocal.
- **Formants**: ressonâncias do trato vocal (F1, F2, F3). Identificam vogais.
- **LPC (Linear Predictive Coding)**: modela envelope espectral via regressão AR. Base de speech coding clássico.

Librosa (Python) é ferramenta canônica para extrair essas.

## Pré-processamento

### Normalização

- **Peak normalize**: max(|x|) = 1.
- **RMS normalize**: RMS = target.
- **LUFS / loudness** (EBU R128, ITU-R BS.1770): padrão broadcast.

### Filtragem

- **Passa-altas**: remover DC, rumble, pop. Corte típico 80 Hz em voz.
- **Passa-baixas**: anti-aliasing, de-emphasis.
- **Passa-banda**: voz (300-3400 Hz é telefonia histórica).
- **Notch**: 60/50 Hz (hum elétrico).
- **Filtros adaptativos** (Wiener, LMS): cancelamento de ruído.

### Resampling

Mudar fs. `librosa.resample`, `sox`, `ffmpeg`. Cuidado com filtros anti-aliasing.

### Silence removal / trimming

Limiar de energia; ou VAD (abaixo).

### Dithering

Adicionar ruído pequeno antes de reduzir bit depth — evita quantização audível em fades.

### Augmentation (para ML)

- **SpecAugment**: masking em bandas tempo/freq no mel-spec.
- **Time stretch, pitch shift** (independentes).
- **Add noise** (SNR controlado, ruído real de dataset).
- **Reverb, EQ, band limiting**.
- **Mixup, audio mixing**.

## Análise Musical

### Onset Detection

Momento em que nota começa. Métodos: spectral flux, high-frequency content, phase deviation, deep (OnsetNet).

### Beat Tracking / Tempo

Estimar BPM + pulsação. Ellis (madmom), librosa `beat_track`. Deep: TCN-based (Böck).

### Pitch Tracking

Monofônico: YIN, PYIN, CREPE.
Polifônico: multipitch é muito mais difícil; NN-based (MT3, Basic Pitch).

### Key Detection

Estimar tonalidade. Krumhansl-Schmuckler template matching em chromagram.

### Chord Estimation

CNN-LSTM sobre chromagram/CQT (McFee, Bittner).

### Structure Analysis

Detectar seções (verso/refrão). Self-similarity matrix + segmentation.

### Music Transcription

Áudio → partitura MIDI. Clássico: hard. Modern: Spotify Basic Pitch, MT3, Onsets and Frames (piano), MT3 polifônico.

### Music Information Retrieval (MIR)

Campo abrangente:
- Genre classification
- Mood/emotion
- Instrument recognition
- Cover detection
- Music similarity / recommendation
- Query-by-humming
- Audio fingerprinting (Shazam — abaixo)

Conferências: ISMIR.

## Detecção e Classificação

### Voice Activity Detection (VAD)

Detectar presença de fala. Classic: energy + ZCR. Moderno: Silero-VAD, WebRTC VAD. Essencial em ASR streaming, telefonia.

### Keyword Spotting (KWS)

"Hey Siri", "OK Google". Always-on; baixo consumo em MCU. Modelos compactos (TinyML): DS-CNN, MatchboxNet, Google's Speech Commands benchmark.

### Automatic Speech Recognition (ASR)

Áudio → texto.

- **HMM-GMM**: histórico (Kaldi).
- **Hybrid HMM-DNN**: acústico DNN + decoding HMM.
- **End-to-end**: CTC, RNN-T, attention-based (LAS).
- **Transformer-based**: Whisper (OpenAI), Canary (Nvidia), Conformer.
- **Modelos multilingual**: MMS (Meta, 1100+ idiomas), Whisper.
- **Streaming vs batch**: latência vs accuracy.

Métricas: **WER (Word Error Rate)** = (S + D + I) / N.

### Speaker Identification / Verification

"Quem está falando?" (identification, 1:N) ou "é quem diz ser?" (verification, 1:1).

- **x-vectors, i-vectors**: embeddings clássicos.
- **ECAPA-TDNN, Resemblyzer**: modernos.
- **Cosine similarity** entre embeddings + threshold.
- Métrica: EER (Equal Error Rate).

### Speaker Diarization

"Quem falou quando?" em áudio com múltiplos speakers. Pyannote, Diart, NeMo. Combinar com ASR → transcript com etiquetas.

### Emotion Recognition

Categorias discretas (raiva/tristeza/alegria) ou dimensional (arousal/valence). Datasets: IEMOCAP, RAVDESS, MELD. Performance moderada — emoção é multimodal.

### Audio Event Detection

Detectar eventos em áudio ambiente: latido, vidro quebrando, bebê chorando, tiro, sirene.

- **AudioSet** (Google): 527 classes, weak labels, 2M samples do YouTube.
- **ESC-50**: 50 classes ambientais balanceadas.
- **UrbanSound8K**: 10 classes urbanas.
- Tarefa: classificação (tag) ou detecção temporal (start/end).
- **DCASE challenges** anuais.

### Acoustic Scene Classification

"Onde foi gravado?" — café, metrô, parque, carro, rua, estação. Bem estabelecido pelo DCASE.

### Anomaly Detection

Som anormal sem rotular falha específica. Autoencoders, flow models, self-supervised. Aplicações: manutenção industrial (ver `CONDITION_MONITORING.md`), segurança, saúde (tosse, respiração).

### Bioacústica

- **Identificação de espécies** por canto de ave, morcego, baleia, anfíbios. BirdNET (Cornell) detecta 6k+ espécies de pássaros.
- **Monitoramento de biodiversidade**: passive acoustic monitoring em florestas, oceanos.
- **Ameaças**: detectar serras em área protegida, caça furtiva.

### Medical / Clinical

- **Cough classification** (COVID, tuberculose).
- **Heart sounds (auscultation)**: S1/S2, sopros.
- **Respiração**: apneia, sibilância.
- **Fonoaudiologia**: avaliação vocal.

### Audio Fingerprinting

Identificar gravação específica por "impressão digital".

- **Shazam algorithm** (Wang, 2003): peaks em espectrograma como constellation; hashes de pares peak. Busca em banco massivo.
- **Chromaprint** (AcoustID): alternativa open.
- Robusto a ruído, compressão, gravação de ambiente.

## Source Separation

Separar fontes misturadas.

### Speech Enhancement

Remover ruído, reverb.

- **Spectral subtraction, Wiener filter**: clássicos.
- **Deep: DCUNet, DeepFilterNet, demucs (v4), RTX Voice**.
- **Microphone arrays + beamforming**: múltiplos microfones + MVDR/GEV.

### Echo Cancellation

Remover eco acústico em call. WebRTC AEC, Speex AEC.

### Music Source Separation

Separar mistura estéreo em stems (vocals/drums/bass/other).

- **Demucs** (Meta) — HT Demucs é SOTA 2023+.
- **Spleeter** (Deezer).
- **Open-Unmix**.
- MUSDB18 benchmark.

### BSS (Blind Source Separation)

Sem conhecer fontes. ICA (Independent Component Analysis), NMF (Non-negative Matrix Factorization). Desafio em cocktail party.

## Deep Learning para Áudio

### Arquiteturas

- **CNN em mel-spectrogram**: 2D conv como imagem. VGGish, YAMNet, PANNs.
- **CRNN**: CNN + RNN para sequence.
- **Transformer**: AST (Audio Spectrogram Transformer), PaSST.
- **Conformer**: conv + attention; dominante em ASR.
- **Waveform direto**: WaveNet, Wav2Vec, SincNet, EnCodec.

### Self-Supervised Learning

Pré-treino em áudio sem rótulo; fine-tune downstream:

- **Wav2Vec 2.0** (Meta): contrastive em quantized latents.
- **HuBERT**: clustering + masked prediction.
- **WavLM, Data2Vec, BEATs, AudioMAE**.
- **CLAP** (Contrastive Language-Audio Pretraining): áudio ↔ texto.

Modelos SSL + classifier simples dominam benchmarks.

### Foundation Models

- **Whisper** (OpenAI): ASR + tradução multilingual. 680k h treino.
- **SeamlessM4T** (Meta): speech-to-speech, multi-modal.
- **MMS** (Meta): ASR/TTS em 1000+ idiomas.
- **AudioLM, MusicLM, MusicGen, Stable Audio**: geração.
- **VALL-E, NaturalSpeech, VoiceBox**: TTS zero-shot.

## Síntese e Geração

### TTS (Text-to-Speech)

- **Concatenativa**: unidades gravadas coladas (velha escola).
- **Paramétrica**: HMM/DNN gera parâmetros → vocoder.
- **Neural**: Tacotron, FastSpeech, Glow-TTS → vocoder (WaveNet, HiFi-GAN, BigVGAN).
- **E2E**: VITS, NaturalSpeech.
- **Zero-shot voice cloning**: amostras de 3-10s clonam timbre. ElevenLabs, XTTS, VoiceBox. Ver `DEEPFAKES.md`.

### Música

- **Symbolic (MIDI)**: MuseNet, Music Transformer.
- **Waveform**: Jukebox (OpenAI), MusicLM, MusicGen, Stable Audio.
- **Text-to-music**: prompt → áudio.
- **Stem generation, completion**.

### Síntese sonora

- **Aditiva**: soma de senoides.
- **Subtrativa**: filtro sobre rico harmônico (analógico clássico).
- **FM (Frequency Modulation)**: Yamaha DX7, DX7-like.
- **Wavetable**: look-up de formas.
- **Granular**: micro-grãos de amostras.
- **Physical modeling**: simular instrumento.

### Neural vocoders

- **WaveNet** (DeepMind 2016): autoregressive, lento.
- **WaveRNN, WaveGlow, HiFi-GAN, Vocos**: paralelos, tempo real.
- **Neural codecs**: EnCodec, SoundStream — compressão aprendida + vocoder.

## Psicoacústica

Como percebemos, não como medimos.

- **Curvas de Fletcher-Munson (equal loudness)**: percepção de volume varia com frequência. Padrão ISO 226.
- **Masking** (mascaramento): som forte oculta som fraco próximo em freq/tempo. Base de codecs perceptuais (MP3/AAC/Opus).
- **Pitch percepcional**: mel, bark scales.
- **Critical bands**: banda onde um som pode ser mascarado por outro.
- **Binaural hearing**: ITD (interaural time difference), ILD (level) → localização.
- **Cocktail party effect**: atenção seletiva em múltiplos falantes.
- **Timbre**: "cor" do som; combinação de harmônicos + envelope + ruído.

Ver também `NEUROSCIENCE_101.md`.

## Compressão Lossy

Explora psicoacústica para descartar o inaudível.

### Codecs

| Codec | Uso | Bitrate típico |
|---|---|---|
| **MP3** | legado | 128-320 kbps |
| **AAC** | streaming, iTunes, YT | 96-256 kbps |
| **Opus** | WebRTC, Discord, streaming | 6-510 kbps, melhor qualidade/bit |
| **Vorbis** | open, legado | 96-256 kbps |
| **FLAC** | lossless | ~50% WAV |
| **Neural codecs** (EnCodec, SoundStream) | emergente, <6 kbps com qualidade |

### Impactos em análise

- Compressão **remove** informação que análise de alta precisão pode precisar.
- Alguns features (MFCC) tolerantes; fingerprinting menos.
- Re-encoding acumula perdas (generational loss).

## Acústica e Captura

### Microfones

- **Condenser**: plate capacitor; sensível, fidelidade alta; estúdio.
- **Dynamic**: bobina móvel; robusto; palco, broadcast.
- **Ribbon**: som "vintage"; frágil.
- **Electret**: capacitor com carga permanente; smartphone, headset.
- **MEMS**: miniaturizado, smartphone moderno, IoT.

Padrões polares:
- **Omnidirecional**: captura tudo.
- **Cardioide**: frente sim, atrás não.
- **Bidirecional (figure-8)**: frente + atrás.
- **Shotgun**: altamente direcional.

### Microphone arrays

Múltiplos microfones + algoritmo = beamforming, source localization (DOA), noise cancellation.

### Sala

- **Reverb**: reflexões; tempo RT60.
- **Modes**: ressonâncias de sala (especialmente baixos).
- **Treatment**: absorção, difusão, trapping.
- **Anechoic** vs real: pesquisa vs produção.

Acústica arquitetônica pertence a `MECHANICS.md` / civil engineering.

## Ferramentas

### Libs (programação)

- **librosa** (Python): DSP + features clássicas. Referência.
- **torchaudio, audiomentations, soundata**: PyTorch.
- **tensorflow-io-audio**.
- **essentia, aubio, madmom**: MIR.
- **pydub, soundfile, scipy.io.wavfile**: I/O.
- **speechbrain, nemo, espnet, k2**: speech toolkits.
- **pyannote.audio**: speaker diarization.

### Line tools

- **ffmpeg, sox**: conversão, filter.
- **mediainfo**: inspeção.

### DAWs / edição

- **Audacity** (open, leve).
- **Reaper** (barato, scriptable).
- **Pro Tools, Logic, Ableton Live, FL Studio, Cubase**: profissionais.
- **Ardour** (open, Linux).

### ASR/TTS serviços

- **Whisper (self-host), OpenAI, Deepgram, AssemblyAI, Google STT, AWS Transcribe, Azure Speech**.
- **ElevenLabs, PlayHT, OpenAI TTS, Cartesia** — TTS.

### Análise ao vivo

- **Smaart, Room EQ Wizard**: acústica de sala.
- **Spek, Sonic Visualiser**: espectrograma GUI.

## Datasets

### Speech

- **LibriSpeech** (1000h) — ASR inglês.
- **Common Voice** (Mozilla) — multilingual, crowdsourced.
- **VoxCeleb** — speaker ID.
- **TIMIT, WSJ** — clássicos.
- **GigaSpeech, MLS, VoxPopuli** — escala.

### Música

- **MSD (Million Song Dataset)**, **FMA (Free Music Archive)**.
- **MAESTRO, MusicNet** — piano transcription.
- **MUSDB18** — separation.
- **NSynth** (Magenta) — notas isoladas.

### Eventos / scenes

- **AudioSet** — 2M vídeos YouTube rotulados (weak).
- **ESC-50, UrbanSound8K, FSD50K** — curated.
- **DCASE** challenges: anual, variados.

### Bioacústica

- **BirdCLEF, Xeno-Canto**, **Macaulay Library** (Cornell).

### Benchmarks

- **SUPERB** (Speech Universal PERformance Benchmark).
- **HEAR** (Holistic Evaluation of Audio Representations).
- **LibriSpeech test-clean/test-other** para ASR.
- **Librimix** para separação.

## Aplicações em Produção

- **Assistentes** (Siri, Alexa, Google): wake word + ASR + NLU + TTS.
- **Tradução em tempo real**: SeamlessM4T.
- **Podcast / vídeo transcript**: Whisper + diarização + editor assistido.
- **Music recommendation**: Spotify, Apple Music — features + collaborative filtering (ver `SEARCH_AND_RANKING.md`).
- **Creative tools**: Soundraw, Boomy, Suno para música; Adobe Podcast Enhance para limpeza.
- **Call center analytics**: emotion, compliance, quality, summarization.
- **Accessibility**: legendas automáticas, TTS para pessoas com deficiência visual.
- **Saúde**: ausculta digital, screening de apneia, biomarkers vocais.
- **Segurança**: detecção de tiro (ShotSpotter), vidro quebrando, grito, queda.
- **Indústria**: detecção de falha em máquinas por assinatura acústica (ver `CONDITION_MONITORING.md`).
- **Monitoramento ambiental**: biodiversidade, ilegal logging.

## Desafios Práticos

1. **Ruído de fundo variável**: treinar com augmentation + matched conditions.
2. **Reverb / far-field**: modelos treinados close-talk falham em sala. DereverberationsPeech enhancement + multi-condition training.
3. **Código de microfone**: response varia; calibração ou augmentation com device simulation.
4. **Latência em streaming**: ASR low-latency sacrifica accuracy.
5. **Privacidade**: áudio é PII. Processar local quando possível; anonimizar (vocal ID removal).
6. **Language coverage**: inglês domina; outros idiomas, especialmente de baixa recurso, ficam para trás. MMS e Whisper ajudaram.
7. **Accents, dialetos, code-switching**: ASR frequentemente pior.
8. **Dados rotulados são caros** em áudio. SSL ajuda muito.
9. **Avaliação subjetiva** em TTS/síntese: MOS (Mean Opinion Score) — caro e ruidoso.
10. **Deploy edge**: MCU + áudio exige quantização agressiva + modelos tiny.

## Recursos

- **"Fundamentals of Music Processing"** — Meinard Müller. Livro referência em MIR.
- **"Speech and Language Processing"** — Jurafsky & Martin.
- **"Digital Signal Processing: Principles, Algorithms, and Applications"** — Proakis & Manolakis.
- **"Audio Processing and Speech Recognition"** — Rao & Manjunath.
- **Librosa tutorials** (librosa.org).
- **CCRMA (Stanford)** online materials.
- **ISMIR proceedings** — MIR research.
- **ICASSP, Interspeech, ASRU** — speech research.
- **DCASE** challenges + papers.
- **valerio-velardo.com** (YouTube "The Sound of AI") — didático.

## Princípios

1. **Trabalhe no domínio certo**. Tempo para onset grosseiro; frequência para harmonic; mel para percepção; CQT para música.
2. **Mel-spectrogram é o default moderno**. Resolvemos 80% dos problemas de classificação em cima dele.
3. **Sample rate importa**. ASR telefônico em 8 kHz ≠ broadcast 48 kHz; não misture sem resample.
4. **Cuidado com leakage** em janelas curtas e com aliasing em resample — artefatos são silenciosos até que quebrem.
5. **Augmentation é quase obrigatória**. SpecAugment + adição de ruído + RIR simulada.
6. **SSL pré-treinado ganha** em regime de poucos dados. Comece com Wav2Vec/HuBERT/BEATs.
7. **Privacidade em áudio**: voz identifica pessoa. Consent + minimização + edge processing quando possível.
8. **Avalie em condições reais**: dataset limpo ≠ produção com ruído, multi-speaker, microphone mismatch.
9. **Compressão de entrada corrompe features**. Prefira analisar pré-codec; quando não dá, treine com dados codec-corruptos.
10. **Perceptual loss > MSE** em síntese. Ouvido julga diferente de métrica simples.
