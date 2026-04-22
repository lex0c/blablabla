# AppSec no SDLC (DevSecOps)

**AppSec** (Application Security) inserida no **SDLC** (Software Development Life Cycle) é o que se chama modernamente de **DevSecOps**: trazer segurança para cada fase do desenvolvimento em vez de delegar a uma equipe que revisa tudo no final. O objetivo é reduzir tempo até a correção de vulnerabilidades e eliminar classes inteiras por construção — não tapar bugs sem fim.

Complementa `SOFTWARE_SECURITY_GUIDELINE.md`, `WEB_SECURITY.md`, `THREAT_MODELING.md`.

## Shift-Left

Frase-chave do campo: "mover segurança para a esquerda do pipeline" — da produção para o PR, do PR para o design, do design para o treinamento. O custo de corrigir cresce ordens de magnitude conforme descoberta acontece mais tarde:

- Design: barato, só muda requisito.
- Dev: refatoração local.
- QA: bug ticket, novo ciclo.
- Produção: incidente, talvez vazamento, talvez regulador.
- Após vazamento: litígio, multa, reputação.

Shift-left não elimina o lado direito; reduz a carga que chega lá.

## Testes Automatizados de Segurança

### SAST (Static Application Security Testing)

Análise **estática** de código fonte ou bytecode. Procura padrões vulneráveis: concatenação de SQL, `eval(input)`, deserialização insegura, crypto fraca.

- **Pros**: early (pre-commit possível), cobre todos os caminhos do código, bom para linguagens tipadas.
- **Contras**: falsos positivos abundantes em configs ruins; não vê comportamento dinâmico; pode ignorar contexto (taint chega a sink ou não?).
- **Ferramentas**: **Semgrep** (regras em sintaxe simples), **CodeQL** (GitHub, query language poderosa), **SonarQube**, **Checkmarx**, **Snyk Code**, **Fortify**, **Bandit** (Python), **gosec** (Go), **brakeman** (Rails), **eslint-plugin-security** (JS).
- **Regra diferencial**: adote regras específicas do projeto. As padrão pegam o óbvio.

### DAST (Dynamic Application Security Testing)

Teste **dinâmico** — sistema rodando. Fuzz e scan de APIs/endpoints externos.

- **Pros**: encontra bugs que aparecem em runtime; menos FP que SAST; não depende de linguagem.
- **Contras**: precisa ambiente, cobre só caminhos testados, lento.
- **Ferramentas**: **OWASP ZAP**, **Burp Suite Enterprise**, **Nuclei**, **Wapiti**, **Arachni**.

### IAST (Interactive Application Security Testing)

Agent em runtime instrumenta a aplicação e observa fluxos durante testes funcionais.

- **Pros**: combina visibilidade estática (caminho) com dinâmica (execução real); baixo FP.
- **Contras**: acoplado a linguagem; performance em staging.
- **Ferramentas**: **Contrast Security**, **Seeker**, **Checkmarx CxIAST**.

### RASP (Runtime Application Self-Protection)

Similar a IAST mas em produção, **bloqueando** ataques. Polêmico — alguns veem como WAF glorificado.

### SCA (Software Composition Analysis)

Identifica dependências e verifica **vulnerabilidades conhecidas** (CVEs).

- **Pros**: pega a maior parte dos issues reais em apps modernos (dependências).
- **Contras**: volume de ruído; CVE nem sempre atinge sua path; "vulnerabilidade sem exploit" vs crítica.
- **Ferramentas**: **Dependabot** (GitHub), **Snyk Open Source**, **Trivy**, **Grype**, **OSV-Scanner**, **Renovate**.
- **Reachability analysis**: ferramentas modernas (Snyk, Semgrep) tentam determinar se código efetivamente chama a função vulnerável — corta ruído drasticamente.

### Secret Scanning

Detecta chaves, tokens, senhas em código / histórico Git.

- **Ferramentas**: **GitLeaks**, **TruffleHog**, **GitHub Secret Scanning** (builtin, gratuito em repos públicos; com push protection).
- **Cobertura**: pré-commit hooks, CI, scan histórico completo.
- **Rotação imediata ao achar**. Segredo em Git público se considera queimado em minutos — bots varrem commits em tempo real.

### Fuzzing

Ver `TESTS.md` e `BINARY_EXPLOITATION.md`. Em AppSec SDLC:

- **Coverage-guided** em código nativo (AFL++, libFuzzer).
- **Property-based** em funções puras / parsers.
- **API fuzzing**: RESTler, KiteRunner.
- **Continuous fuzzing**: OSS-Fuzz (para projetos open-source críticos), ClusterFuzz.

### IaC Scanning

Infraestrutura como código (Terraform, CloudFormation, Helm charts, K8s manifests) também tem misconfig. Scannear antes de aplicar.

- **Checkov**, **tfsec**, **KICS**, **Terrascan**, **kube-linter**, **kubesec**, **Snyk IaC**.
- Bloqueio em CI: PR com S3 bucket público é rejeitado.

### Container Scanning

Ver `CONTAINER_SECURITY.md`. Imagens em CI e no registry.

## Pipeline Integrado

Um pipeline moderno típico:

