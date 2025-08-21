# Software Security Guideline

If your code were guarding nuclear launch codes, you wouldn’t secure it with a sticky note password. You’re not building Fort Knox, but at least don’t leave the door open with a neon sign that says “Free Data.” These rules will keep your software from being the low-hanging fruit in the hacker orchard.

Follow these rules if you want your codebase to go from “hack me for fun” to “meh, too much work”.

1. **All input is hostile**
   Validate and sanitize every byte of input. Schemas, type checks, length limits. Treat user data like it’s laced with anthrax.

2. **Authentication is more than a login form**
   Use proven libraries. No DIY password hashing. Rotate tokens, support MFA, and assume someone will steal your cookie.

3. **Authorization must be explicit**
   Default deny. Enforce permissions server-side, every time. No `if (isAdmin)` scattered like confetti.

4. **Secrets belong in vaults, not in code**
   Use a secret manager. Rotate keys. Block secrets from ever reaching Git, logs, or Slack.

5. **Cryptography is not DIY craft time**
   Use TLS everywhere, hash passwords with Argon2 or bcrypt, and never invent your own crypto “because it’s fun.”

6. **Files and uploads are bombs until proven otherwise**
   Check type, size, extension, and magic bytes. Store outside the web root. Assume someone will upload a PHP shell disguised as a JPEG.

7. **Rate limit like your uptime depends on it**
   Limit by IP, account, and endpoint. Special rules for login, password reset, and other juicy targets. Abuse must be expensive.

8. **Browser defenses are mandatory**
   Set a CSP. Lock cookies with HttpOnly, Secure, SameSite. No tokens in localStorage unless you enjoy XSS Russian roulette.

9. **Dependencies are supply chain landmines**
   Pin versions, scan for CVEs, and update regularly. One abandoned package is enough to make you tomorrow’s ransomware headline.

10. **Logs are forensics, not confessions**
    Record security events, but never credentials or PII. Centralize, encrypt, monitor, and alert. If no one’s watching, it didn’t happen.

11. **SQL is not a suggestion box**:
    Never concatenate strings into queries. Use parameterized statements or ORM bindings only. Escape nothing manually. If you’re building SQL by string concatenation, you’re basically handing attackers a Sharpie and telling them to finish your query.
