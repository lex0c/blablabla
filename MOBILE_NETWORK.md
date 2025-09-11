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

## Comunicação

A comunicação entre celulares envolve uma complexa interação de hardware, software, protocolos de sinalização e infraestrutura de rede.

1. **Início da Comunicação**
   - Quando você decide fazer uma chamada ou enviar uma mensagem, seu celular inicia o processo se comunicando com a torre de celular mais próxima (ou estação base) usando ondas de rádio.

2. **Conexão com a Rede**
   - A estação base é conectada a uma infraestrutura maior de rede. Ela encaminha sua solicitação (seja para uma chamada, SMS ou uso de dados) para um controlador, que gerencia várias estações base.

3. **Roteamento**
   - Se você está fazendo uma chamada ou enviando uma mensagem, o controlador consulta os registros da rede para determinar a localização do dispositivo de destino. Para chamadas e mensagens, isso envolve consultar registros como o Home Location Register (HLR) ou Visitor Location Register (VLR).

4. **Estabelecendo a Comunicação**
   - Uma vez determinada a localização do dispositivo de destino, a rede estabelece um canal de comunicação entre os dois dispositivos. No caso de uma chamada, você ouvirá um tom de chamada enquanto isso está acontecendo.

5. **Comunicação**
   - Uma vez estabelecido o canal, os dados podem fluir entre os dois dispositivos. Isso pode ser voz, texto (SMS) ou outros dados.

6. **Encerramento**
   - Quando a comunicação é concluída (por exemplo, você encerra a chamada), a rede desmonta o canal de comunicação e libera os recursos associados.

7. **Comunicação de Dados**
   - Para serviços de dados (como navegação na web), o processo é um pouco diferente. Em vez de rotear a comunicação para outro dispositivo celular, a rede conecta o seu dispositivo a um gateway que fornece acesso à Internet. A partir daí, seu dispositivo pode se comunicar com servidores e outros recursos na web.

8. **Roaming**
   - Se você estiver em uma área que não é atendida por sua operadora original (por exemplo, viajando para outro país), sua comunicação é roteada através de redes locais. Isso é possível graças aos acordos de roaming entre operadoras e à sinalização global, como o SS7, que permite que redes diferentes em diferentes localizações se comuniquem e cooperem.

## LTE

