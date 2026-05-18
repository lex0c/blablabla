# SELF_UPDATE

Estratégia de atualização do binário `agent` no `AGENTIC_CLI`. Como o runtime é trocado **sem corromper sessão, sem perder audit, sem virar vetor de RCE silenciosa**.

Premissa raiz: *meça duas vezes, corte uma*. Auto-updater é literalmente **remote code execution autorizado** — se a medição (assinatura, checksum, manifest válido, canal correto) falha, o "corte" (swap da versão) tem que **não acontecer**. Update silencioso que pula verificação é a maneira mais elegante de transformar uma CLI confiável em backdoor entregue por CDN.

Estratégia escolhida: **launcher fixo + diretórios de versão + symlink atômico + manifest assinado + swap no próximo launch**. Inspirado em Rustup, Volta, `uv`, e nos updaters de Cursor/VSCode/Claude Code. Diferente deles em uma coisa: toda transição de versão é evento de audit assinado, igual a qualquer outra ação com side effect persistente.

---

## 0. Princípios (não-negociáveis)

1. **Nunca sobrescreva o binário em execução.** Baixa paralelo, troca symlink, aplica no próximo launch. Windows trava arquivo aberto; Unix permite mas resulta em comportamento inconsistente se a sessão paginar páginas do binário antigo após overwrite.
2. **Atomic swap ou nada.** Symlink rename é atomic em POSIX; copy-and-replace **não é**. Update parcial = binário corrompido = perda de confiança maior que ficar uma versão atrás.
3. **Assinatura é obrigatória, checksum é mínimo.** Checksum sozinho protege contra corrupção, não contra ataque. CDN comprometida entrega checksum válido pra binário malicioso.
4. **Update é evento de audit.** Versão antes, versão depois, manifest hash, signer key id, canal, trigger — tudo em `AUDIT.md` append-only.
5. **Package-managed install não faz self-update.** Se instalado via brew/winget/apt/dnf/npm, o updater **detecta e delega**. Mutar pacote gerenciado quebra invariante do PM e gera conflito na próxima reconciliação.
6. **Canal é declarado, nunca inferido.** `stable` por default. Outros canais só com opt-in explícito. Nunca promova canal silenciosamente.
7. **Rollback é first-class.** Versão anterior fica em `versions/` por N releases. `agent rollback` é comando, não procedimento de recovery.
8. **Update nunca interrompe sessão ativa.** Download é background, swap é no próximo launch. Sessão em andamento termina na versão que começou.
9. **Sem auto-update silencioso por default.** Notifica, baixa em background, aplica no restart. Updater que decide sozinho quando reiniciar a CLI é hostile.
10. **Self-update opt-out, não opt-in.** Disponível por default, mas `AGENT_NO_SELF_UPDATE=1` e `--no-update` desligam — ambiente air-gapped, CI determinístico, ou usuário que prefere PM precisam disso.

---

## 1. Arquitetura

### 1.1 Layout de filesystem

```text
~/.agent/
  versions/
    0.4.1/
      agent-runtime    # binário real (Bun compile, Go, Rust — qualquer)
      manifest.json
      manifest.sig
    0.5.0/
      agent-runtime
      manifest.json
      manifest.sig
  current  -> versions/0.5.0    # symlink ativo
  previous -> versions/0.4.1    # rollback target
  channels/
    stable.json        # último manifest do canal escolhido
  state/
    updater.db         # SQLite — histórico, checks, pending swaps, skipped versions
  install-method       # plain text: curl-installer | homebrew | apt | ...
```

### 1.2 Launcher fixo

`/usr/local/bin/agent` (ou `~/.local/bin/agent`) é stub mínimo. Responsabilidades:

1. Resolver `~/.agent/current/agent-runtime`.
2. Verificar assinatura do runtime resolvido (defesa contra tampering no FS).
3. Se houver `state/swap-pending`, executar swap antes de prosseguir.
4. `exec` no runtime, passando argv intacto.
5. **Nada mais.** Sem network, sem update check inline — isso fica no runtime, em background.

Launcher é a **única coisa** que um PM (brew, apt) escreve. Updater só mexe em `~/.agent/`.

### 1.3 Separação launcher ↔ runtime

