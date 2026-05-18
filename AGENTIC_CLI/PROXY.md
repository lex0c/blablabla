# PROXY

Suporte a proxy no `AGENTIC_CLI`. Não é "feature" — é requisito operacional para qualquer agent que pretenda viver fora do laptop do dev. Empresas, redes restritas, self-hosted infra, ambientes paranoicos (com razão) — todos exigem isso.

> **Princípio raiz:** *transport-aware by default.* O agent não fala HTTP direto; fala através de uma camada de transporte que entende proxy, CA, retry, streaming. Cada provider/MCP/tool herda gratuitamente.

Sem proxy support, cedo ou tarde alguém descobre:

```text
"o agent funciona em casa, mas morre atrás do firewall corporativo"
```

Clássico. A internet enterprise continua sendo arqueologia viva de appliances e middleboxes traumatizados.

---

## 0. Princípios

1. **Transport centralizado, não por-cliente.** OpenAI client, Anthropic client, MCP client não lidam com proxy sozinhos. Tudo passa pela mesma `HttpTransport`.
2. **Env vars padrão são lei.** `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`, `NO_PROXY` honrados sem config extra. Docker, curl, npm, git fazem isso — o agent também faz.
3. **Per-provider override > global.** Empresas roteiam diferente; alguns providers são bloqueados; compliance/região manda. Config explícita sobrepõe env.
4. **Custom CA é first-class.** TLS interception corporativo é a regra, não exceção. CA bundle plugável, sem `ignoreUnauthorized` escondido como default.
5. **Credentials nunca em log.** `user:pass@proxy` desaparece de logs, stack traces, telemetry. Redaction antes de qualquer sink.
6. **NO_PROXY é sagrado.** Localhost, MCP local, Ollama, self-hosted inference *não* podem vazar pro proxy corporativo. Excelente forma de irritar infra.
7. **Diagnostics built-in.** Debugging de proxy corporativo é experiência espiritual negativa; o agent precisa de `doctor network` que mostre tudo.

---

## 1. Camada mínima — env vars padrão

Obrigatório honrar, na ordem de precedência usual:

```bash
HTTP_PROXY      # proxy para requests http://
HTTPS_PROXY     # proxy para requests https://
ALL_PROXY       # fallback (também usado por SOCKS)
NO_PROXY        # comma-separated; hostnames/CIDRs que bypassam proxy
```

Regras:

- Variantes lowercase (`http_proxy` etc.) também aceitas; lowercase tem precedência baixa quando ambas existem (mesma convenção do curl).
- `NO_PROXY` aceita:
  - hostnames exatos (`api.internal`)
  - sufixos de domínio (`.internal`, `.corp.local`)
  - IPs (`127.0.0.1`, `::1`)
  - CIDRs (`10.0.0.0/8`, `192.168.0.0/16`)
  - `*` (bypass total)
- Sempre incluir implicitamente em `NO_PROXY`: `localhost`, `127.0.0.1`, `::1`, `0.0.0.0`. Sobrescrevível, mas é o default são.

---

## 2. Schemes suportados

| Scheme       | Suporte | Notas |
|--------------|---------|-------|
| `http://`    | obrigatório | HTTP CONNECT tunneling para HTTPS upstream |
| `https://`   | obrigatório | proxy com TLS próprio (raro mas existe) |
| `socks5://`  | obrigatório | inclui DNS resolution remota (`socks5h://`) |
| `socks4://`  | opcional | legado, sem auth moderna |
| `pac+http://`| futuro | PAC file via HTTP; parsing JS é dor — fase 2 |

`socks5h://` vs `socks5://`: `h` resolve DNS no proxy. Importante para ambientes onde DNS interno não é acessível do cliente.

---

## 3. Per-provider routing

Config explícita sobrepõe env. Exemplo (`config.toml`):

```toml
[transport]
proxy = "http://proxy.internal:8080"        # default global
no_proxy = ["*.internal", "10.0.0.0/8"]

[providers.openai]
proxy = "http://proxy-us.internal:8080"

[providers.anthropic]
proxy = "socks5h://127.0.0.1:9050"          # ex: routing via Tor/VPN

[providers.ollama]
proxy = false                                # localhost, bypass total

[mcp.local-fs]
proxy = false
```

Resolução por request:

1. Override explícito no provider/MCP config.
2. `NO_PROXY` match → sem proxy.
3. Per-scheme env (`HTTPS_PROXY` etc).
4. `ALL_PROXY`.
5. Direct.

---

## 4. Authentication

Formas suportadas:

- **Basic inline:** `http://user:pass@proxy.internal:8080` — clássico, evitar em config commitada.
- **Basic via env:** `PROXY_USER` / `PROXY_PASS` separados (recomendado).
- **NTLM / Negotiate / Kerberos:** enterprise reality. Suporte via lib externa (curl/libcurl-style) ou delegação a `cntlm` local. Documentar como integrar, não embutir.
- **Bearer / custom header:** alguns gateways (Zscaler, Netskope) injetam headers de auth próprios. Permitir `proxy_headers = { "X-Auth-Token" = "..." }`.

