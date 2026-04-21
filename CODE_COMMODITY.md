# O fim do “dev que escreve código”

> Ninguém está te pagando para lembrar sintaxe. Estão te pagando para resolver problemas.

Escrever código virou uma habilidade de baixo diferencial, não irrelevante. Ainda importa, mas como “digitação especializada”. A comparação útil é com matemática básica: ninguém paga alguém só por saber somar, mas todo mundo precisa saber somar para fazer algo mais valioso.

Se você ainda mede seu valor por:

- quantas linguagens “domina”
- quantos frameworks sabe de cabeça
- quão rápido escreve código sem consultar nada

você está se otimizando para um mundo que já acabou...

## O que mudou

Durante décadas, escrever código tinha custo alto:

- buscar referência era lento
- documentação era inconsistente
- exemplos eram escassos

Então memorizar virou vantagem competitiva.

Agora, com ferramentas como ChatGPT e Claude:

- gerar código é trivial
- testar hipóteses é rápido
- explorar alternativas custa quase zero

Ou seja: o custo de produzir código colapsou.

Quando o custo de produção cai, o valor migra.

## O “novo gargalo” nunca foi novo

Arquitetura, modelagem, concorrência, IO, rede, banco, consistência isso sempre foi o gargalo.

A diferença é que antes:

- poucos conseguiam construir sistemas grandes
- os erros apareciam mais cedo (porque desenvolver era lento)

Agora:

- qualquer um monta algo distribuído em uma tarde
- os erros escalam silenciosamente

O problema é que software não falha por falta de sintaxe correta.

Ele falha por:

- modelagem ruim
- decisões arquiteturais frágeis
- falta de entendimento de domínio
- acoplamento desnecessário
- concorrência mal definida
- ausência de contratos claros

Se antes dava para sobreviver sabendo “como fazer”, agora isso virou detalhe operacional.

O que realmente diferencia é entender como as coisas funcionam por baixo.

Não em nível acadêmico decorativo, mas no nível que permite tomar decisões que não explodem em produção.

## O paradoxo atual

IA facilita construir sistemas.

E ao mesmo tempo, torna mais fácil construir sistemas ruins em escala.

Você consegue:

- montar uma API em minutos
- integrar serviços rapidamente
- gerar código “correto o suficiente”

Mas sem entendimento, você também consegue:

- criar gargalos invisíveis
- introduzir falhas de concorrência
- abrir vulnerabilidades básicas
- gerar sistemas impossíveis de evoluir

## O que “entender como funciona” significa

Não é saber teoria. É saber prever comportamento.

IA reduz o custo de escrever código. Ela não reduz o custo de decidir errado.

E decisão errada quase nunca nasce de um `for` mal escrito. Nasce de ignorância estrutural:

- não entender como o navegador renderiza, reflow, event loop, cache, storage, rede
- não entender HTTP, latência, retries, timeouts, proxies, TLS, CDN
- não entender banco, índices, locks, cardinalidade, planner, concorrência
- não entender arquitetura, fronteiras, acoplamento, observabilidade, falhas
- não entender segurança, auth, sessão, trust boundary, input/output validation
- não entender IA, contexto, limitação, custo, alucinação, avaliação

Quanto mais fácil ficou gerar implementação, mais valioso ficou saber julgar a implementação.

Antes, muito dev sobrevivia com conhecimento local:

“sei Node”, “sei React”, “sei framework X”.

Agora isso é pouco. O problema não é mais sobre “como escrevo isso?”.

O problema é:

- isso deveria existir?
- isso escala?
- isso quebra onde?
- isso é seguro?
- isso é observável?
- isso mantém consistência?
- isso degrada bem?

Quem entende sistemas toma decisões melhores porque enxerga consequências de segunda ordem. Vê o efeito cascata antes da merda chegar em produção. Sabe que abstração ruim cobra juros.

## Conclusão

IA não corrige incompetência. Ela a escala.

Antes, existia um limite físico para o estrago:

um dev ruim produzia pouco código ruim, devagar.
O sistema degradava lentamente. Havia tempo para perceber, discutir, corrigir.

Agora...

Um dev ruim, com IA, consegue:

gerar centenas de linhas por hora
cobrir múltiplos arquivos sem entender nenhum
introduzir abstrações desnecessárias com confiança
criar complexidade acidental em escala.

O risco não é “IA substituir devs”.

O risco é times inteiros produzirem sistemas:

- mais rápidos de construir
- mais difíceis de entender
- mais caros de manter
- mais frágeis em produção

Tudo com sensação de progresso.

Porque código existe.
Mas entendimento não.

E sem entendimento, o colapso é só questão de tempo.