| Componente | Muda | Quem atualiza |
|---|---|---|
| Launcher (`bin/agent`) | raramente (ABI break, novo path resolver, key rotation) | package manager OU instalador inicial |
| Runtime (`versions/X.Y.Z/`) | toda release | self-updater |
| Plugins/playbooks | independente | mecanismo próprio (ver §7) |

Isso é a razão de "binário único" funcionar: o launcher é tão simples que **não precisa atualizar** com frequência.

---

## 2. Update flow

### 2.1 Check

```text
startup (background, após handshake do provider)
  → fetch channels/stable.json (HEAD com If-Modified-Since)
  → if 304: done
  → if 200:
      verify signature
      compare semver com current
      if newer e não está em skipped: enqueue download
```

Check é assíncrono, **nunca bloqueia primeiro prompt**. Falha de network → log de DEBUG, sem retry agressivo. Próximo startup tenta de novo.

### 2.2 Download

```text
download → ~/.agent/versions/.staging/X.Y.Z/
  → verify sha256
  → verify signature (Ed25519, chave pinned no launcher)
  → atomic mv → ~/.agent/versions/X.Y.Z/
  → write state/swap-pending marker
  → notify UI: "Update X.Y.Z ready. Restart to apply."
```

Staging dir separado evita que swap parcial polua `versions/`. Nada vira "versão instalada" antes de **assinatura validada + atomic rename**.

### 2.3 Swap

Acontece em **dois pontos**, nunca durante sessão:

- **Próximo launch** se há `swap-pending` marker. Launcher resolve marker antes de `exec`.
- **Comando explícito** `agent update --apply` (encerra sessão atual com warning + checkpoint).

Swap operacional:

```text
read state/swap-pending  → versão alvo
ln -sfn versions/X.Y.Z current.new
mv current.new current     # POSIX atomic rename via renameat2
mv current previous        # se previous existia, GC após N gerações
clear swap-pending
emit audit event
```

`ln -sfn` + `mv` é o padrão atomic-symlink-update. Em Windows: junctions + `MoveFileEx(MOVEFILE_REPLACE_EXISTING)`.

### 2.4 Rollback

```text
agent rollback                     # volta para previous
agent rollback --to 0.4.0          # volta para versão específica
agent versions                     # lista instaladas
agent versions --prune --keep 3    # GC manual
```

Rollback é **mesma mecânica do swap**, só com target invertido. Audit registra `trigger: rollback` com motivo opcional (`--reason "regressão em pipeline X"`).

---

## 3. Manifest e assinatura

### 3.1 Manifest do canal

```json
{
  "channel": "stable",
  "released_at": "2026-05-14T12:00:00Z",
  "latest": "0.5.0",
  "versions": [
    {
      "version": "0.5.0",
      "url": "https://releases.agent.dev/0.5.0/agent-linux-x64",
      "sha256": "abc123...",
      "size": 28442112,
      "min_launcher": "0.1.0",
      "deprecates": ["0.4.x"],
      "notes_url": "https://..."
    }
  ]
}
```

Manifest é assinado (detached, `.sig`). Updater valida com **chave pública embarcada no launcher** — não com chave baixada (chicken-and-egg).

### 3.2 Key rotation

Launcher carrega **até 3 chaves** simultaneamente (current + 2 rotation slots). Mudança de chave é versão de launcher, distribuída via PM ou reinstall. **Nunca** atualize chave via self-update — esse é o golden path do supply chain attack.

### 3.3 Pinning

Sessão pode pinar versão: `agent --version 0.4.1 ...`. Útil para:
- Reproduzir bug em versão específica.
- CI que quer determinismo absoluto.
- Eval comparando duas versões.

Pin não move `current`; só seleciona ad-hoc.

---

## 4. Canais

Em V1, apenas **`stable`**. Mudança de canal entra em V2 (ver §13).

Default declarado em `~/.agent/config.toml`:

```toml
[updater]
channel = "stable"
auto_check = true
auto_download = true
```

Nunca promova canal por heurística (e.g., "user has flag X enabled, switch to nightly"). Canal é configuração consciente — ver `FEATURE_FLAGS.md §1.2`.

---

## 5. Package-managed installs

Detecção via metadata gravada no install inicial:

