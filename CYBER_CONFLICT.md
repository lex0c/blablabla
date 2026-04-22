# Conflito Cibernético

**Conflito cibernético** é o uso de operações em rede para alcançar objetivos políticos, militares ou econômicos entre estados e atores não-estatais com capacidade estatal-like. Distingue-se de **crime comum** por autoria (estado ou proxy), alvo (infra crítica, defesa, economia estratégica) e motivação (estratégica, não lucro). Evolui de "hacking" amador dos 90s para **quinto domínio** militar formal na doutrina de EUA, OTAN, Rússia, China, Israel e várias outras potências.

Complementa `THREAT_INTEL.md`, `INCIDENT_RESPONSE.md`, `MALWARE_ANALYSIS.md`, `GEOPOLITICS.md`, `OSINT.md`.

## Atribuição Rápida

Vocabulário canônico de grupos por país:

### Rússia

- **APT28 / Fancy Bear** (GRU Unit 26165): eleições EUA 2016, DNC hack, WADA, Bundestag.
- **APT29 / Cozy Bear / Midnight Blizzard** (SVR): SolarWinds (Sunburst), Microsoft/Anthropic attempts.
- **Sandworm / APT44 / Voodoo Bear** (GRU Unit 74455): BlackEnergy (Ucrânia 2015), Industroyer (2016), NotPetya (2017), Olympic Destroyer (2018), VPNFilter, contínua operação em Ucrânia.
- **Turla / Venomous Bear** (FSB): espionagem long-haul sofisticada (Snake, Uroburos).
- **Gamaredon / Primitive Bear**: foco Ucrânia, volume alto, pouco sofisticado.

### China

- **APT1 / Comment Crew** (PLA Unit 61398): exposto por Mandiant 2013; IP theft industrial.
- **APT10 / Menupass / Stone Panda** (MSS): Cloud Hopper (MSPs 2016-18).
- **APT31 / Zirconium**, **APT40 / Leviathan**, **APT41 / Winnti / Barium**: misturam espionagem estatal + lucrativos. APT41 acusado por DOJ EUA 2020.
- **Mustang Panda**: Asia/Europe espionagem.
- **Volt Typhoon**: 2023 revelação — pré-posicionamento em infra crítica dos EUA (water, power, transport). Paradigma "living off the land".
- **Salt Typhoon**: 2024 — compromissos em ISPs/telcos EUA com acesso a intercepção legal.

### Coreia do Norte

- **Lazarus Group** (RGB): Sony Pictures 2014, WannaCry 2017, Bangladesh Bank SWIFT 2016 ($81M), cripto exchanges (Ronin $600M, Lazarus stole ~$3B+ em crypto desde 2017). Funding regime.
- **Kimsuky**: espionagem de defectores, pesquisa nuclear.
- **Andariel, APT37 / Reaper / ScarCruft**.

### Irã

- **APT33 / Elfin, APT34 / OilRig, APT35 / Charming Kitten, MuddyWater**: espionagem, wipers (Shamoon 2012 em Saudi Aramco).
- **Pioneer Kitten / Fox Kitten**: initial access broker.

### Israel / 5-Eyes

Menos publicamente atribuído (menos leaks; melhor OPSEC; sem incentivo para expor):

- **Equation Group** (NSA TAO): exposto pelo The Shadow Brokers 2016-17 — EternalBlue, arsenal de exploits. Stuxnet atribuído a EUA + Israel.
- **Unit 8200** (Israel): Stuxnet, Duqu, Flame. Alumni formam boa parte da indústria israelense de cyber.

### Outros

- **Vietnam (APT32 / OceanLotus)**, **India (SideWinder, Patchwork)**, **Pakistan (Transparent Tribe)**, **Turkey (StrongPity)**: ativos regionais.

Atribuição é sempre com graus de confiança — atacantes usam false flags (Turla roubando infra de APT34, Lazarus plantando código russo em Olympic Destroyer). Ver `THREAT_INTEL.md`.

## Operações Emblemáticas

### Stuxnet (descoberto 2010)

- **Alvo**: centrífugas de enriquecimento de urânio iranianas (Natanz).
- **Método**: worm que infectava Windows, procurava Siemens S7 PLCs, reprogramava frequência dos motores, registrava estado "normal" antes para enganar operadores.
- **Impacto**: ~1000 centrífugas destruídas; atraso estimado de 1-2 anos no programa iraniano.
- **Autor**: EUA (NSA) + Israel, segundo reporting subsequente. Marco: **primeiro ataque cibernético que causou dano físico em larga escala** documentado publicamente.
- Usou **4 zero-days Windows** simultâneos — sinal de recurso estatal.

Legado: todo estudo sério de cyberwar começa aqui.

### Shamoon (2012)

