# Raspberry Pi

O [Raspberry Pi](https://www.raspberrypi.com/), devido ao seu tamanho compacto, baixo custo e flexibilidade, é utilizado em uma ampla variedade de aplicações. Alguns dos usos mais comuns incluem:

1. **Computador de Aprendizagem**: Por ser acessível, o Raspberry Pi é frequentemente usado em escolas e programas de educação para ensinar programação e conceitos básicos de informática.

2. **Estação de Trabalho Leve**: Equipado com um sistema operacional como o Raspberry Pi OS (anteriormente conhecido como Raspbian), ele pode ser usado para navegação web, processamento de texto e outras tarefas básicas de computação.

3. **Media Center**: Usando software como o Kodi ou Plex, o Raspberry Pi pode ser transformado em um poderoso media center para reprodução de vídeos, músicas e imagens.

4. **Estação de Jogos Retrô**: Com emuladores como o RetroPie, é possível jogar uma vasta quantidade de jogos clássicos em um Raspberry Pi.

5. **Automatização Residencial**: O Raspberry Pi pode ser usado para controlar luzes, termostatos, câmeras e outros dispositivos em uma casa inteligente.

6. **Servidores Leves**: Seja como um servidor web básico, servidor de arquivos usando o software Samba ou um servidor VPN, o Raspberry Pi pode lidar com várias tarefas de servidor em pequena escala.

7. **Estação de Monitoramento**: Pode ser usado para monitorar condições ambientais, como temperatura e umidade, ou como uma câmera de segurança com o módulo de câmera Raspberry Pi.

8. **Aprendizado de Máquina e IA**: Com a crescente disponibilidade de ferramentas e frameworks, o Raspberry Pi tem sido usado para projetos básicos de inteligência artificial e aprendizado de máquina.

9. **Robótica**: O Raspberry Pi é frequentemente usado como o cérebro de pequenos robôs, drones ou veículos controlados remotamente.

10. **Agricultura Inteligente**: Monitoramento de condições do solo, irrigação automatizada e monitoramento de estufas são algumas das aplicações na agricultura.

11. **Projetos DIY e Artes**: Seja em instalações interativas, arte eletrônica ou projetos faça-você-mesmo, o Raspberry Pi encontra seu lugar em inúmeras criações artísticas e inventivas.

12. **Estudos e Pesquisas**: Por ser de baixo custo, é comum encontrar Raspberry Pi em pesquisas acadêmicas e projetos de estudantes, especialmente quando há necessidade de coleta de dados ou prototipagem.

13. **Navegação e Rastreamento**: Com o acréscimo de módulos GPS, o Raspberry Pi pode ser usado em aplicações de rastreamento e navegação.

Estes são apenas alguns dos muitos usos do Raspberry Pi. A verdadeira extensão de suas aplicações é limitada apenas pela imaginação e criatividade do usuário. O Raspberry Pi revolucionou a forma como os entusiastas e profissionais abordam a computação embarcada e continua a inspirar uma ampla gama de projetos inovadores.

## Server

O Raspberry Pi é bastante adequado para servir como um pequeno servidor Linux para várias aplicações, graças ao seu desempenho decente, baixo consumo de energia e custo acessível.

### Vantagens:

1. **Desempenho**: O Raspberry Pi 4 Model B, com até 8 GB de RAM e um CPU quad-core, oferece desempenho suficiente para muitas aplicações de servidor de pequena escala.

2. **Consumo de Energia**: O Raspberry Pi consome relativamente pouca energia, tornando-o eficiente para operação contínua 24/7.

3. **Custo**: O Raspberry Pi é uma opção de baixo custo comparado a um servidor tradicional ou mesmo a muitos mini PCs.

4. **Comunidade e Suporte**: Há uma comunidade vibrante e ativa em torno do Raspberry Pi. Isso significa que há uma abundância de recursos, tutoriais e especialistas que podem ajudar a resolver problemas e melhorar seu servidor.

5. **Flexibilidade**: Você tem a flexibilidade de escolher entre várias distribuições Linux que são otimizadas para o Raspberry Pi, como Raspberry Pi OS, Ubuntu Server e muitos outros.

### Aplicações Comuns:

- **Servidor Web**: Hospedar um website, blog, ou aplicativo web pessoal.
- **Servidor de Mídia**: Como um servidor Plex ou Emby para streaming de mídia.
- **Servidor de Arquivos**: Usando Samba ou NFS para compartilhar arquivos na rede local.
- **VPN Server**: Para acesso seguro à sua rede doméstica enquanto estiver fora.
- **Servidor de Impressão**: Para gerenciar e compartilhar impressoras na rede.
- **Laboratório de Testes**: Para testar e experimentar com software em um ambiente isolado.
- **Automação Residencial**: Rodando plataformas como Home Assistant ou openHAB.

### Considerações:

- **Desempenho Limitado**: Para tarefas muito exigentes em termos de CPU, RAM ou I/O, um servidor mais potente pode ser mais adequado.
- **Armazenamento**: O Raspberry Pi usa cartões microSD para armazenamento, que podem não ser tão duráveis ou rápidos quanto as soluções de armazenamento tradicionais, como SSDs ou HDDs.
- **Redundância e Backup**: O Raspberry Pi não oferece funcionalidades incorporadas para RAID ou outras formas de redundância de dados. Planeje uma estratégia de backup adequada.

## Segurança

Manter a segurança de um servidor é crucial, especialmente se estiver exposto à internet.

### 1. **Atualizações e Patches**:
   - Mantenha o sistema e todos os pacotes instalados atualizados para garantir que todas as vulnerabilidades conhecidas estejam corrigidas.

### 2. **Senha Forte**:
   - Use senhas fortes e únicas para todas as contas, especialmente para o usuário root e outros usuários com privilégios de superusuário.

### 3. **SSH Secure**:
   - Se estiver usando SSH, desative o login como root e use a autenticação por chave em vez de senhas.
   - Considerar a mudança da porta SSH padrão (22) para uma porta não padrão para evitar ataques automatizados.

### 4. **Firewall**:
   - Configure um firewall para restringir o acesso a serviços e portas. `UFW` (Uncomplicated Firewall) é uma boa opção para usuários de Ubuntu.
   - Permita apenas as portas necessárias para os serviços que você está executando.

### 5. **Reduzir Serviços**:
   - Execute apenas os serviços necessários. Menos serviços significam menos vetores de ataque.
   - Desabilite e desinstale serviços que não são necessários.

### 6. **Controle de Acesso**:
   - Configure permissões de arquivo e diretório apropriadamente para restringir o acesso.
   - Use o princípio do menor privilégio ao configurar permissões de usuário e grupo.

### 7. **Monitoramento e Auditoria**:
   - Configure a auditoria e o monitoramento para acompanhar o acesso e as alterações no sistema.
   - Ferramentas como `fail2ban` podem ajudar a proteger contra ataques de força bruta.

### 8. **Backups**:
   - Mantenha backups regulares de dados importantes e configurações do sistema.
   - Certifique-se de que os backups estejam seguros e protegidos contra acesso não autorizado.

### 9. **HTTPS/TLS**:
   - Se estiver executando um servidor web, considere configurar HTTPS para criptografar o tráfego entre o servidor e os clientes.
   - Let’s Encrypt oferece certificados TLS gratuitos e é fácil de configurar.

### 10. **Segurança de Rede**:
   - Proteja a rede em que o Raspberry Pi está conectado, garantindo que o roteador e outros dispositivos estejam seguros.
   - Considere colocar o Raspberry Pi em uma VLAN separada se a rede suportar.

### 11. **Testes de Segurança**:
   - Considere realizar testes de penetração ou scans de vulnerabilidade para identificar e corrigir possíveis vulnerabilidades.

Implementar essas práticas e continuamente manter-se atualizado com as melhores práticas de segurança e vulnerabilidades conhecidas ajudará a manter seu sistema Raspberry Pi mais seguro.

## Pi-hole

[Pi-hole](https://pi-hole.net) é uma aplicação popular que permite que você configure um servidor DNS ad-blocker na sua rede local, bloqueando anúncios em todos os dispositivos da sua rede sem a necessidade de instalar extensões ou aplicativos de bloqueio de anúncios em cada dispositivo individualmente.

### 1. **Conecte o Raspberry Pi à Rede e Acesse o Terminal**:
   - Certifique-se de que o Raspberry Pi esteja conectado à sua rede local e que você possa acessar o terminal, seja diretamente ou via SSH.

### 2. **Atualize o Sistema**:
   - Mantenha o sistema atualizado executando os comandos:
     ```
     sudo apt update
     sudo apt upgrade
     ```

### 3. **Instale o Pi-hole**:
   - Você pode instalar o Pi-hole usando o seguinte comando curl:
     ```
     curl -sSL https://install.pi-hole.net | bash
     ```
   - O script de instalação será executado, e você será guiado através do processo de configuração.

### 4. **Configuração**:
   - Durante a instalação, o Pi-hole irá pedir-lhe para configurar várias opções, como a escolha do upstream DNS provider, seleção de interfaces de rede e definição de IP estático. Siga as instruções na tela para completar a configuração.

### 5. **Acesso à Interface Web**:
   - Após a instalação, você deve ser capaz de acessar a interface web do Pi-hole usando um navegador web, indo para o endereço IP do seu Raspberry Pi na porta 80.
   - Por exemplo, `http://<IP_DO_SEU_RASPBERRY_PI>/admin`

### 6. **Configurar Dispositivos para Usar o Pi-hole**:
   - Você pode configurar individualmente cada dispositivo para usar o Pi-hole como seu DNS, ou configurar o roteador para usar o Pi-hole como DNS, forçando todos os dispositivos na sua rede a usá-lo automaticamente.

### 7. **Manutenção e Atualização**:
   - Regularmente, verifique se há atualizações para o Pi-hole e mantenha-o atualizado para garantir a melhor performance e segurança.

### Considerações Importantes:

- **IP Estático**: Certifique-se de que o Raspberry Pi tenha um IP estático configurado para evitar problemas se o IP mudar.
- **Senha de Administração**: Defina uma senha forte para a interface web do Pi-hole para prevenir acessos não autorizados.
- **HTTPS**: Considere configurar HTTPS para a interface web para aumentar a segurança.

O Pi-hole é uma ferramenta poderosa para melhorar a privacidade e a performance da rede, reduzindo a quantidade de anúncios e rastreadores que são capazes de acessar seus dispositivos.

## Static IP

Definir um IP estático para o seu Raspberry Pi na sua rede local pode ser feito de duas maneiras principais: configurando o Raspberry Pi para solicitar sempre o mesmo IP ou configurando o seu roteador para atribuir sempre o mesmo IP ao Raspberry Pi (reserva de DHCP). Aqui estão as instruções para ambos os métodos:

### **Método 1: Configurar IP Estático no Raspberry Pi**

1. **Abra o terminal** no seu Raspberry Pi e edite o arquivo de configuração `dhcpcd.conf` usando um editor de texto. Você pode usar o `nano`:
   ```bash
   sudo nano /etc/dhcpcd.conf
   ```

2. **Vá até o final do arquivo** e adicione as seguintes linhas:
   ```bash
   interface eth0
   static ip_address=192.168.1.100/24
   static routers=192.168.1.1
   static domain_name_servers=192.168.1.1
   ```
   - `interface eth0`: Especifica que você está configurando a interface Ethernet. Use `wlan0` para Wi-Fi.
   - `static ip_address`: O IP estático que você deseja atribuir ao Raspberry Pi.
   - `static routers`: O endereço IP do seu roteador/gateway.
   - `static domain_name_servers`: O endereço IP do seu DNS (geralmente o mesmo que o roteador).

3. **Salve e feche o arquivo**. Se você estiver usando `nano`, pressione `CTRL+X`, depois `Y` para confirmar as mudanças e `ENTER` para sair.

4. **Reinicie o Raspberry Pi** para aplicar as configurações:
   ```bash
   sudo reboot
   ```

### **Método 2: Reserva de DHCP no Roteador**

1. **Acesse as configurações do roteador**: Abra um navegador web e entre no endereço IP do seu roteador, geralmente algo como `192.168.1.1`.

2. **Localize as configurações de DHCP**: Navegue pelas configurações até encontrar a seção de DHCP ou algo similar.

3. **Adicione uma Reserva de DHCP**: 
   - Procure uma opção para adicionar uma reserva de DHCP ou IP estático.
   - Você precisará fornecer o endereço MAC do Raspberry Pi e o endereço IP que deseja reservar.

4. **Salve as configurações e reinicie o roteador se necessário**.

5. **Reinicie o Raspberry Pi** para garantir que ele obtenha o novo endereço IP reservado.

O método da reserva de DHCP é geralmente preferido porque permite gerenciar todos os IPs estáticos em um único lugar e reduz a possibilidade de conflitos de IP.

## Domain

Domínio ou hostname personalizado em rede local para acessar o Pi-hole ou outros serviços no seu Raspberry Pi, você pode fazer isso configurando um servidor DNS local ou editando o arquivo hosts nos dispositivos da sua rede. Aqui estão as opções:

### **Opção 1: Configurando um Servidor DNS Local**

1. **Use o Pi-hole como seu DNS Server**:
   - Certifique-se de que todos os dispositivos na sua rede estejam configurados para usar o Pi-hole como seu servidor DNS.

2. **Adicionar Entrada DNS Customizada no Pi-hole**:
   - Acesse a interface de administração do Pi-hole.
   - Vá até "Local DNS Records" e adicione um novo registro com o nome de domínio desejado e o endereço IP do seu Raspberry Pi.

### **Opção 2: Editando o Arquivo Hosts nos Dispositivos**

- **Windows**:
   1. Navegue até `C:\Windows\System32\drivers\etc`.
   2. Abra o arquivo `hosts` com um editor de texto como administrador.
   3. Adicione uma linha com o IP do Raspberry Pi seguido pelo domínio desejado, por exemplo:
      ```
      192.168.1.100 pihole.local
      ```

- **Mac e Linux**:
   1. Abra o terminal.
   2. Edite o arquivo hosts usando um editor de texto, por exemplo usando o comando:
      ```
      sudo nano /etc/hosts
      ```
   3. Adicione uma linha como no exemplo do Windows.

### **Opção 3: Usando um Router com Funcionalidades de DNS**

- Alguns roteadores permitem que você adicione entradas DNS customizadas.
   1. Acesse as configurações do seu roteador.
   2. Procure por configurações DNS ou DHCP e adicione uma entrada apontando para o IP do seu Raspberry Pi.

## HTTPS

Para configurar o HTTPS na interface web do Pi-hole, você pode usar um certificado SSL/TLS. O Let’s Encrypt é uma autoridade certificadora que fornece certificados SSL/TLS gratuitos e é amplamente utilizado para esse propósito. Aqui está um guia passo a passo sobre como configurar o HTTPS no Pi-hole usando o Let’s Encrypt:

### 1. **Instale o Certbot**:
   - Instale o Certbot, uma ferramenta que automatiza a obtenção e a renovação de certificados SSL/TLS do Let’s Encrypt.
     ```
     sudo apt update
     sudo apt install certbot
     ```

### 2. **Obtenha um Certificado SSL/TLS**:
   - Execute o Certbot para obter um certificado. Você precisará ter um domínio apontando para o IP do seu Raspberry Pi.
     ```
     sudo certbot certonly --standalone --preferred-challenges http -d your_domain.com
     ```

### 3. **Configurar Lighttpd para Usar o Certificado**:
   - Edite o arquivo de configuração do Lighttpd (o servidor web usado pelo Pi-hole).
     ```
     sudo nano /etc/lighttpd/external.conf
     ```
   - Adicione ou edite as seguintes linhas para configurar SSL:
     ```conf
     $SERVER["socket"] == ":443" {
         ssl.engine = "enable"
         ssl.pemfile = "/etc/letsencrypt/live/your_domain.com/combined.pem"
     }
     ```
   - Combine o certificado e a chave privada em um único arquivo:
     ```
     sudo cat /etc/letsencrypt/live/your_domain.com/fullchain.pem /etc/letsencrypt/live/your_domain.com/privkey.pem > /etc/letsencrypt/live/your_domain.com/combined.pem
     ```

### 4. **Reinicie o Servidor Web Lighttpd**:
   - Reinicie o Lighttpd para aplicar as mudanças.
     ```
     sudo service lighttpd restart
     ```

### 5. **Automatize a Renovação do Certificado**:
   - Certificados Let’s Encrypt são válidos por 90 dias. Automatize a renovação editando o crontab.
     ```
     sudo crontab -e
     ```
   - Adicione uma linha para renovar o certificado e reiniciar o Lighttpd automaticamente:
     ```
     0 2 * * * /usr/bin/certbot renew --quiet && cat /etc/letsencrypt/live/your_domain.com/fullchain.pem /etc/letsencrypt/live/your_domain.com/privkey.pem > /etc/letsencrypt/live/your_domain.com/combined.pem && service lighttpd restart
     ```

### 6. **Verifique a Configuração**:
   - Acesse a interface web do Pi-hole usando https e o domínio que você configurou.
     ```
     https://your_domain.com/admin
     ```

### Notas Importantes:
- **Domínio**: Você deve ter um domínio apontando para o endereço IP do seu Raspberry Pi.
- **Portas**: Certifique-se de que as portas 80 e 443 estejam abertas no seu roteador e redirecionadas para o Raspberry Pi.
- **Segurança**: Usar HTTPS ajuda a proteger as informações transmitidas entre o seu navegador e o Pi-hole, especialmente útil se você planeja acessar a interface administrativa fora da sua rede local.

---

Depois de configurar, você deverá ser capaz de acessar o Pi-hole ou outro serviço no seu Raspberry Pi usando o domínio ou hostname personalizado que você configurou. Tenha em mente que essas configurações são locais e só funcionarão na sua rede local.
