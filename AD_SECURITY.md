# Segurança em Active Directory

**Active Directory (AD)** é o diretório de identidade dominante em ambientes corporativos Windows desde o início dos anos 2000. É também o alvo canônico de qualquer red team: comprometer AD = comprometer a organização. Este arquivo cataloga as técnicas centrais que red teams usam e que defensores precisam conhecer para proteger.

## Arquitetura Rápida

- **Domain**: unidade lógica. `corp.example.com`.
- **Forest**: um ou mais domínios compartilhando schema e trust.
- **Domain Controller (DC)**: servidor que hospeda o AD.
- **Objects**: users, computers, groups, GPOs, OUs.
- **Kerberos**: protocolo de autenticação padrão.
- **NTLM**: legado, ainda presente.
- **LDAP**: protocolo de consulta/modificação.

Tudo em AD é um **objeto** com **atributos** e **ACL**. A segurança de AD é, em última análise, auditoria dessas ACLs.

## Kerberos

Três partes: cliente, servidor alvo, KDC (Key Distribution Center, rodando no DC).

Fluxo simplificado:

1. Cliente → KDC: `AS-REQ` (pedido de TGT). Autentica com hash da senha.
2. KDC → cliente: `AS-REP`, contendo **TGT** criptografado com chave do `krbtgt`.
3. Cliente quer acessar serviço. → KDC: `TGS-REQ` com TGT.
4. KDC → cliente: `TGS-REP`, contendo **ST (Service Ticket)** criptografado com a chave do serviço.
5. Cliente → serviço: apresenta ST. Serviço valida.

Peculiaridades que viabilizam ataques:

- Tickets são **blobs criptografados** que o serviço alvo valida localmente sem consultar o DC.
- TGTs são válidos por default ~10h, renováveis por 7 dias.
- Serviço é identificado por **SPN** (Service Principal Name).
- **PAC** (Privilege Attribute Certificate) dentro do ticket carrega grupos do usuário.

## Ataques Clássicos

### Kerberoasting

Qualquer usuário autenticado pode solicitar **ST para qualquer SPN**. O ST é criptografado com a **senha do service account** associada. Atacante coleta STs → bruteforce offline.

- **Alvo**: service accounts (usuários com `servicePrincipalName` setado).
- **Fraco**: contas com senhas ruins. TGS em RC4 (tipo 23) é mais rápido de quebrar que AES.
- **Ferramentas**: `GetUserSPNs.py` (impacket), `Rubeus kerberoast`.
- **Mitigação**: **gMSA** (Group Managed Service Accounts — senha de 120 chars rotacionada pelo AD), remover RC4, monitorar TGS-REQs atípicos.

### AS-REP Roasting

Usuários com "**Do not require Kerberos preauth**" (`DONT_REQ_PREAUTH`) recebem `AS-REP` sem autenticação prévia. `AS-REP` contém valor criptografado com a chave do usuário → bruteforce offline.

- **Alvo**: usuários com a flag setada.
- **Ferramentas**: `GetNPUsers.py`, `Rubeus asreproast`.
- **Mitigação**: desabilitar `DONT_REQ_PREAUTH` em todos os usuários.

### Pass-the-Hash (PtH)

Em NTLM, o hash da senha é suficiente — cliente nunca envia senha em claro. Atacante com hash pode autenticar sem conhecer a senha.

- **Cenário**: mimikatz dumpa hashes de `lsass`; atacante usa pth para acessar outros hosts.
- **Ferramentas**: `mimikatz`, `smbexec.py`, `wmiexec.py`, `psexec.py` da impacket.
- **Mitigação**: desabilitar NTLM onde possível; Credential Guard; LAPS (senhas únicas de admin local por máquina); Protected Users group (restringe NTLM para esses usuários).

### Pass-the-Ticket (PtT)

Mesma ideia mas com tickets Kerberos. Injeta TGT ou ST na sessão atual.

- **Ferramentas**: `Rubeus ptt`, `mimikatz kerberos::ptt`.
- **Mitigação**: difícil; ingredientes são criptograficamente válidos. Defesa é em redução de privilégio (não deixar ticket de admin ficar em memória de workstation) e detecção.

### Golden Ticket

O Santo Graal. `krbtgt` é a conta cuja chave cifra todos os TGTs. Se atacante tem o **hash NT do krbtgt** (obtido via DCSync — abaixo), pode forjar TGT **arbitrário** para **qualquer usuário**, **qualquer grupo**, **10 anos de validade**.