### 4.1 Redaction obrigatória

```text
http://user:pass@proxy.internal:8080
```

vira, em qualquer log/telemetry/stack trace:

```text
http://***:***@proxy.internal:8080
```

Regra: redactor roda **antes** do sink, não como camada de "esperança". Test coverage mandatório.

---

## 5. TLS — custom CA e enterprise interception

Enterprise proxies fazem TLS interception. Sintomas:

- `UNABLE_TO_VERIFY_LEAF_SIGNATURE`
- `SELF_SIGNED_CERT_IN_CHAIN`
- `CERT_HAS_EXPIRED` (porque o CA injetado está velho)

### 5.1 Config

```toml
[transport.tls]
ca_bundle = "/etc/ssl/certs/corp-bundle.pem"   # adicional, não substitui system store
ca_bundle_mode = "append"                       # append | replace
client_cert = "/etc/forja/client.pem"           # mTLS opcional
client_key  = "/etc/forja/client.key"
min_version = "TLSv1.2"
```

Honrar também:

- `SSL_CERT_FILE` / `SSL_CERT_DIR` (OpenSSL conventions).
- `NODE_EXTRA_CA_CERTS` (compatibilidade ecossistema Node).
- `REQUESTS_CA_BUNDLE` (compatibilidade ecossistema Python).

### 5.2 `insecure` é opt-in explícito

```toml
[transport.tls]
insecure = true   # NUNCA default. Warning loud no startup.
```

Quando true:

- Log warning a cada startup.
- Telemetria marca a session como `tls_unsafe = true`.
- Bloqueado em modo `--strict` / production profile.

---

## 6. Transport abstraction

```ts
interface HttpTransport {
  fetch(req: Request, ctx?: RequestCtx): Promise<Response>
  stream(req: Request, ctx?: RequestCtx): AsyncIterable<Chunk>
  capabilities(): TransportCapabilities
}

interface TransportCapabilities {
  proxy: ProxyKind | false
  sse: boolean                  // proxy permite SSE end-to-end?
  chunked: boolean              // chunked transfer sobrevive?
  http2: boolean
  ws: boolean
}

interface RequestCtx {
  provider?: string             // pra resolver per-provider config
  no_proxy_override?: boolean
  timeout_ms?: number
  retry_policy?: RetryPolicy
}
```

Tudo passa por aqui:

- Provider adapters (OpenAI, Anthropic, Gemini, Ollama, llama.cpp remote)
- MCP clients (remote)
- WebFetch / WebSearch tools
- Telemetry / OTLP exporters
- Update checker, skill registry sync, etc.

Resultado: adicionar suporte a novo proxy/CA/scheme é **um lugar**, não vinte.

---

## 7. Streaming vs proxy — armadilha clássica

Muitos proxies corporativos:

- Bufferizam SSE inteiro até `\n\n` final → token streaming aparenta travar.
- Quebram `Transfer-Encoding: chunked` → response chega em bloco único.
- Fecham conexões idle > N segundos → long-running generations falham no meio.
- Removem headers `Connection: keep-alive`.

### Estratégia

1. **Capability detection no startup:** request canário pequeno via SSE; mede se chunks chegam progressivamente. Cacheia resultado por origin + proxy.
2. **Fallback automático:** se SSE quebrado, alternar para *poll-based* completion (provider permitindo) ou non-streaming endpoint.
3. **Retry policy SSE-aware:** retry só se chunks ≥ N foram recebidos *ou* zero foram. Meio-fluxo é caso ambíguo — escalar pra usuário.
4. **Timeout duplo:** `connect_timeout` curto, `idle_chunk_timeout` longo. Não usar `total_timeout` para streams.

Subagents que dependem de streaming sem fallback travam silenciosamente — sintoma é spinner eterno, não erro. Garantir telemetria que detecta "stream sem chunks por > X segundos".

---

## 8. `forja doctor network`

Comando diagnóstico. Sai por aqui:

```text
$ forja doctor network

[proxy]
  HTTP_PROXY        = http://proxy.internal:8080
  HTTPS_PROXY       = http://proxy.internal:8080
  NO_PROXY          = localhost,127.0.0.1,*.internal
  effective for api.anthropic.com → http://proxy.internal:8080
  effective for localhost:11434   → DIRECT (NO_PROXY match)

[dns]
  api.anthropic.com → 160.79.104.10  (resolved in 12ms)
  api.openai.com    → 162.159.140.245 (resolved in 8ms)

[tls]
  system store      = /etc/ssl/certs/ca-certificates.crt
  extra ca_bundle   = /etc/forja/corp.pem (3 certs, expires 2027-04-12)
  chain api.anthropic.com:
    CN=*.anthropic.com → CN=Corp-MITM-CA (intercepted)
    ⚠ TLS interception detected — using corp CA bundle ✓

[reachability]
  api.anthropic.com:443        ✓  214ms
  api.openai.com:443           ✓  198ms
  generativelanguage.googleapis.com:443  ✗  blocked by proxy (403)
  localhost:11434              ✓  2ms

[streaming]
  SSE end-to-end via proxy     ✗  buffered (proxy holds chunks)
  fallback: non-streaming      ✓ available
```

