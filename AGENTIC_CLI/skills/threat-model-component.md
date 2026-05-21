---
name:        threat-model-component
description: Threat-model a system or component with a DFD + STRIDE, prioritize threats, and decide a mitigation per threat.
version:     1
trigger_keywords: [threat model, stride, dfd, security review, trust boundary, attack surface]
tools:       [edit]
requires:    [THREAT_MODELING]
source:      project_shared
created_at:  2026-05-21
updated_at:  2026-05-21
expires:     null
---

## When to use

Goal-shape: "threat-model this", "what can go wrong with this design", "review the attack surface of this feature". Best applied at **design time** or on a PR that touches a trust boundary (auth, input handling, crypto, tenant isolation).

Not a use case: finding specific bugs in running code (that's pentest / `security-review`); a generic OWASP Top 10 checklist with no concrete system in front of you — that's security theater. This skill produces *decisions about threats*, not secure code by itself.

## Prerequisites

- A concrete component/system to model — a feature, service, or data flow, not "the whole company".
- Knowledge of its assets (what has value), actors (who might attack), and where it stores/moves data.
- A place to write the output (e.g. `docs/threat-model.md`, versioned with the code).

## Procedure

Four guiding questions (Shostack): *What are we building? What can go wrong? What do we do about it? Did we do a good job?*

1. **Scope it.** List assets (PII, secrets, availability, reputation), actors (authenticated abuser, external attacker, insider, supply chain), and trust boundaries (internet→DMZ, user→admin, tenant A→tenant B, service→DB).
2. **Draw the DFD.** Processes (squares), data stores (cylinders), data flows (arrows), trust boundaries (dashed lines). Ugly is fine; concrete is mandatory.
3. **Apply STRIDE per element** — list plausible threats, don't filter yet:
   - Processes: all six (Spoofing, Tampering, Repudiation, Information disclosure, DoS, Elevation of privilege).
   - Data stores: T, I, R, D.
   - Data flows: T, I, D.
   - External entities: S, R.
4. **Prioritize.** Each threat gets *likelihood × impact* — simple high/med/low is enough.
5. **Decide a mitigation** per prioritized threat — exactly one of: **mitigate** (control/redesign), **transfer** (insurance, third party), **accept** (low risk, high fix cost), **eliminate** (drop the feature).
6. **Write the conclusion**: threat → decision → owner → deadline. Commit it next to the code.
7. **Schedule a revisit** — threat models decay; review on each major architectural change.

## Verification

- Every DFD element has been run through the STRIDE categories that apply to it — no element skipped.
- Every prioritized threat has an explicit decision with a named owner (not "TODO").
- The document is committed in the repo, not left on a whiteboard photo.
- A mitigation claimed as "done" (e.g. rate limiting) has a test or evidence — not just an assertion.

## Anti-cases

- Enumerating the generic OWASP Top 10 without grounding it in the DFD → security theater; restart from step 2.
- Modeling only the external attacker → insiders and supply chain are real; cover them.
- Expensive control for a low-likelihood, low-impact threat → overengineering; the mitigation cost is part of the risk math.
- Treating DoS/availability as "not security" → it is; include capacity and rate-limiting threats.
