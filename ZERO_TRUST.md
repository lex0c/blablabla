# Zero Trust

**Zero Trust** é um modelo de segurança cujo princípio central é: **nunca confie, sempre verifique** (*never trust, always verify*). Rejeita a hipótese de que existe uma rede "interna" confiável e uma "externa" hostil. Em vez disso, cada requisição é autenticada, autorizada e encriptada, independente da origem.

Desenvolvido ao longo dos anos 2010 (Forrester, Google BeyondCorp, NIST SP 800-207) em resposta a: (1) mobilidade — o "perímetro" não faz sentido com laptops fora da rede; (2) cloud — workloads em provedores terceiros; (3) APTs — atacantes que já estão dentro; (4) insider threats.

## Princípios (NIST SP 800-207)

1. **Todos os data sources e computing services são recursos**.
2. **Toda comunicação é segura, independente da localização**. Mesmo tráfego "interno" é criptografado.
3. **Acesso a recursos é concedido por sessão** — não acesso perpétuo.
4. **Decisão de acesso é dinâmica e baseada em política**, usando múltiplos sinais: identidade, device posture, comportamento, risco, tempo.
5. **Monitoramento e medição contínuos** da postura de segurança dos ativos.
6. **Autenticação e autorização estritas** antes de acesso.
7. **Coleta máxima de informação** do estado atual para melhorar decisão.

Resumindo a mensagem: **rede não é identidade**. Estar "dentro" não implica autorização.

## Perímetro Tradicional vs Zero Trust

| | Tradicional (castle-and-moat) | Zero Trust |
|---|---|---|
| Confiança | Rede interna = trusted | Nenhuma rede é trusted |
| Autenticação | Uma vez no VPN | Cada requisição |
| Autorização | Rede/IP como sinal | Identidade + device + contexto |
| Blast radius em breach | Grande | Contido |
| Adequado a cloud/mobile | Mal | Nativo |

Breach interno em modelo tradicional = passeio livre. Em ZT, atacante precisa re-autenticar em cada recurso com telemetria que pode flagear anomalia.

## BeyondCorp

**BeyondCorp** é a implementação ZT do Google — resposta ao Aurora (2010). Tornada pública em papers (2014+). Pilares:

- **Device inventory**: todo device é conhecido, identificado (cert), e tem postura avaliada.
- **User identity service**: OIDC/SSO centralizado com MFA.
- **Access proxy**: frontend que autentica user + device, aplica policy, roteia a serviço backend.
- **Trust tiers**: device/user classificados em tiers; acesso permitido por tier exigido do recurso.
- **Sem VPN**: usuário acessa recursos internos pela Internet via access proxy.

Produtos comerciais do conceito: **Google Cloud IAP**, **Cloudflare Access**, **Zscaler Private Access**, **Tailscale**, **Netskope**, **Palo Alto Prisma Access**.

## Componentes

### PDP / PEP

Ver `AUTHENTICATION.md`. Em ZT:

- **Policy Decision Point (PDP)**: recebe contexto, decide.
- **Policy Enforcement Point (PEP)**: intercepta requests, chama PDP, aplica decisão.
- **Policy Information Points (PIP)**: fontes de sinais (device management, threat intel, user risk score).

PEP pode ser: access proxy, API gateway, service mesh sidecar, endpoint agent.

### Identity Provider (IdP)

Centraliza autenticação. OIDC/SAML-compliant — Okta, Entra ID, Google Workspace, Ping, JumpCloud.

Sinais emitidos: claims de identidade, grupos, MFA status, last login anomalies.

### Device Trust

- **Inventário**: CMDB, MDM (Intune, Jamf, Kandji).
- **Posture**: antivírus atualizado? disco criptografado? patch level?
- **Device certificate**: instalado pelo MDM; prova que o device é corporativo. mTLS no access proxy.
- **EDR integration**: telemetry alimenta risk score.

### Context / Risk Engine

Combina sinais para decisão dinâmica:

- IP / geo → normal para o usuário?
- Device posture adequada?
- Tempo de dia / dia da semana consistente?
- User behavior analytics (UBA) → anomalia?
- Threat intel → IP/ASN suspeito?

Decisão contextual: mesmo user + mesmo recurso, pode ser allow / step-up MFA / deny dependendo do contexto.

### mTLS entre Workloads

Para comunicação service-to-service:

- **Service mesh** (Istio, Linkerd, Consul Connect): sidecar injeta mTLS, identidade SPIFFE.
- **SPIFFE/SPIRE**: padrão para identidade de workload. SVID (SPIFFE Verifiable Identity Document) é cert X.509 ou JWT com ID `spiffe://trust-domain/workload`.
- **Cloud-native identity**: AWS IAM, GCP Workload Identity, Azure Managed Identity — para chamadas a APIs gerenciadas.

## Arquiteturas

### Client-based

Agent no endpoint (Tailscale, Cloudflare WARP, Zscaler Client Connector) roteia tráfego por sessão autenticada.

### Proxy-based (clientless)

Access proxy em frente a apps web; browser redireciona para auth, depois passa. Google IAP, Cloudflare Access zero-trust no web.

### Mixed

Híbrido: agent para TCP/non-HTTP + proxy para web. Típico em ofertas ZTNA (Zero Trust Network Access).