Subcomandos úteis:

- `forja doctor network --provider anthropic` — escopo único.
- `forja doctor network --probe-sse` — força teste de streaming.
- `forja doctor network --json` — output estruturado pra suporte/CI.

---

## 9. Falhas comuns e mensagens

Mensagem de erro genérica ("fetch failed") é o suficiente pra usuário culpar o agent. Mapear erros pra hipóteses:

| Sintoma | Hipótese provável | Sugestão de ação |
|---|---|---|
| `ECONNREFUSED` em proxy host | proxy down ou porta errada | verificar `HTTPS_PROXY` |
| `407 Proxy Authentication Required` | auth faltando/expirada | configurar `PROXY_USER`/`PROXY_PASS` |
| `UNABLE_TO_VERIFY_LEAF_SIGNATURE` | TLS interception sem CA bundle | adicionar CA corporativo via `ca_bundle` |
| `ETIMEDOUT` em `api.*` mas DNS ok | proxy bloqueia domínio | testar via `doctor network`, verificar policy |
| stream "trava" sem erro | proxy bufferizando SSE | fallback non-streaming, reportar ao infra |
| localhost requests via proxy | `NO_PROXY` mal configurado | adicionar `localhost,127.0.0.1` |
| `403` consistente em provider X | provider bloqueado por policy | per-provider proxy ou region routing |

Cada um destes vira hint específico no error output. Sem hint, usuário abre issue.

---

## 10. Segurança

### 10.1 Credentials

- Nunca em logs (já mencionado §4.1).
- Nunca em telemetry payloads (mesmo redacted — preferir não enviar).
- Nunca em error messages user-facing (proxy URL ok; credentials não).
- Stack traces: scrub no formatter, não confiar em libs upstream.

### 10.2 CA bundles

- CA injetado por corp não é "do agent" — é confiança transitiva. Logar fingerprint do CA no startup pra auditabilidade.
- Mudança de CA bundle → log loud. Não silencioso. É vetor de ataque clássico ("alguém substituiu nosso CA bundle").

### 10.3 Proxy como exfiltration vector

Proxy malicioso vê todo tráfego (mesmo HTTPS se for interceptado). Implicações:

- Em modo `--strict`, recusar TLS interception detectada sem CA pré-aprovada (fingerprint pinning).
- Documentar threat model: "se você usa proxy interceptante, ele vê seus prompts e respostas".
- Skills sensíveis (credenciais, código proprietário) podem ter flag `disallow_intercepted_tls = true`.

---

## 11. Fora de escopo (fase atual)

- **PAC file parsing.** JS engine + Mozilla `FindProxyForURL` semantics — dor real. Marcar como futuro; recomendar workaround (cntlm, polipo, env vars manuais).
- **WPAD auto-discovery.** Vetor de ataque conhecido; deixar fora por default.
- **HTTP/3 / QUIC via proxy.** Maioria dos proxies enterprise ainda não suporta. Fallback automático HTTP/2 ou HTTP/1.1 é aceitável.
- **Per-request dynamic proxy selection via tool call.** Tentação grande, vetor de ataque maior. Não.

---

## 12. Test matrix

Cobrir em CI/integration:

| Cenário | Mecânica |
|---|---|
| direct (sem proxy) | sanity |
| `HTTP_PROXY` env | container com squid |
| `HTTPS_PROXY` auth | squid + basic auth |
| `socks5` | dante ou 3proxy |
| `NO_PROXY` bypass | request a `localhost` com proxy configurado, assert direct |
| TLS interception | mitmproxy + CA bundle |
| SSE buffering proxy | nginx com `proxy_buffering on` |
| chunked stripping | proxy que reescreve `Transfer-Encoding` |
| credential redaction | log assertion sobre `user:pass` |
| per-provider override | dois providers, dois proxies, asserts em destination |

Sem essa matrix, "support proxy" é declaração sem prova — igual "multi-provider" sem eval (ver `PROVIDERS.md`).

---

## 13. Resumo

Suporte robusto a proxy é exatamente o tipo de detalhe:

- invisível em demos
- crítico em produção

Ninguém faz keynote sobre proxy auth, CA bundles, transport abstraction. Mas é isso que separa **"demo AI"** de **sistema operacional utilizável**.

Engenharia de software real voltou ao chat. Surpresa coletiva.