```text
~/.agent/install-method
  homebrew | winget | apt | dnf | npm-global | manual | curl-installer
```

Comportamento do updater por método:

| Método | Auto-check | Auto-download | Auto-apply | `agent update` |
|---|---|---|---|---|
| `manual`, `curl-installer` | ✓ | ✓ | swap pending | full self-update |
| `homebrew` | ✓ (version check) | ✗ | ✗ | sugere `brew upgrade agent` |
| `winget`, `apt`, `dnf` | ✓ | ✗ | ✗ | sugere comando do PM, não executa |
| `npm-global` | ⚠ desabilitado | ✗ | ✗ | sugere reinstalar via curl |

`npm-global` é caso especial: histórico de quebrar em permissões, PATH, e versões parciais. Manter como **install method suportado mas sem self-update** evita classe inteira de bugs.

---

## 6. Background updates

UX alvo (notificação no startup, **depois** do primeiro prompt do usuário, não antes):

```text
Update available: 0.4.1 → 0.5.0  (stable)
Downloading in background...
Signature verified.
Update will apply on next launch.

  [v] view changelog       [s] skip this version       [a] apply now (restart)
```

Regras:

- Download só durante **idle** (sem step ativo, sem tool em flight).
- Aborta se rede mudar (laptop em hotspot pago → não termina download de 30MB).
- `skip` registra versão pulada em `state/updater.db`; não pergunta de novo até nova release.
- `apply now` warning explícito: "Sessão atual será encerrada. Continuar? (checkpoint salvo)".

---

## 7. O que **não** atualiza via self-update

| Componente | Mecanismo |
|---|---|
| Playbooks (subagents) | versionados em `.claude/` ou repo do usuário |
| Plugins/MCP servers | ver `MCP.md` — capability negotiation + hash |
| Configuração (`~/.agent/config.toml`) | nunca tocada pelo updater |
| Memória, audit log, sessões | nunca |
| Chaves de API, credenciais | nunca |
| Launcher (`bin/agent`) | package manager ou reinstall manual |

Update mexe **só** em `versions/`. Tudo mais é dado do usuário.

---

## 8. Audit integration

Cada transição é evento append-only (ver `AUDIT.md`):

```json
{
  "event": "self_update.applied",
  "ts": "2026-05-14T08:42:17Z",
  "from": "0.4.1",
  "to": "0.5.0",
  "channel": "stable",
  "manifest_sha256": "...",
  "signer_key_id": "ed25519/2026-01",
  "trigger": "next_launch_swap",
  "session_id": null,
  "install_method": "curl-installer"
}
```

Eventos cobertos:
- `self_update.check` (com `result: up_to_date | newer_found | error`)
- `self_update.download_started` / `download_completed` / `download_failed`
- `self_update.signature_failed` (**alerta de segurança**, ver `SECURITY_GUIDELINE.md`)
- `self_update.applied`
- `self_update.rollback`
- `self_update.skipped` (com versão e motivo)

`signature_failed` **não é warning** — é incident. Updater desabilita auto-download até intervenção manual.

---

## 9. Permission integration

Self-update toca FS fora de `~/.agent/`? **Não.** Toca rede? **Sim** (manifest, download). Logo:

- Updater roda em **escopo de permission próprio**, declarado em `PERMISSION_ENGINE.md`.
- Endpoints permitidos: lista pin de hosts de release (ex.: `releases.agent.dev`). Redirect cross-origin **rejeitado**.
- Se policy do usuário desabilita network sem prompt, updater check vira opt-in manual via `agent update --check`.

---

## 10. CLI surface

```bash
agent update                    # check + download em foreground se TTY (background sem TTY)
agent update --check            # só check, sem download
agent update --apply            # força swap agora (warning, encerra sessão se ativa)
agent update --no-update        # bypass total nesta invocação
agent rollback                  # volta para previous
agent rollback --to 0.4.0
agent versions                  # lista instaladas + current + previous
agent versions --prune --keep 3
agent --version                 # versão do runtime
agent --version --verbose       # runtime + launcher + canal + install method + última checagem
```

Env vars:
- `AGENT_NO_SELF_UPDATE=1` — desliga toda lógica de update (check, download, swap).
- `AGENT_UPDATE_CHANNEL=stable` — override do canal por sessão.

