# OSINT — Open-Source Intelligence

**OSINT** é a coleta e análise de informação a partir de fontes **publicamente acessíveis** — sem exploração técnica, sem acesso privilegiado. Em segurança ofensiva, é a primeira fase de reconhecimento (ver `PENTESTING.md`). Em defensiva, threat intel (ver `THREAT_INTEL.md`) e caça a exposição própria. Também é ferramenta jornalística (Bellingcat é a referência pública) e investigativa.

Complementa `G_DORKS.md` (técnicas Google), `WEB_BEACONS.md` (tracking), `SOCIAL_ENGINEERING.md` (o uso frequente de OSINT).

## Disciplinas

OSINT se divide em múltiplas disciplinas, cada uma com fontes e técnicas:

- **SOCMINT** — Social Media Intelligence.
- **GEOINT** — Geospatial (satélite, imagens).
- **IMINT** — Imagery (fotos, CCTV, drones).
- **SIGINT** aberto — ADS-B, AIS (dados de tráfego aéreo/marítimo públicos).
- **FININT** aberto — filings, relatórios regulatórios.
- **HUMINT** aberto — Reddit, fóruns.

## Princípios

1. **Minimize pegada**. Observe sem alterar — use contas descartáveis, VPN, navegadores isolados.
2. **Verifique**. Informação fácil não significa verdadeira. Triangule fontes.
3. **Documentação contemporânea**. Screenshot + URL + timestamp + hash. Conteúdo vai sumir; seu registro persiste.
4. **Aspectos legais e éticos**: acessível ≠ utilizável. LGPD, GDPR, termos de uso. Coleta sistemática pode ser crime em algumas jurisdições.

## Infraestrutura de Reconnaissance

### DNS e Domínios

- **WHOIS**: registrante, data, nameservers. Privacy protection moderna esconde muito, mas dados históricos (Whoxy, DomainTools) revelam.
- **Reverse WHOIS**: domínios com mesmo email/nome.
- **DNS records**: `dig`, `nslookup`, dnsdumpster.com.
- **Passive DNS**: SecurityTrails, RiskIQ, Mnemonic, Farsight DNSDB. Histórico de resoluções.
- **Certificate Transparency**: crt.sh. Todo cert emitido → visível. Revela subdomínios.
- **Subdomain enumeration**: Amass, subfinder, assetfinder, chaos. Combina fontes passivas + brute.
- **Shodan, Censys, FOFA, ZoomEye**: portas/serviços expostos, indexados por varredura contínua. Busca por headers, banners, ports, favicons.

### Corporate / Entidades

- **Receita Federal, JUCESP** (Brasil): CNPJ, sócios, quadro societário.
- **OpenCorporates, Sayari**: bases globais.
- **SEC EDGAR** (EUA): filings.
- **LinkedIn**: funcionários, tech stack (ferramentas mencionadas), hierarquia.
- **Relatórios anuais** de empresas listadas.

### Pessoas

- **LinkedIn, Twitter/X, Facebook, Instagram, Reddit, TikTok**.
- **GitHub, Stack Overflow**: código, interesse técnico.
- **HIBP (Have I Been Pwned)**: e-mails em vazamentos.
- **Reverse image search**: Google Images, TinEye, Yandex — Yandex é surpreendentemente melhor em rostos.
- **PimEyes, FaceCheck**: face search comercial (bordas éticas).
- **Skype/Discord username lookups**.
- **Lusha, Apollo, Hunter.io**: e-mails corporativos.
- **Pipl, Spokeo**: agregadores de dados pessoais (nos EUA).

### Infraestrutura Técnica

- **BuiltWith, Wappalyzer**: tech stack do site.
- **Favicon hash** (MurmurHash): buscar em Shodan sites com o mesmo favicon → descobre infra relacionada.
- **Wayback Machine (archive.org)**: versões antigas; frequentemente arquivam configs/paths expostos brevemente.
- **Google cache**: páginas removidas ainda em cache.
- **Codigo em GitHub**: `trufflehog`, `gitleaks` procuram segredos em repos públicos.

### GEOINT

- **Google Earth, Maps, Street View**.
- **Satellite imagery**: Sentinel (free ESA), Planet Labs, Maxar (pago).
- **Flightradar24, ADS-B Exchange**: voos em tempo real.
- **MarineTraffic, VesselFinder**: AIS de navios.
- **SunCalc, suncalc.org**: sombras para dating de fotos.
- **Weather history** (wunderground): clima num momento exato.
- **OpenStreetMap, Mapillary**: dados geo comunitários.

### Leaked / Breach data

- **HIBP** (haveibeenpwned): e-mail em breach? (sem conteúdo).
- **Dehashed, Leak-Lookup, IntelX**: conteúdo de vazamentos (zona cinza legalmente).
- Discord/Telegram de brokers — ilegal comercialmente na maior parte dos países.

### Dark Web

- **TOR** para acessar `.onion` sites.
- **Ahmia, Tor66**: motores de busca.
- **Market monitoring**: forums, onde credenciais/dados corporativos aparecem à venda.
- **OpSec extremo**: uso de VMs dedicadas, Tails OS, identidades separadas.

## Fluxo de Investigação — Pessoa

Exemplo de pivot em cadeia:

