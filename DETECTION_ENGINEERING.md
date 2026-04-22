# Detection Engineering

**Detection Engineering** é a disciplina de criar, manter e melhorar detecções que identificam atividade maliciosa em um ambiente. É para a blue team o que engenharia de software é para qualquer outra área: aplicar rigor, versionamento, testes e métricas a algo que antes era ad-hoc.

Complementa `INCIDENT_RESPONSE.md` (o que fazer após detectar), `MALWARE_ANALYSIS.md` (entender os indicators), `THREAT_INTEL.md` (inputs para detecção) e `OBSERVABILITY.md` (fundação técnica).

## Por Que Engenharia, Não Só "Regras"?

Sem disciplina, detecções viram:

- Centenas de regras, ninguém sabe o que cada uma faz.
- Falsos positivos crônicos → fadiga → alertas reais ignorados.
- Mudança no ambiente quebra detecção; ninguém percebe.
- Atacante adapta; detecções não.

**Detection as Code** traz: versionamento em Git, PR e code review, testes automatizados, pipeline de deploy, métricas.

## Pirâmide de Dor (Bianco)

David Bianco ordena IOCs pelo quão doloroso é para o adversário mudar:

```
         TTPs            (muito doloroso)
       Tools
     Network / Host Artifacts
   Domain Names
 IP Addresses
Hash values                (trivial)
```

- **Hash**: atacante recompila → novo hash. Valor efêmero.
- **IP**: troca de hospedagem → novo IP.
- **Domain**: DGA, domain hopping.
- **Artifacts** (registry keys, filenames, mutexes): exigem re-engineer de malware.
- **Tools**: recriar toolkit é caro.
- **TTPs**: mudar o modus operandi é custoso e lento.

**Conclusão estratégica**: detecções baseadas em comportamento (TTPs) envelhecem melhor que IOCs.

## Camadas de Detecção

### 1. Signature-based (hash, YARA, regex)

- **Pró**: zero falso positivo; rápida.
- **Contra**: atacante muda um byte → evade. Só pega o que já viu antes.
- **Uso**: AV, YARA em arquivos (ver `MALWARE_ANALYSIS.md`), blocklists.

### 2. Heuristic-based

Regras complexas, múltiplos sinais. Ex.: "PowerShell com `-enc` de string longa base64, de um processo sem assinatura, em horário noturno" → flag.

### 3. Anomaly / behavior-based

- **Baseline + desvio**: usuário A normalmente loga de SP; agora está logando do exterior.
- **UEBA (User and Entity Behavior Analytics)**: ML sobre eventos, detecta anomalias.
- **Pró**: detecta o inédito.
- **Contra**: falsos positivos em mudanças legítimas; explicabilidade.

### 4. Threat hunting

Humano em busca **hipótese-dirigida**: "se atacante usou Kerberoasting esta semana, veríamos X no log Y". Não é detecção automática; é caça proativa.

## Sigma

Linguagem aberta, neutra de plataforma, para regras de detecção em logs. Traduz para queries específicas (Splunk SPL, Elastic DSL, Sentinel KQL, etc.).

```yaml
title: Suspicious PowerShell Encoded Command
id: 12345-...
status: experimental
description: Detects PowerShell with encoded command
author: equipe
date: 2026-04-21
logsource:
  product: windows
  service: sysmon
detection:
  selection:
    EventID: 1
    Image|endswith: '\powershell.exe'
    CommandLine|contains|any:
      - '-enc '
      - '-encodedcommand '
  filter_legit:
    CommandLine|contains: 'OurInternalDeployTool'
  condition: selection and not filter_legit
level: medium
tags:
  - attack.execution
  - attack.t1059.001
```

Regras: GitHub `SigmaHQ/sigma`. Projetos como `uncoder.io` (SOC Prime) convertem para backends.

## MITRE ATT&CK como Mapa

Ver `MALWARE_ANALYSIS.md`. Cada detecção deve estar **ligada** a uma técnica/sub-técnica. Permite:

- **Cobertura por TTP**: onde estão os buracos?
- **Priorização**: técnicas usadas por adversários relevantes.
- **Medição**: % de técnicas top-20 com detecção vs sem.
- **Comunicação blue-red**: mesma linguagem, menos ambiguidade.

Ferramentas: **DeTT&CT** (mapear detecção vs ATT&CK), **ATT&CK Navigator**, **Atomic Red Team** (testes vermelhos por técnica).

## Fontes de Dados (Log Sources)

A detecção depende da telemetria disponível. Sem log, sem detecção.

### Windows

- **Windows Event Logs** padrão — básico, ruído alto.
- **Sysmon** com configuração madura (Swift on Security, Olaf Hartong) — telemetria rica de processos, rede, imagens carregadas, WMI.
- **PowerShell Script Block Logging** (4104) — deobfusca e loga o script efetivamente executado.
- **AMSI** events.
- **ETW (Event Tracing for Windows)** para acesso a sources que EDRs consomem.

### Linux

- **auditd** — syscalls e arquivos.
- **systemd-journald**.
- **eBPF** — Falco, Tracee. Observa kernel eventos com baixo overhead.
- **sysmon for Linux** (Microsoft) — port.

### Rede

- **Zeek** — transforma tráfego em logs ricos (conn, dns, http, ssl, files).
- **Suricata** — IDS + logs de fluxo.
- **NetFlow/IPFIX**.
- **DNS logs** — ver `DNS.md`.