Wiper iraniano contra Saudi Aramco — 30 mil máquinas comprometidas, operações afetadas por semanas. Retornos em 2016, 2018.

### Sony Pictures (2014)

Lazarus em resposta ao filme "The Interview". Exfiltração massiva + wipe. Resposta dos EUA incluiu sanctions públicas.

### Ucrânia — grid elétrico

- **BlackEnergy (2015)**: primeiro corte de grid via cyber confirmado. ~225k pessoas sem energia.
- **Industroyer / CrashOverride (2016)**: sofisticado; protocol-aware (IEC 61850, 60870).
- **Industroyer2 (2022)**: variante evoluída, deployed durante invasão.
- **FrostyGoop (2024)**: dedicated ICS malware.

### WannaCry (maio 2017)

Ransomware-wormy de Lazarus; usou EternalBlue (vazado pelo Shadow Brokers, abril 2017). 200k+ máquinas, incluindo NHS (Reino Unido), Renault, Telefónica. Stopado por kill-switch encontrado por Marcus Hutchins. Prejuízo global bilhões.

### NotPetya (junho 2017)

Não era ransomware — **wiper disfarçado**. Sandworm (GRU). Vetor inicial: software contábil ucraniano M.E.Doc comprometido (supply chain). Pivotou globalmente, devastou Maersk, Merck, FedEx TNT, Mondelez, Saint-Gobain. **Estimado em $10B**, o ataque cibernético mais caro da história. Trump imposed sanctions 2018; EUA e UK atribuíram oficialmente à Rússia.

### Cloud Hopper (2016-2018)

APT10 alvo: MSPs (Managed Service Providers). Uma vez dentro do MSP, acesso a dezenas de clientes corporativos. Escala industrial de IP theft.

### VPNFilter (2018)

Sandworm em roteadores SOHO (Linksys, MikroTik, Netgear, TP-Link, QNAP). 500k+ devices, 54 países. Seu takedown pelo FBI foi a primeira grande operação de sinkhole coordenada.

### SolarWinds / SUNBURST (dez 2020)

APT29. Comprometimento do **pipeline de build** da SolarWinds, trojanizou update do Orion. Distribuído a ~18k clientes, incluindo múltiplas agências federais dos EUA (Tesouro, DOJ, DoE, DHS), Microsoft, FireEye (que detectou primeiro). Paradigma do supply chain attack estratégico.

### Microsoft Exchange / Hafnium (março 2021)

Zero-days (ProxyLogon) em Exchange. China exploitou ~30k servers antes de patch; cascata de atacantes oportunistas seguiu.

### Colonial Pipeline (maio 2021)

Ransomware DarkSide; não um estado, mas afiliado russo. Pipeline de combustível EUA East Coast shut down. Pânico, filas, Biden assinou EO sobre cyber. $4.4M pagos; FBI recuperou ~metade.

### Kaseya (julho 2021)

REvil explorou VSA (RMM) — MSP supply chain. ~1500 empresas afetadas.

### Log4Shell / log4j (dez 2021)

CVE-2021-44228 em biblioteca onipresente. Trivial RCE via string de log contendo `${jndi:ldap://...}`. Frenesi global de scanning e exploit por semanas. Expôs fragilidade de supply chain open-source.

### ViaSat KA-SAT (fev 2022)

Sandworm, no primeiro dia da invasão da Ucrânia. Wiper derrubou modems por toda Europa. Ver `SATELLITE_HACKING.md`.

### Volt Typhoon (2023+)

China pré-posicionando em critical infra EUA. Living off the land em routers/firmware. FBI+NSA+CISA alertas públicos indicando preocupação com **preparação para conflito cinético**.

### MOVEit (2023)

Cl0p ransomware exploitou zero-day no MOVEit Transfer (Progress). ~2,700+ organizações, >77M indivíduos afetados.

### XZ Utils backdoor (março 2024)

Social engineering long-con: "Jia Tan" assumiu maintainership de xz por 2+ anos, plantou backdoor na liblzma afetando sshd. Descoberto por Andres Freund (Microsoft) quase por acidente antes de atingir distros estáveis. Paradigma perigoso de supply chain via contributor takeover.

### Salt Typhoon (2024)

China em ISPs/telcos EUA acessou intercepção legal (CALEA) — aplicação de vigilância. Marco em espionagem tier-0.

## Doutrinas

### EUA

- **US Cyber Command (USCYBERCOM)** desde 2010; elevado a combatant command 2017.
- **"Defend forward"** (2018+): atuar na rede adversária antes que ataque chegue.
- **"Persistent engagement"**: contato contínuo ao invés de eventos discretos.
- **Cyber Mission Force**: ~133 equipes.
- **Joint Cyber Force** proposta 2024-25.

### Rússia

