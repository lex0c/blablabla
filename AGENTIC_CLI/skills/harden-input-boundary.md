---
name:        harden-input-boundary
description: Make untrusted input safe at the point it reaches a sink (SQL, shell, path, HTML, deserializer) using the sink-correct defense.
version:     1
trigger_keywords: [input validation, sanitize, sql injection, command injection, escape, untrusted input, boundary, injection]
tools:       [bash, edit]
requires:    [SOFTWARE_SECURITY_GUIDELINE, WEB_SECURITY]
source:      project_shared
created_at:  2026-05-21
updated_at:  2026-05-21
expires:     null
---

## When to use

Goal-shape: "this takes user input", "is this safe from injection", "sanitize this", or any code where untrusted data reaches an interpreter/sink. Apply on the lines where untrusted data crosses into a sink — not as a generic audit.

Not a use case: threat-modeling a whole design (`threat-model-component`); authentication/authorization logic (a different concern); finding a known exploit in running code (pentest).

## Prerequisites

- Knowledge of which data is untrusted (anything that originated outside your trust boundary — see step 1).
- The specific sink the data reaches, and the ability to edit and run the code.

## Procedure

The defense depends on the **sink**, not the source. Validation alone is never the primary fix.

1. **Find the boundary and the sink.** Trace untrusted data — HTTP param/body/header/cookie, file content, IPC/queue message, DB content that originated from a user, third-party API response — to where it is *used*: a query, a command, a path, markup, a deserializer.
2. **Classify the sink and apply its defense:**
   - **SQL / NoSQL / ORM raw** → parameterized query / prepared statement. Never string-concatenate values into the query.
   - **Shell / OS command** → avoid the shell; pass an argv array to exec. Never interpolate into a command string. If a shell is unavoidable, allowlist the value.
   - **Filesystem path** → canonicalize first (step 4), then assert the result is inside an allowed base directory.
   - **HTML / DOM** → contextual output encoding; rely on framework auto-escaping; never `innerHTML`/raw render of untrusted data.
   - **LDAP / XPath** → library escaping for that grammar; parameterize where supported.
   - **Deserialization** → never deserialize untrusted bytes into rich/typed objects; use a data-only format (JSON) parsed against an explicit schema.
   - **Template engine** → pass user input as *data* to the template; never render it *as* a template.
   - **Regex** → bound input length; avoid patterns with catastrophic backtracking.
3. **Validate at entry as defense in depth.** Allowlist by type, length, range, and format. Prefer *reject* over *sanitize*. This backs up the sink defense — it does not replace it.
4. **Canonicalize before validating.** Decode once, normalize encoding, resolve the path/URL, then validate the canonical form — so a double-encoded or `../`-laden payload can't slip past.
5. **Fail closed.** Reject invalid input with a generic error; do not echo the bad value back; log the rejection.
6. **Cover every path to that sink.** One validated entry point plus three raw ones leaves the sink unprotected — find all callers.

## Verification

- Each untrusted→sink path uses the **sink-correct** defense (parameterize/argv/canonicalize/encode), not only input validation.
- A known malicious payload for that sink — `' OR 1=1--`, `; rm -rf /`, `../../etc/passwd`, `<script>` — is rejected or rendered inert. Tested, not assumed.
- No string concatenation or interpolation builds a query, command, or path from untrusted data.
- Entry validation is an allowlist, not a denylist.

## Anti-cases

- Denylisting "bad characters" → always bypassable by encoding or an unlisted char; allowlist instead.
- Stripping characters instead of parameterizing → reordering or re-encoding defeats it.
- Validating once at a route and trusting the value forever → it is untrusted again at each new sink; defend at the sink.
- Escaping for the wrong context (HTML-escaping a value going into SQL) → zero protection.
- "It's internal, so it's trusted" → DB rows, other services, and env vars routinely carry user-origin data.
