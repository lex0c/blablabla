# Compliance em Segurança

**Compliance** é a conformidade com frameworks regulatórios, contratuais ou voluntários. Diferente de "estar seguro", compliance é **poder demonstrar** conformidade com critérios externos — via evidência, auditoria, certificação.

Relação com segurança:

- **Compliance ⊂ Segurança ≠ Segurança ⊂ Compliance**. Uma organização compliant pode ter vulnerabilidades reais; uma organização segura pode falhar auditoria por papelada.
- **Ponto de partida útil**: frameworks cobrem controles mínimos consensuais.
- **Ponto de chegada pobre**: atender só ao checklist é atender ao auditor, não ao adversário.

## Principais Frameworks / Regulações

### Horizontal (aplicáveis a muitos setores)

- **ISO/IEC 27001**: gestão de segurança da informação. Baseado em Annex A com 93 controles. Requer SGSI (Sistema de Gestão). Certificação auditada.
- **ISO/IEC 27002**: guia de implementação dos controles.
- **SOC 2** (AICPA): **Trust Services Criteria** — Security, Availability, Processing Integrity, Confidentiality, Privacy. Relatório Type I (ponto no tempo) ou Type II (período de ~6-12 meses). Não é certificação; é *attestation* por auditor licensed.
- **NIST Cybersecurity Framework (CSF) 2.0**: Identify, Protect, Detect, Respond, Recover, Govern. Voluntário; referência forte.
- **NIST SP 800-53 / 800-171**: catálogo de controles (FISMA, governo EUA) — 800-171 para contratantes.
- **CIS Controls**: 18 controles com implementation groups. Concreto, acessível.

### Privacidade

- **LGPD** (Lei 13.709/2018, Brasil). Princípios alinhados ao GDPR; ANPD é o regulador.
- **GDPR** (UE). Multas até 4% do faturamento global ou €20M, o maior.
- **CCPA/CPRA** (Califórnia).
- **HIPAA** (saúde EUA).
- **PIPL** (China).

### Setoriais

- **PCI DSS**: transações com cartão. v4.0 (2022+), muito detalhado. Tier 1-4 por volume. Violações caras — multas + perda de merchant account.
- **HIPAA** (saúde EUA).
- **GLBA** (financeiro EUA).
- **SOX Section 404**: controles internos sobre financial reporting (listadas EUA).
- **NERC CIP**: setor elétrico EUA.
- **TISAX**: automotivo europeu.
- **ISO 13485** (dispositivos médicos), **IEC 62443** (ICS), **DORA** (financeiro EU 2025+).

### Governo e Defesa

- **FedRAMP**: cloud para governo EUA. High/Moderate/Low.
- **CMMC**: DoD contractors.
- **IL4/IL5/IL6**: DoD impact levels.

## LGPD — Resumo

Lei Geral de Proteção de Dados do Brasil (2018, em vigor 2020):

### Bases Legais para Tratamento

Art. 7:

1. Consentimento.
2. Obrigação legal/regulatória.
3. Execução de políticas públicas.
4. Estudos por órgão de pesquisa.
5. Execução de contrato.
6. Exercício regular de direitos.
7. Proteção da vida/incolumidade física.
8. Tutela da saúde.
9. **Interesse legítimo**.
10. Proteção do crédito.

**Dados sensíveis** (art. 11): origem racial/étnica, convicção religiosa, opinião política, saúde, sexual, biometria, genética — exigências mais estritas.

### Direitos do Titular (art. 18)

- Confirmação de tratamento.
- Acesso.
- Correção.
- Anonimização, bloqueio, eliminação.
- Portabilidade.
- Eliminação de dados tratados com consentimento.
- Informação sobre compartilhamento.
- Revogação de consentimento.

### Papéis

- **Controlador**: decide sobre o tratamento.
- **Operador**: trata em nome do controlador.
- **Encarregado (DPO)**: ponte com titulares e ANPD.

### Incidentes