---

## 11. Failure modes (ver `FAILURE_MODES.md`)

| Falha | Detecção | Recovery |
|---|---|---|
| Download corrompido | sha256 mismatch | descarta staging, retry uma vez, depois desiste |
| Assinatura inválida | sig verify fail | **incident**: desabilita auto-update, registra audit, exige `--force-signature-bypass` opt-in com warning crítico |
| Manifest com clock no futuro (>24h) | `released_at` no futuro | warning, mas não bloqueia (pode ser clock errado local) |
| Swap em FS read-only | `mv` falha | log + notify; sessão segue na versão atual |
| Permissões erradas em `~/.agent/` | acesso negado em check | fallback pra read-only mode (sem update, sem state writes) |
| Disk full durante download | `ENOSPC` | aborta, mantém versão atual, sugere `agent versions --prune` |

---

## 12. Anti-patterns

### ❌ `auto-update` silencioso durante sessão

Mata sessão sem aviso, perde contexto, viola princípio 8.

### ❌ overwrite in-place do binário

Windows: file lock. Unix: páginas paginadas do binário antigo causam crash imprevisível. Use rename atomic ou nada.

### ❌ checksum sem assinatura

Checksum protege contra corrupção em trânsito. **Não** protege contra CDN comprometida — atacante recalcula checksum trivialmente.

### ❌ chave de assinatura distribuída via self-update

Chicken-and-egg. Comprometeu chave → updater aceita chave nova maliciosa → game over. Chave **só** vem com launcher.

### ❌ canal auto-promovido por heurística

"Você usa flag experimental, vou te colocar em canal nightly." Não. Canal é decisão consciente — ver `FEATURE_FLAGS.md`.

### ❌ self-update em install gerenciado por package manager

Conflita com PM, gera versão dual, próxima `brew upgrade` desfaz. Detecte e delegue.

### ❌ rollback como procedimento manual oculto

Se rollback exige editar symlink na mão, ninguém faz quando precisa. `agent rollback` é comando first-class.

### ❌ delta updates como V1

Tentador, complexidade gigante. Versões inteiras + atomic swap é simples e correto. Delta vira V3 quando o tamanho do binário justificar — ver §13.

### ❌ "background updater daemon"

Processo separado que roda fora da CLI é vetor de complexidade: lifecycle, permissões, telemetria, init system. Updater roda **dentro** do processo principal, em idle.

### ❌ auto-rollback em crash sem evidência

Tentação: "runtime novo crashou 3x, volta sozinho". Problema: crash pode ser do provider, da rede, do prompt do usuário — não da nova versão. Auto-rollback mascara bug real. Em V1, rollback é **sempre manual** com `agent rollback`.

---

## 13. Roadmap

### V1 (MVP)

- Launcher fixo + `versions/` + symlink `current`/`previous`.
- Manifest assinado (Ed25519), chave pinned no launcher.
- Canal `stable` only.
- Check no startup, download background, swap no próximo launch.
- `agent update`, `agent rollback`, `agent versions`.
- Detecção de install method, delegação pra PMs.
- Audit completo.

### V2

- Canais `nightly` e `experimental` com opt-in explícito.
- `agent update --apply` com checkpoint de sessão.
- Skip versions com persistência granular (skip patch vs skip minor).
- Auto-rollback opt-in em crash repetido com evidência clara (mesmo crash signature 3x em 60s).

### V3

- Delta updates (bsdiff/zstd) para releases minor.
- Plugin runtime separado (runtime vs plugins atualizam independente).
- Multi-launcher coexistence (sistema + user-local).

---

## 14. Insight

A "atualização do binário único mágico" que usuários veem é, na verdade:

> um pequeno gerente de versões disfarçado de executável simples.

Software moderno prefere **trocar versões inteiras de forma atômica** a **mutar o binário em execução**. A primeira estratégia tem invariantes simples (assinatura, atomic rename, rollback óbvio). A segunda tem zero invariantes e infinitos bugs.

A CLI agentic herda essa decisão arquitetural sem cerimônia: launcher fixo, versões em diretórios, symlink atômico, manifest assinado, audit append-only. Tudo o que aparenta ser "atualizador inteligente" é, no fundo, **um symlink que se move sob assinatura verificada**.
