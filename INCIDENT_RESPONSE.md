# Resposta a Incidentes

**Incident Response (IR)** é o processo disciplinado que uma organização aplica quando detecta um evento de segurança — intrusão, ransomware, exfiltração, DDoS, insider. Não é apenas "apagar o fogo"; é conter o dano, recuperar capacidade, aprender, e preservar evidências para análise e, quando aplicável, processo jurídico.

## O Ciclo PICERL

Formalização clássica (NIST SP 800-61, SANS):

1. **Preparation** (Preparação)
2. **Identification** (Identificação)
3. **Containment** (Contenção)
4. **Eradication** (Erradicação)
5. **Recovery** (Recuperação)
6. **Lessons Learned** (Lições aprendidas)

As fases não são lineares rígidas; frequentemente voltam — identificar novos hosts comprometidos reabre contenção.

## 1. Preparação

A fase mais importante — e a única **antes** do incidente.

### Políticas e plano

- **IR plan** documentado, atualizado, testado. Define papéis (IC, engenheiro, comunicação, jurídico), canais, critérios de severidade.
- **Runbooks** por cenário: ransomware, credencial comprometida, phishing bem-sucedido, insider, exfiltração.
- **Matriz de escalada**: quem notificar, quando.
- **Contatos externos**: vendors, CERTs, autoridades, seguradora, assessoria jurídica.

### Tecnologia

- **EDR/XDR** em endpoints — telemetria é o insumo básico.
- **SIEM** / log centralizado com retenção suficiente (90d mínimo; melhor 365d).
- **Imutabilidade de logs** — atacante vai tentar apagar.
- **Backups offline/imutáveis** — ransomware apaga backups online. Regra 3-2-1.
- **Segmentação de rede** — limita movimento lateral quando necessário.
- **Ferramentas de IR prontas**: imagens forenses (FTK Imager, dd), análise (Volatility, KAPE), coleta (Velociraptor, GRR).

### Pessoas

- **CSIRT** (Computer Security Incident Response Team) designado — interno, terceirizado, ou híbrido.
- **On-call** claro.
- **Treinamento** — tabletop exercises, simulações. Ver `CHAOS_ENGINEERING.md` para game days defensivos.
- **Relação prévia com especialistas externos** (Mandiant, CrowdStrike, Secureworks, boutiques). Primeiro contato em incidente é tarde demais.

### Base legal e regulatória

- Políticas de preservação de evidência.
- Obrigações de notificação: **LGPD** (72h em incidentes significativos no Brasil), **GDPR** (72h), **breach notification** por estado nos EUA, setoriais (saúde, financeiro).
- Acordos de compartilhamento de inteligência.

## 2. Identificação

Evento → incidente. Quando um alerta vira investigação?

### Fontes de detecção

- **EDR** (detecção em endpoint).
- **SIEM** (correlação de logs).
- **NDR / IDS** (rede).
- **Usuários reportando** (canal aberto, não punitivo).
- **Terceiros** — fornecedor, cliente, governo, pesquisador, imprensa. Infelizmente comum.
- **DFIR rotineira** (caça ativa — threat hunting).

### Avaliação inicial

- **Escopo estimado**: quantos hosts, quais dados potencialmente afetados?
- **Gravidade**: tiers (P1/P2/P3) conforme impacto potencial, exposição de dado, risco de propagação.
- **Kill chain**: em que estágio está (initial access, execution, persistence, lateral movement, exfiltration, impact)? Ver MITRE ATT&CK em `MALWARE_ANALYSIS.md`.

### Declarar incidente

Critério explícito. Iniciar timeline, criar war room (canal dedicado, ponte telefônica), definir **Incident Commander (IC)** — única pessoa decide, outros aconselham.

## 3. Contenção

Parar o dano sem destruir evidência.

### Curto prazo (horas)

- **Isolar host**: desconectar rede (EDR faz em um clique modernos), não desligar — preserva memória volátil.
- **Revogar credenciais comprometidas**: trocar senha, invalidar tokens, rotacionar chaves, sair de sessões.
- **Bloquear IOCs**: IPs, domínios, hashes em firewall, proxy, EDR.
- **Desabilitar contas** de usuários afetados.
- **Bloquear regras de e-mail** de encaminhamento criadas pelo atacante (truque comum em BEC).

### Médio prazo

- **Segmentar**: isolar subnets, desligar VPN regional, cortar link com parceiro comprometido.
- **Aplicar patch emergencial** se a origem é vulnerabilidade conhecida.
- **Reforçar MFA** — frequentemente atacante já está dentro porque MFA não estava.

**Dilema clássico**: contenção cedo alerta o atacante → some (e volta). Contenção tardia → mais dano. Decisão do IC conforme risco.

## 4. Erradicação

Remover **toda** presença adversária. Se ficar um backdoor, volta em dias.

- **Reimagem** de máquinas críticas (reinstalação completa; limpar ≠ erradicar).
- **Revogar** certificados comprometidos.
- **Rotacionar** **todos** os segredos acessíveis pelo host comprometido. *Assume breach* das credenciais que passavam por ali.
- **Fechar** vulnerabilidade inicial (patch, config, desabilitar recurso).
- **Mapear persistências** — scheduled tasks, services, run keys, WMI subs, cron, systemd. `PersistenceSniper` e `Autoruns` ajudam.
- **Caça residual**: procurar hosts ainda comprometidos não identificados.

## 5. Recuperação

Voltar a produção segura.

- **Restaurar** de backup limpo se necessário.
- **Validar** que persistências não retornam.
- **Aumentar monitoramento** no período pós-incidente (30-90 dias) — atacante tenta voltar.
- **Comunicação** a usuários/clientes/reguladores conforme exigência legal e de relacionamento.
- **Reset em massa**: frequentemente todas as senhas de funcionários, se credenciais de diretório comprometidas.

