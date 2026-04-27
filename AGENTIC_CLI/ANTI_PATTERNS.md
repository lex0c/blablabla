# ANTI_PATTERNS

Lista canônica de padrões que o `AGENTIC_CLI` **decidiu não fazer**, com motivo. Toda decisão aqui é uma **âncora contra scope creep**: se aparecer PR sugerindo um destes, este doc é o link de fechamento.

Anti-patterns são tão load-bearing quanto princípios. Princípios definem o que perseguir; anti-patterns definem o que rejeitar mesmo quando parece útil. Sem este doc, cada implementador re-discute do zero, e o protocolo erode.

Cada item: **o que é → por que rejeita → quando reconsiderar**.

---

## 1. Identity / persona

### 1.1 Undercover mode (não-se-apresentar-como-IA)

**O que é:** modo onde o agente se passa por humano em interações (chat, email, comentários).

**Por que rejeita:**
- Conflita com princípio 11 ("Confiança explícita, nunca implícita").
- Quebra `[system] identity / role marker` (`CONTEXT_TUNING.md §1.2`) — modelo recebe instrução contraditória, qualidade cai silenciosamente.
- Audit trail fica inverificável: se o agente mente sobre o que é, nada que ele declara é confiável.
- Risco regulatório (EU AI Act art. 52, similar em outras jurisdições).

**Quando reconsiderar:** nunca em escopo do projeto. Casos legítimos (red team, pentest) são tools dedicadas com policy explícita, não modo do agente.

### 1.2 Persona tuning ("you are an expert X who...")

**O que é:** prompts longos descrevendo personalidade, expertise, tom emocional do agente.

