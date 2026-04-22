# Commits

Fazer bons commits é uma prática fundamental quando se trabalha com sistemas de controle de versão, como o Git. Aqui estão algumas razões pelas quais é importante fazer bons commits e algumas características de um "bom" commit:

### 1. Histórico claro e intuitivo
Bons commits fornecem um histórico claro e intuitivo do projeto. Eles permitem que desenvolvedores, novos e experientes no projeto, entendam a evolução do código, as razões por trás das mudanças e os problemas que foram resolvidos.

### 2. Facilita a revisão do código
Commits claros e bem estruturados tornam o processo de revisão do código mais eficiente. Revisores podem entender rapidamente o propósito de uma mudança, o que facilita a identificação de problemas ou melhorias.

### 3. Rastreabilidade
Em caso de problemas, é mais fácil rastrear quando e por que uma mudança específica foi introduzida. Isso é especialmente útil quando se está tentando identificar a causa de um bug ou quando se está avaliando o impacto de uma mudança específica.

### 4. Simplifica a integração e resolução de conflitos
Commits menores e focados são mais fáceis de integrar em outras branches e tendem a causar menos conflitos. Quando surgem conflitos, eles são geralmente mais simples de resolver se o commit estiver bem definido.

### 5. Facilidade de rollback
Se descobrir que uma determinada mudança introduziu um problema, bons commits permitem que você reverta facilmente essa mudança específica sem afetar as demais.

### 6. Documentação e comunicação
Os commits atuam como documentação do código, ajudando a comunicar as intenções do desenvolvedor para a equipe e para si mesmo no futuro. Mensagens de commit claras podem reduzir a necessidade de documentação adicional ou esclarecimentos.

### 7. Responsabilidade
Bons commits também estabelecem responsabilidade. Eles indicam claramente quem fez uma mudança específica, o que pode ser útil em equipes grandes ou distribuídas.

## Características de Bons Commits

1. **Focado**: Cada commit deve representar uma mudança lógica única. Evite misturar várias mudanças em um único commit.
2. **Mensagem clara**: A primeira linha da mensagem do commit deve ser um resumo conciso da mudança, seguido por uma descrição mais detalhada, se necessário.
3. **Consistente**: Mantenha um estilo consistente para suas mensagens de commit. Isso ajuda a manter o histórico do projeto limpo e legível.
4. **Referência a issues ou tickets**: Se estiver usando um sistema de rastreamento de issues ou tarefas, referencie-os no commit.
5. **Pequeno**: Commits menores são mais fáceis de entender e revisar.

## Mensagens de Commit

Mensagens de commit são comentários curtos que acompanham cada commit (ou conjunto de alterações) em um repositório. Essas mensagens desempenham um papel fundamental na documentação da história do projeto e ajudam outros membros da equipe a entender o propósito e o contexto das mudanças feitas.

### Importância de boas mensagens de commit

- **Clareza**: Uma mensagem clara ajuda a equipe a entender rapidamente o que foi feito e por quê.
- **Rastreabilidade**: Em projetos grandes ou com muitos colaboradores, mensagens bem estruturadas auxiliam na identificação de quando e por que determinadas alterações foram feitas.
- **Revisão de código**: Torna mais fácil para os revisores entenderem o contexto das alterações.
- **Documentação**: Em muitos casos, a história do commit serve como uma forma de documentação para o código.

### Características de uma boa mensagem de commit

- **Sumário conciso**: A primeira linha da mensagem deve ser um resumo conciso (geralmente com menos de 50 caracteres) da mudança.
- **Descrição detalhada (opcional)**: Após o sumário, pode-se incluir uma descrição mais detalhada, explicando o que foi feito e por que foi feito.
- **Uso de linguagem imperativa**: Frequentemente, é uma convenção escrever mensagens de commit no modo imperativo (por exemplo, "Adicione função de login" ou "Corrija bug no carregamento de imagens").
- **Referências a issues ou tickets**: Se as mudanças estiverem relacionadas a uma issue ou ticket em um sistema de rastreamento, referencie-os na mensagem.

