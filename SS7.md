# SS7

O [SS7 (Signaling System No. 7)](https://en.wikipedia.org/wiki/Signalling_System_No._7) é um protocolo internacional que define como os elementos da rede se comunicam em redes de telecomunicações públicas. Ele foi desenvolvido na década de 1970 para suportar redes de comutação de circuitos e desde então tem sido a espinha dorsal da sinalização em redes de telecomunicações em todo o mundo. Aqui está um resumo completo:

## Componentes Chave

- **STP (Signal Transfer Point):** Atua como um roteador para mensagens SS7, direcionando-as através da rede para seus destinos corretos.
  
- **SCP (Service Control Point):** Uma espécie de banco de dados que fornece informações que auxiliam no roteamento de chamadas, como traduzir números gratuitos em números de telefone reais.
  
- **SSP (Service Switching Point):** Centrais telefônicas que iniciam, encaminham ou terminam chamadas. Eles interagem com o sistema SS7 para realizar funções como estabelecer ou encerrar chamadas.

## Funcionalidades

1. **Estabelecimento de Chamada:** Quando um usuário inicia uma chamada, o SSP de origem envia uma mensagem através do SS7 para o SSP de destino para configurar a chamada. Esta sinalização inclui etapas como solicitando o estabelecimento da chamada, indicando quando um telefone está tocando e confirmando quando a chamada é atendida.

2. **Serviços de Valor Agregado:** O SS7 permite recursos como chamada em espera, redirecionamento de chamadas e identificação de chamadas.

3. **Mensagens SMS:** O protocolo é usado para rotear mensagens SMS entre telefones através de centros de mensagens curtas (SMSCs).

4. **Mobilidade em Redes Móveis:** Em redes celulares, o SS7 gerencia funções como localização de um dispositivo móvel e transferência de um dispositivo entre torres de celular (handover).

5. **Consulta de Informações:** O SCP pode ser consultado para obter informações específicas necessárias para rotear uma chamada ou fornecer um serviço específico.

## Vulnerabilidades

Por décadas, o SS7 tem sido o padrão para sinalização em redes de telecomunicações. No entanto, foi projetado em uma época em que as redes eram mais fechadas e os riscos de segurança eram diferentes. Como resultado:

- Atacantes que obtêm acesso ao SS7 podem, potencialmente, interceptar chamadas e mensagens, rastrear a localização de um telefone e até mesmo redirecionar serviços.
  
- O protocolo não verifica se as mensagens originadas são legítimas ou se vêm de uma fonte confiável.

- Há uma crescente conscientização sobre estas vulnerabilidades, e muitas operadoras e organizações estão implementando medidas de segurança para mitigar os riscos associados ao SS7.

### Spoofing

- **Spoofing de Localização:** Um atacante pode solicitar a localização atual de um número de telefone específico, fazendo parecer que a solicitação vem de uma operadora legítima. Isso pode ser usado para rastrear a localização de um indivíduo sem seu conhecimento ou consentimento.

- **Spoofing de Chamadas:** Um atacante pode iniciar chamadas que parecem vir de um número de telefone específico. Isso significa que eles podem fazer uma chamada para alguém e fazer parecer que a chamada está vindo de um número diferente do real.

- **Spoofing de SMS:** De maneira semelhante ao spoofing de chamadas, um atacante pode enviar mensagens SMS que parecem vir de qualquer número de telefone. Isso pode ser usado para enganar, defraudar ou confundir o destinatário.

- **Interceptação de Chamadas e Mensagens:** Embora isso não seja exatamente "spoofing", a capacidade de interceptar chamadas e mensagens é uma consequência das vulnerabilidades do SS7. Um atacante pode redirecionar chamadas ou mensagens destinadas a um número para outro número. Isso pode ser feito fazendo o sistema acreditar que o telefone do alvo está em uma localização geográfica diferente da real.