Notificação à ANPD + titulares em **prazo razoável** (ANPD emitiu guidance sugerindo 2 dias úteis em casos relevantes) quando há risco/dano aos titulares.

### Sanções

- Advertência, multa (até 2% do faturamento Brasil, teto R$50M por infração), bloqueio ou eliminação de dados relacionados, suspensão do banco.

## GDPR — Complementos

- **Aplicação extraterritorial**: empresa fora da UE tratando dados de residentes europeus.
- **Notificação em 72h** à autoridade em breach com risco.
- **DPIA** obrigatório em casos de alto risco.
- **DPO** obrigatório em certos cenários (tratamento sistemático em larga escala, core activity, dados sensíveis).
- **Transferências internacionais**: adequação, SCCs, BCRs; Schrems II.

## SOC 2 — Essencial

- **Trust Services Criteria (TSC)**: Security (obrigatório), mais opcionais (Availability, Confidentiality, Processing Integrity, Privacy).
- **Common Criteria (CC)**: ~150+ critérios distribuídos por TSCs.
- **Period**: Type I é ponto no tempo; Type II é 6-12 meses (mais valorizado).
- **Report**: não público por default; compartilhado sob NDA.
- **Fee típico**: US$ 25-100k+ primeira vez; US$ 15-50k continuação.
- **Ferramentas (GRC/automation)**: Vanta, Drata, Secureframe, Tugboat Logic, AuditBoard.

## ISO 27001 — Essencial

- **SGSI**: Information Security Management System; processo contínuo.
- **PDCA**: Plan-Do-Check-Act.
- **Annex A (93 controles)**: organizacional, pessoas, físico, tecnológico.
- **Statement of Applicability (SoA)**: quais controles se aplicam, justificativa.
- **Certificação** por CB acreditado (BSI, Bureau Veritas, DQS, etc.). Ciclo de 3 anos com surveillance anual.
- **Alternativa**: implementar sem certificar, útil para referência interna.

## PCI DSS — Essencial

Se você "toca" dado de cartão (armazenar, processar, transmitir):

- **12 requerimentos** principais; centenas de sub-requisitos.
- **SAQ** (Self-Assessment Questionnaire) para merchants pequenos.
- **QSA (Qualified Security Assessor)** para Nível 1.
- **Escopo**: reduzir o ambiente que tem dado → **tokenização** é aliada principal. Use processadores PCI-compliant (Stripe, Braintree) e mantenha seu ambiente fora do escopo.
- **v4.0** (2022): requerimentos mais rigorosos, transição até 2025-2025.

## Mapeamento Entre Frameworks

Controles se sobrepõem muito — investimento em um traz valor para outros.

- **SOC 2 + ISO 27001**: ~80% overlap.
- **NIST CSF**: mapeia quase tudo.
- Ferramentas como **Vanta** implementam muitos frameworks com mesmo evidence.
- **Unified Compliance Framework (UCF)** oferece crosswalks.

## Evidência e Auditoria

Compliance é muito sobre **evidência**. Controles sem prova de aplicação não passam auditoria.

### Tipos de evidência

- **Políticas**: documentos escritos, aprovados, revisados anualmente.
- **Screenshots** de configurações.
- **Logs**: auth events, changes, access.
- **Tickets**: provando processo (IR, access request, change management).
- **Training records**.
- **Pentest reports, scans**.

### Automação de evidência (GRC modernas)

- **Vanta, Drata, Secureframe, Scrut, Sprinto**: integram com cloud/SaaS, coletam evidência contínua (MFA status, logs habilitados, usuários, devices).
- Reduz fricção em audit prep de meses para semanas.
- Cuidado: automatização faz parecer que controle funciona — mas se não tem efeito operacional, é security theater.

## Governança

Compliance depende de estrutura organizacional:

- **CISO / Security Officer**: responsabilidade executiva.
- **Comitê de Segurança**: cross-functional.
- **Políticas** aprovadas pelo topo.
- **Treinamento** recorrente (anual + phishing simulations).
- **Risk Register**: riscos identificados, avaliados, ação-owner.
- **Exceções documentadas** com justificativa e expiration.

