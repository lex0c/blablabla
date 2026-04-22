# Privacy Engineering

**Privacy Engineering** é a disciplina de construir sistemas que **protegem** a privacidade por design, não como afterthought. Enquanto `THREAT_MODELING.md` foca em integridade/disponibilidade/confidencialidade contra atacantes, privacy engineering foca em **uso legítimo**: como limitar o que você coleta, processa e compartilha, mesmo que tenha autorização técnica.

Complementa `COMPLIANCE.md` (o lado regulatório) e `CRYPTO_ADVANCED.md` (primitivas técnicas).

## Por Que Privacy é Engenharia

Privacy **não é só política**. É:

- **Decisão arquitetural**: onde os dados fluem, quem retém.
- **Escolha de primitiva**: hash, pseudonimização, anonimização, DP.
- **Trade-off de utilidade**: mais privacidade frequentemente reduz qualidade analítica.
- **Métrica**: como medir "o quão privado" um sistema é?

Sem engenharia, privacidade vira declaração vazia na política de privacidade.

## Princípios (FTC FIPPs / GDPR / LGPD)

Comuns às grandes frameworks:

1. **Minimização**: colete só o necessário.
2. **Finalidade limitada**: use apenas para o propósito declarado.
3. **Transparência**: diga o que faz com os dados.
4. **Consentimento válido**: informado, específico, revogável, livre.
5. **Precisão**: dados corretos; mecanismos de correção.
6. **Retenção limitada**: apague quando não precisar mais.
7. **Segurança**: proteja (ver todos os outros arquivos de seg).
8. **Accountability**: responsabilidade documentada.
9. **Data subject rights**: acesso, retificação, apagamento, portabilidade, oposição.

## Privacy by Design (Cavoukian)

Sete princípios:

1. **Proativa, não reativa** — preventiva, não remediativa.
2. **Privacy as default** — sem ação do usuário, máxima proteção.
3. **Privacy embedded in design** — não add-on.
4. **Full functionality** — soma positiva, não zero.
5. **End-to-end security** — ciclo de vida inteiro.
6. **Visibility and transparency** — auditável.
7. **Respect for user privacy** — centrado no usuário.

Framework útil para revisar design. Cuidado: pode virar checklist sem profundidade; use como espelho, não bingo.

## Classificação de Dados

Não todos são iguais. Classificação típica:

1. **Públicos**: marketing, institucional.
2. **Internos**: comunicação corporativa.
3. **Confidenciais**: strategic, comercial.
4. **Sensíveis/PII**: nome + identificador.
5. **Altamente sensíveis**: saúde, financeiro, orientação, religião, biometria — **dados sensíveis** na LGPD/GDPR.
6. **Regulados**: PCI (pagamentos), PHI (saúde nos EUA), crédito, minors.

Classificação dirige controles. Ferramentas: **Macie (AWS), DLP API (GCP), Purview (Azure)**, **classificadores** baseados em regex/ML.

## PII, Quasi-Identifier, Anonimização

- **Identificador direto**: CPF, email, telefone.
- **Quasi-identifier**: CEP + idade + sexo. Três quasi combinam para identificar 87% da população americana (Sweeney, 2000). Cuidado.
- **Atributos sensíveis**: diagnóstico, orientação.

### Pseudonimização

Substituir identificador por pseudônimo (hash, UUID) mantendo reversibilidade via tabela de mapeamento.

- Dados pseudonimizados **ainda são pessoais** por definição legal — reversão é possível.
- Protege contra vazamento casual de PII direto em logs/analytics.
- Tabela de mapeamento precisa proteção extra.

### Anonimização

Remover informação até não ser possível identificar *mesmo com dados externos*.

**Desafio**: "dados anonimizados" frequentemente são reidentificáveis:

- **Netflix Prize** (2006): ratings "anônimos" re-identificados com IMDB.
- **AOL search logs** (2006): buscas "anônimas" revelaram identidades por busca conjunta.
- **Strava heatmap** (2018): padrões revelaram bases militares.

