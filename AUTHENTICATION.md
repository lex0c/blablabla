# Autenticação e Autorização

Dois problemas frequentemente confundidos:

- **Autenticação (authN)**: *quem* é o usuário? Prova de identidade.
- **Autorização (authZ)**: *o que* esse usuário pode fazer? Política de acesso.

Um sistema pode ter autenticação forte e autorização quebrada (IDOR), ou o contrário. Separar os dois na cabeça é o primeiro passo para acertar os dois na prática.

## Fatores de Autenticação

1. **Algo que você sabe**: senha, PIN.
2. **Algo que você tem**: celular (TOTP), chave de hardware (FIDO2), smartcard.
3. **Algo que você é**: biometria (digital, face, íris).
4. **Algum lugar onde você está**: geofencing, IP corporativo (fraco, facilmente contornável).

**MFA (Multi-Factor Authentication)** combina ao menos dois fatores **distintos**. Senha + SMS código é MFA. Senha + pergunta de segurança **não é**.

## Senhas

Se você vai implementar senhas, saiba o mínimo:

- **Hash com KDF resistente a GPU/ASIC**: Argon2id (estado da arte), scrypt, ou bcrypt. **Nunca** MD5, SHA-1, SHA-256 puros — calculáveis em bilhões por segundo em hardware dedicado.
- **Salt único por senha**: impede rainbow tables.
- **Pepper**: segredo global adicional guardado separado (em HSM/secret manager). Defesa extra se o banco vaza sem o pepper.
- **Sem limites de caracteres absurdos**: se não aceita 64+ chars ou todos os unicode, algo está errado.
- **Validar contra listas de senhas vazadas** (HIBP Pwned Passwords via k-anonymity API).
- **Lockout ou rate limit exponencial** após tentativas falhas, por conta E por IP.
- **Nunca** log da senha em lugar algum.

## Sessões

Após autenticar, o servidor mantém estado (cookie de sessão, token) de que o usuário "está logado".

### Cookies de sessão

- ID opaco (~128 bits aleatórios) armazenado server-side.
- Cookie com `HttpOnly`, `Secure`, `SameSite=Lax|Strict`.
- Expiração explícita; invalidar no logout; rotacionar após mudança de privilégio.

### Tokens (JWT / Opaque)

Alternativa stateless:

- **Opaque tokens**: string aleatória, servidor consulta banco. Simples, fácil de revogar.
- **JWT (JSON Web Token)**: claims assinados digitalmente; servidor verifica sem consultar DB. Rápido, escala bem. Revogação difícil — exige blocklist ou janela curta de expiração.

### JWT — armadilhas

1. **`alg: none`**: bibliotecas antigas aceitavam token assinado com algoritmo "nenhum" — ou seja, não assinado. Nunca aceitar `none`. **Validar `alg`** explicitamente.
2. **Key confusion (HS256 vs RS256)**: servidor espera RSA, atacante envia HS256 assinado com a **chave pública** (que é texto conhecido). Mitigação: fixar `alg` por chave.
3. **Expiração longa**: JWT sem TTL é credencial permanente. Combine com refresh tokens (opacos e revogáveis).
4. **Dados sensíveis em payload**: JWT é *assinado*, não *criptografado*. Qualquer um decodifica.
5. **`kid` header injeção**: se o servidor carrega chave pelo `kid` do token, atacante aponta para arquivo qualquer no sistema.
6. **Tamanho**: JWT com muitas claims enche headers e quebra proxies.

## OAuth 2.0

Framework de **autorização delegada**. Permite que um usuário conceda a um app de terceiros acesso a uma API, sem compartilhar a senha.

### Atores

- **Resource Owner**: o usuário.
- **Client**: a aplicação que quer acessar.
- **Authorization Server**: quem emite tokens (ex.: Google accounts).
- **Resource Server**: a API protegida (ex.: Google Drive API).

### Grant Types

1. **Authorization Code (com PKCE)**: padrão para apps web/mobile/SPA. Redireciona usuário → consent → código → troca por token no backend. **PKCE** obrigatório para clientes públicos; hoje recomendado para todos.
2. **Client Credentials**: machine-to-machine. Sem usuário envolvido.
3. **Device Code**: dispositivos sem teclado (TVs, IoT).
4. **Resource Owner Password (ROPC)**: **depreciado**. Cliente recebe a senha direto.
5. **Implicit**: **depreciado**. Vulnerável e redundante com Auth Code + PKCE.

### Escopos

Tokens carregam `scope` limitando o que podem fazer. `drive.readonly` ≠ `drive.full`. Princípio do menor privilégio aplicado.

### Token types

- **Access token**: curto (minutos-horas), usado na API.
- **Refresh token**: longo, trocável por novo access token. Armazenar com cuidado extra.
- **Rotation**: refresh token é descartado a cada uso; servidor emite novo. Detecta vazamento (dois usos simultâneos).

## OpenID Connect (OIDC)

**Camada de autenticação sobre OAuth 2.0**. OAuth delega autorização; OIDC adiciona "quem é o usuário" via **ID Token** (um JWT com claims identificando o usuário). Se você quer "login com Google", é OIDC, não só OAuth.

Endpoints-chave: `/.well-known/openid-configuration`, `/authorize`, `/token`, `/userinfo`, `/jwks` (chaves públicas para verificar assinaturas).

## SAML

