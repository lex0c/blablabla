# Threat Intelligence (CTI)

**Threat Intelligence** (TI), ou **Cyber Threat Intelligence** (CTI), é informação sobre adversários, suas capacidades, intenções e operações, processada e contextualizada para decisão. É diferente de **dados** soltos (IOCs em listas) — é **inteligência** quando responde a uma pergunta e direciona ação.

## Definição — quatro níveis

1. **Strategic**: para executivos. Quem são os atacantes de alto nível que ameaçam nosso setor? Em que direção evoluem? Relatórios trimestrais.
2. **Operational**: para SOC/IR. Quais TTPs estão em uso agora pelo grupo X? Como se defender?
3. **Tactical**: para defensores técnicos. Assinaturas, regras, detections específicas.
4. **Technical**: IOCs puros — hashes, IPs, domínios.

Cada nível tem audiência, cadência, formato próprios. Confundi-los é a causa mais comum de TI irrelevante.

## Ciclo de Inteligência

Clássico (emprestado da inteligência tradicional):

1. **Planning & Direction**: que pergunta queremos responder? (Requirements)
2. **Collection**: OSINT, IOCs, feeds, análise própria.
3. **Processing**: normalização, enriquecimento, deduplicação.
4. **Analysis**: transformar dados em inteligência — contextualizar, hipotetizar.
5. **Dissemination**: formatos apropriados à audiência; no momento certo.
6. **Feedback**: a inteligência gerou decisão? Ajustar.

Pular passos gera "feeds para ninguém". TI boa começa por **perguntar**.

## Fontes

### OSINT

- **Blogs de vendors**: Mandiant, Microsoft Threat Intelligence, Red Canary, Crowdstrike, ESET, Kaspersky, Palo Alto Unit 42, Cisco Talos.
- **Reports de governos**: CISA advisories, ACSC, NCSC UK.
- **Research groups**: Citizen Lab, Volexity, GroupIB.
- **Twitter/X, Mastodon**: pesquisadores compartilham IOCs rapidamente — `@malwrhunterteam`, `@vxunderground`, etc.
- **Public malware databases**: MalwareBazaar, URLhaus, ANY.RUN reports, Joe Sandbox public.
- **Pastebin/telegram channels** que divulgam leaks.

### Fechadas / Comerciais

- **Recorded Future, Mandiant Advantage, Flashpoint, Intel 471, Group-IB, Analyst1**: feeds com análise e IOCs.
- **ISAC** (Information Sharing and Analysis Centers): FS-ISAC (financeiro), H-ISAC (saúde), etc. Sharing comunitário setorial.
- **CTI Services** específicas: Digital Shadows (SearchLight), CybelAngel, Recorded Future Threat Intelligence Module.

### Internas

- **IR passada**: a própria org é rica fonte. Incidentes geram TTPs observados.
- **Honeypots / deception**: T-Pot, Thinkst Canary — captura de atividade maliciosa.
- **Dark web monitoring**: próprios dados, credenciais à venda.
- **Brand protection**: typosquatting em CT logs; fakes em redes sociais.

## Modelos de Análise

### MITRE ATT&CK

Já coberto em `MALWARE_ANALYSIS.md` e `DETECTION_ENGINEERING.md`. Vocabulário comum para descrever TTPs. Pergunta típica TI: "Qual a cobertura de detecção para técnicas usadas por UNC2452 (SolarWinds actor)?".

### Diamond Model (Caltagirone, 2013)

Um evento de intrusão tem 4 vértices:

- **Adversary**: quem.
- **Capability**: o quê (malware, exploit, técnica).
- **Infrastructure**: de onde (IP, domínio, C2).
- **Victim**: contra quem.

Conexões entre vértices definem analytics. Múltiplos diamantes conectados formam *campaigns*.

### Kill Chain (Lockheed Martin)

7 fases:

1. Reconnaissance
2. Weaponization
3. Delivery
4. Exploitation
5. Installation
6. C2
7. Actions on Objectives

Modelo criticado por ser muito linear (intrusões modernas são iterativas, fora de ordem), mas ainda útil para comunicação macro.

### Pyramid of Pain