[LTE](https://en.wikipedia.org/wiki/LTE_(telecommunication)) (Long-Term Evolution) é uma padrão de comunicação móvel de alta velocidade e alta capacidade destinado a dispositivos móveis e terminais de dados sem fio. É popularmente conhecido como 4G LTE, representando a quarta geração de tecnologias de telefonia móvel. Aqui estão alguns aspectos e características do LTE:

### 1. **Velocidade e Capacidade:**
   - O LTE oferece velocidades de download e upload significativamente mais rápidas em comparação com as tecnologias anteriores (3G, 2G).
   - Ele suporta navegação na web de alta velocidade, streaming de vídeo HD, jogos online e outras atividades de dados intensivos.

### 2. **Eficiência Espectral:**
   - LTE é projetado para ser mais eficiente espectralmente, permitindo maior transferência de dados em uma faixa de espectro limitada.

### 3. **Conectividade Melhorada:**
   - Ele proporciona uma melhor experiência de usuário com menor latência, o que significa um atraso reduzido durante a transmissão de dados.
   - Melhor desempenho em movimento, permitindo uma conexão estável mesmo em altas velocidades, como quando se viaja de carro ou trem.

### 4. **Arquitetura Simplificada:**
   - A arquitetura de rede LTE é mais simples e mais plana em comparação com as redes 3G, o que ajuda a reduzir os custos e a latência.

### 5. **Suporte a Múltiplas Frequências:**
   - O LTE pode operar em várias bandas de frequência, proporcionando flexibilidade para as operadoras e melhor cobertura para os usuários.

### 6. **Tecnologia MIMO:**
   - O LTE usa a tecnologia MIMO (Multiple Input Multiple Output) para aumentar a capacidade e desempenho da rede, utilizando múltiplas antenas para transmissão e recepção.

### 7. **Segurança:**
   - O LTE tem várias camadas de segurança para proteção contra várias ameaças, como interceptação e interferência.

### Vulnerabilidades:
   - Como qualquer tecnologia de comunicação sem fio, o LTE também tem suas vulnerabilidades, incluindo potenciais ataques de interceptação, rastreamento de localização e ataques de negação de serviço.
   - Há também preocupações com a segurança relacionadas à implementação de redes 5G, onde o LTE ainda desempenhará um papel, funcionando em conjunto com novas tecnologias 5G.

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

## Baseband

Baseband é o “cérebro de rádio” do seu celular. Ele fala com a torre, negocia criptografia, mantém a sessão de dados viva e decide quando sua ligação cai. Você raramente o vê, mas se ele falhar você vira um tijolo com bateria.

### O que é e onde vive

* **Chip dedicado** no SoC ou separado, com **firmware próprio** e um **RTOS** minimalista. Nada de Android ou iOS ali. É C em cima de C, com otimizações agressivas e zero paciência para bug.
* **Isolado** do processador de apps, mas não tanto. Conversa com o sistema via canais de IPC como **QMI** em Qualcomm, **RIL** no Android, filas em memória compartilhada, interfaces seriais e drivers no kernel.

### O que ele faz

1. **Camadas de rádio**

   * **PHY**: modulação, codificação, MIMO, beamforming.
   * **MAC**: agendamento de uplink e downlink.
   * **RLC/PDCP**: retransmissão, cifragem e compactação.
   * **RRC**: controle do rádio, estabelecimento de conexão, handover.
   * **NAS**: autenticação com a rede, mobilidade, gestão de sessão.
2. **Sessão de dados**

   * Cria **bearers** de dados com QoS, abre o **PDP context** e conecta ao seu **APN**. Há bearer default e dedicados para tráfego sensível.
3. **Voz**

   * 2G e 3G usam circuito, 4G e 5G usam **IMS** com **SIP** e **RTP**. VoLTE é baseband + pilha IMS.
4. **Mobilidade e energia**

   * Decide **handover**, DRX, quando dormir e acordar o rádio para economizar bateria.
5. **SIM e autenticação**

   * Fala com **SIM e eSIM**. Executa o **AKA** com a rede, deriva chaves e mantém segredos fora do Android iOS.

### Segurança em 2G 3G 4G 5G

* **2G**: cifragem fraca ou inexistente. Aceita **downgrade**. Perfeito para museu de vulnerabilidades.
* **3G**: melhora a autenticação e integridade. Ainda tem heranças.
* **4G LTE**: chaves derivadas fortes. Cifras típicas **AES** e **SNOW 3G** para cifragem e integridade.
* **5G**: autenticação reforçada, **SUCI** para esconder IMSI, novos domínios de chaves e separação melhor de planos.

Tradução sem marketing: 4G e 5G são muito melhores, contanto que você **não caia para 2G** e que o firmware não seja uma peneira.

### Caminho do tráfego

* **Plano de controle**: RRC e NAS ajustam rádio, autenticação, mobilidade.
* **Plano de usuário**: pacotes de dados passam por PDCP cifrados e seguem para a rede da operadora. O sistema operacional só vê pacotes depois do baseband.

### Por que é alvo valioso

* Recebe **entradas do ar** de qualquer torre válida ou fake.
* Roda **código privilegiado** com pouco sandboxing.
* Se cair, pode expor chamadas, SMS e localização sem pop-up de permissão.
* Correção é lenta. Patches dependem de fornecedor do modem, OEM e operadora. Boa sorte na fila.

### Superfícies de ataque típicas

Sem o manual do crime, só o mapa conceitual:

* **Sinalização RRC NAS malformada**: parser errado, buffer overflow. Clássico.
* **SMS especiais e WAP Push legados**: menos comum hoje, ainda existe.
* **Downgrade e BTS falsa**: força 2G, manipula autenticação fraca, injeta lixo.
* **Drivers de IPC**: ponte entre baseband e o sistema. Bug aí vira escalada para o kernel do AP.
* **SIM Toolkit e provisioning**: canais de configuração que já renderam CVEs.

### Como o SO conversa com ele

* **Android**: apps falam com **Telephony Framework**, que fala com **RIL**, que fala com o **vendor daemon** que fala com o **baseband** via QMI ou equivalente.
* **iOS**: arquitetura parecida, ainda mais fechada, com sandbox e políticas rígidas entre o modem e o userland.

### Atualizações e cadeia de confiança

* **Secure Boot** do modem valida firmware assinado pelo fornecedor.
* Atualizações chegam embaladas com o update do sistema ou do vendor. Muitas vezes atrasadas. Em aparelhos baratos, às vezes nunca.

### Diagnóstico e visibilidade

* Usuário vê quase nada. Pesquisadores usam **SDR** e stacks open source para observar a sinalização, o que exige laboratório e dinheiro. Não é “abrir um app e ver a mágica”.

### O que melhora sua vida na prática

* **Desative 2G** se seu aparelho e operadora permitirem. 2G é armadilha.
* **Mantenha VoLTE e 5G**. Mais segurança que cair para 3G 2G.
* **Compre aparelhos com política de update longa** e chipset recente.
* **Não instale lixo**. Se o atacante não quebrar o rádio, vai tentar quebrar você via APK.
* **Cuidado com eSIM de fontes aleatórias**. Provisionamento também é superfície.

## Vulnerabilidades

Smartphones, redes móveis e SMS têm uma variedade de vulnerabilidades que podem ser exploradas por atores maliciosos.

### Smartphones

**1. Sistemas Operacionais Desatualizados:** Muitos smartphones não têm as últimas atualizações de segurança, tornando-os suscetíveis a várias ameaças conhecidas.

**2. Aplicativos Maliciosos:** Aplicativos maliciosos podem ser baixados de fontes não confiáveis ou até mesmo de lojas oficiais de aplicativos que não conseguiram detectar o software malicioso.

**3. Phishing e Engenharia Social:** Os usuários podem ser enganados para baixar aplicativos, clicar em links ou fornecer informações pessoais.

**4. Interfaces Expostas:** Muitos smartphones têm interfaces, como Bluetooth, NFC e Wi-Fi, que, se não forem protegidas adequadamente, podem ser exploradas.

**5. Permissões excessivas:** Muitos aplicativos solicitam mais permissões do que realmente precisam, o que pode levar a violações de privacidade e outras ameaças.

### Redes Móveis

**1. Ataques Man-in-the-Middle:** Ataques em que um adversário intercepta a comunicação entre duas partes sem que elas saibam.

**2. Falsificação de Torre de Celular (Stingrays):** Dispositivos que se passam por torres de celular legítimas para interceptar comunicações móveis.

**3. SS7 (Signalling System No. 7) e vulnerabilidades Diameter:** Os protocolos de sinalização usados em redes móveis têm vulnerabilidades conhecidas que podem permitir a interceptação de chamadas e mensagens, bem como rastrear a localização do dispositivo.

### SMS

**1. Phishing por SMS (Smishing):** Ataques que usam mensagens SMS para enganar os usuários a clicar em links maliciosos ou fornecer informações pessoais.

**2. Interceptação:** Se a rede não estiver usando criptografia robusta ou se estiver vulnerável (por exemplo, através das vulnerabilidades SS7 mencionadas), as mensagens SMS podem ser interceptadas.

**3. Falsificação de SMS:** Ataques onde mensagens são enviadas parecendo ser de fontes confiáveis, mas são de atores maliciosos.

**4. Vulnerabilidades em Aplicativos de SMS:** Alguns aplicativos de mensagens têm vulnerabilidades que podem ser exploradas para executar código malicioso ou interceptar mensagens.

Para se proteger contra essas e outras vulnerabilidades, é importante manter os dispositivos e aplicativos atualizados, ter cuidado com as permissões dos aplicativos, ser cético em relação a links e mensagens desconhecidas e considerar o uso de soluções de segurança adicionais, como VPNs, firewalls e software antivírus.

