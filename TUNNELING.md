# Tunneling

O [tunneling](https://en.wikipedia.org/wiki/Tunneling_protocol) em redes de computadores é uma técnica que permite a transmissão de dados entre redes usando protocolos que normalmente não seriam suportados. Essencialmente, o tunneling encapsula um tipo de tráfego de rede dentro de pacotes transportados por outro protocolo. Isso é frequentemente usado para atravessar redes que não suportam o protocolo original ou para proporcionar uma camada adicional de segurança. Aqui estão algumas técnicas e protocolos de tunneling comuns:

1. **VPN (Virtual Private Network)**: Uma das aplicações mais comuns do tunneling é na criação de redes privadas virtuais. VPNs usam o tunneling para encapsular e criptografar dados, permitindo que eles transitem de forma segura através de redes públicas, como a Internet. Protocolos comuns de VPN incluem PPTP, L2TP, IPSec e OpenVPN.

2. **SSL/TLS**: O Secure Sockets Layer (SSL) e seu sucessor, o Transport Layer Security (TLS), são usados para criar um canal criptografado entre dois dispositivos. Eles são amplamente usados para segurança na web, como em conexões HTTPS.

3. **IPsec**: Usado principalmente para estabelecer conexões seguras em uma rede IP, o IPsec pode operar em dois modos: modo de transporte e modo de túnel. No modo de túnel, o IPsec encapsula um pacote inteiro em outro pacote e o criptografa, proporcionando segurança para a transmissão de dados.

4. **GRE (Generic Routing Encapsulation)**: O GRE é um protocolo simples de tunneling que pode encapsular uma variedade de tipos de protocolos de rede dentro de conexões virtuais ponto a ponto. É frequentemente usado em conjunto com o IPsec para adicionar uma camada de segurança.

5. **SSH (Secure Shell)**: Embora mais conhecido por fornecer terminais seguros para administração de sistemas, o SSH também pode ser usado para tunneling de dados. Pode-se encaminhar portas de rede locais para um servidor remoto através de uma conexão SSH criptografada.

6. **Tor (The Onion Router)**: O Tor é um exemplo de um sistema de tunneling complexo destinado a anonimizar a conexão do usuário na Internet. Ele encaminha o tráfego da Internet através de uma rede global de servidores, ocultando a localização e o uso do usuário.

Cada técnica de tunneling tem suas próprias características, vantagens e desvantagens. A escolha do método de tunneling apropriado depende das necessidades específicas de segurança, desempenho e compatibilidade da rede.

## SSH Tunneling

O SSH (Secure Shell) Tunneling é uma técnica poderosa usada para encaminhar tráfego de rede de forma segura através de um túnel criptografado. 

### 1. Encaminhamento de Porta Local (Local Port Forwarding)

Suponha que você queira acessar remotamente a interface web de um servidor de banco de dados que está executando na porta 8080, mas essa porta não é acessível fora da rede interna da sua empresa. Você pode criar um túnel SSH para encaminhar a porta 8080 do servidor para a porta 9090 em sua máquina local:

```bash
ssh -L 9090:localhost:8080 usuario@servidorSSH
```

Neste comando:
- `ssh`: é o comando para iniciar uma conexão SSH.
- `-L 9090:localhost:8080`: indica que o tráfego na porta local 9090 deve ser encaminhado para a porta 8080 no servidor.
- `usuario`: é o seu nome de usuário no servidor SSH.
- `servidorSSH`: é o endereço do servidor SSH (pode ser um endereço IP ou nome de domínio).

### 2. Encaminhamento de Porta Remota (Remote Port Forwarding)

Imagine que você quer permitir que alguém acesse um servidor web rodando na sua máquina local na porta 3000. Você pode encaminhar essa porta para a porta 8080 no servidor SSH remoto:

```bash
ssh -R 8080:localhost:3000 usuario@servidorSSH
```

Neste comando, `-R 8080:localhost:3000` significa que a porta 8080 no servidor SSH remoto será encaminhada para a porta 3000 na sua máquina local.

### 3. Encaminhamento de Porta Dinâmico (Dynamic Port Forwarding)

Para criar um proxy SOCKS em sua máquina local que encaminha o tráfego através de um servidor SSH:

```bash
ssh -D 1080 usuario@servidorSSH
```

Neste caso, `-D 1080` especifica que o ssh deve criar um proxy SOCKS na porta 1080 da sua máquina local. Você pode então configurar aplicações, como um navegador web, para usar este proxy e todo o tráfego da aplicação será encaminhado de forma segura através do servidor SSH.

### Desativar

Para desativar o tunneling em um servidor SSH, você precisa editar a configuração do serviço SSH, que geralmente está localizada no arquivo `sshd_config`.

1. **Acesse o arquivo de configuração SSH**: 
   Abra o arquivo `sshd_config` com um editor de texto.
   ```bash
   sudo nano /etc/ssh/sshd_config
   ```

2. **Desativar Encaminhamento de Porta**:
   - Para desativar todo o encaminhamento de porta, procure pelas seguintes linhas e altere ou adicione-as conforme necessário:
     ```
     AllowTcpForwarding no
     AllowStreamLocalForwarding no
     ```
   - Para desativar especificamente o encaminhamento de porta dinâmica (proxy SOCKS), adicione ou modifique a seguinte linha:
     ```
     PermitOpen none
     ```
   - Para desativar o encaminhamento de portas para usuários específicos, grupos ou endereços, você pode usar as diretivas `Match User`, `Match Group` ou `Match Address` para aplicar as configurações apenas a certas condições.

3. **Salvar e Fechar o Arquivo**: 
   Após fazer as alterações, salve o arquivo e feche o editor.

4. **Reinicie o Serviço SSH**:
   Para que as mudanças entrem em vigor, você precisará reiniciar o serviço SSH. Isso pode ser feito com um comando como:
   ```bash
   sudo systemctl restart sshd
   ```

5. **Verificar as Configurações**:
   Após reiniciar o serviço, é uma boa prática verificar se as configurações estão funcionando conforme esperado. Tente estabelecer um túnel SSH do seu cliente e veja se as restrições estão em vigor.

### Logs

Para verificar se há atividade de tunneling em um servidor SSH, você pode analisar os logs do SSH. 

1. **Visualizar o Log**:
     ```sh
     sudo tail -f /var/log/auth.log
     ```

2. **Procurar por Entradas de Tunneling**:
   - Nos logs, procure por linhas que indicam a criação de túneis SSH. Isso pode incluir palavras-chave como "port forwarding", "tunneled", ou detalhes específicos das portas encaminhadas.
   - Por exemplo, uma linha contendo "port forwarding" pode indicar uma tentativa de tunneling.

3. **Configurar o Nível de Log**:
   - Se você não encontrar informações suficientes, pode ser necessário ajustar o nível de log do SSH. Isso é feito editando o arquivo de configuração SSH (`/etc/ssh/sshd_config`) e alterando ou adicionando a linha `LogLevel`.
   - Níveis comuns incluem `INFO`, `VERBOSE`, `DEBUG`, etc. Por exemplo:
     ```
     LogLevel VERBOSE
     ```
   - Lembre-se de reiniciar o serviço SSH após alterar essa configuração para que as mudanças tenham efeito.

4. **Analisar Logs para Atividades Suspeitas**:
   - Ao analisar os logs, esteja atento a qualquer atividade incomum ou padrões que possam indicar uso indevido, como um grande número de tentativas de conexão, uso de portas incomuns para encaminhamento, ou padrões de tráfego anormal.
  
   ...
