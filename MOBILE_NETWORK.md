# Redes Móveis

As [redes móveis](https://en.wikipedia.org/wiki/Cellular_network) representam um complexo ecossistema de infraestrutura, padrões e tecnologias desenvolvidas ao longo dos anos para permitir a comunicação sem fio entre dispositivos móveis e sistemas fixos.

### 1G (Primeira Geração)

- **Tecnologia:** Analógica.
- **Função Principal:** Voz.
- **Características:** A primeira geração de telefonia móvel introduziu a comunicação sem fio, mas tinha uma qualidade de chamada limitada, era suscetível a interferências e não tinha criptografia.

### 2G (Segunda Geração)

- **Tecnologia:** Digital.
- **Função Principal:** Voz e texto (SMS).
- **Padrões Principais:** GSM (Global System for Mobile Communications), CDMA (Code Division Multiple Access).
- **Características:** Introdução do SMS, maior segurança através da criptografia, melhor qualidade de chamada e serviços como MMS e GPRS (General Packet Radio Service) para dados.

### 2.5G e 2.75G

- **Tecnologias:** GPRS e EDGE (Enhanced Data rates for GSM Evolution), respectivamente.
- **Função Principal:** Melhorias na transmissão de dados.
- **Características:** Estas são extensões do 2G que ofereceram velocidades de transmissão de dados mais rápidas, preparando o caminho para a internet móvel.

### 3G (Terceira Geração)

- **Tecnologia:** Digital com foco em dados de alta velocidade.
- **Função Principal:** Dados e voz.
- **Padrões Principais:** UMTS (Universal Mobile Telecommunications System) baseado em WCDMA (Wideband Code Division Multiple Access), CDMA2000.
- **Características:** Internet móvel de alta velocidade, chamadas de vídeo, maior capacidade e cobertura.

### 4G (Quarta Geração)

- **Tecnologia:** IP totalmente orientado a dados.
- **Função Principal:** Dados de alta velocidade.
- **Padrões Principais:** LTE (Long-Term Evolution), WiMAX.
- **Características:** Velocidades de internet muito mais rápidas (até 10 vezes mais rápidas que 3G), VoLTE (Voice over LTE), baixa latência.

### 4.5G (LTE-A)

- **Tecnologia:** Melhorias no LTE.
- **Função Principal:** Velocidades de dados ainda mais rápidas e melhorias de capacidade.
- **Características:** Agregação de portadoras, MIMO (Multiple Input, Multiple Output) e modulação avançada para aumentar a capacidade e a velocidade.

### 5G (Quinta Geração)

- **Tecnologia:** Novas faixas de frequência e infraestrutura.
- **Função Principal:** Ultra alta velocidade, baixa latência e capacidade massiva de dispositivos.
- **Características:** Suporte para IoT (Internet das Coisas), comunicação M2M (máquina a máquina), latência extremamente baixa, velocidades de até 10 Gbps ou mais.

## SS7

O [SS7 (Signaling System No. 7)](https://en.wikipedia.org/wiki/Signalling_System_No._7) é um protocolo internacional que define como os elementos da rede se comunicam em redes de telecomunicações públicas. Ele foi desenvolvido na década de 1970 para suportar redes de comutação de circuitos e desde então tem sido a espinha dorsal da sinalização em redes de telecomunicações em todo o mundo. Aqui está um resumo completo:

### Componentes Chave

- **STP (Signal Transfer Point):** Atua como um roteador para mensagens SS7, direcionando-as através da rede para seus destinos corretos.
  
- **SCP (Service Control Point):** Uma espécie de banco de dados que fornece informações que auxiliam no roteamento de chamadas, como traduzir números gratuitos em números de telefone reais.
  
- **SSP (Service Switching Point):** Centrais telefônicas que iniciam, encaminham ou terminam chamadas. Eles interagem com o sistema SS7 para realizar funções como estabelecer ou encerrar chamadas.

### Funcionalidades

1. **Estabelecimento de Chamada:** Quando um usuário inicia uma chamada, o SSP de origem envia uma mensagem através do SS7 para o SSP de destino para configurar a chamada. Esta sinalização inclui etapas como solicitando o estabelecimento da chamada, indicando quando um telefone está tocando e confirmando quando a chamada é atendida.

2. **Serviços de Valor Agregado:** O SS7 permite recursos como chamada em espera, redirecionamento de chamadas e identificação de chamadas.

3. **Mensagens SMS:** O protocolo é usado para rotear mensagens SMS entre telefones através de centros de mensagens curtas (SMSCs).

4. **Mobilidade em Redes Móveis:** Em redes celulares, o SS7 gerencia funções como localização de um dispositivo móvel e transferência de um dispositivo entre torres de celular (handover).

5. **Consulta de Informações:** O SCP pode ser consultado para obter informações específicas necessárias para rotear uma chamada ou fornecer um serviço específico.

### Vulnerabilidades

Por décadas, o SS7 tem sido o padrão para sinalização em redes de telecomunicações. No entanto, foi projetado em uma época em que as redes eram mais fechadas e os riscos de segurança eram diferentes. Como resultado:

- Atacantes que obtêm acesso ao SS7 podem, potencialmente, interceptar chamadas e mensagens, rastrear a localização de um telefone e até mesmo redirecionar serviços.
  
- O protocolo não verifica se as mensagens originadas são legítimas ou se vêm de uma fonte confiável.

- Há uma crescente conscientização sobre estas vulnerabilidades, e muitas operadoras e organizações estão implementando medidas de segurança para mitigar os riscos associados ao SS7.

#### Spoofing

- **Spoofing de Localização:** Um atacante pode solicitar a localização atual de um número de telefone específico, fazendo parecer que a solicitação vem de uma operadora legítima. Isso pode ser usado para rastrear a localização de um indivíduo sem seu conhecimento ou consentimento.

- **Spoofing de Chamadas:** Um atacante pode iniciar chamadas que parecem vir de um número de telefone específico. Isso significa que eles podem fazer uma chamada para alguém e fazer parecer que a chamada está vindo de um número diferente do real.

- **Spoofing de SMS:** De maneira semelhante ao spoofing de chamadas, um atacante pode enviar mensagens SMS que parecem vir de qualquer número de telefone. Isso pode ser usado para enganar, defraudar ou confundir o destinatário.

- **Interceptação de Chamadas e Mensagens:** Embora isso não seja exatamente "spoofing", a capacidade de interceptar chamadas e mensagens é uma consequência das vulnerabilidades do SS7. Um atacante pode redirecionar chamadas ou mensagens destinadas a um número para outro número. Isso pode ser feito fazendo o sistema acreditar que o telefone do alvo está em uma localização geográfica diferente da real.

## Diameter

O [Diameter](https://en.wikipedia.org/wiki/Diameter_(protocol)) é um protocolo de sinalização de autenticação, autorização e contabilidade (AAA) que foi desenvolvido para substituir o protocolo mais antigo chamado "RADIUS" (Remote Authentication Dial-In User Service). Enquanto o RADIUS foi predominantemente usado para acesso dial-up, o Diameter foi projetado para atender a requisitos mais modernos e para ser mais versátil em ambientes de rede, especialmente nas redes 3G, 4G (LTE) e futuras redes móveis.

### SS7
- **Chamadas de Voz (2G e 3G):** Em redes 2G (GSM) e 3G (UMTS), o SS7 é a principal tecnologia de sinalização utilizada para estabelecer, gerenciar e encerrar chamadas de voz.
  
- **SMS (2G e 3G):** O SS7 também é usado para enviar e receber mensagens SMS em redes GSM e UMTS. Quando uma mensagem SMS é enviada, ela passa por um centro de serviço de mensagens curtas (SMSC) que utiliza SS7 para rotear e entregar a mensagem.

### Diameter
- **4G (LTE) e além:** À medida que as redes evoluem para 4G e além, o protocolo Diameter começa a substituir o SS7 para a sinalização de serviços de dados. No entanto, para serviços de voz e SMS em redes 4G, a situação é um pouco mais complexa.
  
- **Voz sobre LTE (VoLTE):** Em redes 4G que suportam VoLTE (Voice over LTE), o protocolo Diameter é usado para a sinalização de chamadas de voz. No entanto, muitas redes 4G não possuem VoLTE e, em vez disso, regridem para 2G ou 3G para chamadas de voz (usando SS7 para sinalização).

- **SMS em 4G:** Mesmo em redes 4G, o SMS ainda pode usar a sinalização SS7, especialmente quando a rede regride para 2G/3G para serviços de voz e mensagens. No entanto, em redes com VoLTE, o SMS pode ser enviado sobre IP usando o protocolo Diameter para sinalização.

Resumindo: enquanto o Diameter desempenha um papel crescente na sinalização de redes móveis modernas, especialmente para serviços de dados e VoLTE, o SS7 ainda é amplamente utilizado para chamadas de voz e SMS, especialmente em redes 2G e 3G, e em redes 4G sem suporte a VoLTE.