Padrão pré-OAuth, dominante em SSO corporativo (empresas Microsoft/Okta/Azure AD). Troca assertions XML assinadas. Mais complexo que OIDC; muitas vulnerabilidades históricas (XML Signature Wrapping, canonicalização). Para greenfield, prefira OIDC. Para interoperar com ERP/SAP, vai ter que falar SAML.

## WebAuthn / FIDO2 / Passkeys

Padrão moderno de autenticação baseado em criptografia de chave pública.

- Registro: autenticador gera par de chaves; chave pública vai ao servidor.
- Login: servidor manda desafio; autenticador assina com chave privada (nunca sai do dispositivo).
- Resistente a phishing por construção — a origem é parte do que é assinado. Phishing site ≠ site real, assinatura inválida.
- **Passkeys**: WebAuthn + sincronização da chave entre dispositivos do usuário (iCloud Keychain, Google Password Manager). Substitui senha.

Se você está desenhando auth em 2026+, passkeys são o alvo. TOTP continua como fallback.

## TOTP

Códigos de 6 dígitos baseados em tempo (RFC 6238). Popular via Google Authenticator, Authy. Vulnerável a phishing em tempo real (atacante pede ao usuário o código). WebAuthn é melhor.

## mTLS

**Mutual TLS**: além do servidor apresentar certificado (TLS normal), o cliente também. Cliente prova sua identidade com chave privada; servidor valida contra CA. Usado em:

- Comunicação entre microsserviços (service mesh: Istio, Linkerd).
- APIs bancárias (Open Banking).
- Zero trust em redes internas (ver `ZERO_TRUST.md`).

Pros: forte, sem senha, sem tokens em jogo. Contras: gestão de certificados (emissão, rotação, revogação). SPIFFE/SPIRE padronizam identidade de workload.

## Autorização

### RBAC (Role-Based Access Control)

Usuário tem **papéis** (admin, editor, viewer); papéis têm **permissões**. Simples; dominante na maioria das aplicações. Sofre quando:

- Papéis explodem em número ("admin de X, editor de Y se Z").
- Precisa-se de decisão dependente do recurso ("usuário pode editar *seus próprios* posts").

### ABAC (Attribute-Based Access Control)

Decisão baseada em **atributos**: do usuário, do recurso, do ambiente. "Permitir se `user.department == resource.department && time.hour between 9 and 18`". Flexível, expressivo, complexo de auditar.

XACML é o padrão formal; na prática, muitos sistemas usam OPA (Open Policy Agent) com Rego.

### ReBAC (Relationship-Based Access Control)

Autorização baseada em **relações** entre entidades. "Pode editar o documento se é membro do grupo dono do documento". Popularizado pelo paper **Zanzibar** do Google (2019), que roda bilhões de checks/s.

Implementações: SpiceDB (AuthZed), Permify, OpenFGA (Auth0/Okta), Warrant.

Ideal para domínios com hierarquia/compartilhamento complexos (Docs, Drive, GitHub).

### PDP/PEP

Separação útil:

- **PDP (Policy Decision Point)**: decide "permitir ou negar?" dado um request.
- **PEP (Policy Enforcement Point)**: intercepta request, consulta PDP, aplica decisão.

OPA como PDP + gateway/sidecar/middleware como PEP é arquitetura comum.

## Logout

- **Local logout**: limpa sessão no cliente.
- **Global logout (SLO)**: invalida em todos os dispositivos. Exige estado server-side (blocklist de JWTs ou sessões server-side).
- **OIDC back-channel logout**: IdP notifica RPs de que a sessão expirou.

## Recuperação de Senha

Vetor frequentemente explorado. Cuidados:

- **Token de reset** aleatório, TTL curto (15-30min), único por pedido, invalidado após uso.
- **Enviar via e-mail registrado** (e avisar em outros canais).
- **Não vazar existência do e-mail**: "se a conta existe, enviaremos link". Caso contrário, enumera-se usuários.
- **Sem perguntas de segurança fracas** ("nome do cachorro" é OSINT).
- **Re-autenticar antes de mudar senha** se usuário já estiver logado.

## Enumeração de Usuários

Respostas diferentes para "usuário existe" e "não existe" (em login, reset, signup) permitem enumeração. Responda igual em ambos os casos, com timing consistente.

## Rate Limiting

Obrigatório em:

- Login (por IP, por user, global).
- Reset de senha.
- Endpoints de OTP.
- Qualquer endpoint caro computacionalmente.

Ver `RELIABILITY_PATTERNS.md` para algoritmos.

## Gestão de Segredos

- Senhas de banco, API keys, chaves de assinatura: **nunca em código**. Use secret manager (Vault, AWS SM, GCP SM) ou sealed secrets em K8s.
- **Rotação periódica** obrigatória para chaves de longa duração.
- **Short-lived credentials** > long-lived (ex.: AWS STS em vez de access keys permanentes).

## Princípios

1. **Use bibliotecas consagradas, não roles seu próprio auth**. Acertar auth é matematicamente difícil e sutil.
2. **Hash senhas com Argon2id / bcrypt**. Sempre. Sem exceção.
3. **MFA é não-negociável** para contas sensíveis. Passkeys > TOTP > SMS.
4. **Valide tokens inteiramente**: assinatura, emissor, audiência, expiração, não-antes-de, escopo.
5. **Autorização em cada acesso a recurso**, não só "está logado?".
6. **Menor privilégio** em tudo — papéis, tokens, APIs, serviços.
7. **Auditoria**: log de login, mudança de senha, elevação de privilégio — sem logar *a senha ou token*.