Anonimização real é difícil e frequentemente inviável sem perder utilidade.

### k-Anonymity

Cada registro é indistinguível de pelo menos **k-1** outros no conjunto de quasi-identifiers. Proposto por Sweeney.

- Generalização (idade exata → faixa etária).
- Supressão (apagar o outlier).

**Limitações**: **l-diversity** (ataque de homogeneidade — todos os k têm mesmo diagnóstico), **t-closeness** (distribuição). Ainda vulnerável a **background knowledge attack**.

### Differential Privacy (DP)

Garantia matemática: presença ou ausência de um indivíduo no dataset quase não muda output de uma query.

**Definição formal**: um mecanismo M é ε-DP se para datasets D e D' diferindo em 1 registro e qualquer output S: `Pr[M(D) ∈ S] ≤ e^ε × Pr[M(D') ∈ S]`.

- **ε (epsilon)** pequeno = mais privacidade, mais ruído; grande = pouco ruído, pouca privacidade.
- **Laplace mechanism**: adicionar ruído Laplace ao resultado de query.
- **Gaussian mechanism**: usado em ML.
- **Budget**: cada query consome `ε`; composição soma.

Adotado em: **US Census 2020** (controversial; afetou utilidade), **Apple** em telemetria, **Google** em Chrome, **Meta**, **Microsoft**. Biblioteca: **OpenDP, Google DP, IBM Diffprivlib**.

### Federated Learning

Modelo treina em dispositivos/clientes; só **updates agregados** (gradientes) saem. Dados crus ficam local.

- Combina bem com DP (secure aggregation + noise).
- Usado em Google Keyboard (teclado), Apple (sugestões), health.

Não é panaceia — updates ainda vazam informação residual.

### Synthetic Data

Gerar dados artificiais que preservam propriedades estatísticas sem pertencer a ninguém.

- **GANs, VAEs, diffusion** para geração.
- Utilidade para testes, ML em domínios sensíveis (saúde).
- **Cuidado**: memorização — modelos podem reproduzir amostras reais.

### MPC / FHE / ZKP

Ver `CRYPTO_ADVANCED.md`. Computação privada em múltiplas partes sem revelar entradas. Produção: lenta mas viável em casos específicos.

## Privacy Threats — LINDDUN

Threat model específico para privacy (ver `THREAT_MODELING.md` para STRIDE):

- **Linkability**: ações do mesmo usuário linkáveis entre si.
- **Identifiability**: sujeito identificado a partir de dados.
- **Non-repudiation**: usuário não pode negar ação (pode ser ruim em privacidade).
- **Detectability**: sabe-se que o registro existe mesmo sem ler.
- **Disclosure of information**: acesso não autorizado.
- **Unawareness**: usuário não sabe o que acontece.
- **Non-compliance**: violação de política/lei.

LINDDUN GO é versão rápida.

## Consent Management

- **CMP (Consent Management Platform)**: OneTrust, Cookiebot, Didomi.
- **IAB TCF** (Transparency and Consent Framework): padrão publicitário europeu.
- **Design de consent UX**: dark patterns são a norma; design ético exige resistir.
- **Granularidade**: consent por finalidade, não monolítico.
- **Revocation**: tão fácil quanto dar.
- **Registros auditáveis**: quem consentiu o quê, quando.

## Data Subject Rights

Sob LGPD/GDPR:

- **Acesso** ao que você tem sobre ele.
- **Retificação** de dados errados.
- **Apagamento ("right to be forgotten")** com exceções (legal hold, obrigações regulatórias).
- **Portabilidade**: formato estruturado, legível.
- **Oposição** a certos processamentos.
- **Decisões automatizadas**: direito à explicação / intervenção humana.

**Implementação técnica**: endpoints de API por direito; SLAs (30 dias típico); propagação pros processadores/subprocessadores.

## Sistemas que nem deveriam existir

Design ético, não só legal:

- **Não colete** o que não precisa. CPF para newsletter? Não.
- **Não logue** PII. Logs vivem muito; PII em log vaza em próximo incidente.
- **Não retenha** além do necessário. Retenção definida por finalidade, não eterna "caso precise".
- **Não enriqueça** com dados de terceiros sem consent específico.
- **Evite tracking pixel e third-party cookies** onde possível.

## Observabilidade vs Privacidade

- **Métricas agregadas** são menos privadas do que parecem (K-anonymity baixo em dimensões combinadas).
- **Logs estruturados** facilitam filtering de PII. Use `user_id` pseudônimo, não email.
- **Traces distribuídos** (OpenTelemetry) com dados no payload — sanitize.
- **Sampling** reduz exposição.
- **TTL curto** em logs com dados sensíveis.

## Telemetria em Produtos

- **Opt-in** honesto > opt-out enterrado.
- **Crashes**: dados diagnósticos, sem dump de estado com PII.
- **Analytics**: eventos sem PII (`button_clicked`, não `"user@email.com clicked button"`).
- **Cohort analysis** sem individual tracking.

## Implementação Operacional

### DPIA (Data Protection Impact Assessment)

Análise formal quando alto risco:

- Processamento em larga escala.
- Dados sensíveis.
- Vigilância pública.
- Decisões automatizadas que afetam pessoas.

Conteúdo: descrição, necessidade, proporcionalidade, riscos, medidas.

### ROPA (Record of Processing Activities)

Inventário de processamento (art. 30 GDPR, art. 37 LGPD): finalidade, base legal, dados, destinatários, transferências internacionais, retenção.

### DPO (Data Protection Officer)

Papel formal exigido em vários casos. Independente, reporta ao alto escalão, advisor em privacidade.

### Transferências Internacionais

- **Adequacy decisions** (GDPR/LGPD): países aprovados.
- **SCCs (Standard Contractual Clauses)**.
- **BCRs (Binding Corporate Rules)**.
- **Schrems II** (2020): invalidou Privacy Shield EUA-UE. Transferências para EUA exigem avaliação + safeguards. Novo framework Data Privacy Framework (2023+).

## Dark Patterns

Design que manipula usuário contra seu próprio interesse privacidade:

- **Roach motel**: fácil entrar, difícil sair (desativar conta escondido).
- **Confirmshaming**: "Não, prefiro menos segurança" em lugar de "Não".
- **Pre-checked boxes** para consent.
- **Cookie banners que forçam "Aceitar tudo"** e escondem "Rejeitar".

GDPR/LGPD apertam sobre isso. Design ético = diferencial reputacional.

## Ferramentas

- **Amnesia, ARX Anonymization Tool**: k-anonymity na prática.
- **PySyft, TensorFlow Privacy, Opacus**: DP em ML.
- **OpenDP** (Harvard, Microsoft): library.
- **Microsoft Presidio**: detecta e anonimiza PII em texto.
- **Faker, Mimesis**: dados sintéticos para testes.
- **DataGrail, Transcend, DataGuard**: plataformas DSAR.
- **Tableau/PowerBI row-level security**: em analytics.

## Recursos

- **"Privacy Engineering"** — Michelle Finneran Dennedy et al.
- **"The Algorithmic Foundations of Differential Privacy"** — Dwork, Roth.
- **NIST Privacy Framework**.
- **ANPD (Autoridade Nacional de Proteção de Dados — Brasil)** guias.
- **EDPB guidelines** (Europa).
- **Berkman Klein Center**, **Future of Privacy Forum**, **EFF**.

## Princípios

1. **Mínimo coletar**. O dado que não existe não vaza.
2. **Finalidade antes de coleta**. Se não pode justificar, não colete.
3. **Anonimização é difícil**. Trate "anonimizado" com ceticismo; quase sempre é pseudônimo.
4. **DP é a garantia formal**. Use onde utilidade estatística basta.
5. **Privacy by design**. Checklist pós-release é tarde.
6. **Alinhe legal + engenharia**. Ambos precisam se entender.
7. **Trate privacidade como feature**. Usuários valorizam; diferencia em mercados maduros.