## Risk Management

Framework típico:

1. **Identificar** riscos (threat × vulnerability × impact).
2. **Avaliar**: probabilidade × impacto. Matriz 5×5 ou categorias.
3. **Tratar**: mitigate, transfer (seguro), accept (com aprovação), avoid.
4. **Monitorar**: revisar periódico.

Integra-se a `THREAT_MODELING.md`. Outputs alimentam priorização de segurança.

## Vendor / Third-Party Risk

Fornecedores podem comprometer você (ver `SUPPLY_CHAIN.md`).

- **Due diligence** pré-contratação: SOC 2/ISO 27001, questionários (SIG, CAIQ).
- **Contratos**: data processing agreements (LGPD/GDPR Art. 28), SLAs de segurança, right to audit.
- **Monitoramento contínuo**: BitSight, SecurityScorecard, UpGuard — ratings externos.
- **Inventário**: todo vendor com acesso a dados mapeado.

## Penalidades Reais

Não é teoria:

- **Equifax** (2017 breach): ~US$ 1.4B em acordos + FTC order.
- **British Airways** (GDPR): £20M após redução.
- **Meta** (multi-casos GDPR): €1.2B (2023, transferência de dados EUA).
- **Ashley Madison**: $11.2M em settlement.
- **LGPD**: ANPD aplicou primeira multa em 2023 (Telekall); outras em sequência.

Média de custo de breach (IBM Cost of a Data Breach Report 2023): US$ 4.45M globalmente, US$ 1.36M no Brasil.

## Anti-Padrões

1. **Compliance theater**: políticas belas, zero operação.
2. **Audit hopping**: preparar só na época de auditoria.
3. **Checklist minded**: esquecer ameaças não-cobertas.
4. **Overengineer**: compliance "além do necessário", gastando recurso que faria falta em defesa real.
5. **Ignorar risk-based thinking**: aplicar todos os controles igualmente em vez de priorizar por risco real.
6. **Silos**: compliance separado de engineering — documentação nunca casa com realidade.
7. **Vendor lock-in de GRC sem uso real**: só ativa perto da auditoria.

## Implementação Prática (Resumo)

Para empresa sendo criada:

1. **Definir ativos e riscos** (básico).
2. **Políticas mínimas**: security policy, acceptable use, access control, incident response, backup, vendor mgmt.
3. **Controles técnicos**: MFA, logging, encryption, patching, backup, EDR.
4. **Training** anual.
5. **Ferramenta de GRC** desde cedo (evita retrabalho em evidence).
6. **Escolher framework** conforme cliente/mercado: SOC 2 para SaaS B2B EUA, ISO 27001 para B2B global, LGPD/GDPR sempre, PCI se toca cartão.
7. **Audit**: começar com *gap assessment* (auditor amigável) antes do audit oficial.
8. **Continuous**: integrar em dev process; não episódico.

## Recursos

- **"CISO Compass"** — Todd Fitzgerald.
- **NIST CSF 2.0 documentação** (gratuito).
- **CIS Controls v8** (gratuito).
- **OWASP SAMM**.
- **ISACA, ISC2** — certificações: CISSP, CISM, CISA, CRISC.
- **"SOC 2 Simplified"** — guide.
- **Brian Krebs**, **Troy Hunt** (HIBP) — compliance em contexto.

## Princípios

1. **Compliance é piso, não teto**. Pensar defesa acima dos controles mínimos.
2. **Risk-based > checklist**. Priorize pelo que realmente importa a você.
3. **Evidência contínua > burst pré-audit**. Integre em operação.
4. **Automatize coleta**. Humano só revisa.
5. **Políticas refletem prática, não o contrário**. Se não consegue cumprir, revise a política.
6. **Um framework bem > três mal**. Overhead linear; valor decrescente.
7. **Compliance é produto**: habilitador de vendas B2B e confiança. Trate como tal.
