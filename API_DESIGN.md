# Design de APIs

Uma API é um **contrato** entre produtor e consumidor. Diferente do código interno, que se refatora à vontade, uma API pública é difícil de mudar — há clientes dependendo dela, muitas vezes sem canal direto de comunicação. Boa API é escolhida com cuidado antes do deploy, não corrigida depois.

## Princípios Gerais

1. **Menos é mais**: cada endpoint, campo, parâmetro é um compromisso permanente. Remover é caro; adicionar é fácil. Comece mínimo.
2. **Previsibilidade > criatividade**: siga convenções do estilo escolhido (REST, gRPC) até ter uma razão forte para divergir.
3. **Otimize para o leitor, não o escritor**: consumidores leem a doc, escritores escrevem uma vez. Nomes e estruturas devem ser óbvios.
4. **Erros são parte da API**: códigos de erro, mensagens, payloads de erro — tudo é contrato. Clientes vão programar contra eles.
5. **Postel's Law (com cuidado)**: "Be liberal in what you accept, conservative in what you send." Hoje é visto com reservas — aceitar input malformado mascara bugs do cliente e cria dependências implícitas. Modernamente: *strict in, strict out*.

## Estilos

### REST

Baseado em recursos. Cada URI representa um recurso; verbos HTTP representam operações.

- `GET /users/42` — recupera.
- `POST /users` — cria.
- `PUT /users/42` — substitui (idempotente).
- `PATCH /users/42` — atualiza parcialmente.
- `DELETE /users/42` — remove (idempotente).

**Boas práticas**:
- Substantivos no plural para coleções: `/users`, não `/user`.
- Hierarquia para relações: `/users/42/posts/7`.
- Paginação explícita: `?cursor=...&limit=50` (cursor) ou `?page=2&size=50` (offset). Cursor é melhor para datasets grandes e mutáveis.
- Filtros como query string: `GET /orders?status=paid&created_after=2026-01-01`.
- Status codes com significado: `200` (ok), `201` (criado), `204` (sem conteúdo), `400` (erro do cliente), `401` (não autenticado), `403` (não autorizado), `404` (não existe), `409` (conflito), `422` (validação), `429` (rate limit), `500` (erro do servidor), `503` (indisponível).
- HATEOAS é raramente usado na prática; não se culpe por omitir.

**Problemas conhecidos**:
- Múltiplas round-trips para compor UIs (N+1).
- Subtypes forçam vários endpoints ou payloads genéricos.
- Versionamento é desajeitado.

### gRPC

RPC sobre HTTP/2 com contratos em Protobuf. Schema-first; clientes gerados.

**Prós**:
- Binário, compacto, rápido.
- Streaming bidirecional nativo.
- Schema evolui com regras claras (campos numerados, compatibilidade forward/backward).
- Gera código em múltiplas linguagens.

**Contras**:
- Não funciona bem direto do browser (precisa de gRPC-Web ou gateway).
- Debugging é mais difícil que curl em JSON.
- Ferramental HTTP (caches, proxies, logs) entende menos.

Bom para **comunicação interna entre serviços**; pior para APIs públicas voltadas a browsers.

### GraphQL

Cliente declara exatamente os campos que quer; servidor resolve.

**Prós**:
- Elimina over-fetching e under-fetching.
- Um único endpoint.
- Schema auto-documentável, tooling forte (introspection, codegen).

**Contras**:
- Complexidade no servidor (resolvers, N+1 via DataLoader).
- Cache HTTP tradicional não funciona (tudo é `POST /graphql`).
- Risco de queries abusivas: cliente pode pedir consultas caríssimas. Exige query cost analysis, persisted queries, depth limiting.
- Mutations são menos ergonômicas que REST puro.

Bom para **agregadores de dados** em frontends complexos; menos bom para APIs simples de CRUD.

### Quando Escolher

- **REST**: APIs públicas simples, CRUD, integração com ferramental HTTP padrão.
- **gRPC**: microserviços internos, alta performance, streaming.
- **GraphQL**: frontends com necessidades de dados variadas, múltiplos clientes consumindo dados relacionados.

Não é incomum combinar: gRPC interno, GraphQL ou REST para o mundo externo.

## Versionamento

Toda API muda. A questão é como.

### Estratégias

