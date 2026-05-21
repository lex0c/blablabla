---
name:        acquire-forensic-evidence
description: Capture evidence from a host in volatility order, with hashes and chain of custody, working only on copies.
version:     1
trigger_keywords: [forensics, dfir, evidence, acquisition, chain of custody, memory dump, disk image]
tools:       [bash]
requires:    [FORENSICS]
source:      project_shared
created_at:  2026-05-21
updated_at:  2026-05-21
expires:     null
---

## When to use

Goal-shape: "preserve evidence from this host", "image this disk/memory before we touch it", "this incident may need legal defensibility". Use when artifacts from a host you are authorized to examine must be **preserved before** any analysis or cleanup that would alter them.

Not a use case: routine triage where evidence will not be needed (use `investigate-suspicious-host`); systems you are not authorized to image. If in doubt whether evidence is needed — acquire first; you cannot un-destroy it.

## Prerequisites

- Written authorization to acquire from this host.
- Somewhere clean to store images, separate from the source.
- Tools available: `dd`/`dcfldd`/`ewfacquire` (disk), `AVML`/`LiME` (Linux memory), `sha256sum`.
- A log (paper or file) for contemporaneous notes — every action, timestamp, tool, version.

## Procedure

Acquire in **order of volatility** (RFC 3227) — capture what disappears first.

1. **Open the custody log.** Record: who, when, host, authorization reference. Note every command as you run it, not afterward.
2. **Volatile state, in order:**
   - **Memory (RAM)** — disappears on power-off. Capture first:
     ```bash
     sudo avml /evidence/mem.lime          # or LiME
     ```
   - **Network state:** `ss -tunap`, `ip neigh`, `ip route` — redirect each to a file under `/evidence/`.
   - **Running processes:** `ps auxww`, `ls -l /proc/*/exe` → files.
   - **Logged-in users / temp files:** `who`, `w`, listing of `/tmp`, `/var/tmp`.
3. **Disk — work toward a dead, bit-for-bit image:**
   - Prefer **dead acquisition** (powered-off, source disk read-only). Enforce read-only: hardware write blocker, or `sudo blockdev --setro /dev/sdX`.
   - Image bit-for-bit:
     ```bash
     sudo dcfldd if=/dev/sdX of=/evidence/disk.img hash=sha256 hashlog=/evidence/disk.hash
     ```
   - Live acquisition only when unavoidable (encrypted volume mounted, server that cannot stop) — note the reason in the log.
4. **Hash every artifact** immediately after capture, and store the hash separately:
   ```bash
   sha256sum /evidence/* > /evidence/HASHES.txt
   ```
5. **Analyze only copies.** Never touch the original disk or the primary image — duplicate first, verify the copy's hash, work on the duplicate.
6. **Close the custody log:** final inventory of artifacts, their hashes, storage location, and who holds them.

## Verification

- `sha256sum -c /evidence/HASHES.txt` passes — artifacts are intact and unaltered.
- A second hash, taken later, matches the first — integrity holds over time.
- The custody log accounts for every artifact: how acquired, when, by whom, with which tool version.
- The source was demonstrably read-only during disk acquisition (write blocker / `blockdev --setro` recorded).

## Anti-cases

- Running analysis tools on the original → alters timestamps and state; image first, analyze the copy.
- Capturing disk before memory → RAM is gone by the time you get there; follow volatility order.
- No hash at capture time → integrity is unprovable later; the evidence is not defensible.
- Acquiring without written authorization → the evidence and the act itself are compromised.
