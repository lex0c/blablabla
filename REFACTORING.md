# Refatoração

A [refatoração](https://en.wikipedia.org/wiki/Code_refactoring) é o processo de melhorar a estrutura interna do software sem modificar o seu comportamento externo. Pode ser uma simples mudança de nome de variável ou uma reformulação completa do design de uma classe. O objetivo é tornar o código mais eficiente, legível e compreensível, não apenas para você, mas para qualquer desenvolvedor que possa trabalhar nele no futuro.

## Por que Refatorar?

### 1. Melhorar a Legibilidade

O código é escrito e mantido por pessoas. Torná-lo legível e compreensível ajuda toda a equipe a entender o que o código faz, o que, por sua vez, reduz o tempo para adicionar novas funcionalidades ou corrigir bugs.

### 2. Reduzir a Complexidade

Complexidade é inimiga do desenvolvedor. Códigos complexos são difíceis de entender e propensos a erros. Refatorar para reduzir a complexidade ajuda a tornar o código mais gerenciável.

### 3. Eliminar Código Duplicado

Código duplicado é basicamente um convite para bugs e problemas de manutenção. Refatorar para eliminar duplicações torna o código mais limpo e fácil de manter.

### 4. Preparação para Extensões

O código sempre muda. Novas funcionalidades são adicionadas, e os bugs são corrigidos. Um código bem refatorado é mais flexível e fácil de estender.

## Práticas Recomendadas

### Faça Pequenas Mudanças

Grandes mudanças são arriscadas e difíceis de gerenciar. É melhor fazer pequenas mudanças incrementais, testando frequentemente para garantir que você não introduziu novos bugs.

### Use Testes

Uma boa suíte de testes é seu melhor amigo quando você está refatorando. Ela serve como uma rede de segurança, permitindo que você faça alterações com confiança.

### Revise o Código

Peça a um colega para revisar as suas mudanças. Outros olhos podem captar erros ou sugerir melhorias que você não percebeu.

## Princípio do Escoteiro

"Deixe sempre o código melhor do que o encontrou" é uma filosofia adotada por muitos desenvolvedores e é comumente referida como o "Princípio do Escoteiro" no desenvolvimento de software. Este princípio se originou do Código dos Escoteiros, que diz: "Deixe o acampamento mais limpo do que você o encontrou". No contexto do desenvolvimento de software, a ideia é simples, mas poderosa: a cada vez que você interage com o código, seja para adicionar uma nova funcionalidade, corrigir um bug ou fazer qualquer outra modificação, você deve se esforçar para melhorar a qualidade do código, mesmo que seja uma pequena mudança.

## Por que adotar esse princípio?

1. **Melhor manutenção:** Um código que é constantemente melhorado se torna mais fácil de ser mantido. Pequenas melhorias regulares evitam que o código se torne uma "massa de espaguete" ininteligível.

2. **Evita a deterioração do código:** Em projetos de software, especialmente aqueles com muitos desenvolvedores, o código pode se deteriorar rapidamente se as más práticas não forem corrigidas. Adotando o princípio do escoteiro, os desenvolvedores combatem ativamente essa deterioração.

3. **Promove boas práticas:** Ao assumir a responsabilidade de melhorar o código continuamente, os desenvolvedores se tornam mais conscientes das melhores práticas de codificação.

4. **Educação contínua:** Ao refatorar e melhorar o código de outra pessoa, um desenvolvedor pode aprender novas técnicas ou entender padrões diferentes. É uma forma de aprendizado contínuo.

5. **Benefício para a equipe:** Deixar o código em um estado melhor beneficia não apenas o desenvolvedor atual, mas toda a equipe. A próxima pessoa que visitar aquele trecho de código encontrará um ambiente mais amigável e compreensível.

## Como adotar o Princípio do Escoteiro?

1. **Refatoração:** Sempre que você estiver trabalhando em um trecho de código e notar algo que pode ser melhorado, mesmo que não esteja diretamente relacionado à sua tarefa atual, tente refatorá-lo.

2. **Melhorar a legibilidade:** Renomeie variáveis, funções ou classes para tornar o propósito delas mais claro.

3. **Documentação:** Adicione comentários onde for necessário, ou melhore os comentários existentes para que eles sejam mais claros.

4. **Remova código morto:** Se você encontrar código que não está sendo usado ou está obsoleto, remova-o.

5. **Automatize testes:** Se uma parte do código não possui testes, considere adicionar alguns para garantir sua funcionalidade.

## Estratégias para Refatorações Grandes

Em sistemas em produção, refatorações pequenas e locais resolvem-se em um PR. Mas há mudanças estruturais — substituir um banco, extrair um serviço, trocar um framework — que não cabem em um commit nem podem bloquear o desenvolvimento por semanas. Três padrões são essenciais nesses casos.

### Strangler Fig

Nome emprestado da figueira estranguladora, planta que cresce ao redor de uma árvore hospedeira e, gradualmente, substitui-a. Martin Fowler popularizou o termo em software.

**Mecanismo**:

1. Coloca-se uma "fachada" (proxy, roteador, gateway) na frente do sistema legado.
2. Novas features são implementadas no sistema novo, atrás da fachada.
3. Gradualmente, funcionalidades existentes são **reimplementadas** no sistema novo; a fachada redireciona chamada a chamada.
4. Quando tudo foi migrado, o legado é desligado — "estrangulado".

**Prós**:
- Sem big bang. Entrega contínua permanece.
- Rollback granular: uma feature migrada pode voltar ao legado.
- Risco distribuído no tempo.

**Contras**:
- Duas implementações convivem por meses/anos. Complexidade operacional.
- A fachada pode virar um "lugar eterno" se ninguém pressionar o desligamento.

### Branch by Abstraction

Introduzir uma **abstração** (interface) entre o cliente e a implementação, e usá-la para trocar a implementação sem interromper o fluxo principal.

**Passos**:

1. **Criar a abstração** — interface que descreve as operações. Implementação inicial é um wrapper da implementação antiga.
2. **Migrar clientes gradualmente** — todos passam a chamar a abstração, não a implementação antiga direto.
3. **Implementar o novo** — em paralelo, a implementação nova satisfaz a mesma interface.
4. **Trocar** — via configuração/flag, o sistema passa a usar a nova.
5. **Remover o antigo** — quando confiança é alta.

Diferença em relação ao strangler fig: strangler vive **fora** do código (redirecionamento entre sistemas); branch by abstraction vive **dentro** do código (abstração interna).

Trabalha excelente com **feature flags** — a troca pode ser gradual, por percentual de tráfego ou por cohort.

### Parallel Change (Expand-Migrate-Contract)

Também chamado **expand and contract**. Para mudanças de contrato (schema de banco, API, assinatura de função) sem downtime.

**Três fases**:

1. **Expand**: adicione o novo **ao lado** do antigo, sem remover. Ambos aceitos/escritos.
2. **Migrate**: atualize consumidores/produtores para usar o novo. Duplique escritas se necessário (escreve no antigo E no novo).
3. **Contract**: quando ninguém usa mais o antigo, remova-o.

**Exemplos concretos**:

- **Renomear coluna**:
  1. Adicionar coluna `email_address`.
  2. Escrever em ambas (`email` e `email_address`). Migrar leitores para `email_address`.
  3. Parar de escrever em `email`. Remover.

- **Mudar formato de evento**:
  1. Produtor emite em ambos os formatos.
  2. Consumidores migram do antigo para o novo.
  3. Produtor para de emitir o antigo.

É o padrão que permite **zero-downtime migrations** — sem isso, qualquer mudança de schema precisa parar o sistema.

## Flags como Infraestrutura de Refatoração

Feature flags (ou toggles) transformam decisões de deploy em decisões de runtime. Em refatorações:

- **Kill switch**: se o caminho novo tem problema, volta para o antigo sem redeploy.
- **Canary**: 1% → 10% → 50% → 100% do tráfego no caminho novo, com monitoramento.
- **Dark launch**: caminho novo executa *em paralelo* com o antigo, resultado comparado mas o antigo é o que responde. Excelente para ganhar confiança sem risco.

Ferramentas: LaunchDarkly, Unleash, Flipt, Split. Ou uma flag simples na config em projetos pequenos.

**Cuidado**: flags são dívida. Toda flag deve ter **data de remoção planejada**. Flags esquecidas viram spaghetti condicional.

## Refatorações Pequenas (Catálogo)

Além das estruturais, há o repertório clássico de Martin Fowler (*Refactoring*, 2018):

- **Extract Function / Inline Function**.
- **Extract Variable / Inline Variable**.
- **Rename** (variável, função, classe, arquivo).
- **Move Function** entre módulos.
- **Replace Conditional with Polymorphism**.
- **Introduce Parameter Object**.
- **Replace Magic Number with Symbolic Constant**.
- **Split Phase** (separar etapas que estavam misturadas).
- **Replace Loop with Pipeline** (map/filter/reduce).
- **Decompose Conditional** (extrair condição complexa em função nomeada).

IDEs modernas automatizam várias dessas — use a refatoração da IDE, não edição manual, para garantir que referências sejam atualizadas corretamente.

## Quando *Não* Refatorar

1. **Código que vai ser removido em breve**: polir algo que vai morrer é desperdício.
2. **Sem testes e sem tempo de adicionar**: refatorar sem rede de segurança é introduzir bugs com ar de virtude.
3. **Próximo de deadline crítico**: risco > benefício.
4. **Sem entender bem o código**: refatoração sem compreensão transforma bugs sutis em bugs óbvios (ou vice-versa).
5. **"Por estética"**: se não há uso concreto do código refatorado no próximo trimestre, o custo excede o valor presente.

## Conclusão

Refatoração não é apenas uma tarefa de limpeza; é um investimento contínuo na saúde do projeto. Mantenha a prática de revisar e refatorar o código regularmente. Lembre-se, um código limpo hoje economizará horas de dor de cabeça no futuro.
