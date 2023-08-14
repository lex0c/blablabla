# Generating Function (Yield)

As [funções geradoras](https://en.wikipedia.org/wiki/Generator_(computer_programming)) são um conceito interessante em programação e são encontradas em várias linguagens modernas, como Python, JavaScript, C#, entre outras. Elas permitem gerar uma sequência de resultados de maneira preguiçosa, ou seja, os resultados são calculados sob demanda e não todos de uma vez.

Uma função geradora é definida como uma função regular, mas contém uma ou mais expressões [yield](https://developer.mozilla.org/pt-BR/docs/Web/JavaScript/Reference/Operators/yield). Quando uma expressão [yield](https://developer.mozilla.org/pt-BR/docs/Web/JavaScript/Reference/Operators/yield) é alcançada, a função geradora retorna o valor especificado, suspendendo sua execução. A função pode ser retomada posteriormente e continuará a partir do ponto onde parou.

## Exemplo em Python

Veja um exemplo simples de uma função geradora que produz uma sequência de números:

```python
def gera_numeros():
    i = 0
    while True:
        yield i
        i += 1

gerador = gera_numeros()
for numero in gerador:
    print(numero)
    if numero > 5:
        break
```

Nesse exemplo, a função `gera_numeros` continuará gerando números indefinidamente. O loop externo para a iteração quando o número é maior que 5, evitando um loop infinito.

### Vantagens

- **Eficiência de Memória**: Ao gerar valores sob demanda, a função geradora consome memória apenas para o valor atual, não para toda a sequência de valores que ela pode gerar.

- **Código Mais Limpo**: Em alguns casos, o uso de geradores pode levar a um código mais conciso e legível, especialmente quando você está trabalhando com fluxos de dados complexos.

- **Composição**: Geradores podem ser combinados e encadeados, criando fluxos de processamento de dados complexos de maneira mais direta e legível.

## Desvantagens

- **Complexidade**: Para programadores inexperientes, o conceito de geradores pode parecer um pouco confuso no início, especialmente a interação entre a função geradora e o código que a consome.

- **Desempenho**: Embora geradores possam ser mais eficientes em termos de memória, eles podem ser mais lentos em termos de tempo de execução em comparação com outras técnicas, dependendo do caso de uso.

## Eficiência de Memória

A eficiência de memória das funções geradoras pode ser especialmente útil quando se trabalha com grandes conjuntos de dados. Vamos comparar dois exemplos para ilustrar esse ponto, um usando uma lista tradicional e outro usando uma função geradora.

### Utilizando lista tradicional

Se você tentar criar uma lista de todos os números até um número grande, isso consumirá uma quantidade significativa de memória. Por exemplo:

```python
def gera_lista(n):
    return [i for i in range(n)]

# Usando a função para criar uma lista de 1 milhão de números
numeros = gera_lista(1000000)
```

Essa abordagem criará uma lista completa em memória com todos os números de 0 a 999999, consumindo uma quantidade significativa de memória.

### Utilizando função geradora

Agora, vamos fazer o mesmo usando uma função geradora:

```python
def gera_numeros(n):
    i = 0
    while i < n:
        yield i
        i += 1

# Usando a função geradora para iterar sobre 1 milhão de números
for numero in gera_numeros(1000000):
    pass  # Faça algo com cada número
```

Neste exemplo, a função geradora `gera_numeros` produz cada número de 0 a 999999 sob demanda, um de cada vez. Isso significa que apenas um número é mantido na memória de cada vez, em vez de todos os 1 milhão de números. Isso resulta em uma economia significativa de memória.

## Código Limpo

Funções geradoras podem ajudar a tornar o código mais limpo e legível, especialmente quando se trabalha com sequências de dados ou operações que podem ser naturalmente divididas em etapas. Vamos comparar dois exemplos para ilustrar esse ponto.

### Exemplo sem função geradora

Suponha que você queira processar uma série de números, filtrando os números ímpares, elevando-os ao quadrado e, em seguida, somando tudo. Sem utilizar geradores, você pode fazer algo assim:

```python
def processa_numeros(numeros):
    numeros_impares_quadrados = []
    for numero in numeros:
        if numero % 2 != 0:
            numeros_impares_quadrados.append(numero ** 2)
    return sum(numeros_impares_quadrados)

numeros = range(10)
resultado = processa_numeros(numeros)
print(resultado)  # Saída: 165
```

Essa abordagem funciona, mas mistura diferentes etapas do processamento em um único loop, tornando o código menos claro.

### Exemplo com função geradora

Agora vamos dividir as mesmas etapas em funções geradoras separadas, tornando o código mais legível e modular:

```python
def numeros_impares(numeros):
    for numero in numeros:
        if numero % 2 != 0:
            yield numero

def ao_quadrado(numeros):
    for numero in numeros:
        yield numero ** 2

numeros = range(10)
numeros_impares_gen = numeros_impares(numeros)
numeros_ao_quadrado_gen = ao_quadrado(numeros_impares_gen)

resultado = sum(numeros_ao_quadrado_gen)
print(resultado)  # Saída: 165
```

Neste exemplo, dividimos o processamento em duas funções geradoras: `numeros_impares` e `ao_quadrado`. Isso torna o código mais claro e cada função geradora realiza uma tarefa específica. Você pode entender, testar e reutilizar cada etapa do processamento de forma independente.

## Composição

Composição refere-se à capacidade de combinar várias funções geradoras de maneira encadeada para criar uma sequência de processamento mais complexa. Isso pode tornar o código mais modular, reutilizável e fácil de entender.

Vamos estender o exemplo anterior para mostrar como você pode compor várias funções geradoras para realizar tarefas mais complexas.

### Definindo funções geradoras

```python
def numeros_impares(numeros):
    for numero in numeros:
        if numero % 2 != 0:
            yield numero

def ao_quadrado(numeros):
    for numero in numeros:
        yield numero ** 2

def multiplica_por(numeros, fator):
    for numero in numeros:
        yield numero * fator
```

### Compondo funções geradoras

Agora você pode compor essas funções geradoras para criar uma cadeia de processamento. Suponha que você queira pegar os números ímpares de uma sequência, elevar ao quadrado e, em seguida, multiplicar por 5. Você pode fazer isso assim:

```python
numeros = range(10)
numeros_impares_gen = numeros_impares(numeros)
numeros_ao_quadrado_gen = ao_quadrado(numeros_impares_gen)
numeros_multiplicados_gen = multiplica_por(numeros_ao_quadrado_gen, 5)

resultado = sum(numeros_multiplicados_gen)
print(resultado)  # Saída: 825
```

### Benefícios da Composição

- **Modularidade**: Cada função geradora realiza uma tarefa específica e pode ser reutilizada em diferentes partes do código.
- **Legibilidade**: A composição torna o código mais fácil de entender, pois cada etapa do processamento é claramente definida.
- **Flexibilidade**: Você pode facilmente alterar, adicionar ou remover etapas na cadeia de processamento.

A capacidade de compor funções geradoras desta forma torna o código mais declarativo e expressivo, permitindo que você construa fluxos de processamento complexos a partir de blocos de construção simples e reutilizáveis. Isso é particularmente útil em aplicações de processamento de dados, onde você pode precisar realizar várias etapas de transformação em sequência.

## Lazy Reading

A leitura preguiçosa (ou "lazy reading") é um exemplo clássico de onde as funções geradoras podem ser úteis. É especialmente benéfico quando se trabalha com arquivos grandes que podem não caber na memória de uma só vez.

Suponha que você tenha um arquivo de texto grande com números, um em cada linha, e deseja somar todos os números ímpares que são maiores que 10.

### Função geradora para ler o arquivo

```python
def ler_arquivo_preguicosamente(nome_arquivo):
    with open(nome_arquivo, 'r') as arquivo:
        for linha in arquivo:
            yield linha.strip()  # Remove espaços em branco
```

### Função geradora para filtrar números ímpares e maiores que 10

```python
def filtrar_numeros(linhas):
    for linha in linhas:
        numero = int(linha)
        if numero % 2 != 0 and numero > 10:
            yield numero
```

### Função geradora para transformar (opcional)

Suponha que você queira transformar os números de alguma forma antes de agregá-los, por exemplo, multiplicando-os por 2. Você pode criar uma função geradora para isso:

```python
def transformar_numeros(numeros):
    for numero in numeros:
        yield numero * 2
```

### Agregando os resultados

Agora você pode compor essas funções geradoras para ler, filtrar, transformar e somar os números:

```python
nome_arquivo = 'meu_arquivo_grande.txt'
linhas = ler_arquivo_preguicosamente(nome_arquivo)
numeros_filtrados = filtrar_numeros(linhas)
numeros_transformados = transformar_numeros(numeros_filtrados)  # Opcional
resultado = sum(numeros_transformados)

print(f"A soma dos números ímpares maiores que 10, multiplicados por 2, é {resultado}")
```

Este exemplo mostra como você pode criar uma cadeia de processamento de dados usando funções geradoras. Cada função geradora na cadeia realiza uma etapa específica do processamento, tornando o código modular, reutilizável e fácil de entender. Você pode facilmente adicionar, remover ou modificar etapas na cadeia sem alterar o restante do código. Isso também permite que você processe grandes volumes de dados de maneira eficiente, sem a necessidade de carregar todo o conjunto de dados na memória.

### Vantagens

- **Eficiência de Memória**: Ao ler o arquivo uma linha de cada vez, você não precisa carregar o arquivo inteiro na memória. Isso é particularmente útil para arquivos grandes.

- **Simplicidade**: O código é simples e claro, e o gerador encapsula a lógica de leitura do arquivo, facilitando a reutilização em outras partes do seu código.

- **Flexibilidade**: Você pode facilmente combinar essa função geradora com outras para criar uma cadeia de processamento de dados. Por exemplo, você pode adicionar funções geradoras para filtrar, transformar ou agregar as linhas conforme são lidas.

Esta abordagem é uma excelente ilustração de como as funções geradoras podem ser usadas para criar um código mais eficiente e legível, especialmente ao trabalhar com recursos limitados ou grandes volumes de dados.

## Conclusão

O conceito de geradores e a palavra-chave `yield` são poderosos e versáteis, disponíveis em linguagens modernas como Python, C# e JavaScript. Eles ajudam a tornar o código mais eficiente, limpo e expressivo.
