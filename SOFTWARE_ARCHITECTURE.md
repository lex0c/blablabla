# A Arquitetura de Software vai além de Diretórios e Arquivos

É comum para muitos no campo do desenvolvimento de software visualizar inicialmente a arquitetura de software como uma estrutura complexa de pastas e arquivos. Embora isso possa ser parte da história, a realidade é muito mais complexa e conceitual. Na verdade, a arquitetura de software é um conceito multifacetado que engloba a organização de sistemas, o relacionamento entre seus componentes, suas propriedades e os princípios que guiam seu design e evolução.

## O que é Arquitetura de Software?

Arquitetura de software é a organização fundamental dos sistemas. Ela abrange os componentes de um sistema, o relacionamento entre esses componentes, as propriedades de ambos (componentes e relacionamentos) e os princípios que orientam seu design e evolução. Em outras palavras, a arquitetura de software define a espinha dorsal do sistema e estabelece regras e diretrizes para a construção e manutenção desse sistema.

## Componentes, Relacionamentos e Propriedades

Em uma arquitetura de software, os conceitos de componentes, relacionamentos e propriedades são fundamentais. Juntos, eles descrevem as partes constituintes do sistema, como essas partes interagem e as características dessas partes e interações.

### Componentes

Componentes são as unidades básicas da arquitetura de software. Eles representam partes distintas do sistema que realizam funções específicas. Os componentes podem ser variados em tamanho e escopo, desde módulos de código individuais até serviços completos dentro de uma arquitetura orientada a serviços.

Cada componente é geralmente autocontido e possui uma interface bem definida que outros componentes usam para interagir com ele. Os componentes são projetados para serem altamente coesos (realizam uma função bem definida) e fracamente acoplados (dependem minimamente de outros componentes), facilitando a manutenção e a evolução do software.

### Relacionamentos

Relacionamentos em uma arquitetura de software definem como os componentes interagem entre si. Os relacionamentos podem ser na forma de comunicação direta (como uma chamada de método), comunicação indireta (como a publicação/subscrição em um tópico de mensagens), compartilhamento de dados, etc.

Os relacionamentos determinam o fluxo de informação e controle entre os componentes. Eles são fundamentais para a funcionalidade do sistema como um todo. Os relacionamentos também devem ser cuidadosamente gerenciados para evitar a dependência excessiva entre componentes (alto acoplamento), o que pode tornar o sistema difícil de manter e evoluir.

### Propriedades

Propriedades em uma arquitetura de software descrevem as características dos componentes e seus relacionamentos. Elas podem incluir funcionalidades (o que um componente faz), desempenho (quão rápido ou eficiente é um componente), confiabilidade (quão confiável é um componente), segurança (como um componente lida com questões de segurança) e muitos outros.

As propriedades são geralmente a principal consideração quando se escolhe entre diferentes opções de arquitetura de software. Uma arquitetura que oferece as propriedades corretas para um sistema específico (por exemplo, alta performance para um sistema de processamento de transações, alta confiabilidade para um sistema de controle de voo) será mais eficaz do que uma que não o faz.

Compreender componentes, relacionamentos e propriedades é fundamental para a criação de uma arquitetura de software eficaz. Juntos, eles formam a base sobre a qual os sistemas de software são construídos.

## Princípios de Design

Os princípios de design de software são diretrizes estabelecidas para ajudar os desenvolvedores a criar sistemas de software de alta qualidade. Esses princípios ajudam a garantir que o software seja resiliente, robusto e fácil de manter e alterar. Embora existam muitos princípios de design, aqui estão alguns dos mais fundamentais:

- **Modularidade**: A ideia aqui é dividir um sistema em módulos menores ou componentes que possam ser desenvolvidos, testados, alterados e atualizados de forma independente. Isso simplifica o desenvolvimento e a manutenção, melhorando a compreensibilidade do sistema.

- **Encapsulamento**: Este princípio envolve esconder os detalhes internos de uma classe ou módulo e expor apenas o que é necessário. Isso fornece uma maneira de proteger os dados de alterações acidentais e cria uma 'interface' clara para outros módulos.

- **Abstração**: Este princípio é sobre esconder a complexidade fornecendo um nível de 'abstração'. Os usuários ou desenvolvedores podem usar a funcionalidade sem precisar entender todos os detalhes internos.

- **Separation of Concerns (SoC)**: Este princípio sugere que um software deve ser dividido em partes com base na funcionalidade, com cada parte lidando com uma preocupação ou funcionalidade específica. Isso pode levar a um design mais limpo, com menos acoplamento e mais coesão.

- **Reusabilidade**: Em vez de reinventar a roda cada vez, este princípio sugere que os módulos ou funções devem ser projetados de tal forma que possam ser reutilizados em diferentes partes do sistema ou mesmo em diferentes sistemas.

- **Princípio de substituição de Liskov (LSP)**: Este princípio declara que os objetos de uma superclasse devem ser capazes de ser substituídos por objetos de uma subclasse sem afetar a correção do programa.

- **Princípio aberto/fechado**: Este princípio afirma que "as entidades de software (classes, módulos, funções, etc.) devem estar abertas para extensão, mas fechadas para modificação". Em outras palavras, você deve ser capaz de adicionar novos comportamentos sem alterar o código existente.

- **Princípio da responsabilidade única**: Este princípio sugere que uma classe deve ter apenas uma única responsabilidade ou razão para mudar. Isso geralmente leva a um design mais robusto e menos propenso a erros.