- **GRU (Unit 26165, 74455)**, **FSB (16th/18th Centers)**, **SVR**.
- Doutrina **"hybrid warfare"** (Gerasimov): cyber integrado a info ops, militar não-linear, negação plausível.
- **Proxies** (Conti, REvil) com tolerância estatal.

### China

- **Strategic Support Force (SSF)** criada 2015, consolidou cyber + space + electronic warfare. Reestruturada 2024 em Information Support Force e outros.
- **MSS (Ministry of State Security)**: espionagem civil e industrial.
- **Civil-Military Fusion**: contratos com firmas civis, estudantes, associações.
- **Três Guerras** (san zhong zhanfa): psicológica, pública, legal — cyber é vetor.

### Outros

- **OTAN**: cyber como artigo 5 potencial desde 2014; Cyber Defence Centre of Excellence (Tallinn).
- **Israel Unit 8200**, **UK GCHQ / NCSC**, **Germany BND**, **France ANSSI**, **Japan NISC / MOD cyber defense**, **Australia ASD**.

## Taxonomia de Operações

### Espionagem cibernética (CNE — Computer Network Exploitation)

Objetivo: roubar informação. Baixo ruído, persistência. Maior parte do volume estatal.

### Ataque cibernético (CNA — Computer Network Attack)

Degradar, negar, destruir, disruptar. Minoria do volume, alto impacto. Stuxnet, NotPetya, BlackEnergy.

### Information Operations (IO) / Influence Operations

Moldar percepção. Hack-and-leak (DNC 2016), coordinated inauthentic behavior (CIB), bots, deepfakes. Ver `DEEPFAKES.md`.

### Pré-posicionamento

Acesso mantido para uso futuro — opção estratégica. Volt Typhoon, também feito por EUA/aliados.

### Preparação do campo de batalha

Cyber + SIGINT + OSINT para informar ops cinéticas. Bombardeio Gaza 2021 integrou cyber. Invasão Ucrânia preparada por cyber meses antes.

## Zonas Cinzas

Conflito cibernético vive em **zonas legais e éticas mal definidas**:

- **Atribuição lenta** cria incentivo a negação plausível.
- **Proporcionalidade**: resposta a cyber ataque é cyber, econômico, diplomático, cinético?
- **Thresholds**: quando cyber constitui "ataque armado" (UN Charter Art. 2(4))? Tallinn Manual tenta mapear aplicabilidade de jus in bello.
- **Civilian vs militar**: cyber atinge infra dual-use (hospital, energia, banco).
- **Contratados e proxies**: estado hospeda mas não "opera" — responsabilidade difusa.

## Economia do Conflito Cyber

### Zero-day market

- **Legítimo defensivo**: bug bounties (Apple até $2M, Google até $1.5M).
- **Offensive brokers**: Zerodium (até $2.5M iOS chain), Crowdfense, NSO, Intellexa, Candiru.
- **Gray/black market**: venda para APTs; dificil de mapear.

Preços revelam custo e superfície:

| Exploit | Preço (brokers) |
|---|---|
| iOS zero-click RCE + persistence | $2-3M |
| Android zero-click | $2M+ |
| Windows LPE | $50-200k |
| Browser RCE + SBX | $300k-1M |
| Server RCE em produto enterprise | $50-500k |

### Malware-as-a-Service

RaaS (LockBit, ALPHV, Cl0p), loaders (Emotet, Trickbot), IAB (Initial Access Brokers). Economia de afiliados com % sobre pagamentos.

### Crypto + money laundering

Mixers (Tornado Cash sancionado), chain-hopping, OTC. Agencies tracking: Chainalysis, TRM, Elliptic.

## Infra Crítica — Setores-Alvo

- **Energia**: grid elétrico, oil&gas pipelines, nuclear.
- **Água e esgoto**: setor crônicamente sub-investido em cyber; Florida water 2021.
- **Saúde**: hospitais como ransomware target; COVID research espionagem.
- **Financeiro**: SWIFT heists (NK Bangladesh), bolsas, clearing.
- **Transporte**: aviação (Maersk NotPetya), ferrovia, trânsito urbano.
- **Telecom**: ISPs/telcos (Salt Typhoon), CDN, BGP hijacks.
- **Governo**: intel, defesa, diplomacia.
- **Academia / pesquisa**: IP theft.
- **Eleições**: voter registration, electoral infrastructure.

## Ukraine — Laboratório de Cyber-guerra

Desde 2014 (anexação da Crimeia) e especialmente pós-fev 2022, Ucrânia é o caso moderno mais estudado:

