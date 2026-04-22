# Segurança Web

Aplicações web operam em um dos ambientes mais hostis da computação: expostas à internet, atendendo input arbitrário de usuários anônimos, integradas a dezenas de dependências. Segurança web é, em boa parte, o estudo sistemático das classes de bugs que surgem nessa combinação. A referência de fato é o [OWASP](https://owasp.org), em particular o **Top 10** — que reflete as categorias mais impactantes observadas em campo.

## OWASP Top 10 (edição 2021)

1. **A01 — Broken Access Control**: autorização quebrada (IDOR, privilégios verticais/horizontais).
2. **A02 — Cryptographic Failures**: criptografia ausente, fraca ou mal usada.
3. **A03 — Injection**: SQLi, NoSQLi, LDAPi, command injection, SSTI.
4. **A04 — Insecure Design**: falhas arquiteturais (vs. bugs pontuais).
5. **A05 — Security Misconfiguration**: defaults, cabeçalhos ausentes, portas expostas.
6. **A06 — Vulnerable and Outdated Components**: dependências com CVE.
7. **A07 — Identification and Authentication Failures**: login, sessão, credenciais.
8. **A08 — Software and Data Integrity Failures**: CI/CD, deserialização insegura.
9. **A09 — Security Logging and Monitoring Failures**: ausência de detecção.
10. **A10 — Server-Side Request Forgery (SSRF)**: servidor fazendo requisições para onde não deveria.

## Injection

### SQL Injection

Input do usuário é concatenado em uma query. Ex.: `SELECT * FROM users WHERE name = '` + input + `'`. Com input `' OR '1'='1`, a query vira `WHERE name = '' OR '1'='1'` — retorna tudo.

**Variantes**:
- **Union-based**: atacante adiciona `UNION SELECT` para extrair colunas.
- **Boolean-based blind**: sem output direto; inferir por diferença no retorno.
- **Time-based blind**: usar `SLEEP(5)` e observar latência.
- **Out-of-band**: exfiltrar via DNS/HTTP lookup.

**Mitigação definitiva**: **prepared statements** com placeholders. Nunca concatenar. ORMs corretamente usados quase eliminam SQLi — desde que não se use `raw()` com input direto.

### Command Injection

Input entra em shell. `os.system("ping " + host)` com `host = "1.1.1.1; rm -rf /"` é catastrófico. Mitigação: usar APIs que aceitam lista de argumentos (`subprocess.run([...], shell=False)`), nunca string concatenada.

### SSTI (Server-Side Template Injection)

Templates que interpolam input do usuário como expressão. Em Jinja, `{{7*7}}` retorna `49`; `{{config}}` vaza configuração; `{{''.__class__.__mro__...}}` escala a RCE. Mitigação: nunca passar input como template; usar `render_template(..., var=input)` com variáveis explícitas.

### LDAP, XPath, NoSQL

Variantes do mesmo antipadrão: concatenar input em query estruturada. NoSQL sofre com operadores como `{"$gt": ""}` em MongoDB aceitando JSON bruto.

## Cross-Site Scripting (XSS)

Injeção de JavaScript no contexto da vítima. O atacante executa código sob a origem da aplicação — roubando cookies, fazendo ações em nome do usuário, alterando DOM.

### Tipos

1. **Reflected**: o payload está na URL/requisição e é refletido na resposta. Ex.: `/search?q=<script>...`.
2. **Stored (persistent)**: payload salvo no servidor (comentário, perfil) e servido a outros usuários.
3. **DOM-based**: ocorre inteiramente no cliente — JS lê `location.hash` e injeta em `innerHTML` sem sanitizar.

### Mitigações

- **Output encoding por contexto**: HTML-encode ao inserir em HTML, JS-encode ao inserir em script, URL-encode em atributos, CSS-encode em style. Frameworks modernos (React, Vue, Angular) fazem isso automaticamente no JSX/template — **desde que** você não use `dangerouslySetInnerHTML` / `v-html` com input.
- **CSP (Content Security Policy)**: `Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-abc123'`. Mesmo com XSS, script inline não executa sem nonce correto. Não é substituto para encoding, é defesa em profundidade.
- **Cookies `HttpOnly`**: JS não acessa → XSS não rouba cookies de sessão. Combinar com `Secure`, `SameSite`.
- **`X-Content-Type-Options: nosniff`**: impede browser de "adivinhar" content-type que viabiliza XSS.
- **Sanitizar quando precisar renderizar HTML confiável**: DOMPurify é o padrão.

## CSRF (Cross-Site Request Forgery)

Atacante faz o browser da vítima autenticada executar ações em outro site, aproveitando que cookies são enviados automaticamente. Ex.: vítima logada em `bank.com` visita `evil.com`, que contém `<form action="https://bank.com/transfer" method="POST">` + auto-submit. Transferência sai do nome da vítima.

### Mitigações

- **`SameSite=Lax` ou `Strict` em cookies**: navegador não envia cookie em requisições cross-site. `Lax` é default moderno — resolve a maioria dos casos.
- **CSRF tokens**: token único por sessão (ou por requisição) em cada form. Servidor rejeita sem token válido.
- **Double-submit cookie**: token em cookie + header; servidor confere igualdade.
- **Exigir métodos com corpo (POST/PUT/DELETE)** para mutações; nunca `GET`. Simples e efetivo.
- **Re-autenticação para operações críticas** (mudar senha, transferência).

## Broken Access Control / IDOR

**IDOR (Insecure Direct Object Reference)**: `/api/invoices/42` — se servidor só verifica autenticação mas não *autorização* ("este user tem direito a esta invoice?"), basta trocar `42` por `43` para ver dados alheios.

Mitigação: **toda rota verifica ownership/autorização no recurso acessado**, não só se está logado. ABAC ou RBAC formal (ver `AUTHENTICATION.md`). IDs opacos (UUIDs) ajudam contra bruteforce mas **não substituem** autorização.

**BOLA (Broken Object Level Authorization)** e **BFLA (Broken Function Level Authorization)** são a mesma família em APIs REST.

## SSRF (Server-Side Request Forgery)

Servidor faz requisição HTTP a URL controlada pelo atacante. Se a URL aponta para o metadata service da cloud (`http://169.254.169.254/`), o atacante extrai credenciais IAM. Se aponta para serviços internos (`http://internal-admin:8080/`), acessa coisas que não deveria.

**Mitigações**:

- **Allowlist de domínios/IPs**: só permitir URLs para destinos conhecidos.
- **Bloquear ranges privados**: `127.0.0.0/8`, `10.0.0.0/8`, `169.254.0.0/16`, `::1`, etc. Cuidado com DNS rebinding (o domínio resolve público na validação, privado na requisição real).
- **IMDSv2 (AWS)**: exige header + token, o que impede SSRF naive. Sempre prefira.
- **Egress segregado**: o serviço que lida com URLs externas não tem acesso a rede interna.

## Deserialização Insegura

Deserializar dados controlados pelo atacante em formato que permite reconstrução de objetos arbitrários (Java native, Python pickle, PHP unserialize, YAML unsafe load) permite RCE. **Nunca** faça pickle/unserialize em dados não confiáveis. Prefira formatos de dados puros (JSON, Protobuf) que não reconstroem tipos arbitrários.

## XXE (XML External Entity)

Parser XML com entidades externas habilitadas. `<!ENTITY xxe SYSTEM "file:///etc/passwd">` vaza arquivos; `http://internal/` vira SSRF. Mitigação: desabilitar entidades externas e DTD no parser. Moderna: evitar XML em greenfield.

## Clickjacking

Página maliciosa embute a vítima em iframe transparente. Usuário pensa clicar em botão do site do atacante, mas clica em ação na página real. Mitigação: `X-Frame-Options: DENY` ou `Content-Security-Policy: frame-ancestors 'none'`.

## Headers de Segurança

| Header | Função |
|---|---|
| `Content-Security-Policy` | restringe origens de script, frame, etc. |
| `Strict-Transport-Security` (HSTS) | força HTTPS; `max-age=63072000; includeSubDomains; preload` |
| `X-Frame-Options` / `frame-ancestors` | anti-clickjacking |
| `X-Content-Type-Options: nosniff` | impede MIME sniffing |
| `Referrer-Policy: strict-origin-when-cross-origin` | vazamento de referrer |
| `Permissions-Policy` | restringe APIs (geo, camera, usb) |
| `Cross-Origin-Opener-Policy: same-origin` | anti Spectre, isolamento de janela |
| `Cross-Origin-Embedder-Policy: require-corp` | idem, para recursos embedados |
| `Cache-Control: no-store` | em páginas com dado sensível |

Ferramenta: [securityheaders.com](https://securityheaders.com) audita rapidamente.

## CORS

**CORS não é controle de segurança** — é **flexibilização** da Same-Origin Policy. `Access-Control-Allow-Origin: *` não protege nada; permite que qualquer origem chame via fetch. Erros comuns:

- Refletir `Origin` recebido em `Access-Control-Allow-Origin` sem validar → equivale a `*` mas piora, pois combina com `Allow-Credentials`.
- `Allow-Credentials: true` com `Allow-Origin: *` → browser rejeita; tentativa de contornar via refletir Origin é o bug acima.
- Confundir CORS com autenticação; CORS não impede requisição, só restringe leitura da resposta em navegador.

## Cookies

- **`Secure`**: só sobre HTTPS.
- **`HttpOnly`**: inacessível a JS.
- **`SameSite=Lax`** (default) ou **`Strict`**: anti-CSRF. `None` exige `Secure`.
- **`__Host-` prefix**: exige `Secure`, sem `Domain`, `Path=/`. Forte garantia de origem.
- **`__Secure-` prefix**: exige `Secure`.
- **Tamanho**: cookies enormes em todo request custam banda; considere armazenar só um ID de sessão.

## Upload de Arquivos

Vetor clássico. Armadilhas:

- Validar content-type do header é inútil — atacante controla.
- Validar extensão só no cliente é inútil.
- Deixar arquivos em path servido diretamente (`/uploads/evil.php`) pode virar RCE.
- Validar magic bytes + re-encodar (imagem) + servir de domínio separado sem execução.
- Anti-vírus / sandbox para arquivos potencialmente perigosos.

## Race Conditions em Web

Exemplo clássico: "saque de 100 reais" em paralelo. Se check-and-debit não é atômico, o saldo vira negativo. Em HTTP, atacantes enviam dezenas de requisições simultâneas em "single-packet attack" (HTTP/2 multiplexing). Mitigação: transações de banco com locking apropriado, idempotency keys (ver `IDEMPOTENCY.md`).

## Testes Automatizados

- **DAST** (Dynamic Application Security Testing): OWASP ZAP, Burp Suite Pro, Nuclei.
- **SAST** (Static): Semgrep, CodeQL, Snyk Code.
- **SCA** (Software Composition Analysis): dependências com CVE. Dependabot, Snyk, Trivy.
- **Fuzzing de endpoints**: Restler, Wuppiefuzz.
- **Pentest manual**: automação encontra o óbvio; criatividade humana encontra o resto.

## Princípios

1. **Input não é seguro**: todo input passa por validação (allowlist preferencialmente), sanitização ou encoding no output.
2. **Autenticação ≠ autorização**: saber quem é não é saber o que pode.
3. **Defesa em profundidade**: CSP + encoding + validação; HSTS + cookies seguros; WAF + SAST + threat modeling.
4. **Atualize dependências**: vulnerabilidades em libs populares são exploradas em horas, não dias.
5. **Ninguém testa login**: autenticação, recuperação de senha e gestão de sessão são onde os bugs "sutis" mais catastróficos moram. Ver `AUTHENTICATION.md`.