### Cloud / SaaS

- **CloudTrail** (AWS), **Audit Logs** (GCP), **Activity Logs**/**Entra ID** (Azure).
- **M365 Unified Audit Log**.
- **GitHub Audit Log**.
- **Okta System Log** / IdP equivalente.

### Endpoint (EDR)

- CrowdStrike, SentinelOne, Defender, Elastic, Velociraptor.
- Telemetria consolidada: processos, rede, filesystem, módulos, tokens, scripts.

## Ciclo de Vida de uma Detecção

1. **Ideação**: vem de threat intel, pentest report, incident, paper, ATT&CK coverage gap.
2. **Requisitos**: qual técnica? que log source? que precisão esperada?
3. **Desenvolvimento**: escrever Sigma ou native. Dev em dataset histórico.
4. **Teste**:
   - **True positive cases**: reproduzir com Atomic Red Team, Caldera, ou manualmente.
   - **False positive check**: rodar em logs de 30-90 dias → revisar hits. Afinar.
5. **Peer review**: PR.
6. **Deploy**: staging → prod. Canary em parte do ambiente.
7. **Monitorar**: SLOs de precisão. Alertas escalados, ignorados, FP rate.
8. **Manter**: versionar, ajustar, aposentar quando obsoleta.

## Métricas

- **TPR (True Positive Rate / Recall)**: de ataques reais, quantos pegamos?
- **FPR (False Positive Rate)**: do total de alertas, quantos eram ruído?
- **Precisão**: TP / (TP + FP). O alerta é real?
- **MTTD (Mean Time to Detect)**: do comprometimento até alerta.
- **Cobertura ATT&CK**: % de técnicas com detecção documentada.
- **Alert volume per analyst**: carga — acima de X, fadiga.
- **Coverage gap closure rate**: novas detecções / mês.

## Anti-Padrões

1. **"Alert on everything"**: se tudo é crítico, nada é. Priorização dói agora; vira catástrofe depois.
2. **Regras sem dono**: ninguém edita, ninguém responsabiliza, ninguém sabe por que existe.
3. **Copy-paste de regras públicas sem adaptação**: não encaixa no ambiente; FP ou FN.
4. **IOCs permanentes**: hash, IP ficam em regra por ano → nunca acionam, poluem.
5. **Sem ambiente de teste**: regra em produção direta quebra analistas.
6. **Sem documentação**: "o que faço quando isto aciona?" — sem runbook, o alerta some.
7. **Detecção isolada de response**: criou alerta, ninguém planejou o que fazer.

## Threat Hunting

Caça proativa, não reativa. Fluxo padrão (Sqrrl's PEAK, Chris Sanders):

1. **Hipótese**: "atacantes fazem pass-the-hash lateralmente; veríamos sequência de eventos 4624 tipo 3 com hashes compartilhados".
2. **Coleta**: dados relevantes, janela de tempo.
3. **Análise**: querying, visualização, enriquecimento.
4. **Descoberta**: achou adversário real? anomalia? nada? (todos são valiosos).
5. **Automação**: transformar caça bem-sucedida em detecção contínua. Caça sem virar detecção repete trabalho.
6. **Documentação**: hunt report.

Metodologias: **PEAK** (Splunk), **TaHiTI**, **TCA** (Target-Centric).

## Equipes e Interações

- **SOC (Security Operations Center)**: consome alertas, triagem, escalada.
- **DE (Detection Engineering)**: constrói/melhora regras.
- **TH (Threat Hunting)**: caça proativa.
- **IR (Incident Response)**: executa quando há incidente.
- **Red Team**: testa detecções, encontra gaps.
- **Purple Team**: coordena DE + Red para fechar gaps rapidamente.

Organizações menores fundem papéis — mas explicitar funções ajuda mesmo assim.

## Ferramentas

- **SIEM**: Splunk, Elastic Security, Microsoft Sentinel, Chronicle, Sumo Logic, Graylog, QRadar.
- **EDR/XDR**: CrowdStrike, SentinelOne, Defender XDR, Elastic.
- **SOAR**: Palo Cortex XSOAR, Splunk Phantom, Tines, Shuffle (open).
- **Detection as Code**: Panther (plataforma), Elastic Detection Rules repo (exemplo de boas práticas), Sigma repositories.
- **Testing/validation**: Atomic Red Team (Red Canary), Caldera (MITRE), Vectr.
- **Hunting**: HELK, Chainsaw, Hayabusa, APT Hunter (sobre logs).

## Recursos

- **"The Practice of Network Security Monitoring"** — Richard Bejtlich.
- **"Intelligence-Driven Incident Response"** — Roberts & Brown.
- **"Applied Incident Response"** — Steve Anson.
- Livros do Roberto Rodriguez (OSSEM, ThreatHunter-Playbook).
- Blogs: Red Canary, Elastic Security Labs, SpecterOps, Mandiant, Microsoft Threat Intelligence.

## Princípios

1. **Detection as code**. Git, PR, CI, testes. Tudo.
2. **Comportamento > IOC**. Atacante troca hash; muda TTP é caro.
3. **Falso positivo é custo**. Minimizar sem matar recall.
4. **Cobertura mapeada** em ATT&CK. Mensurável, comunicável.
5. **Toda detecção tem runbook**. Senão não tem valor operacional.
6. **Test contínuo com red/purple**. Detecção não testada não existe.
7. **Aposente regras obsoletas**. Manter saudável > manter enorme.
