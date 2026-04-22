# Segurança em Cloud

Segurança em cloud compartilha fundamentos com segurança tradicional mas muda os controles: perímetro deixa de ser físico; IAM vira o plano de controle dominante; erros de configuração substituem muitos vetores clássicos como principal causa de incidente. A frase do Gartner resumiu a década: *"Through 2025, 99% of cloud security failures will be the customer's fault"*.

## Modelo de Responsabilidade Compartilhada

Fundamento conceitual: o provedor é responsável pela segurança **da** cloud; o cliente, pela segurança **na** cloud.

| Camada | IaaS (EC2, GCE) | PaaS (RDS, Cloud Run) | SaaS (Workspace, M365) |
|---|---|---|---|
| Facilities | Cloud | Cloud | Cloud |
| Hardware | Cloud | Cloud | Cloud |
| Virtualização | Cloud | Cloud | Cloud |
| OS | **Cliente** | Cloud | Cloud |
| Runtime | **Cliente** | Cloud | Cloud |
| Dados | **Cliente** | **Cliente** | **Cliente** |
| IAM/Acesso | **Cliente** | **Cliente** | **Cliente** |
| Config | **Cliente** | **Cliente** | **Cliente** |

Em qualquer modelo, **dados, IAM e configuração sempre são do cliente**. A maioria dos incidentes vive aí.

## IAM — o Plano de Controle

Em cloud, identidade é o perímetro. Quase todo ataque moderno passa por credencial/permissão.

### Conceitos

- **Principal**: quem age (user, role, service account, workload).
- **Action / Permission**: o que pode fazer (`s3:GetObject`).
- **Resource**: em quê.
- **Policy**: documento que combina o acima.
- **Effect**: allow / deny.

### Boas práticas

1. **Least privilege sempre**. Começar com nada; adicionar só o necessário. Usar **access analyzer** (AWS) / **policy troubleshooter** (GCP) para detectar sobre-permissionamento.
2. **Roles, não long-lived keys**. Workloads assumem roles com short-lived credentials (STS). Usuários via SSO + role assumption.
3. **MFA obrigatório** para consoles e operações sensíveis.
4. **Rotação** de chaves residuais; idealmente eliminar.
5. **Conditions** em policies: restringir por IP, MFA, time, source VPC.
6. **Separação por conta/projeto**: contas AWS separadas por ambiente (dev/staging/prod) e equipe. Explosion radius contido.
7. **Organizações / SCPs (AWS)**, **Org Policies (GCP)**, **Management Groups (Azure)**: guardrails que nem conta-admin pode burlar.

### Anti-padrões frequentes

- **Usuário IAM com access key em código**. Solução: roles + IMDS; GitHub Actions OIDC; secret manager.
- **Policy com `Action: *` ou `Resource: *`**: quase sempre excessivo.
- **`AdministratorAccess` como default**.
- **Service account em JSON commitado em repositório**.
- **Trust policies muito permissivas**: `Principal: *` sem condição.
- **Permissões escalonáveis**: `iam:CreateRole` + `iam:AttachRolePolicy` = caminho para admin.

### Privilege Escalation Paths

Ferramentas mapeiam caminhos onde permissões aparentemente benignas se combinam em admin. **Pacu** (AWS), **PurplePanda**, **Stormspotter** (Azure), **ScoutSuite**, **Prowler**, **CloudSploit**.

## Storage Exposto

### S3 / GCS / Blob Storage

Vazamentos de buckets públicos são o incidente mais clichê de cloud:

- **Bloquear acesso público** no nível da conta (AWS "Block Public Access" em 4 níveis).
- **Políticas de bucket** explícitas; sem `Principal: *` sem condição.
- **Versionamento + MFA Delete** em buckets críticos.
- **Encryption at rest**: SSE-S3 mínimo, SSE-KMS preferível (controle por CMK). Rotação de CMK.
- **Pre-signed URLs** com TTL curto para compartilhar temporariamente.
- **Logging** (S3 Access Logs, CloudTrail S3 data events).