```
┌─ IDE ──────────┐   ┌─ Pre-commit ──┐   ┌─ CI ────────────────┐   ┌─ Runtime ─────┐
│ Linters        │   │ Secret scan   │   │ SAST                │   │ RASP          │
│ SAST local     │ → │ SAST rápido   │ → │ SCA                 │ → │ WAF           │
│ Typing         │   │ Unit tests    │   │ IaC scan            │   │ CSPM/CWPP     │
│                │   │               │   │ Container scan      │   │ DAST em prod  │
│                │   │               │   │ Unit + integration  │   │ (cautelosa)   │
│                │   │               │   │ DAST em staging     │   │               │
└────────────────┘   └───────────────┘   └─────────────────────┘   └───────────────┘
```

**Princípios**:

1. **Feedback rápido** onde possível. IDE > pre-commit > PR > main merge > staging > prod.
2. **Não bloquear builds por FPs conhecidos**. Tune regras; allowlist documentada.
3. **Triagem automatizada**: mesmo critério: severidade + exposição + reachability → ação.
4. **Severidade define bloqueio**: critical/high bloqueiam; medium/low viram issue.

## Gates vs Observability

Dois estilos:

- **Gate**: CI bloqueia se check falha. Forte; pode criar atrito.
- **Observability**: log/issue criado sem bloquear. Fraco; fácil ignorar.

Geralmente: crítico = gate, resto = observability com SLA de correção. Equipe aceita gates se ferramentas forem precisas e regras justificadas.

## Secure SDLC — Microsoft SDL

Referência clássica: **Security Development Lifecycle** da Microsoft pós-Blaster worm (2002). Fases:

1. **Training**: desenvolvedores treinados em appsec.
2. **Requirements**: definir requisitos de segurança.
3. **Design**: threat modeling (ver `THREAT_MODELING.md`).
4. **Implementation**: SAST, padrões de codificação.
5. **Verification**: DAST, fuzzing, pen test.
6. **Release**: revisão final, plano de resposta.
7. **Response**: processo de gestão de incidente e patches.

Modelos similares: **NIST SSDF** (Secure Software Development Framework), **OWASP SAMM** (Software Assurance Maturity Model), **BSIMM** (benchmarking).

## SAMM / BSIMM / SSDF

- **SAMM**: modelo de maturidade; medir onde a org está e para onde ir.
- **BSIMM**: descritivo (não prescritivo); baseado em observações de empresas reais.
- **SSDF (NIST SP 800-218)**: referência para governo EUA, virando padrão pós-EO 14028.

## Secrets Management no SDLC

- Separação entre **código** (git) e **segredos** (secret manager).
- **CI tokens** de vida curta, escopo limitado.
- **OIDC** entre GitHub Actions/GitLab CI e cloud (sem storing de access keys).
- **Sealed/encrypted secrets** para K8s GitOps.
- **Rotation** automatizada.

## Software Supply Chain

Ver `SUPPLY_CHAIN.md`. Em pipeline:

- **SBOM** (Syft) gerada em build.
- **Assinatura** (Cosign/Sigstore) de artefatos.
- **Provenance** (SLSA framework) — metadados sobre como o artefato foi construído.
- **Verified base images** (Chainguard, Wolfi).
- **Dependency pinning** por hash; `Renovate` ou `Dependabot` para updates controlados.
- **Builds reprodutíveis** (ideal, difícil).

## Vulnerability Management

- **Backlog priorizado**: severidade + exploitabilidade + asset criticality. CVSS + **EPSS** (Exploit Prediction Scoring System) + **KEV** (CISA Known Exploited Vulnerabilities).
- **SLA**:
  - Critical: patch em 48-72h.
  - High: 7-14 dias.
  - Medium: 30-90 dias.
  - Low: backlog técnico.
- **Exceções documentadas**: por quê não corrigir; prazo de revisão; mitigações compensatórias.

## Educação e Champions

- **Training anual** obrigatório; OWASP Top 10, secure coding, phishing simulation.
- **Security Champions** por equipe: engenheiros embedded com interesse, treinados, ponte com appsec. Escala o que time central não consegue.
- **Capture the Flag internos**: aprender explorando em controlled env (ver `CTF.md`).
- **Lunch & learns**, post-mortems públicos de vuln encontrada.

## Bug Bounty / Disclosure Program

Responsible / Coordinated Disclosure:

- **Vulnerability Disclosure Policy (VDP)**: canal público, safe harbor, reconhecimento.
- **Bug Bounty**: VDP + recompensa financeira.
- Plataformas: **HackerOne, Bugcrowd, Intigriti, YesWeHack, Synack**.
- Importante: **safe harbor legal** explícito para pesquisadores que seguem regras.

## Métricas DevSecOps

- **MTTR de vulnerabilidades** por severidade.
- **% de builds com checks passando**.
- **Taxa de falsos positivos** das ferramentas.
- **Cobertura SAST/SCA** por repositório.
- **Training compliance**.
- **Escape rate**: vulns descobertas em produção vs pré-produção.
- **Defect density**: vulnerabilidades por KLOC.

## Princípios

1. **Automatize o que é checkável**. Humano revisa PR; máquina roda SAST.
2. **Shift-left sem fricção**. Developer experience importa — regras precisas, falsos positivos baixos.
3. **SCA antes de SAST em ROI**. A maioria dos bugs em apps é em dependências.
4. **Reachability > CVE count**. Não é o número; é o que chega a código executado.
5. **Threat modeling no design** economiza SAST em volume depois.
6. **Secret em Git é incidente**. Reset imediato; não tente "reescrever history".
7. **Patching contínuo**: atualizações são o único real mitigante a médio prazo.
8. **Cultura > ferramenta**. Developer que entende porquê escreve código seguro; o que não entende fica bicho de estimação de regras de SAST.