- **Efeito**: admin permanente, invisível (não passa pelo AS-REQ — logs diferentes).
- **Ferramentas**: `mimikatz kerberos::golden`, `ticketer.py` (impacket).
- **Mitigação**: **rotação dupla** da senha do krbtgt (dobra — primeiro ok, segundo invalida tickets anteriores). Não eliminar o ataque; só invalidar tickets existentes.

### Silver Ticket

Forjar **ST** para serviço específico usando hash da conta do serviço. Menos poderoso que golden (um serviço só), mas também menos detectável — não passa pelo DC.

### DCSync

Faz o atacante se passar por DC e pedir replicação. Retorna hashes de **todos** os usuários, incluindo krbtgt.

- **Requisitos**: permissões `Replicating Directory Changes`, `... All`, `... in Filtered Set` sobre o domain head. Frequentemente concedidas a Domain Admins, Enterprise Admins, Built-in Administrators — e erroneamente a outros.
- **Ferramentas**: `mimikatz lsadump::dcsync`, `secretsdump.py`.
- **Mitigação**: auditar quem tem essas permissões. Alerta de alta prioridade em logs de replicação vindos de IP não-DC.

### DCShadow

Registra um DC falso temporário; push mudanças ao AD legítimo; remove. Deixa pouco traço no log.

- **Requisitos**: mesmos do DCSync + direitos específicos.
- **Mitigação**: monitoramento de mudanças em schema/replicação; EDR com visibilidade em registros RPC.

### Unconstrained Delegation

Computador marcado com `TRUSTED_FOR_DELEGATION` recebe, em cada autenticação, o **TGT do usuário que conecta**. Atacante comprometendo tal host extrai TGTs de qualquer usuário que conectou — inclusive admin.

- **Mitigação**: substituir por **Constrained Delegation** (limita a serviços específicos) ou **Resource-Based Constrained Delegation** (RBCD, controle no resource). DCs não devem ser unconstrained.
- **Ataque com RBCD**: `CVE-2022-26923` (Certifried) permitia abuse em certificados; mas RBCD ainda é útil para atacantes com `GenericAll` em computer object.

### Kerberos Relaying / PrinterBug / PetitPotam

Coagir DC ou outro host a autenticar no atacante.

- **PrinterBug** (`SpoolSample`): RPC do spooler força outgoing auth.
- **PetitPotam** (MS-EFSRPC): não exige autenticação pré-existente.
- **DFSCoerce**, **ShadowCoerce**: similares.
- Combinado com **NTLM relay** a ADCS (AD Certificate Services) com ESC8 → pwn.

### AD Certificate Services (AD CS) — ESC1-ESC15

Pesquisa de SpecterOps (Certified Pre-Owned). Categorias de misconfiguration em ADCS:

- **ESC1**: template aceita SAN arbitrária + Authenticated Users podem enroll → forjar cert como qualquer usuário.
- **ESC2**: "any purpose" EKU.
- **ESC3**: enrollment agent abusado.
- **ESC6**: `EDITF_ATTRIBUTESUBJECTALTNAME2` no CA.
- **ESC8**: HTTP CertSrv vulnerável a NTLM relay → cert de DC → DCSync.
- **ESC9-15**: descobertos depois, novas variantes.

- **Ferramentas**: `Certipy`, `Certify`, `Ghostpack`.
- **Mitigação**: templates restritivos, PKINIT hardening, desabilitar HTTP auth em CA Web Enrollment, auditoria constante.

### LLMNR / NBT-NS / mDNS Poisoning

Quando um host resolve nome que não está em DNS, cai em broadcast. Atacante responde "sou eu" → coleta hash NTLMv2 da tentativa de autenticação.

- **Ferramentas**: `Responder`.
- **Mitigação**: desabilitar LLMNR/NBT-NS via GPO. DNS correto para todos os nomes usados.

## BloodHound

Ferramenta canônica de ofensiva/defensiva em AD. Coleta dados (via `SharpHound`, `AzureHound`, `BloodHound.py`) e mostra **grafo de caminhos de ataque** — "quem pode, via sequência de permissões, chegar a Domain Admin?".

- **Queries built-in**: shortest path to DA, Kerberoastable users, unconstrained delegations.
- **CypherQL** permite queries customizadas.
- **BloodHound Community Edition** (Bloodhound.io) é plataforma SaaS/self-hosted moderna.

Defensores: rodar sobre próprio AD para **identificar e quebrar** caminhos. É uma das ferramentas defensivas mais valiosas disponíveis.

## Tiering

Modelo Microsoft para segmentar admin:

- **Tier 0**: controla identidades/KDC (DCs, ADCS, servidores de identidade). Admin tier 0 nunca loga em outros tiers.
- **Tier 1**: servidores de aplicação/dados.
- **Tier 2**: workstations e usuários.

Regra inviolável: **credenciais de tier superior não tocam sistemas de tier inferior**. Senão, comprometer workstation rende DA.

## LAPS

**Local Administrator Password Solution**. Cada máquina tem senha de admin local **única e aleatória**, rotacionada, armazenada em AD com ACL controlada. Mata pass-the-hash em cascata (senha local diferente por host).

Versão moderna: **Windows LAPS** (2023+), built-in; rotaciona senhas de contas do domínio e locais, com suporte a criptografia e cloud.

## Monitoramento e Detecção

Eventos de interesse (já com Sysmon / logs de segurança):

- **4624 / 4625** — logon sucesso/falha (especialmente tipo 3 lateral, tipo 10 RDP).
- **4768, 4769** — TGT / ST requests (Kerberos). TGS com RC4 a serviços high-value = red flag.
- **4672** — logon com privilégios especiais.
- **4662** — operação em AD object (DCSync deixa trilha aqui).
- **4776** — NTLM validation.
- **4688** — process creation (com command line habilitado).
- **5136, 5137** — LDAP modify/add.
- **4738, 4720** — user account modified/created.

Ferramentas:

- **ATA / Defender for Identity** (Microsoft): detecção comportamental em AD. Pega Golden Ticket, DCSync, pass-the-ticket, reconhaissance.
- **Sysmon + EDR** consumidos em SIEM.
- **Sigma rules** específicas de AD.

## Hardening Baseline

Checklist obrigatório:

1. **Disable NTLMv1**; restringir NTLMv2 a onde for estritamente necessário.
2. **Disable LLMNR e NBT-NS** via GPO.
3. **LAPS** em 100% dos endpoints.
4. **Protected Users group** para admins + restrição de credencial.
5. **Credential Guard** (Windows 10+ / Server 2016+) em endpoints sensíveis.
6. **Rotação dupla de krbtgt** periódica (~1x ao ano; após incidente).
7. **Remover** contas inativas, SPNs obsoletos, delegation unconstrained.
8. **Auditoria de ACL** do domain head e objetos high-privilege.
9. **Separação de contas admin vs user** por administrador (conta `.adm`).
10. **Tiering** estrito; PAW (Privileged Access Workstation) para Tier 0.
11. **MFA** em logins privilegiados e expostos.
12. **Backup do AD offline** e testado — ransomware pode criptografar DCs.
13. **Azure AD / Entra ID** se híbrido — Azure AD Connect é vetor crítico; atenção ao `MSOL_` account e permissões de sync.
14. **Reduzir domain admins** ao mínimo absoluto (sub-5 idealmente).
15. **ADCS**: templates auditados; desabilitar Web Enrollment HTTP; habilitar EPA para HTTPS.

## Ferramentas de Referência

| Uso | Ferramenta |
|---|---|
| Enum/auditoria | BloodHound, PingCastle, Purple Knight, ADRecon |
| Exploit | Impacket, Rubeus, Mimikatz, Certify, Certipy |
| Framework | CrackMapExec / NetExec |
| Relay | ntlmrelayx.py |
| Coercion | PetitPotam, SpoolSample, Coercer |
| Detection | Defender for Identity, Elastic/Splunk + Sigma |
| Audit permission | BloodHound CE, ADACLScanner |

## Recursos

- **SpecterOps blog** — pesquisas canônicas.
- **adsecurity.org** (Sean Metcalf) — referência compreensiva.
- **"PowerShell for Pentesters"**, **"The Hacker Playbook"**, curso **"Certified Red Team Operator"**.
- SANS FOR508/SEC565.
- **Microsoft Security Compliance Toolkit** com baselines.

## Princípios

1. **Tier 0 é sagrado**. DCs e identidade crítica vivem isolados; admins tier 0 têm PAW dedicado.
2. **Assuma comprometimento de workstation**. Projeto de AD deve tolerar isso sem escalar a DA.
3. **Reduza superfície de ataque**: contas obsoletas, SPNs não usados, delegations amplas, ACLs overpermissive.
4. **BloodHound periódico como auditoria**. Quebre caminhos antes que atacante descubra.
5. **Kerberos monitorado**. TGS-REQs atípicos, tickets AS-REP sem preauth, DCSync — todos têm padrão detectável.
6. **Rotacione krbtgt**. Procedimentalmente, anualmente.
7. **MFA em escalação**. Admin login sem MFA é receita de desastre.