- **URL**: `/v1/users`, `/v2/users`. Simples, visível, fácil de rotear. Penaliza evolução (breakings viram releases grandes).
- **Header**: `Accept: application/vnd.api+json; version=2`. Mais limpo URL; pior ergonomia, mais difícil testar com curl.
- **Query string**: `?version=2`. Raro.
- **Sem versão, evolução contínua**: campos novos são adicionados, nunca removidos; comportamentos antigos mantidos. Abordagem do Stripe. Exige disciplina enorme — mas é a experiência mais suave para clientes.

### Compatibilidade

- **Backward-compatible change** (não quebra clientes velhos): adicionar campo opcional, adicionar endpoint, adicionar enum value (com cuidado — clientes podem dar panic).
- **Breaking change**: remover campo, renomear, mudar tipo, tornar opcional → obrigatório, mudar comportamento, mudar formato de erro.

Regra de ouro: **breaking changes exigem versão nova ou migração coordenada**. Nunca quebre silenciosamente.

## Erros

Formato consistente é mais importante que formato "bonito". Um esquema útil (variante de RFC 7807 "Problem Details"):

```json
{
  "error": {
    "code": "payment_declined",
    "message": "Payment was declined by the issuer.",
    "details": { "issuer_code": "51" },
    "request_id": "req_abc123"
  }
}
```

- `code` é **estável** e **programático** (snake_case, curto, sem pontuação). É contra ele que clientes codificam.
- `message` é humano, mutável.
- `request_id` permite correlacionar com logs em suporte.
- Sem stack traces, sem caminhos de arquivo, sem nomes internos — vazam e dão pistas para ataques.

Evite misturar validation errors (campo a campo) com erros semânticos num mesmo formato plano. Use sub-estrutura:

```json
{ "error": { "code": "validation_failed", "fields": { "email": "invalid_format" } } }
```

## Paginação

- **Offset/limit**: `?offset=500&limit=50`. Simples, mas pula/duplica em datasets mutáveis; custo O(N) no banco para offsets altos.
- **Cursor-based**: `?cursor=abc&limit=50`. Cursor opaco (base64 do último ID + ordering). Estável, rápido, idiomático. Use isso por padrão.
- **Keyset / seek**: variante de cursor usando colunas indexadas (`WHERE id > last_id ORDER BY id LIMIT 50`). Rápido mesmo em datasets enormes.

Sempre devolva `next_cursor` explícito; não peça ao cliente para inferir.

## Autenticação e Autorização

- **Bearer token** (`Authorization: Bearer ...`) é o padrão HTTP. Tokens podem ser JWT, opacos (referência a registro server-side), ou ambos.
- **JWT** tem trade-offs: fácil de validar sem consultar banco, mas difícil de revogar. Combine com janela curta (`exp`) + refresh token.
- **API keys** para autenticação server-to-server simples, com rotação explícita.
- **OAuth 2.0 / OIDC** para delegação (usuário autoriza app terceiro).
- **Nunca** passe credencial em URL (vai para logs, histórico, referer). Sempre em header ou body.

## Consistência e Timing

- Use **ISO 8601** em UTC para timestamps (`2026-04-21T14:32:00Z`). Nunca timestamp Unix em produção de APIs públicas — ilegível, ambíguo de unidade.
- Booleans explícitos (`true`/`false`), nunca `"yes"`/`"no"` ou `0`/`1`.
- Dinheiro **não** em float. Use string decimal (`"10.00"`) ou menor unidade inteira (`1000` centavos) + `currency`.
- `null` vs campo ausente: decida e documente. Em JSON, `{ "x": null }` e `{}` são diferentes — seu parser pode fazer distinção.

## Documentação

- **OpenAPI (REST) / Protobuf IDL (gRPC) / GraphQL SDL** são contratos versionados. Tratam-se como código: em PR, review, CI.
- Exemplos de request e response reais. Mais valor que descrição prosa.
- `curl` exemplos que funcionam copy/paste.
- **Changelog** explícito. Nunca mudança silenciosa.

## Checklist Rápido

1. Recursos nomeados com consistência; verbos HTTP com semântica correta.
2. Erros com `code` estável + `request_id`.
3. Paginação por cursor; nunca retornar "tudo".
4. Rate limit documentado (`X-RateLimit-*` headers).
5. Idempotência onde escrita pode ser retentada (ver `IDEMPOTENCY.md`).
6. Autenticação em header; sem credencial em URL.
7. Timestamps em UTC/ISO 8601; dinheiro sem float.
8. Versionamento definido antes do primeiro release.
9. Schema em arquivo sob controle de versão, gera tipos para clientes.
10. Um exemplo funcional por endpoint na doc.