Ferramentas que encontram buckets públicos: **grayhatwarfare.com**, `bucket-stream`, `awesome-s3-buckets`.

## Metadata Service

Serviço interno que fornece credenciais/config para instâncias. SSRF contra ele é catastrófico.

### AWS IMDS

- **IMDSv1**: GET simples. Vulnerável a SSRF. Migre.
- **IMDSv2**: exige `PUT` para obter token + header com token em requests. Evita SSRF naive.
- **Configuração**: force IMDSv2 (`HttpTokens: required`), reduza hop limit (`HttpPutResponseHopLimit: 1`) para impedir container children.

### GCP / Azure

- GCP `metadata.google.internal` exige header `Metadata-Flavor: Google` — já resiste a SSRF simples.
- Azure IMDS `169.254.169.254` exige header `Metadata: true`.

Em qualquer caso, bloqueie egress para `169.254.0.0/16` de workloads que não precisam.

## Segredos

Nunca em repositórios, env vars em plain, ou config files.

- **AWS Secrets Manager, Parameter Store**.
- **GCP Secret Manager**.
- **Azure Key Vault**.
- **HashiCorp Vault** (multi-cloud).
- **External Secrets Operator** em K8s.

**Rotação automatizada** para DB creds. Auditar acesso (quem leu qual segredo quando).

Secret scanning: **gitleaks, trufflehog, detect-secrets, GitHub Advanced Security** em CI.

## Rede

### VPC/VNet basics

- **Subnets públicas vs privadas**.
- **Security Groups / NSGs**: firewall por instância. Stateful.
- **NACLs** (AWS): stateless; menos usado, para defesa extra em subnet.
- **PrivateLink / Private Service Connect**: acesso a serviços sem sair para internet.
- **VPC endpoints**: acesso a S3/DynamoDB sem internet. Reduz ataque de SSRF + data exfil em muitos casos.

### Perímetro

- **Egress controlado**: proxy ou NAT gateway com allowlist. Exfiltração depende de egress.
- **WAF** (CloudFront, Cloud Armor, Azure Front Door): regras gerenciadas (OWASP core), custom rules, rate limit.
- **DDoS protection**: AWS Shield (Standard free, Advanced pago), Cloudflare, Google Cloud Armor.

### Zero Trust

Ver `ZERO_TRUST.md`. mTLS entre services, identity-aware proxies, JIT access. BeyondCorp como blueprint.

## Compute

### EC2 / VMs

- **AMIs/images** endurecidas (CIS benchmarks). Ou imagens minimizadas (distroless).
- **Patching automatizado** (SSM Patch Manager, Azure Update Manager).
- **Desabilitar SSH pela internet**: Bastion host, SSM Session Manager, IAP (GCP) — acesso sem IP público.
- **Encryption de EBS/disks** por padrão.
- **Instance profile** = role assumida — sem access key na máquina.

### Serverless (Lambda, Functions)

- **Role por função** (não um genérico); least privilege.
- **Timeout curto**: limita blast radius se comprometida.
- **Ephemeral storage**: `/tmp` é persistente entre invocações do mesmo container — cuidado com segredos.
- **Dependências**: mesma disciplina que qualquer código (SCA).
- **VPC attach** apenas quando necessário — adiciona latência.

### Containers / K8s

Ver `CONTAINER_SECURITY.md`. Tópicos cloud-específicos:

- **IRSA / Workload Identity**: pods assumem IAM role via OIDC. Elimina access keys em pods.
- **Control plane**: API server exposto? RBAC? Audit logs?
- **Managed nodes** (EKS, GKE, AKS) automatizam patches mas ainda exigem hardening de pods.
- **Service mesh** (Istio, Linkerd, AWS App Mesh): mTLS, authz entre services.
- **GKE Autopilot, EKS Fargate**: reduzem node attack surface, terceirizam para cloud.

## Data