### Dicas para boas mensagens de commit

- **Evite mensagens vagas**: "Correções" ou "Atualizações" não são descritivos o suficiente.
- **Evite mencionar arquivos específicos**: A menos que seja extremamente relevante para o contexto.
- **Pense em "Por que"**: Em vez de apenas descrever o que você fez, pense em por que fez aquela mudança. Isso é particularmente útil para outros colaboradores que estão tentando entender a motivação por trás do commit.

## Revisão de Commits

Revisar commits é uma prática essencial em muitas equipes de desenvolvimento de software. Envolve a análise das mudanças feitas no código antes que essas alterações sejam integradas à base de código principal (ou à "branch" principal). A revisão de commits, muitas vezes chamada de revisão de código, tem múltiplos benefícios, incluindo a garantia de qualidade, a promoção da colaboração e o compartilhamento de conhecimento.

### Benefícios da revisão de commits

- **Prevenção de bugs**: Antes do código ser integrado, ele é verificado por outras pessoas, reduzindo a chance de que falhas ou erros não percebidos pelo autor original sejam introduzidos.
- **Melhoria da qualidade do código**: Revisores podem sugerir melhorias ou otimizações.
- **Compartilhamento de conhecimento**: A revisão permite que a equipe esteja ciente das alterações e compreenda diferentes partes do projeto.
- **Padronização**: Garante que o código siga as convenções e padrões estabelecidos pela equipe.

### Como realizar a revisão de commits

- **Ferramentas de revisão de código**: Plataformas como GitHub, GitLab e Bitbucket têm ferramentas incorporadas para revisão de código em pull requests ou merge requests.
- **Pequenos commits**: Commits menores e focados são mais fáceis de revisar do que grandes conjuntos de alterações.
- **Verificar conformidade**: Certifique-se de que o código segue as diretrizes e padrões estabelecidos pela equipe.
- **Executar o código**: Além de revisar o código visualmente, pode ser útil verificar o código em execução, especialmente para mudanças mais significativas.
- **Comentários construtivos**: Dê feedback de forma clara e construtiva. A revisão de código deve ser uma experiência de aprendizado, não uma crítica negativa.

### O que Procurar durante a revisão

- **Erros lógicos ou bugs**: Alguma coisa parece estranha ou contraintuitiva? Pode ser um sinal de um problema.
- **Qualidade do código**: O código segue as melhores práticas? Está otimizado?
- **Legibilidade**: O código é fácil de entender? Variáveis e funções têm nomes descritivos?
- **Testes**: Se aplicável, os testes estão incluídos? Eles cobrem os casos essenciais?

## Dicas para Revisores

- **Seja Empático**: Lembre-se de que todos estão tentando fazer o melhor. Seja gentil e construtivo em seus comentários.
- **Pergunte**: Em vez de afirmar que algo está errado, faça perguntas para entender as decisões do autor.
- **Fique atento às suas limitações**: Se não estiver certo sobre algo, admita e talvez peça a opinião de outra pessoa.

### Para os Autores dos Commits

- **Descreva suas mudanças**: Ao criar um pull request ou merge request, inclua uma descrição clara do que foi feito e por quê.
- **Seja receptivo ao feedback**: A revisão de código é uma oportunidade de aprendizado. Aceite feedbacks construtivos e esteja disposto a fazer alterações.

## Conventional Commits

