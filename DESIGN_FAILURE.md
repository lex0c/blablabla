# Design para Falhas

Em um mundo ideal, todos os sistemas funcionariam perfeitamente, o tempo todo. Mas vivemos no mundo real, onde falhas são inevitáveis. Se você está lidando com grandes volumes de tráfego e infraestruturas complexas, a pergunta não é "se" uma falha ocorrerá, mas "quando". É aqui que entra o conceito de "[Design para Falhas](https://en.wikipedia.org/wiki/Fail-fast)", uma abordagem que nos ajuda a construir sistemas mais resilientes e confiáveis.

## O Que é Design para Falhas?

"Design para Falhas" é a arte de projetar sistemas de modo a antecipar, tolerar e se recuperar de falhas. Em vez de ver a falha como uma exceção, ela é tratada como um aspecto inerente e inevitável do sistema. Isso muda completamente a abordagem de engenharia, desde a arquitetura até a entrega e operação de software.

## Estratégias para Design de Sistemas Resilientes

### Redundância

Sempre tenha um plano B. Se um servidor falhar, outro deve estar pronto para assumir. Redundância não é apenas uma boa prática; é um requisito para garantir alta disponibilidade.

### Replicação

Para bancos de dados e outros sistemas de armazenamento, a replicação mantém cópias dos dados em locais diferentes. Se um servidor falhar, a aplicação ainda poderá acessar uma [cópia dos dados](https://en.wikipedia.org/wiki/Backup).

### [Failover](https://en.wikipedia.org/wiki/Failover) Automático

O failover automático garante que, em caso de falha de um componente, outro assumirá instantaneamente, mantendo a continuidade do serviço.

### [Circuit Breakers](https://microservices.io/patterns/reliability/circuit-breaker.html)

Essa técnica é como um disjuntor que "desliga" um serviço ao detectar condições que provavelmente levariam a falhas, evitando sobrecarga e permitindo a recuperação.

### [Retries com Backoff Exponencial](https://en.wikipedia.org/wiki/Exponential_backoff)

Quando uma operação falha, é tentada novamente após um pequeno intervalo. Se continuar falhando, o tempo entre as tentativas aumenta exponencialmente. Isso dá ao sistema uma chance de se recuperar.

### [Balanceamento de Carga](https://en.wikipedia.org/wiki/Load_balancing_(computing))

Distribuir o tráfego de forma eficaz pode evitar pontos únicos de falha e maximizar a utilização dos recursos.

### Monitoramento e Alertas

Não podemos corrigir o que não conhecemos. Monitorar o sistema e ter alertas bem configurados são fundamentais para uma rápida intervenção em caso de problemas.

### [Testes de Caos](https://en.wikipedia.org/wiki/Chaos_engineering)

Esta abordagem envolve injetar falhas intencionais no sistema para testar sua resiliência. É uma forma extrema, mas eficaz, de preparar o sistema para falhas inesperadas.

## Idempotência

Idempotência é um conceito amplamente utilizado na ciência da computação e na matemática e refere-se à propriedade pela qual uma operação pode ser aplicada várias vezes sem alterar o resultado além da aplicação inicial. Em outras palavras, realizar a operação uma ou múltiplas vezes tem o mesmo efeito que realizá-la apenas uma vez.

**Exemplo Clássico**: Imagine um botão de elevador. Se você pressionar o botão para ir ao 10º andar uma vez ou várias vezes, o resultado é o mesmo: o elevador vai ao 10º andar.

### Idempotência em Sistemas de Computador

A idempotência é particularmente relevante em sistemas distribuídos e na construção de APIs. Vejamos alguns contextos:

- **APIs REST**: Uma requisição HTTP `GET` para um serviço RESTful deve ser idempotente. Isso significa que não importa quantas vezes você faça a mesma requisição `GET`, o resultado (ou efeito colateral) deve ser o mesmo. Do mesmo modo, métodos como `PUT` e `DELETE` também são esperados para serem idempotentes. No entanto, `POST` não é necessariamente idempotente, o que significa que enviar a mesma requisição `POST` várias vezes pode resultar em diferentes efeitos colaterais.

- **Retentativas em Sistemas Distribuídos**: Em sistemas distribuídos, falhas de rede ou timeouts podem resultar em incerteza sobre se uma operação foi concluída. Se uma operação for idempotente, o sistema pode tentar a operação novamente sem preocupação, sabendo que não haverá efeitos colaterais negativos indesejados.

- **Operações de Banco de Dados**: Idempotência pode ser útil em transações de banco de dados. Se uma transação for interrompida (por exemplo, devido a uma falha de sistema), sabendo que a operação é idempotente permite que o sistema tente a transação novamente sem medo de duplicar efeitos.

### Por que é importante?

A idempotência é crucial para garantir a consistência e a confiabilidade, especialmente em sistemas distribuídos onde a comunicação entre os componentes pode ser incerta ou falível. Ao projetar operações que são idempotentes, os engenheiros podem construir sistemas que são mais robustos frente a falhas, e que podem se recuperar graciosamente quando as falhas ocorrem.

## Conclusão

Design para falhas não é apenas uma técnica; é uma mudança de mentalidade. Em vez de gastar recursos tentando evitar todas as falhas – um objetivo inatingível –, focamos em como podemos nos recuperar delas. Dessa forma, criamos sistemas mais resilientes, robustos e, em última análise, mais confiáveis.

Portanto, da próxima vez que você estiver projetando um sistema, não pergunte apenas como evitar falhas; pergunte também como você vai lidar com elas quando ocorrerem. Afinal, um sistema verdadeiramente resiliente não é aquele que nunca falha, mas aquele que sabe como se recuperar depois de falhar.
