# Testes de Software

No mundo em constante evolução da tecnologia, garantir a [qualidade do software](https://en.wikipedia.org/wiki/Software_quality) é mais crucial do que nunca. Uma das principais estratégias que as organizações empregam para assegurar a robustez do software é o teste.

## Testes Manuais

Os testes manuais são executados por seres humanos, onde os testadores interagem diretamente com o software, seguindo um plano de teste pré-definido.

**Vantagens:**
1. **Flexibilidade:** Os testadores podem mudar seu curso de ação baseado no que observam, permitindo identificar erros inesperados.
2. **Intuição:** A capacidade de perceber problemas sutis ou questões de usabilidade que uma máquina pode não detectar.
3. **Baixo custo inicial:** Não exige investimento em ferramentas de automação ou na criação de scripts.

**Desvantagens:**
1. **Velocidade:** É demorado, principalmente em grandes aplicações.
2. **Inconsistência:** A possibilidade de erro humano pode levar à inconsistência nos testes.
3. **Repetibilidade:** Cada vez que uma alteração é feita, os testes precisam ser refeitos manualmente.

## Testes Automatizados

Os testes automatizados usam ferramentas e scripts específicos para executar testes em uma aplicação automaticamente.

**Vantagens:**
1. **Velocidade:** Os testes podem ser realizados rapidamente, muitas vezes em questão de minutos.
2. **Consistência:** Garantem que cada teste seja executado da mesma maneira todas as vezes.
3. **Reusabilidade:** Podem ser realizados várias vezes, em diferentes versões ou configurações do software.
4. **Integração com CI/CD:** Eles se encaixam perfeitamente em ambientes de Integração e Entrega Contínua.

**Desvantagens:**
1. **Custo inicial:** Exige tempo e recursos para configurar.
2. **Manutenção:** À medida que a aplicação muda, os scripts de teste também precisam ser atualizados.

## Testes de Regressão

Imagine o cenário: uma equipe de desenvolvimento corrigiu um bug em uma aplicação ou introduziu uma nova funcionalidade. Enquanto o novo código pode funcionar bem isoladamente, ele pode, inadvertidamente, afetar outras partes do sistema. Aqui é onde os testes de regressão entram em jogo. Eles asseguram que o "antigo" código continua a funcionar como esperado após a introdução do "novo" código.

### Por que são importantes?

1. **Garantia de qualidade:** Asseguram que as mudanças não introduzam novos defeitos em áreas não relacionadas.
2. **Reduzem riscos:** Minimizam o risco de problemas no software após cada atualização.
3. **Fornecem feedback rápido:** Permitem aos desenvolvedores detectar e corrigir problemas imediatamente.

### Como são conduzidos?

1. **Automatização:** Dado que os testes de regressão precisam ser executados frequentemente, eles são candidatos ideais para automação. Com ferramentas de teste automatizadas, é possível executar uma suíte de testes de regressão sempre que o código é alterado, garantindo que as mudanças não quebrem funcionalidades existentes.
2. **Suítes de teste:** Nem todos os testes são executados todas as vezes. As suítes de testes podem ser adaptadas para focar nas áreas do software que foram modificadas, bem como áreas que têm maior risco de serem afetadas pelas mudanças.
3. **Integração com CI/CD:** Muitas equipes integram testes de regressão em seus pipelines de Integração Contínua e Entrega Contínua (CI/CD) para garantir que os testes sejam executados automaticamente a cada merge ou deploy.

### **Desafios:**

1. **Manutenção:** Testes de regressão automatizados precisam ser atualizados regularmente para refletir as mudanças no software.
2. **Volume:** Em sistemas grandes, a suíte de testes de regressão pode se tornar enorme, levando muito tempo para ser executada.
3. **Falsos positivos:** Pode haver situações em que os testes falhem, não devido a um bug real, mas devido a alterações no ambiente ou configuração.

## Scripts de Teste

Um script de teste é um conjunto de instruções (escritas em uma linguagem de script de teste ou em uma linguagem de programação) que são executadas em um software para validar que ele se comporta conforme as especificações e requerimentos. O script pode interagir diretamente com a interface do software (como em testes de UI) ou com partes do código (como em testes unitários).

### Tipos de scripts de teste

1. **Manuais:** São passos detalhados que um testador deve seguir manualmente. Eles são frequentemente usados ​​em situações onde a automação pode ser muito cara ou desnecessária.
2. **Automatizados:** São escritos para serem executados por ferramentas de teste automatizadas, como Jest, JUnit, Selenium, entre outras. Estes scripts podem ser executados rapidamente e repetidamente, proporcionando feedback rápido para a equipe de desenvolvimento.

### Componentes típicos de um script de teste

1. **Precondições:** Descrevem os estados iniciais ou configurações que o sistema ou módulo deve ter antes de iniciar o teste.
2. **Passos de teste:** Sequência de ações a serem realizadas durante o teste.
3. **Dados de teste:** Inputs necessários para a execução do teste.
4. **Resultados esperados:** O resultado que você espera após a execução do teste. Isso é usado para determinar se o teste foi bem-sucedido ou não.
5. **Resultados atuais e Observações:** Os resultados obtidos após a execução do teste e qualquer observação relevante.
6. **Pós-condições:** Descreve o estado final esperado do sistema após a execução do teste.

### Benefícios dos scripts de teste

1. **Repetibilidade:** Permitem que os testes sejam repetidos de maneira consistente em diferentes ciclos e ambientes.
2. **Documentação:** Fornecem uma descrição clara e concisa do cenário de teste.
3. **Automatização:** Scripts automatizados podem ser executados frequentemente e rapidamente, ajudando a identificar regressões.
4. **Cobertura:** Permitem que as equipes rastreiem a cobertura dos testes, garantindo que todas as funcionalidades do software sejam testadas.

### Desafios

1. **Manutenção:** À medida que o software evolui, os scripts de teste também precisam ser atualizados.
2. **Complexidade:** Criar scripts para cenários complexos pode ser desafiador e demorado.
3. **Ambiente:** Diferentes ambientes podem resultar em diferentes comportamentos, exigindo ajustes nos scripts.

## Cobertura de Testes

A cobertura de testes refere-se à medida de quão bem o código de um software é executado por um conjunto de testes. Ter uma alta cobertura de testes é essencial para a sustentabilidade do software a longo prazo, e aqui está o porquê:

### 1. Detecção precoce de erros
Quanto maior a cobertura de testes, maior a probabilidade de você identificar bugs e problemas no código antes que ele chegue ao ambiente de produção. Isso pode economizar tempo e recursos, evitando problemas caros ou danos à reputação que podem ocorrer quando os problemas são descobertos por usuários finais.

### 2. Facilita refatorações e extensões
Com uma boa cobertura de testes, os desenvolvedores podem fazer refatorações ou adicionar novas funcionalidades ao código com confiança, sabendo que qualquer regressão ou erro será rapidamente identificado pelos testes.

### 3. Documentação do comportamento do software
Testes bem escritos atuam como documentação, mostrando como o sistema deve se comportar sob diferentes condições. Novos desenvolvedores ou membros da equipe podem usar esses testes como um guia para entender a lógica e as funcionalidades do software.

### 4. Melhora a colaboração
Em equipes grandes ou distribuídas, a alta cobertura de testes garante que o código que está sendo integrado no repositório principal seja de alta qualidade e livre de bugs conhecidos, facilitando a colaboração e reduzindo conflitos de integração.

### 5. Confiança na entrega contínua
Para equipes que praticam integração contínua (CI) e entrega contínua (CD), uma alta cobertura de testes é fundamental. Ela garante que as mudanças enviadas automaticamente para produção (ou para estágios avançados do pipeline) sejam sólidas e confiáveis.

### 6. Reduz o custo total de propriedade
Embora escrever e manter testes exija um investimento inicial, a longo prazo, essa prática pode reduzir os custos associados à manutenção, suporte e correção de bugs no software.

### 7. Melhora a moral da equipe
Quando os problemas são identificados e corrigidos precocemente, a equipe não precisa passar por ciclos estressantes de correções de última hora. Isso pode melhorar a satisfação e a produtividade da equipe.

## Qualidade dos Testes

Quando falamos em [testes de software](https://en.wikipedia.org/wiki/Software_testing), simplesmente alcançar uma alta percentagem de cobertura de testes não é suficiente. Os testes em si devem ser de alta qualidade e relevantes para as funcionalidades e cenários que estão testando.

### Qualidade dos Testes

1. **Precisão**: Os testes devem corretamente refletir o comportamento desejado do sistema. Um teste impreciso pode passar mesmo quando há um bug ou falhar mesmo quando o código está correto.
2. **Completo**: Os testes devem cobrir todos os possíveis cenários de uso, incluindo casos de borda e situações adversas que possam surgir.
3. **Mantível**: O código de teste deve ser bem organizado, bem documentado e fácil de entender. Testes que são difíceis de entender ou modificar se tornam um fardo e perdem sua utilidade ao longo do tempo.
4. **Isolado**: Um bom teste não deve depender de outros testes. Cada teste deve ser capaz de ser executado de forma independente e em qualquer ordem.
5. **Determinístico**: Cada vez que você executa um teste, o resultado deve ser o mesmo, seja ele passar ou falhar. Testes que produzem resultados inconsistentes (às vezes passam, às vezes falham) podem ser difíceis de diagnosticar e corrigir.

### Relevância dos Testes

1. **Cenários de uso real**: Priorize a escrita de testes que reflitam como os usuários realmente interagem com o sistema. Isso garante que as funcionalidades mais críticas e frequentemente usadas sejam robustas e livres de erros.
2. **Evite testes redundantes**: Testes que cobrem o mesmo cenário repetidas vezes, sem qualquer variação significativa, não acrescentam valor e desperdiçam recursos. Identifique e elimine redundâncias.
3. **Teste códigos de caminho crítico**: Priorize a testagem de funcionalidades e códigos que são críticos para o negócio ou que têm um alto risco associado em caso de falha.
4. **Evite testes de baixo valor**: Nem todo código precisa do mesmo nível de rigor em testes. Alguns códigos, como simples getters/setters ou CRUDs simples, podem não necessitar de testes extensivos.
5. **Considere o ambiente de produção**: Os testes devem ser relevantes para o ambiente em que o software será executado. Isso pode incluir a testagem em diferentes dispositivos, sistemas operacionais ou condições de rede.

## Mutation Testing

A cobertura de testes diz apenas que uma linha foi **executada**, não que seu comportamento foi **verificado**. É possível ter 100% de cobertura e uma suíte que não detecta regressões reais — por exemplo, testes com assertions fracas como `expect(result).toBeDefined()`. O *mutation testing* (teste de mutação) ataca exatamente esse falso conforto: mede quão bons os testes são em detectar mudanças no código.

### Como funciona

Uma ferramenta de mutação introduz pequenas alterações automáticas no código-fonte. Cada versão alterada é chamada de **mutante**. Em seguida, a suíte de testes é executada contra cada mutante:

1. Se algum teste **falha** → o mutante foi *morto* (killed). Bom sinal: a suíte detectou a mudança de comportamento.
2. Se todos os testes **passam** → o mutante *sobrevive*. Indica uma lacuna na suíte: o código mudou e ninguém percebeu.

A métrica resultante é o **mutation score** = `mutantes mortos / mutantes totais`. É muito mais rigorosa que cobertura de linha — não é incomum ver projetos com 90% de cobertura e mutation score abaixo de 50%.

### Exemplos de mutadores comuns

| Original | Mutante |
|---|---|
| `a > b` | `a >= b`, `a < b` |
| `a + b` | `a - b` |
| `return x` | `return null` / `return 0` |
| `if (cond)` | `if (true)` / `if (false)` |
| `x && y` | `x \|\| y` |
| remoção de chamada `void` | *(nada)* |

Se o teste continua passando quando `a > b` vira `a >= b`, provavelmente o caso de borda `a == b` nunca foi testado.

### Desafios práticos

1. **Custo computacional:** para N mutantes e M testes, a suíte pode ser executada milhares de vezes. Ferramentas modernas mitigam isso com análise incremental, *mutant schemata* (um único binário com switches) e filtragem por cobertura (só roda os testes que tocam a linha mutada).
2. **Mutantes equivalentes:** mutações que não alteram o comportamento observável (ex.: `i < 10` vs `i <= 9` em um loop). Nunca morrem, mas não são falha do teste. A detecção automática é indecidível no caso geral.
3. **Ruído:** mutações em logs, métricas ou código defensivo geram falsos positivos. Ferramentas sérias permitem excluir arquivos ou operadores específicos.

### Ferramentas por ecossistema

- **Java:** [PIT (pitest)](https://pitest.org) — padrão de facto, integra Maven/Gradle.
- **JS/TS:** [StrykerJS](https://stryker-mutator.io) — também suporta .NET e Scala.
- **Python:** `mutmut`, `cosmic-ray`.
- **Go:** `go-mutesting`, `gremlins`.
- **Ruby:** `mutant`.
- **C/C++:** `mull`, `Dextool Mutate`.

### Quando vale a pena

1. **Código crítico:** parsers, lógica financeira, bibliotecas reutilizáveis — onde regressões silenciosas são caras.
2. **Auditoria de suítes legadas:** antes de uma refatoração grande, para saber em quais áreas a suíte realmente protege.
3. **Modo incremental em PRs:** rodar apenas sobre os arquivos modificados (`--since`) costuma ser viável em CI; rodar na suíte inteira, raramente.

Não substitui outros tipos de teste — é uma camada sobre a suíte existente, focada em medir (e melhorar) a qualidade das assertions.

## Property-Based Testing

Em testes tradicionais (*example-based*), você escreve casos específicos: `add(2, 3) == 5`. Em **property-based testing**, você descreve **propriedades** que devem valer para **qualquer entrada**, e a ferramenta gera centenas de entradas aleatórias para tentar refutar.

### Exemplos de propriedades

1. **Invertibilidade**: `decode(encode(x)) == x` para qualquer `x`.
2. **Idempotência**: `f(f(x)) == f(x)`.
3. **Comutatividade**: `a + b == b + a`.
4. **Invariantes estruturais**: após ordenar, o tamanho não muda; todo elemento de A ainda está em B; B é monotônica.
5. **Oráculo alternativo**: `nossaImplementação(x) == implementaçãoDeReferência(x)`.

### Shrinking

Quando uma propriedade falha, a ferramenta tenta **encolher** a entrada até o menor contraexemplo. Em vez de "falhou em `[42, -17, 999, ...]`", você recebe "falhou em `[0, 0]`". Shrinking é o que torna PBT prático — sem ele, contraexemplos são ilegíveis.

### Ferramentas

- **Haskell**: QuickCheck (o original, 1999).
- **JS/TS**: fast-check.
- **Python**: Hypothesis.
- **Rust**: proptest, quickcheck.
- **Java**: jqwik.
- **Scala**: ScalaCheck.

### Quando vale

- Lógica com muitos casos de borda (parsers, serializadores, algoritmos numéricos).
- Código com **invariantes claras** (estruturas de dados, protocolos).
- Reescrita / refatoração onde existe implementação de referência.

Não substitui testes example-based — complementa. Um bug descoberto pelo PBT deve virar teste example-based para garantir não regressão.

## Snapshot Testing

Captura a saída atual de uma função/componente e compara com um "snapshot" salvo. Se diferir, o teste falha — exigindo atualização explícita do snapshot.

**Popular em**: testes de UI (React via Jest/Vitest), CLI output, schemas gerados, HTML renderizado.

**Prós**:
- Rápido de escrever — não precisa descrever o output, só gravá-lo.
- Pega mudanças não-intencionais em output estruturado.

**Contras**:
- Snapshots são **opacos**: reviewer aprova mudança sem entender o que mudou.
- Fáceis de "atualizar cegamente" (`--updateSnapshot`), neutralizando o teste.
- Frágeis: pequena mudança cosmética causa diff enorme.
- Não substitui assertions explícitas sobre comportamento.

**Uso responsável**: reserve para output que é *naturalmente* um snapshot (screenshot de UI, config gerada). Para lógica, prefira assertions.

## Contract Testing

Em arquiteturas distribuídas, dois serviços têm um contrato: "quando eu chamo seu `/users/:id`, você responde com `{id, name, email}`". Testes de integração verificam isso ponta-a-ponta, mas são lentos, frágeis e exigem ambientes.

**Contract testing** divide o problema:

1. O **consumer** define o que espera do provider (em teste).
2. Isso gera um **contrato** (arquivo).
3. O **provider** valida que sua implementação satisfaz o contrato (em teste).

Nenhuma das execuções precisa que ambos estejam vivos ao mesmo tempo. Rápido, estável, detecta breaking changes cedo.

**Ferramenta dominante**: [Pact](https://pact.io). Variantes: Spring Cloud Contract.

**Quando vale**: APIs internas entre equipes diferentes, microserviços, evolução independente de serviços.

## Fuzz Testing

Gera entradas **aleatórias ou malformadas** (intencionalmente inválidas) e procura crashes, hangs, leaks, corrupções de memória. Diferente de PBT em foco: PBT procura violação de propriedade; fuzzing procura **falha catastrófica** (crash, UB, exploração de segurança).

### Variantes

- **Coverage-guided fuzzing**: a ferramenta observa quais caminhos de código cada entrada ativa e evolui entradas para explorar mais caminhos. AFL (American Fuzzy Lop), libFuzzer, honggfuzz.
- **Mutational**: parte de um corpus de entradas válidas e as muta.
- **Generational**: gera entradas a partir de gramática/schema (para protocolos estruturados).

### Integrações nativas

- `go test -fuzz` (Go 1.18+).
- `cargo fuzz` (Rust + libFuzzer).
- Java: Jazzer.
- Python: Atheris.

### Quando vale

Parsers, decodificadores, crypto, código que processa entrada não confiável (upload de arquivo, deserialização, network). Fuzzing descobriu milhares de CVEs em projetos como libpng, OpenSSL, ffmpeg. Para lógica de negócio comum, o ROI é menor.

## Taxonomia de Test Doubles

Termo geral "mock" é frequentemente usado para tudo que substitui uma dependência em teste — mas há categorias distintas (Gerard Meszaros, *xUnit Test Patterns*):

1. **Dummy**: passado mas nunca usado. Só para preencher parâmetro obrigatório.
2. **Stub**: retorna valores fixos predefinidos. "Quando chamarem `getUser(42)`, retorne este objeto." Não verifica nada.
3. **Fake**: implementação real mas simplificada. Ex.: repositório em memória no lugar do banco. Funciona de verdade, só não é production-grade.
4. **Spy**: como um stub, mas também **registra** como foi chamado (quantas vezes, com que argumentos). Permite asserções *depois* do teste.
5. **Mock**: spy que, além de registrar, tem **expectativas pré-declaradas**. Se a chamada esperada não acontece, o próprio mock falha o teste. "Eu *espero* que `save` seja chamado uma vez com X."

A confusão é comum porque frameworks como Mockito chamam tudo de "mock" independente do papel. A distinção importa:

- **Stub/fake** testam *saída* (state-based testing).
- **Mock/spy** testam *interações* (behavior-based testing).

Mocks em excesso acoplam o teste à implementação — refatoração que preserva comportamento mas muda sequência de chamadas quebra testes. Regra de ouro: **prefira fakes e stubs**; use mocks apenas quando a interação *é* o comportamento a verificar (ex.: garantir que um e-mail foi enviado).

## Conclusão

Escrever testes de alta qualidade e relevância requer um profundo entendimento do software, do domínio do negócio e dos usuários. Embora possa ser tentador focar apenas na cobertura de testes como métrica, a verdadeira eficácia dos testes é determinada por sua capacidade de identificar e prevenir problemas reais que possam afetar os usuários e o negócio.