Recuperação é fase operacional mais pesada — recriar infra, comunicar, restaurar confiança.

## 6. Lições Aprendidas

Fase **mais negligenciada e mais valiosa** — se a equipe não aprende, o próximo incidente é igual.

### Postmortem blameless

- **Timeline** factual: quando o quê aconteceu, quem descobriu, quando agiu.
- **Causas raízes**: técnicas E organizacionais. "Faltou MFA" E "não priorizamos habilitar porque causaria fricção com time X".
- **O que funcionou bem**: replicar.
- **O que falhou**: corrigir.
- **Ações** com owners e prazos: detecção, hardening, runbook, treinamento.
- **Cultura blameless**: "quem errou" ≠ "o que falhou no sistema que permitiu o erro". Pessoas punidas escondem detalhes; sistema seguro depende de honestidade.

### Compartilhamento

- Internamente: toda a engenharia se beneficia das lições.
- Externamente (com cuidado): CERTs, ISACs, parceiros. Contribui para defesa coletiva.

## Papéis Durante Incidente

- **Incident Commander (IC)**: decisão. Não executa — coordena.
- **Tech Lead**: conduz investigação técnica.
- **Scribe**: mantém timeline detalhada em canal dedicado.
- **Communications Lead**: stakeholders internos, externos, imprensa.
- **Legal/Compliance**: obrigações regulatórias, preservação para litígio, privilégio ACP.
- **Executive Sponsor**: autorizações difíceis (ex.: desligar produto, pagar ransomware).

Em organizações pequenas, uma pessoa acumula papéis. Explicitar ajuda mesmo assim — evita que o IC pare para responder imprensa.

## Comunicação

### Interna

- **Canal separado** do ambiente potencialmente comprometido. Se atacante está no Slack corporativo, war room no Signal.
- **Updates periódicos** (a cada hora no início) mesmo que seja "sem novidade".
- **Need-to-know** para detalhes sensíveis; amplo para status geral.

### Externa

- **Linha cuidadosa** entre transparência e riscos. Rascunho + revisão jurídica + comms.
- **Não especular publicamente** sobre atribuição.
- **Clientes afetados**: notificação honesta; minimização em massa destrói confiança.
- **Reguladores**: dentro do prazo; formato específico.
- **Imprensa**: porta-voz treinado, não engenheiro no canal.

## Ransomware — caso especial

- **Nunca pagar** antes de discussão jurídica + seguro + realidade de descriptografia.
- Pagamento pode violar sanções (OFAC). Consulte advogado.
- Mesmo pagando, decryptor pode não funcionar ou não recuperar tudo. Estatística: ~30% dos que pagam não recuperam completamente.
- **Double extortion**: atacante ameaça vazar dados exfiltrados mesmo se pagar restauração.
- Decryptors públicos (No More Ransom) cobrem várias famílias antigas — sempre verifique.

## BEC (Business Email Compromise)

Fraude por e-mail de conta legítima. **Não é intrusão técnica profunda** — social + credencial. Ações:

- Reset de credenciais + invalidar tokens + sair de sessões.
- Remover regras de encaminhamento/caixa oculta criadas pelo atacante.
- Auditar logs de envio — o que saiu, para quem.
- Avisar parceiros que podem ter sido alvo de fraude nessa conta.
- **Dinheiro transferido** — avisar banco em horas importa; há janela para reversão.

## Insider Threat

Complexidade extra: atacante tem acesso legítimo, conhece organização, pode estar presente durante investigação.

- **Cuidado com comunicação** — não avisar o alvo da investigação.
- **Coleta de evidência** conforme política de RH e privacidade.
- **Coordenação jurídica e RH** obrigatória antes de ação.

## Ferramentas Essenciais

- **EDR**: CrowdStrike Falcon, SentinelOne, Microsoft Defender, Elastic.
- **Forensics**: FTK, EnCase, X-Ways (comerciais); Autopsy/Sleuthkit, Volatility, KAPE (open).
- **Coleta live**: Velociraptor, GRR, KAPE.
- **SIEM**: Splunk, Elastic, Sentinel, Sumo Logic, Chronicle.
- **SOAR**: Palo Alto Cortex XSOAR, Splunk Phantom, Tines — automação de playbooks.
- **Comunicação**: Slack/Teams dedicado ou Signal para casos sensíveis.
- **Gestão de incidente**: Jira, PagerDuty, Squadcast — mas muitos usam doc compartilhado.

## Métricas

- **MTTD (Mean Time to Detect)**: do comprometimento à detecção.
- **MTTR (Mean Time to Respond)**: da detecção à contenção.
- **Dwell time**: tempo que atacante ficou no ambiente sem ser detectado. Benchmarks IBM/Mandiant variam; estado-da-arte é horas a dias, média ainda é dezenas de dias.
- **Taxa de falsos positivos** em alertas.

Medir e reduzir sistematicamente.

## Princípios

1. **Preparação é a única intervenção barata**. Todo o resto é caro e sob pressão.
2. **Preserve evidência antes de "limpar"**. Memória, disco, logs — imagens antes de reimagem.
3. **Contenção rápida, erradicação completa**. Pressa em contenção; rigor em erradicação.
4. **Blameless ou nada**. Cultura de punição destrói capacidade de IR.
5. **Assume breach**. Opere todos os dias assumindo que alguém está dentro; facilita detectar e responder.
6. **Pratique**. Tabletop trimestral mínimo; simulação anual completa. O plano que nunca foi testado não é plano.