- **BlackEnergy, Industroyer, Industroyer2**: grid.
- **NotPetya (2017)**: destruidor disfarçado.
- **WhisperGate, HermeticWiper, CaddyWiper, AcidRain (ViaSat)**: wipers pré/durante invasão.
- **SSU / CERT-UA**: defesa; partnership com Microsoft/Google/Cloudflare/AWS/Mandiant migrando workloads para cloud estrangeiro.
- **Offensive "IT Army"**: voluntários ucranianos + estrangeiros atacando alvos russos. Precedente de mobilização civil em cyber.
- **FrostyGoop, BlackJack, Nearest Neighbor (2024)**: inovações técnicas.

Lições: resiliência por disseminação (cloud), parcerias público-privadas, mobilização civil controlada, cyber nem decide guerra nem é irrelevante — é vetor adicional.

## Cybernorms — Esforços de Governança

Tentativas de limitar:

- **UN GGE / OEWG**: processos multilateral. Consensos magros.
- **Tallinn Manual 1.0 (2013), 2.0 (2017)**: academic, aplicação de direito internacional a cyber.
- **Budapest Convention** (CoE): criminal cooperation.
- **Paris Call (2018)**: não-binding; muitos signatários.
- **Cyber Peace Institute, Microsoft Digital Peace**: pressão por normas.

Acordos vinculantes entre grandes potências rivais: **não há**.

## Defesa Estratégica

### Nacional

- **CERT/CSIRT nacional**, informação compartilhada com setores.
- **ISAC** setoriais (FS-ISAC, E-ISAC, H-ISAC).
- **Regulação obrigatória** (NIS2 EU, executive orders EUA).
- **Public-private partnerships**: CISA JCDC, Microsoft/Google threat intel sharing.
- **Offensive capability** como dissuasão.

### Organizacional

Ver `INCIDENT_RESPONSE.md`, `DETECTION_ENGINEERING.md`, `THREAT_INTEL.md`. Priorizar por **adversário provável** do setor, não só OWASP.

### Resiliência em guerra

- Cloud geograficamente diverso.
- Backups offline/imutáveis (NotPetya mata backups online; imutabilidade é defesa).
- **Capacidade de degradação graceful** em systems críticos.
- **Exercícios** (tabletop, game days). CyberStorm, Cyber Europe.

## Tendências 2025-2030

1. **Pré-posicionamento em infra crítica** como nova norma. Volt Typhoon paradigma.
2. **AI-enabled ops**: autonomia cresce; phishing personalizado em escala; deepfakes.
3. **Supply chain**: foco em software dependency chains (xz case), MSPs, CI/CD.
4. **Satélites e espaço** como alvo emergente.
5. **OT convergência**: sistemas industriais em rede corporativa — superfície maior.
6. **Cyber + cinético** integrado cada vez mais em doutrinas.
7. **Ransomware geopolítico**: ambiguidade criminal/estado.
8. **Defensor DC cloud vs atacante LOLBin**: assimetria econômica.

## Ética e Deontologia

Para profissional de cyber:

- **Legalidade**: ofensivo sem autorização oficial é crime.
- **Proporcionalidade e discriminação**: princípios jus in bello aplicam-se analogamente.
- **Dual-use**: pesquisa de vulnerabilidade; disclosure responsável vs venda a broker.
- **Empregador**: trabalhar para intel/defesa é escolha pessoal; fazê-lo com ética exige reflexão.
- **Disclosure**: responsabilidade a clientes e sociedade.

## Recursos

- **"Sandworm"** — Andy Greenberg.
- **"This Is How They Tell Me the World Ends"** — Nicole Perlroth.
- **"Dark Territory"** — Fred Kaplan.
- **"Countdown to Zero Day"** — Kim Zetter (Stuxnet).
- **"The Hacker and the State"** — Ben Buchanan.
- **"Cybersecurity and Cyberwar"** — P.W. Singer, Allan Friedman.
- **DEF CON, Black Hat, CyberwarCon, RSA Conference**.
- **Tallinn Manual**, **CyberPeaceInstitute**, **CyberLaw Toolkit**.
- **Mandiant M-Trends, Verizon DBIR, CrowdStrike Global Threat Report, Microsoft DFR**.

## Princípios

1. **Cyber é domínio estratégico**, não puro TI. Design arquitetural reflete premissas geopolíticas.
2. **Atribuição é política**, não puramente técnica. Graus de confiança; mobilização política complexa.
3. **Dissuasão em cyber é frágil**. Low cost of entry, ambiguidade, proxies — cost-benefit atacantes é assimetricamente favorável.
4. **Pré-posicionamento é o padrão**. Assume adversário competente está dentro.
5. **Resiliência > prevenção absoluta**. Recovery rápido é capacidade estratégica.
6. **Supply chain é vetor dominante**. Integridade não termina no seu perímetro.
7. **Infra crítica merece governança própria**. OT/ICS não é IT; regulação convergindo.
8. **Cyber-norms ainda fracos**. Ambiente normativo imaturo favorece agressores.