1. **Username** → busca em múltiplas plataformas: `sherlock`, `whatsmyname.app`, `namechk`.
2. **E-mail descoberto** → HIBP, Dehashed (se permitido), reverse WHOIS.
3. **Telefone** (se exposto) → Truecaller.
4. **Foto de perfil** → reverse image. Mesma foto em outro contexto.
5. **Posts** → metadata EXIF em imagens upadas (se preservada — muitas plataformas strip).
6. **Padrão linguístico, timezone, menções** → refinar.
7. **Relações** → grafo de seguidores/amigos; pessoas em comum.

## Fluxo de Investigação — Empresa

1. **Domínio** → subdomínios (amass) + CT logs.
2. **IPs** → Shodan: portas, serviços, banners.
3. **Tech stack** (BuiltWith) → CVEs conhecidos.
4. **E-mails corporativos** (Hunter.io, LinkedIn) → fuzzing de padrão + verificação.
5. **GitHub org** → repos públicos, secrets vazados.
6. **Funcionários-chave** (LinkedIn): devs, SRE, security — alvos de spearphishing.
7. **Documentos expostos** (Google dorks, Wayback) → PDFs internos, manuais, diagramas.
8. **Buckets abertos** → grayhatwarfare.
9. **Historical breaches**: tokens antigos ainda válidos?
10. **Cloud assets**: AWS, Azure, GCP IPs associados (BGP.tools).

## Ferramentas "Hub"

Agregadores que orquestram várias fontes:

- **Maltego**: grafo visual. Transforms conectam fontes (WHOIS, social, DNS). Padrão em investigações sérias.
- **SpiderFoot**: automação de recon massiva; configurable.
- **recon-ng**: framework Python com módulos por fonte.
- **theHarvester**: e-mails, subdomínios, hosts.
- **OSINT Framework** (osintframework.com): diretório visual de fontes por categoria.
- **Intel X (intelx.io)**: search engine comercial que cobre pastebins, darkweb, leaked docs.

## Verificação

Informação falsa é tão abundante quanto verdadeira.

- **Triangulação**: 3+ fontes independentes.
- **Geolocalização de fotos**: SunCalc, landmarks, shadows.
- **Análise de metadata** de imagem: EXIF (FotoForensics, ExifTool), mas redes sociais strippam.
- **Reverse image search**: essa foto já apareceu antes?
- **Análise temporal**: datas plausíveis? árvore com folhas não combina com "dezembro em São Paulo".
- **Perícia forense** de mídia: Forensically, JPEGsnoop.

Bellingcat tem manuais públicos excelentes para verificação.

## OpSec do Investigador

Ver `OPSEC.md`. Em OSINT:

- **VMs** descartáveis — snapshot + reset.
- **VPN** para não expor IP real a alvos.
- **Contas pesquisa** com identidade falsa, isoladas de pessoais.
- **Cuidado com DMs** — enviar mensagem avisa o alvo.
- **Atenção em plataformas sociais**: "quem visualizou meu perfil" existe.
- **Uploads**: ferramenta online pode reter sua amostra.

## Automatização

- **Script Python** para search engines + parsing.
- **APIs oficiais** (Shodan, Censys) quando exigem consulta em massa.
- **Google CSE** (Custom Search Engine) para dorks em batch.
- **Cron + diff**: monitoramento contínuo de alvo/próprio ativo.
- **Webhooks**: notificação quando certo domínio aparece em CT log, pastebin, breach.

## OSINT Defensivo

Para sua organização — antes que atacante faça:

- **Monitorar CT** para certs emitidos em seus domínios (pinning misconfig, typosquatting).
- **Monitorar pastebin/Telegram** para dados vazados (ferramentas: IntelX, dehashed alerts, Recorded Future).
- **Brand protection**: nomes/logos em phishing pages (Phishtank, URLScan).
- **Dark web**: aparições de credenciais, menção de empresa.
- **GitHub secret scanning** em orgs (built-in + comercial).
- **Shodan alertas** por IP/domínio: novo serviço aberto.
- **Google dorks automatizados**: arquivos sensíveis aparecendo no índice.

## Ética e Aspectos Legais

OSINT opera em zona delicada:

- **Dados públicos não são livres**. Scraping massivo viola ToS; agregar dados de fontes separadas pode criar perfil PII regulado por LGPD/GDPR.
- **Stalkerware / doxxing** é crime em várias jurisdições.
- **Nivel de detalhe** pertinente à investigação legítima vs intrusão em privacidade — linha cinza.
- **Uso em emprego / pentest**: escopo autorizado; caso contrário, nada.
- **Jornalismo investigativo**: proteções específicas em muitas leis.

## Recursos

- **Bellingcat Online Investigation Toolkit**.
- **OSINT Curious** (podcast, blog).
- **Michael Bazzell — IntelTechniques** (curso, livros "Open Source Intelligence Techniques").
- **Jake Creps, OSINT Combine, SANS SEC487**.
- **Trace Labs CTFs** — missing persons OSINT CTFs (impacto real).

## Princípios

1. **Comece largo, estreite depois**. Cast wide net, filter.
2. **Documente enquanto coleta**. Conteúdo desaparece, screenshots permanecem.
3. **Verifique ou qualifique a confiança**. "Alta, média, baixa" é útil; "é verdade" raramente se justifica.
4. **OpSec do investigador**. Não se queime em primeira busca.
5. **Ética como filtro**: "faria isto se fosse observado?" Se não, reconsidere.
6. **Automatize repetitivo** mas revise manualmente os achados — ferramentas geram falsos positivos.
