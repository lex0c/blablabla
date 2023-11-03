# Criptografia

A [criptografia](https://en.wikipedia.org/wiki/Cryptography) é a prática e o estudo de técnicas para comunicação segura na presença de adversários. Tradicionalmente, a criptografia está associada à conversão de informações de uma forma legível para algo que parece ser aleatório e indecifrável, e vice-versa. Ela permite que informações sejam transmitidas ou armazenadas de forma segura, de modo que apenas as partes que possuem a "chave" correta possam ler o conteúdo.

A criptografia se divide em duas grandes categorias com base no tipo de chaves que utiliza:

1. **Criptografia Simétrica**: Também conhecida como criptografia de chave secreta, utiliza a mesma chave para criptografar e descriptografar a mensagem. O emissor e o receptor da mensagem devem ambos possuir a chave e mantê-la em segredo. Exemplos de algoritmos de criptografia simétrica incluem AES (Advanced Encryption Standard) e DES (Data Encryption Standard).

2. **Criptografia Assimétrica**: Também chamada de criptografia de chave pública, utiliza um par de chaves: uma pública e uma privada. A chave pública pode ser compartilhada abertamente para permitir que outros usuários criptografem mensagens, enquanto a chave privada é mantida em segredo pelo proprietário e é usada para descriptografar mensagens. Exemplos incluem RSA, ECC (Elliptic Curve Cryptography) e DSA (Digital Signature Algorithm).

A criptografia moderna envolve vários conceitos e técnicas avançadas, como:

- **Funções de Hash Criptográficas**: Funções que convertem uma quantidade arbitrária de dados em uma string de tamanho fixo, que atua como uma impressão digital dos dados. Qualquer pequena alteração nos dados originais produzirá um hash diferente. Exemplos são SHA-256 e MD5.

- **Protocolos de Comunicação Segura**: Como TLS/SSL, que são usados para segurança na Internet, garantindo que os dados transmitidos entre o navegador da web e o site sejam criptografados e seguros.

- **Assinaturas Digitais**: Um meio de verificar a autenticidade e integridade de uma mensagem ou documento digital, baseado em criptografia assimétrica.

- **Criptoanálise**: O estudo de métodos para derrotar a criptografia, ou seja, obter o significado de informações criptografadas sem acesso à chave secreta.

A criptografia é uma ferramenta vital em muitas aplicações de tecnologia da informação, como bancos online, comunicações confidenciais, e-commerce, senhas de acesso e muitos outros sistemas que dependem da segurança da informação.

## Assinatura Digital

A [assinatura digital](https://en.wikipedia.org/wiki/Digital_signature) é usada para verificar a autenticidade e integridade de mensagens digitais ou documentos. Uma assinatura digital garante que a mensagem ou documento foi emitido pelo remetente declarado e que não foi alterado após a assinatura. Aqui está uma visão geral de como funciona e por que é importante:

### Como Funciona

1. **Criação de Assinatura**:
   - O emissor da mensagem gera um resumo (hash) da mensagem original usando uma função de hash criptográfica.
   - Este resumo é então criptografado com a chave privada do emissor para criar a assinatura digital.
   - A mensagem original junto com a assinatura digital são então enviadas para o receptor.

2. **Verificação da Assinatura**:
   - O receptor da mensagem usa a chave pública do emissor para descriptografar a assinatura digital e recuperar o resumo da mensagem.
   - Em paralelo, o receptor gera um novo resumo da mensagem recebida usando a mesma função de hash utilizada pelo emissor.
   - Se o resumo descriptografado da assinatura corresponde ao resumo gerado pelo receptor, a assinatura é considerada válida, confirmando que a mensagem foi enviada pelo detentor da chave privada correspondente e não foi alterada.

### Importância

- **Autenticação**: As assinaturas digitais fornecem uma prova matemática de que a mensagem ou documento veio do emissor declarado.
- **Integridade**: Qualquer alteração na mensagem após a assinatura resultará em um resumo de hash diferente, indicando que a mensagem foi alterada.
- **Não Repúdio**: O emissor não pode negar a autoria da mensagem assinada, pois a assinatura só pode ser gerada com a chave privada do emissor, que é presumivelmente mantida em segredo.

### Aplicações

- **Documentos legais eletrônicos**: Contratos e documentos legais podem ser assinados digitalmente para dar-lhes autenticidade legal.
- **Software e atualizações**: Distribuidores de software frequentemente assinam seus produtos digitalmente para provar que o software vem de uma fonte confiável.
- **Transações online**: No e-commerce e em serviços bancários online, as assinaturas digitais ajudam a garantir que as transações sejam seguras e que as partes não possam negar a transação.
- **Sistemas de correio eletrônico**: Assinaturas digitais são usadas para verificar que os e-mails não foram forjados ou alterados.

### Tecnologias Envolvidas

- **RSA**: Um dos primeiros e mais utilizados algoritmos para assinaturas digitais.
- **ECDSA (Elliptic Curve Digital Signature Algorithm)**: Uma variante do algoritmo de assinatura digital que usa curvas elípticas para proporcionar um nível de segurança comparável com RSA, mas com chaves menores.
- **PGP (Pretty Good Privacy)**: Um programa que fornece criptografia e assinaturas digitais, comumente usado para e-mails seguros.

Assinaturas digitais são um pilar da segurança digital e são fundamentais para a confiança e a segurança em ambientes digitais.

...