Estes são apenas alguns dos muitos princípios de design que ajudam os desenvolvedores a criar software de alta qualidade. Eles servem como uma bússola para orientar as decisões de design durante o processo de desenvolvimento.

## Arquitetura de Software vs. Estrutura de Diretórios e Arquivos

A arquitetura de software é muito mais do que a organização de arquivos e pastas. Enquanto a estrutura do sistema no nível do sistema de arquivos é importante, ela não captura muitos dos aspectos críticos da arquitetura de software.

A estrutura de arquivos e diretórios é apenas uma representação física do código do sistema. Embora possa refletir aspectos da arquitetura, como a organização de código em módulos ou pacotes, ela não fornece informações sobre as interações entre os componentes, as propriedades do sistema ou os princípios de design subjacentes à organização do sistema.

## Arquitetura Evolutiva (Evolutionary Architecture)

Sistemas duram anos; requisitos mudam em meses. A visão tradicional de "arquitetar no início e executar a seguir" falha porque supõe que entendemos tudo antecipadamente — raramente é o caso. **Arquitetura evolutiva**, formalizada por Neal Ford, Rebecca Parsons e Patrick Kua (livro homônimo, 2017), trata a arquitetura como algo que **responde a mudança** de forma controlada, não como um monólito projetado uma vez.

### Princípios

1. **Mudança incremental guiada**: a arquitetura muda em passos pequenos, mensuráveis, reversíveis — não em grandes saltos. Isso casa com entregas contínuas e padrões como *strangler fig* (ver `REFACTORING.md`).
2. **Múltiplas dimensões arquiteturais**: a arquitetura não é só "estrutural". Há *-ilities* que competem entre si: performance, segurança, escalabilidade, operabilidade, evolutibilidade, custo. Mudanças em uma dimensão afetam as outras.
3. **Ciclos de feedback curtos**: quanto menor o ciclo entre "mudança feita" e "efeito observado", mais rápido a arquitetura aprende. Testes, observabilidade, deploy contínuo são as máquinas desse feedback.

### Última decisão responsável

Princípio ágil clássico: adie decisões arquiteturais **até o último momento responsável** — o ponto em que mais tarde custa mais que decidir agora com informação incompleta. Decidir cedo demais congela opcionalidade; decidir tarde demais trava progresso. Engenharia sênior é, em grande parte, intuir esse ponto.

## Fitness Functions

Conceito central da arquitetura evolutiva: uma **fitness function** é uma verificação automatizada que mede quão bem a arquitetura satisfaz uma **característica desejável**. Emprestado de algoritmos evolutivos, mas aplicado à arquitetura como *testes contínuos de propriedades arquiteturais*.

Enquanto testes unitários verificam **comportamento**, fitness functions verificam **propriedades transversais**: "o módulo de domínio não importa nada de infraestrutura?", "o p99 de latência está abaixo de 300ms?", "não há ciclo de dependências?".

### Exemplos

1. **Dependências arquiteturais**: "nenhuma classe em `domain/` depende de `infrastructure/`". Verificável com ArchUnit (Java), arch_unit (Python), dependency-cruiser (JS), go-arch-lint (Go).
2. **Performance**: benchmark automático falha se a latência passar de um limite.
3. **Cobertura por módulo**: módulo crítico exige cobertura ≥ 90%.
4. **Tamanho de classe / função**: limites como guarda contra "deus classes".
5. **Ciclos**: detecção automática de ciclos de importação.
6. **Observabilidade**: todo handler HTTP tem tracing e log estruturado (lint/test).
7. **Segurança**: nenhuma dependência com CVE alto na suite (integrar a CI).
8. **API contract**: schema não sofreu breaking change sem bump de versão.

### Tipos (taxonomia do livro)

- **Atômica vs holística**: atômica mede uma característica isolada; holística mede interação entre várias.
- **Triggered vs contínua**: disparada por evento (PR, deploy) vs avaliada o tempo todo em produção.
- **Estática vs dinâmica**: valor fixo como alvo vs valor derivado do contexto atual.
- **Automatizada vs manual**: ideal é automatizar; alguns aspectos (UX, elegância) precisam de julgamento humano.

### Anti-patterns

1. **Fitness function teatral**: teste que passa sempre, ninguém confere. Cheque que ela já tenha falhado legitimamente pelo menos uma vez.
2. **Muitas fitness functions**: fadiga; perde-se de vista o que importa. Menos, relevantes, mantidas.
3. **Ignorar falhas regulares**: se uma FF falha toda semana e ninguém age, ela deixou de ser contrato; virou ruído.

## Integração com Outras Práticas

- **`ADR.md`**: ADRs documentam *por que* da arquitetura atual; fitness functions **protegem** a arquitetura de desviar sem decisão consciente.
- **`TESTS.md`**: fitness functions são testes — de propriedades arquiteturais, não de comportamento.
- **`OBSERVABILITY.md`**: SLOs são fitness functions sobre performance e disponibilidade em produção.
- **`REFACTORING.md`**: arquitetura evolutiva exige o catálogo de estratégias (strangler, branch by abstraction, parallel change) para executar mudanças sem parar o mundo.

## Conclusão

A arquitetura de software é uma disciplina complexa e abstrata que envolve muito mais do que a organização física do código-fonte. Ela aborda questões fundamentais sobre a estrutura, a organização, a performance, a segurança e a evolução de um sistema de software. Uma compreensão completa da arquitetura de software é fundamental para a criação de sistemas de software robustos, eficientes e manuteníveis. Então, da próxima vez que você pensar em arquitetura de software, lembre-se de que ela vai muito além da estrutura de diretórios e arquivos.