Já citada em `DETECTION_ENGINEERING.md`. Ordena IOCs pelo custo de mudança para o adversário: TTP > tool > artifact > domain > IP > hash.

### Courses of Action Matrix

Para cada fase da kill chain × ação defensiva (detect, deny, disrupt, degrade, deceive, destroy), mapear capacidades organizacionais. Ferramenta de gap assessment.

## Atribuição

**Perigosa, cara e frequentemente errada**. Sinais típicos:

- **Infrastructure overlap**: mesmos IPs/C2 em operações.
- **Code reuse**: bibliotecas, strings, comentários.
- **TTPs**: sequência de técnicas característica.
- **Timing**: timezone, feriados nacionais (atacantes folgam).
- **Language artifacts**: comentários, path de build.
- **Crypto specifics**: parâmetros (implementação de TLS), construção de key.
- **OpSec failures**: meta, email, credencial pessoal.

**False flag** é real — estado pode plantar artefatos de outro estado. Grupos como Turla (Rússia) já reusaram infra de Iranians (APT34). Atribuição responsável: com graus de confiança, aberta a revisão.

Taxonomias de grupos:

- **Mandiant**: APT29, FIN7, UNC1234 (UNC = unclassified cluster).
- **CrowdStrike**: "animals by region" — BEAR (Rússia), PANDA (China), KITTEN (Irã), CHOLLIMA (Coreia do Norte), BUFFALO (Vietnã).
- **Microsoft**: "weather" — Midnight Blizzard, Forest Blizzard.
- **Kaspersky, ESET, Talos**: cada um sua taxonomia. **MITRE ATT&CK Groups** consolida mappings.

## Formatos e Padrões

### STIX / TAXII

**STIX (Structured Threat Information eXpression)**: modelo JSON para representar TI. Objects: Indicator, Malware, Threat Actor, Campaign, Tool, Attack Pattern, Relationship.

**TAXII**: protocolo de transporte/sharing (request/response, pub/sub).

Padrão quando múltiplas orgs trocam TI.

### MISP

**MISP (Malware Information Sharing Platform)**: plataforma open-source para sharing. Events com IOCs, correlations, taxonomies. Integração com feeds comerciais e comunidade.

### OpenIOC, YARA, Sigma

- **OpenIOC** (Mandiant, original): XML, legado.
- **YARA**: assinaturas em arquivos.
- **Sigma**: regras em logs.

Convergindo: todas podem coexistir. STIX como wrapper comum.

## TLP — Traffic Light Protocol

Semáforo para indicar até onde um item de inteligência pode ser compartilhado:

- **TLP:RED** — destinatários específicos; não compartilhar além.
- **TLP:AMBER** — dentro da organização / parceiros need-to-know.
- **TLP:AMBER+STRICT** — só dentro da organização.
- **TLP:GREEN** — comunidade; mas não público.
- **TLP:CLEAR** (formalmente TLP:WHITE, renomeado): público.

Respeitar TLP é cultura — violação mata sharing futuro.

## IOC vs IOA

- **IOC (Indicator of Compromise)**: evidência estática — hash, IP, domínio, registry key. "Já aconteceu".
- **IOA (Indicator of Attack)**: evidência comportamental — sequência de ações indicando ataque em progresso, independente de artefato específico. "Está acontecendo".

IOC envelhece rápido; IOA é mais durável. Modernos detections focam IOA (ATT&CK-based).

## Enriquecimento

Dado bruto (IP suspeito) vale pouco; **enriquecido** com contexto vale mais:

- **Geolocation** (MaxMind).
- **ASN / owner**: cloud provider? hosting conhecido de abuso?
- **Reverse DNS**.
- **Passive DNS history**: que domínios já apontaram para este IP?
- **Certificate history**.
- **VirusTotal reputação**.
- **WHOIS** (para domínios).
- **Shodan / Censys**: serviços.
- **Sandbox reports** para hashes.

Ferramentas de enriquecimento automático: **Cortex (TheHive), MISP modules, TIP platforms** (Recorded Future, Anomali).

## Threat Intelligence Platform (TIP)