[Conventional Commits](https://www.conventionalcommits.org) é uma convenção leve para padronizar a primeira linha da mensagem, tornando-a **processável por máquina** (para geração de changelog, bump de versão automático, etc).

### Formato

```
<tipo>(<escopo opcional>): <descrição>

<corpo opcional>

<rodapés opcionais>
```

### Tipos comuns

- **feat**: nova funcionalidade.
- **fix**: correção de bug.
- **docs**: mudança só em documentação.
- **style**: formatação, ponto-e-vírgula, espaços — sem mudança de comportamento.
- **refactor**: mudança no código que não adiciona feature nem corrige bug.
- **perf**: otimização de performance.
- **test**: adição ou correção de testes.
- **chore**: tarefas de manutenção (deps, config, tooling).
- **build**, **ci**, **revert**: menos comuns mas úteis.

### Exemplos

```
feat(auth): adiciona suporte a login via SSO
fix(checkout): corrige recálculo de frete em CEPs do AM
refactor(orders): extrai PricingService de OrderService
```

### Breaking changes

Indicar com `!` após o tipo ou com um rodapé `BREAKING CHANGE:`:

```
feat(api)!: remove endpoint /v1/users

BREAKING CHANGE: clientes devem migrar para /v2/users.
```

### Benefícios

- **Automação**: ferramentas como `standard-version`, `semantic-release`, `git-cliff` geram changelog e bump de versão sem intervenção.
- **Scan visual**: consegue-se ler `git log --oneline` e entender natureza das mudanças rapidamente.
- **Filtragem**: `git log --grep="^fix"` lista só correções.

### Quando não vale

Projeto pequeno, solo, sem pipeline de release automatizado. Convenção sem uso é só burocracia.

## Versionamento Semântico (SemVer)

Commits informam versões. [SemVer 2.0](https://semver.org) é o padrão dominante:

- **MAJOR.MINOR.PATCH** (ex.: `2.4.7`).
- **MAJOR**: mudança incompatível na API.
- **MINOR**: feature nova, compatível com versões anteriores.
- **PATCH**: correção de bug, compatível.

Conventional Commits mapeiam naturalmente: `fix` → patch, `feat` → minor, `BREAKING CHANGE` → major. Por isso a combinação é popular.

**Pré-release**: sufixos como `-alpha.1`, `-rc.3`, `-beta`. **Build metadata**: `+build.123` — ignorado em ordenação.

**Armadilha comum**: versões `0.x.y` são, por convenção, consideradas "instáveis" — mudanças quebrando podem acontecer em `MINOR`. Muitas libs ficam em `0.x` por anos para ter essa liberdade.

## Trailers

A última seção da mensagem pode conter **trailers** — pares chave-valor estruturados, ao estilo de headers de e-mail. `git interpret-trailers` lida com eles. Convenções populares:

- `Co-authored-by: Nome <email>` — GitHub reconhece e credita pair programming.
- `Signed-off-by: Nome <email>` — Developer Certificate of Origin (DCO). Usado no kernel Linux e outros projetos que rejeitam CLA tradicional. `git commit -s` adiciona automaticamente.
- `Reviewed-by:`, `Acked-by:`, `Tested-by:` — kernel Linux.
- `Refs: #123`, `Closes: #456`, `Fixes: #789` — linkagem a issues; `Closes` fecha issue ao merge no GitHub.

Trailers são parseáveis. Evite misturar com prosa no corpo.

## Commits Bisect-Friendly

`git bisect` faz busca binária no histórico para encontrar o commit que introduziu um bug. Funciona bem quando cada commit no histórico:

1. **Compila**: se um commit intermediário não compila, `bisect` quebra.
2. **Passa nos testes**: ou pelo menos roda o que precisa para o bisect.
3. **É autocontido**: uma mudança lógica completa, não metade de uma refatoração.

Isso implica alguns hábitos:

- **Não commitar estados intermediários quebrados**. Se o trabalho for naturalmente incremental, use `git commit --amend` ou `git rebase -i` antes do push para consolidar.
- **Separar refatoração de mudança comportamental**. Commit 1: refatora sem mudar comportamento. Commit 2: muda comportamento com a estrutura nova. Bisect aponta para o culpado certo.
- **Evitar "WIP" no histórico principal**. WIP vive em branches, não em `main`.

Quando um PR é grande, um rebase interativo antes do merge (ou squash com mensagem cuidadosa) produz um histórico linear de mudanças atômicas.

## Conslusão

Fazer bons commits não é apenas uma boa prática de codificação, mas também uma parte essencial da colaboração eficaz em equipe e da gestão de projetos de software.