- **Encryption at rest**: default em storage moderno, mas confirme. Com CMK gerenciada pelo cliente (CMEK) para compliance.
- **Encryption in transit**: TLS em tudo; mTLS entre services.
- **Classificação**: sabe quais dados são sensíveis? **Macie** (AWS), **DLP API** (GCP), **Purview** (Azure).
- **Retenção**: ver `COMPLIANCE.md`; regras legais (LGPD, GDPR).
- **Backup**: imutável (Object Lock / Blob immutability) contra ransomware em cloud.
- **Cross-region replication** para disaster recovery.

## Logging e Monitoramento

- **CloudTrail** / **Audit Logs** / **Activity Logs**: quem fez o quê.
- **VPC Flow Logs** / **VPC Flow Logs** / **NSG Flow Logs**: tráfego.
- **GuardDuty, Security Command Center, Microsoft Defender for Cloud**: detecção gerenciada pelo provedor.
- **CSPM (Cloud Security Posture Management)**: Wiz, Prisma Cloud, Orca, Lacework. Varredura contínua de misconfig.
- **CWPP (Cloud Workload Protection)**: runtime em workloads.
- **CNAPP** = CSPM + CWPP + CIEM etc. Converge o mercado.

Agregar em SIEM central. Ver `DETECTION_ENGINEERING.md`.

## Ataques Cloud Comuns

1. **Credential leak em GitHub** → acesso imediato. Bots varrem commits.
2. **SSRF → metadata → IAM creds** → pivot.
3. **Bucket público** → leak de PII.
4. **Takeover de DNS órfão** (dangling CNAME → subdomain takeover).
5. **Over-permissioned role** + comprometimento de workload → amplificação.
6. **Privilege escalation via cadeia de permissões IAM benignas**.
7. **Lambda/Function com data source envenenada** → RCE serverless.
8. **K8s API exposto** (6443/8443) com anonymous enabled.
9. **AMI pública com segredos** (prática desleixada).
10. **Lateral via Assume Role** entre contas (trust policies amplas).

## CIS Benchmarks

Configurações mínimas audit-friendly para AWS, Azure, GCP, K8s. Úteis como referência; seguir cegamente gera fricção — adapte.

## Ferramentas

- **Scanning e posture**: Prowler, ScoutSuite, CloudSploit, Wiz, Prisma, Orca.
- **Exploitation / assessment**: Pacu (AWS), MicroBurst (Azure), GCP-IAM-Privilege-Escalation, ROADtools.
- **IaC scan**: Checkov, tfsec, Terrascan, Snyk IaC — validar Terraform/CloudFormation antes de deploy.
- **SAST cloud**: Semgrep com regras cloud, regulamentação por provider.
- **Secret scanning**: GitLeaks, TruffleHog, GitHub secret scanning.

## Boas Práticas para Ambiente

1. **Multiple contas/projetos**: ambiente, equipe. Consolidado via Organizations.
2. **Guard rails** via SCP / Org Policies: ninguém, nem admin, pode desabilitar CloudTrail.
3. **IaC versionado**: Terraform/Pulumi/CDK. Mudanças via PR com code review. Nada de "cliquei no console".
4. **CI com scans**: IaC scan + secret scan + SCA + SAST.
5. **Drift detection**: config muda fora do IaC → alerta.
6. **Centralized logging**: todas as contas → bucket/acc central imutável.
7. **Break-glass**: conta/role de emergência auditada, nunca usada em rotina.
8. **Documentação de ownership**: toda conta/projeto tem dono documentado.

## Princípios

1. **Identidade é o perímetro**. Todo o resto decorre de IAM bem-feito.
2. **Least privilege, realmente**. `*:*` é um cheiro, não uma permissão.
3. **Automatize guard rails**. Humano errará; máquina não.
4. **Crypto at rest + in transit**. Sem exceções em dado sensível.
5. **Assume breach** no modelo de rede: Zero Trust.
6. **Logs centralizados e imutáveis**. Sem logs, sem detecção, sem IR, sem forense.
7. **Teste regularmente**. CSPM contínuo + pentest de cloud anual.
