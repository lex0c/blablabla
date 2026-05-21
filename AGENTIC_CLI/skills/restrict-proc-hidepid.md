---
name:        restrict-proc-hidepid
description: Harden a Linux host by mounting /proc with hidepid so users cannot see each other's processes.
version:     1
trigger_keywords: [hidepid, proc, hardening, process visibility, linux, fstab]
tools:       [bash, edit]
source:      project_shared
created_at:  2026-05-21
updated_at:  2026-05-21
expires:     null
---

## When to use

Goal-shape: "stop users from seeing each other's processes", "harden /proc", "hide process command lines on a multi-user host". Use on shared/multi-tenant Linux hosts where process arguments may leak secrets.

Not a use case: single-user machines (no benefit); containers where `/proc` is managed by the runtime; hiding processes from root (hidepid does not).

## Prerequisites

- Root access on the target host.
- Know the modes: `hidepid=0` (default, all visible), `hidepid=1` (others' dirs exist but their details are hidden), `hidepid=2` (others' process dirs are fully hidden). `hidepid=2` is the hardened choice.
- Optional: `gid=<group>` mount option exempts a monitoring group from the restriction.

## Procedure

1. **Edit `/etc/fstab`** to make the change persistent. Add or modify the `proc` line:
   ```
   proc /proc proc defaults,hidepid=2 0 0
   ```
2. **Remount** to apply without rebooting:
   ```bash
   sudo mount -o remount /proc
   ```
3. **Verify** as a non-root user (see Verification below).
4. **Reboot** once to confirm the fstab entry is correct and the setting survives:
   ```bash
   sudo reboot
   ```

## Verification

- As a non-root user, `ps aux` shows only that user's own processes (plus root's, depending on mode).
- `cat /proc/<pid-of-another-user>/cmdline` is denied or empty.
- `findmnt /proc` shows `hidepid=2` among the mount options.
- After the reboot, re-check `findmnt /proc` — the option is still present.

## Anti-cases

- A monitoring agent (node_exporter, etc.) breaks after enabling → it needs process visibility; add `gid=<monitoring-group>` to the mount options instead of reverting.
- Expecting hidepid to hide processes from root → it does not; root always sees everything.
