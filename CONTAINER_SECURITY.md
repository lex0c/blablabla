# Segurança em Containers

Containers **não são VMs** — compartilham kernel com o host. Isso muda modelo de ameaça radicalmente: uma vulnerabilidade no kernel ou um escape de container leva ao host. A segurança de containers combina três frentes: **imagem** (o que é distribuído), **runtime** (o que executa), **orquestração** (como e onde).

Complementa `APP_SANDBOX.md`, `CLOUD_SECURITY.md` e `SUPPLY_CHAIN.md`.

## Modelo Técnico

Container em Linux é um processo com **namespaces** (pid, mount, net, uts, ipc, user, cgroup) isolando visibilidade e **cgroups** limitando recursos. Compartilha **kernel** com outros containers e com host.

- **Namespaces** dão ilusão de sistema próprio.
- **Cgroups** impõem limites.
- **Capabilities** (veremos) restringem o que o processo pode fazer como root.
- **Seccomp** filtra syscalls.
- **LSM (AppArmor/SELinux)** aplica MAC.

Quando ausentes ou mal configurados, container é privilégio com verniz.

## Imagens

### Base image

- **Distroless** (Google) ou **scratch**: mínimo possível. Sem shell, sem package manager — mata muitas TTPs de pós-exploração.
- **Alpine**: pequeno; usa musl (diferenças sutis vs glibc; problemas históricos com DNS e timezones).
- **Ubuntu/Debian slim**: maior, familiar.
- **Chainguard Images**: moderno, zero-CVE como princípio de design.

### Princípios

1. **Menor possível**: cada binário extra é superfície.
2. **Multi-stage build**: compile em imagem grande, copie binário para runtime mínima.
3. **Pin de tags imutáveis**: `alpine:3.19` ok, `alpine:latest` nunca. Idealmente pin por digest (`alpine@sha256:abc...`).
4. **Sem credenciais na imagem**: nunca `COPY id_rsa .` ou `.env` com secret. Layer é imutável; camada anterior não some com `rm`.
5. **Usuário não-root**: `USER 1000` em Dockerfile. Root em container continua sendo privilégio.
6. **Read-only filesystem** em runtime, quando possível.

### Scanning

Vulnerabilities em pacotes e bibliotecas:

- **Trivy** (Aqua) — open, rápido, excelente default.
- **Grype** (Anchore).
- **Snyk, Docker Scout, Sysdig, Prisma, Wiz**: comerciais.
- Scan em CI antes do push; scan contínuo no registry (imagens envelhecem).

### SBOM

**Software Bill of Materials**: inventário do que a imagem contém. Formatos: **SPDX, CycloneDX**. Ferramentas: `syft`, `trivy sbom`.

SBOM habilita resposta rápida a CVE novo ("tenho log4j afetado?"). Regulações começam a exigir (EO 14028, EUA).

### Assinatura de Imagens

- **Cosign** / **Sigstore**: assina artefatos, verificável sem PKI própria.
- **Notation** / **Notary v2**: padrão OCI.
- **Admission controllers** em K8s bloqueiam imagens não assinadas.

Fecha vetor de supply chain (ver `SUPPLY_CHAIN.md`).

## Runtime

### Capabilities

Root Linux foi dividido em ~40 capabilities. Alguns exemplos:

- `CAP_NET_ADMIN` — config de rede.
- `CAP_SYS_ADMIN` — "o novo root"; concede muito.
- `CAP_NET_BIND_SERVICE` — bind em porta <1024.
- `CAP_SYS_PTRACE` — debugar outros processos.

Docker default: conjunto reduzido. Em K8s pod spec: `securityContext.capabilities.drop: ["ALL"]` e adicionar só o necessário.

### Seccomp

Filtra syscalls que o container pode fazer. Docker tem profile default que bloqueia ~60 syscalls perigosas. K8s: `RuntimeDefault` no `seccompProfile`. Customizar para apps críticos reduz ainda mais.

### AppArmor / SELinux

MAC (Mandatory Access Control) complementa DAC. Container tem perfil dedicado que restringe syscalls, filesystem, network. Red Hat/CentOS/Fedora: SELinux ativo por default em CRI-O; muitos esquecem que ajuda.

### User Namespaces

Mapear root do container para UID não-privilegiado no host. Se container escapa, não é root do host. Docker: `--userns-remap`. K8s 1.30+: `hostUsers: false`.

### Rootless

Executar daemon e containers como usuário comum. **Podman** é rootless by default; Docker tem modo rootless. Reduz drasticamente impacto de bugs no daemon.

### Read-only root filesystem

`readOnlyRootFilesystem: true` em pod spec. Escritas só em volumes explícitos. Trava muitas TTPs (dropar binário, editar configs).

## Flags Perigosas

Nunca (salvo casos específicos bem-justificados):

- **`--privileged`**: dá quase todas as capabilities + acesso a devices. É "quase root no host".
- **`--pid=host`**: compartilha namespace de pid — vê processos do host; pode matar.
- **`--net=host`**: sem isolamento de rede.
- **Mount de `/` do host**: acesso total ao filesystem.
- **Mount do Docker socket** (`/var/run/docker.sock`): um pod pode criar outros containers privilegiados no host. Equivale a root.
- **`hostPath` em K8s**: com paths perigosos (`/`, `/etc`, `/var/run/docker.sock`, `/proc`).

## Escape Techniques

Vocabulário dos adversários — saber para defender:

1. **Privileged container com device access**: acesso direto a `/dev/*` do host.
2. **Kernel vulnerability**: bug em syscall handler, eBPF, namespace. CVE-2022-0185 (legacy_parse_param), CVE-2022-0492 (cgroup release_agent), DirtyPipe (CVE-2022-0847).
3. **CAP_SYS_ADMIN** → abuse de `mount`, cgroups release_agent, core_pattern.
4. **Docker socket mount** → criar container privilegiado.
5. **Shared PID + ptrace** → injeção em processo do host.
6. **runc vulnerabilities**: CVE-2019-5736 (sobrescrever runc binary do host).

Defesa: camadas — seccomp + apparmor + user ns + patch kernel regularmente.

## Kubernetes Security

### Pod Security

- **Pod Security Admission** (PSA): 3 níveis — `privileged`, `baseline`, `restricted`. Aplicar `restricted` em namespaces de produção.
- **Pod Security Policies (PSP)**: depreciado em K8s 1.25; substituído por PSA + OPA Gatekeeper/Kyverno.

### RBAC

- Roles e ClusterRoles com verbos e resources específicos.
- Service accounts por workload (não usar `default`).
- Evite `cluster-admin` para workloads.
- Auditar com `rbac-lookup`, `kube-hunter`, **Krane**.

### Network Policies

Kubernetes por default: tudo fala com tudo. Network Policies restringem por selector/namespace/IP. Exige CNI que suporte (Calico, Cilium, Antrea). Sem elas, lateral movement é trivial.

### Service Mesh

**Istio, Linkerd, Cilium Service Mesh**: mTLS automático, authz entre services, telemetria. Zero trust no mesh. Ver `ZERO_TRUST.md`.

### Admission Controllers

Policy-as-code:

- **OPA Gatekeeper** — Rego, expressivo.
- **Kyverno** — YAML nativo, mais acessível.
- **Validation e Mutation**: bloquear imagens sem assinatura, exigir resource limits, proibir `hostPath`, forçar labels.

### Secrets

- Secrets do K8s são **base64, não criptografados** por padrão.
- Habilitar **encryption at rest** no etcd com KMS externa.
- **External Secrets Operator** + Secret Manager (ver `CLOUD_SECURITY.md`).
- **Sealed Secrets** (Bitnami) para GitOps.
- Nunca `kubectl create secret` via CLI com histórico.

### Runtime Security

- **Falco** (Sysdig) — eBPF/kernel-based detection de comportamento suspeito em containers.
- **Tetragon** (Cilium) — similar, mais integrado a Cilium.
- **Tracee** (Aqua).

Detectam: shell spawn, conexão suspeita, edição de bins, escape attempts.

### Auditing

- **Audit log do API Server** — habilitar com policy adequada. Ingerir em SIEM.
- **Kubernetes Audit Benchmark** (CIS) — configuração baseline.

### Outros vetores

- **Kubelet API** (10250): não expor sem auth.
- **etcd**: acesso direto = acesso total. Encryption + auth + TLS + firewall.
- **API Server**: TLS + authn forte + RBAC.
- **Dashboard**: legado; raramente necessário e frequentemente mal-configurado.

## Supply Chain do Container

Ver `SUPPLY_CHAIN.md`. Específico a containers:

1. **Registry comprometido**: use registry privado com auth; verifique assinatura.
2. **Typosquatting em base images**: `nginxx` em vez de `nginx`.
3. **Build pipeline comprometido**: runner com credenciais demais; injeção no pipeline.
4. **Dependências transitivas em imagens**: apt/apk/pip trazem mundo.
5. **Reprodutibilidade**: build reprodutível + SBOM + provenance (SLSA).

SLSA (Supply-chain Levels for Software Artifacts) define 4 níveis de integridade de build.

## Compliance

- **CIS Docker Benchmark, CIS Kubernetes Benchmark**: checks auditáveis. `kube-bench` automatiza.
- **NSA Kubernetes Hardening Guide**: leitura obrigatória.
- **NIST SP 800-190** (Application Container Security Guide).

## Ferramentas

- **Scanning**: Trivy, Grype, Clair, Snyk, Docker Scout.
- **SBOM**: Syft.
- **Assinatura**: Cosign, Notation.
- **Runtime**: Falco, Tetragon, Tracee, Sysdig Secure, Aqua.
- **Policy**: OPA Gatekeeper, Kyverno.
- **K8s hardening**: kube-bench, kube-hunter, kubeaudit, Polaris, Krane.
- **Secrets**: ESO + Vault / AWS SM / GCP SM / Azure KV.
- **Service mesh**: Istio, Linkerd, Cilium.

## Anti-Padrões

1. **Container "seguro porque é container"**. Container não é sandbox por si só.
2. **Root em container**. "É só root no container" vira root no host em bom número de CVEs.
3. **Base image enorme**. Mais superfície, mais CVE, mais ruído.
4. **Construção na própria máquina do dev**. CI reprodutível + assinada.
5. **Secrets em ENV via Dockerfile**. Vazam em history/inspect.
6. **Sem Network Policy**. Lateral movement trivial.
7. **PSA permissive em prod**. `restricted` deveria ser default.
8. **Docker socket mountado**. Sinal vermelho permanente.

## Princípios

1. **Defense in depth**: imagem mínima + runtime restrito + policy + runtime detection.
2. **Tudo em código**: Dockerfile, K8s manifests, policies — versionados, reviewed.
3. **Imutabilidade**: container não muda em runtime; release é nova imagem.
4. **Least privilege**: capabilities drop ALL; readonly root; no host mounts.
5. **Zero trust entre workloads**: mTLS + authz, não confie em rede.
6. **Supply chain verificada**: assinatura + SBOM + registry privado.
7. **Runtime detection**: Falco/Tetragon captura o que imagem/policy não preveniu.
