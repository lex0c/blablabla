# Threat Modeling

*Threat modeling* (modelagem de ameaças) é o processo estruturado de identificar **o que pode dar errado** em um sistema — antes dos atacantes o façam. É a atividade de segurança com melhor custo-benefício em engenharia: uma hora na frente de um quadro branco evita semanas de incident response depois.

Quatro perguntas orientam o exercício (Adam Shostack):

1. **O que estamos construindo?**
2. **O que pode dar errado?**
3. **O que vamos fazer sobre isso?**
4. **Fizemos um bom trabalho?**

## Escopo e Pré-requisitos

Antes de listar ameaças, entenda:

- **Ativos**: o que tem valor? Dados (PII, segredos, financeiros), disponibilidade, reputação.
- **Atores**: quem pode atacar? Usuário malicioso autenticado, atacante externo, insider, parceiro comprometido, cadeia de suprimentos (ver `SUPPLY_CHAIN.md`).
- **Trust boundaries**: onde o nível de confiança muda? Internet → DMZ, DMZ → backend, usuário comum → admin, tenant A → tenant B, serviço → banco. Toda boundary é lugar óbvio para checagem.
- **Data flow**: como os dados atravessam o sistema? Requisição → API → fila → worker → banco → dashboard. Cada seta é oportunidade de ameaça.

Ferramenta de base: **Data Flow Diagram (DFD)** — quadrados (processos), cilindros (data stores), setas (fluxos), linhas tracejadas (trust boundaries). Simples e suficiente.

## STRIDE

Framework da Microsoft. Seis categorias, mnemônico prático:

| | Categoria | Propriedade violada | Exemplo |
|---|---|---|---|
| **S** | Spoofing | Autenticidade | Atacante finge ser outro usuário; token roubado. |
| **T** | Tampering | Integridade | Modificação de dado em trânsito ou em repouso. |
| **R** | Repudiation | Não-repúdio | Usuário executa ação e nega depois; ausência de logs. |
| **I** | Information disclosure | Confidencialidade | Vazamento de PII, de stack trace, de segredos. |
| **D** | Denial of service | Disponibilidade | Exaustão de recursos, amplificação, lentidão induzida. |
| **E** | Elevation of privilege | Autorização | Usuário comum vira admin; sandbox escapada. |

Aplicar STRIDE a cada elemento do DFD:

- **Processos**: todas as 6 aplicam.
- **Data stores**: T, I, R, D (spoofing/elevation não fazem sentido em storage passivo).
- **Data flows**: T, I, D (em trânsito).
- **External entities**: S, R.

## Variantes

### LINDDUN (foco em privacidade)

Linkability, Identifiability, Non-repudiation (aqui ruim: não desejamos que ações privadas sejam rastreáveis), Detectability, Disclosure of information, Unawareness, Non-compliance. Útil quando o produto lida com PII e regulações como LGPD/GDPR.

### PASTA (Process for Attack Simulation and Threat Analysis)

Processo de 7 etapas mais pesado; combina threat modeling com risk management. Adequado a organizações maduras com programa formal de segurança.

### Attack Trees (Bruce Schneier)

Modelam objetivos do atacante como raiz e decomposições como ramos. Cada folha recebe custo/probabilidade. Útil para decidir onde gastar defesa: atacar qual caminho é mais barato?

### VAST, OCTAVE, Trike

Alternativas menos comuns; citadas por completude.

## Princípios de Defesa

### Defense in Depth

Não confiar em uma única camada. Um atacante que passa por uma barreira deve encontrar outra. Ex.: autenticação + autorização + validação + criptografia em repouso + monitoração. Cada camada tem falhas; a interseção é pequena.

### Least Privilege

Todo componente (usuário, serviço, processo) deve ter **o mínimo** de permissão necessária. Aplicar em:

- IAM (roles AWS/GCP por serviço, não um role global).
- Banco (usuário de aplicação só `SELECT`/`INSERT` no necessário; jobs de migração em usuário separado).
- Sistema operacional (processos em usuários não-root).
- Rede (segmentação, security groups estritos).

### Fail Secure

Em erro, falhe em estado **fechado**, não aberto. Se o serviço de autorização cai, negue acesso — não libere "por tolerância". Exceção consciente: graceful degradation de features não-críticas.

### Separation of Duties

Nenhuma pessoa/serviço sozinho pode executar uma ação crítica. Ex.: deploy exige 2 aprovadores; operação financeira precisa de maker+checker. Limita dano por insider ou conta comprometida.

### Assume Breach

Projete como se já estivessem dentro. Segmentação interna, detecção, rotação de credenciais, monitoramento de movimento lateral. "Zero trust" é uma aplicação dessa ideia.

## Fluxo Prático

1. **Reúna quem sabe**: engenheiro responsável, segurança, SRE, produto. Whiteboard.
2. **Desenhe o DFD**: componentes, fluxos, data stores, trust boundaries. Não precisa ser bonito.
3. **Aplique STRIDE por elemento**: liste ameaças plausíveis. Não filtre ainda.
4. **Priorize**: cada ameaça tem *probabilidade* e *impacto*. DREAD ou simples alto/médio/baixo funciona.
5. **Decida mitigação**: por ameaça priorizada, uma de quatro respostas:
   - **Mitigar** (controles, redesign).
   - **Transferir** (seguro, terceirizar).
   - **Aceitar** (risco baixo, custo alto).
   - **Eliminar** (remover feature/componente).
6. **Escreva** a conclusão: ameaça, decisão, owner, prazo.
7. **Revisite**: threat model decai. Revisar em cada mudança arquitetural grande e periodicamente (trimestral/semestral).

## Integração com o Ciclo de Desenvolvimento

- **Design phase**: threat model antes de escrever código — o momento mais barato para mudar arquitetura.
- **PR review**: mudanças em boundaries sensíveis (auth, input, crypto) merecem pergunta explícita "quais ameaças isso introduz ou mitiga?".
- **Pentest/red team**: complementa, não substitui. Pentest encontra bugs específicos; threat model encontra classes de problema.
- **Security review** (checklists, OWASP): camada adicional, não a mesma coisa.

## Armadilhas Comuns

1. **Teatro de segurança**: reunião onde se enumera OWASP Top 10 genérico sem aterrar no sistema real. Use o DFD concreto.
2. **Foco só no atacante externo**: insiders e cadeia de suprimentos são ameaças reais. `SUPPLY_CHAIN.md` detalha.
3. **Threat model que vira documento morto**: se não é revisitado em mudanças, perde valor. Versione junto com o código (`docs/threat-model.md`).
4. **Mitigação sem validação**: "adicionamos rate limiting" não prova que funciona. Testar.
5. **Overengineering**: controle de segurança caro para ameaça improvável e baixo impacto. Risco = probabilidade × impacto; custo da mitigação também entra na conta.
6. **Ignorar disponibilidade**: DoS é ameaça de segurança tão real quanto vazamento. Capacidade, rate limiting, WAF, CDN são controles.

## Recursos

- "Threat Modeling: Designing for Security" — Adam Shostack.
- OWASP Threat Modeling Cheat Sheet.
- MITRE ATT&CK para catalogar técnicas reais usadas em ataques.
- Microsoft Threat Modeling Tool (se preferir tooling dedicado).

## Resumo

Threat modeling não produz software seguro; produz **consciência** das ameaças e decisões explícitas sobre elas. Uma decisão ruim documentada é melhor que uma decisão boa acidental — porque a decisão pode ser revisitada; o acidente, não.