## Just-in-Time Access (JIT)

Privileged access é concedido **temporariamente**, sob demanda, com aprovação e expiração.

- **AWS IAM Identity Center Assume Role** com sessions curtas.
- **PIM** (Privileged Identity Management, Azure) — role ativada por N horas com MFA.
- **sudo com MFA** em servers.
- **Break-glass** documentado e auditado.

Elimina "admin permanente" — superfície temporal minimizada.

## Microsegmentação

Rede dividida em pequenas zonas com policies específicas. Lateral movement entre workloads exige cruzar controles.

- **Cloud**: Security Groups, NSG, Network Policies K8s (ver `CLOUD_SECURITY.md`, `CONTAINER_SECURITY.md`).
- **On-prem**: VMware NSX, Cisco ACI, Illumio, Guardicore.
- **SDN**: políticas por label/tag, não IP. Adapta-se a movimento de workload.

## Telemetria e Monitoramento

ZT é alimentada por dados. Sem telemetria, PDP decide no escuro.

- Logs de autenticação (IdP).
- Logs de acesso (proxy, API gateway).
- Device posture (EDR, MDM).
- Network flows (service mesh).
- Anomalias via UEBA.

Tudo fluí para SIEM; alimenta risk engine; informa detecção (ver `DETECTION_ENGINEERING.md`).

## SASE / SSE

**SASE (Secure Access Service Edge)** — Gartner, 2019. Convergência de SD-WAN + segurança de rede (SWG, CASB, ZTNA, FWaaS) em cloud.

**SSE (Security Service Edge)** = SASE sem SD-WAN. Subset focado em security.

Fornecedores: Netskope, Zscaler, Palo Alto Prisma, Cloudflare One, Cisco Umbrella.

## Maturidade — CISA Zero Trust Maturity Model

CISA (Cybersecurity and Infrastructure Security Agency) publicou modelo com 5 pilares × 4 estágios:

**Pilares**:
1. Identity
2. Devices
3. Networks
4. Applications & Workloads
5. Data

**Estágios**: Traditional → Initial → Advanced → Optimal.

Transversais: Visibility & Analytics, Automation & Orchestration, Governance.

Empresas avaliam onde estão em cada célula e priorizam investimento.

## Anti-Padrões

1. **"Fazemos ZT porque temos MFA"**. MFA é um sinal; ZT é modelo arquitetural completo.
2. **Substituir VPN por ZTNA sem revisitar autorização**. Só mudou o caminho do perímetro, não eliminou.
3. **Ignorar device posture**. Sem avaliar device, identidade sozinha é adivinhação.
4. **Policies hard-coded em IPs**. ZT desacopla policy de localização. Policies por identidade de workload.
5. **Big-bang migration**. Levar 18 meses incrementalmente > 6 meses paralisando operação.
6. **Sem logging/monitoramento**. ZT pressupõe visibilidade; cego = ineficaz.
7. **Confiar em sinais imutáveis (IP corporativo)**. Sinais são inputs para risco, não permissões finais.

## Implementação — Roadmap Prático

1. **Catalogar identidades**: users, services, devices. Federar em IdP.
2. **MFA universal** em logins humanos; preferir phishing-resistant (FIDO2).
3. **Inventariar dispositivos**: MDM, device cert.
4. **Aplicar mTLS em comunicação crítica** (API pública, service mesh interno).
5. **Access proxy** para apps web sensíveis.
6. **Reduzir VPN** conforme migração.
7. **JIT access** para privilegiado.
8. **Microsegmentação** baseada em identidade.
9. **Telemetria em SIEM** + risk engine contextual.
10. **Políticas dinâmicas**, reavaliadas continuamente.

## Ferramentas

| Função | Ferramenta |
|---|---|
| IdP | Okta, Entra ID, Google Workspace |
| MFA forte | YubiKey (FIDO2), Duo |
| MDM | Intune, Jamf, Kandji |
| Device cert | MDM-managed or SCEP |
| Access Proxy / ZTNA | Cloudflare Access, Zscaler, Tailscale, Netskope |
| Service Mesh | Istio, Linkerd, Consul |
| Workload Identity | SPIFFE/SPIRE, cloud IAM |
| EDR | CrowdStrike, SentinelOne, Defender |
| Privileged | CyberArk, HashiCorp Boundary, Teleport |
| Policy | OPA, Cedar (AWS) |

## Recursos

- **NIST SP 800-207** — documento base.
- **Google BeyondCorp papers** (beyondcorp.com).
- **"Zero Trust Networks"** — Gilman & Barth.
- **CISA Zero Trust Maturity Model**.
- **John Kindervag blogs** (cunhou "Zero Trust" na Forrester).

## Princípios

1. **Identidade > rede**. Autorização parte de quem, não de onde.
2. **Verifique sempre**. Acesso perpétuo é anti-padrão.
3. **Contexto dinâmico**. Mesmo usuário, contexto diferente → decisão diferente.
4. **Menor privilégio, always**. Padrão de toda a segurança moderna.
5. **Assume breach**. Contenha lateral movement por design.
6. **Automatize**: políticas declarativas, reavaliação automática.
7. **Visibilidade maior = decisão melhor**. Telemetria é oxigênio.