**Por que rejeita:**
- `CONTEXT_TUNING.md §1.2` declara **role-as-tool, não persona**. Persona consome tokens estáveis (cache breakpoint #1) sem retorno mensurável.
- Eval mostra que constraints **negativas** (§1.6) afetam mais comportamento do que afirmações de identidade.
- Persona vira teatro: dois implementadores escrevem dois textos diferentes, ambos passam eval, regressão fica invisível.

**Quando reconsiderar:** se eval comparativo (`PERFORMANCE.md`) mostrar que persona melhora ≥ 5% em workflow específico, com p < 0.05. Até lá, role marker curto (3-5 linhas) é o teto.

### 1.3 Verbosity drift como tuning

**O que é:** ajustar verbosity do output via instruções vagas no system prompt ("be concise", "explain thoroughly").

**Por que rejeita:**
- Drift silencioso entre versões de modelo (Anthropic admitiu publicamente em 2026 que Claude Code regressou em verbosity sem mudança intencional).
- Não-mensurável sem eval dedicado.
- Sintoma de não saber qual é o output esperado.

**Substituição:** `CONTEXT_TUNING.md §1.5` (output format expectations literais) + `TOKEN_TUNING.md` (output budget hard cap). Verbosity é consequência de schema + budget, não de adjetivo.

**Quando reconsiderar:** nunca como adjetivo. Sempre como schema ou budget.

---

## 2. Prompts e knowledge

### 2.1 Prompt-as-IP (system prompt secreto)

**O que é:** tratar system prompts como propriedade intelectual fechada, não versionada, não auditável.

**Por que rejeita:**
- Princípio 8 ("permissões e hooks como dado, não como `if`") se estende a prompts.
- Sem versionamento (`AUDIT.md` §prompt_versions), regressão de qualidade não é rastreável a um commit.
- Open-source: prompts ficam em `~/.config/agent/playbooks/*.md` versionáveis e diff-áveis.

**Inversão consciente:** o valor migra de **prompt-quality** para **prompt-curation-workflow** (eval-driven tuning, ver `TOKEN_TUNING.md`).

**Quando reconsiderar:** nunca em escopo do projeto. Se um usuário quer manter playbook próprio privado, isso é um arquivo em `.agent/` gitignored — escolha do usuário, não default.

### 2.2 Vector DB no v1

**O que é:** indexar code/memory/docs em embedding store (Pinecone, Chroma, etc) e fazer similarity search.

**Por que rejeita:**
- Princípio 12 ("sem cargo cult"). `MEMORY.md §0.9` declara: markdown auditável, diff-able, grep-able vence embedding pra os tamanhos típicos (~1500 linhas de memória).
- Embedding = solução pra problema errado quando dataset é < 100k docs ou estrutura é navegável (filesystem, repo).
- Custo: dependência runtime + reindex + drift entre embedding e fonte.

**Quando reconsiderar:** quando dor real existir (memória > 50k entries, busca lexical falha em > 20% dos casos medidos). Não antes.

### 2.3 Multi-model router por task

**O que é:** routing automático entre modelos por tipo de task ("usa Haiku pra X, Opus pra Y").

**Por que rejeita:**
- Princípio 12. Eval-driven choice por playbook (`PROVIDERS.md`) é mais simples e auditável.
- Router dinâmico vira black box: por que escolheu Haiku pra esse turno? Reproduzível?
- Otimização prematura: a heurística é sempre pior que a escolha consciente do usuário no playbook.

**Quando reconsiderar:** quando ≥ 4 playbooks tiverem o mesmo padrão de routing manual e eval mostrar economia ≥ 30% em custo sem regressão de qualidade.

---

## 3. Loop e orquestração

### 3.1 Auto-retry infinito

**O que é:** loop que re-tenta tool failure sem teto até "dar certo".

**Por que rejeita:**
- `ORCHESTRATION.md §1.6` (loop degenerado detection) e `maxToolErrors=5` existem porque retry sem teto é como o orçamento de tokens evapora silenciosamente.
- Mascara bugs: se `bash` falha 50x com mesmo erro, o problema é a invocação, não a flakiness.

**Substituição:** retry com backoff e teto explícito; após teto, falha vira input pro modelo decidir.

### 3.2 Auto-commit pós-edit

**O que é:** agente commita automaticamente após cada edit "porque parece útil".

**Por que rejeita:**
- Princípio 11. Commit é ação compartilhada, não ação do worktree do usuário.
- Memória do projeto: feedback registrado é "só commitar quando pedido explícito; nunca após criar/editar".
- Reverter commits feitos sem permissão é caro; não-commitar é trivialmente reversível.

**Quando reconsiderar:** modo `--auto-commit` opt-in explícito, com policy declarada. Nunca como default.

### 3.3 Stateful tools (tool com memória entre invocações)

**O que é:** tool que mantém estado interno entre chamadas (conexão aberta, cursor, contador).

**Por que rejeita:**
- `CONTRACTS.md §2`: "sem state global mutável entre invocações (tool é função, não classe com estado)".
- Quebra replay (`AGENTIC_CLI.md` princípio 7): mesmo input → mesmo output só vale se tool é pura.
- Side effects implícitos viram bugs intermitentes.

**Substituição:** estado externalizado em SQLite ou FS, tool lê/escreve explicitamente.

---

## 4. UX e output

### 4.1 Spinner sem informação

**O que é:** "Thinking..." / "Working..." sem dizer **o quê**.

**Por que rejeita:**
- `UI.md` e `DESIGN_SYSTEM.md` exigem progress label informativo. Spinner mudo é UI placebo.
- Em sessão > 10s, usuário precisa saber se vale interromper.

**Substituição:** label do step atual ("running tests...", "reading src/auth.ts...").

### 4.2 Markdown rico em --json

**O que é:** modo `--json` retornando strings com markdown embedado (asteriscos, backticks).

**Por que rejeita:**
- `AGENTIC_CLI.md §2.2`: "em modo `--json`, **nada** vai pra stdout que não seja JSONL válido". Markdown vira lixo pro consumidor.
- Quebra pipe-ability (princípio 9).

**Substituição:** campos estruturados (`code: string`, `language: string`); cliente renderiza.

### 4.3 Disclaimers, prefácios, "I'll start by..."

**O que é:** prosa decorativa antes/depois da ação real ("Sure! I'll help you with that. Let me start by...").

**Por que rejeita:**
- `CONTEXT_TUNING.md §1.5`: "Sem prefácio. Sem disclaimers. Sem 'I'll start by...'. Aja."
- Custa tokens estáveis no output, polui scrollback, atrasa primeira ação útil.
- Sintoma de RLHF mal calibrado herdado do modelo base.

**Substituição:** uma frase de status quando há mudança de direção; senão, ferramenta direto.

---

## 5. Audit e segurança

### 5.1 Logging silencioso de PII

**O que é:** gravar tool inputs/outputs crus em audit sem redação.

**Por que rejeita:**
- `AUDIT.md` exige PII redaction **antes-de-persistir**, não no read-time.
- Vazamento via backup, replay, ou recap é vetor real.

**Substituição:** redactor declarativo por tabela (já especificado em `AUDIT.md`).

### 5.2 Trust transitivo

**O que é:** "este diretório é confiável → tudo dentro dele é confiável".

**Por que rejeita:**
- `CLAUDE.md` é input não-confiável (princípio 11) mesmo dentro de diretório confiado — pode vir de fork, paste, supply chain.
- Trust em `AGENTIC_CLI` é por **superfície** (FS read, FS write, network, exec), não por path.

**Substituição:** policy YAML com globs explícitos (`SECURITY_GUIDELINE.md`).

---

## 6. Quando este doc muda

Este doc é deliberadamente **append-mostly**. Remover anti-pattern requer:
1. PR com motivo (eval, mudança de premissa, ou bug do anti-pattern original).
2. Atualização do princípio relacionado em `AGENTIC_CLI.md` se aplicável.
3. Migração documentada se houver implementação afetada.

Adicionar anti-pattern requer apenas: nome, motivo, substituição. Mais fácil rejeitar que reconsiderar — by design.