- **Recorded Future, Anomali, EclecticIQ, ThreatConnect** (comerciais).
- **MISP + TheHive + Cortex** (open-source stack).

Função: agregar feeds → deduplicar → enriquecer → prover via API → integrar com SIEM/EDR/SOAR.

## Ameaças — Tipologia

- **Nation-state / APT**: Rússia (APT28/29, Sandworm, Turla), China (APT10/41, Winnti, Mustang Panda), Coreia do Norte (Lazarus, Kimsuky, Andariel), Irã (APT33/34/35, Charming Kitten), outros (OceanLotus, equações EUA/"Tailored Access Operations").
- **Cybercrime**: ransomware as a service (LockBit, ALPHV/BlackCat, Clop, Play, Akira, RansomHub), infostealers (Redline, Raccoon, Vidar, StealC, Lumma), bankers, crypters.
- **Hacktivists**: Anonymous, GhostSec, regional.
- **Insider**: colaboradores maliciosos ou negligentes.
- **Script kiddies / opportunistic**: varrem Internet, exploram o que acham.

Priorização por **provável adversário do seu setor**, não todo o catálogo.

## Priority Intelligence Requirements (PIR)

Perguntas estratégicas que guiam coleta e análise. Exemplos:

- "Qual o risco de ransomware afetar nosso setor nos próximos 6 meses?"
- "UNC2452 ainda visa nossa vertical?"
- "Novas técnicas em abuso de OAuth — há IOCs específicos?"
- "Funcionário alvo de spearphishing recente — contexto?"

PIRs precisam ser concretos, atualizados, alinhados com risco. Sem PIR, TI vira consumo passivo.

## Integração com Operações

1. **SOC**: IOCs alimentam regras de detection; relatórios informam enhancements.
2. **IR**: triagem usa reputation; investigação cruza com TI histórica.
3. **Red Team**: emulação de adversário — baseada em TTPs reais (Atomic Red Team, Caldera).
4. **Risk / GRC**: input para análise de risco, compliance reports.
5. **Executivos**: strategic intel; decisões de investimento.
6. **Desenvolvimento / Security Engineering**: hardening contra técnicas conhecidas.

## Métricas

- **Relevância**: % IOCs que se materializam no ambiente.
- **Timeliness**: tempo entre publicação e consumo.
- **Actionability**: intel gerou decisão?
- **Coverage ATT&CK**: % de técnicas relevantes cobertas.
- **False positive rate** em detections alimentadas por TI.

"Quantos IOCs ingerimos" é vaidade, não métrica de valor.

## Armadilhas

1. **Feed fatigue**: mil feeds, zero insight. Priorize qualidade.
2. **IOCs sem TTL**: hash de 2019 em regra ativa = ruído eterno.
3. **Atribuição pública com baixa confiança**: mata credibilidade.
4. **TI desacoplada de ação**: relatório que ninguém lê ou implementa.
5. **Confundir ameaça com relevância**: "Laos com APT russos" pode não ser seu risco.
6. **Automação cega**: enriquecimento automático em lote pode importar IOCs errados (DNS públicos, serviços legítimos). Curadoria é necessária.

## Recursos

- **"Intelligence-Driven Incident Response"** — Scott J. Roberts, Rebekah Brown.
- **"The Threat Intelligence Handbook"** — Recorded Future (free).
- **SANS FOR578, FOR610**.
- **Blogs**: Mandiant, Unit42, Talos, Microsoft Threat Intelligence, Red Canary, vxunderground.
- **Podcasts**: The CyberWire Daily Briefing, Risky Business, Darknet Diaries (narrativo).

## Princípios

1. **Inteligência > dados**. Contexto, análise, direcionamento.
2. **PIRs primeiro**. Sem pergunta, sem resposta útil.
3. **Tempo é dimensão**. Intel atrasada = história.
4. **Confiança explícita**: alta/média/baixa, nunca certeza absoluta.
5. **Compartilhe — com TLP**. Defesa coletiva é multiplicador.
6. **Atribuição cautelosa**. False flags são reais.
7. **Automatize enriquecimento, humano faz análise**. Escalar o braço, não o julgamento.
