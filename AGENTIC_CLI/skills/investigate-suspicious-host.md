---
name:        investigate-suspicious-host
description: Triage a possibly-compromised Linux host — connections, owning process, persistence, logs — and conclude with IOCs.
version:     1
trigger_keywords: [compromise, suspicious, intrusion, malware, beacon, triage, incident, ioc]
tools:       [bash]
requires:    [NETWORK_MONITORING]
source:      project_shared
created_at:  2026-05-21
updated_at:  2026-05-21
expires:     null
---

## When to use

Goal-shape: "investigate this suspicious connection/process", "is this host compromised", "triage this alert". Use for **defensive triage** of a host you own or are authorized to inspect — confirm or rule out compromise and produce indicators of compromise (IOCs).

Not a use case: attacking or scanning hosts you do not own; full forensic acquisition for legal use (use `acquire-forensic-evidence` — and do *that* before destructive cleanup if evidence matters); long-term monitoring setup.

## Prerequisites

- Authorized access to the host (root/sudo gives full process and socket visibility).
- A baseline notion of what is *normal* for this host — without it, "unusual" is unmeasurable.
- If evidence may be needed legally: stop, run `acquire-forensic-evidence` first — triage commands alter state.

## Procedure

1. **Active connections.** List sockets and spot what does not belong:
   ```bash
   sudo ss -tunap                       # all TCP/UDP sockets + owning process
   sudo ss -tnp state established       # established connections only
   ```
   Flag: unknown remote IPs, high/non-standard ports, beaconing (regular interval to one IP), unexpected listeners.
2. **The process behind a suspect socket.** `ss -p` gives the PID; expand it:
   ```bash
   ps -p <PID> -f
   cat /proc/<PID>/status /proc/<PID>/cmdline
   ls -l /proc/<PID>/exe /proc/<PID>/cwd      # real binary path; deleted exe is a red flag
   ```
3. **Persistence — how it survives reboot.** Check the common footholds:
   ```bash
   for u in $(cut -f1 -d: /etc/passwd); do crontab -l -u "$u" 2>/dev/null; done
   ls -la /etc/cron.* /etc/cron.d/ ; cat /etc/crontab
   systemctl list-units --type=service --state=running
   systemctl list-timers --all
   ls -la ~/.bashrc ~/.profile /etc/rc.local
   ```
4. **Logs.** Correlate to a timeline:
   ```bash
   sudo grep -E 'Failed|Accepted' /var/log/auth.log | tail -50   # brute force / logins
   sudo journalctl --since "today" -p warning
   last -20 ; lastb -20                                          # login / failed-login history
   ```
5. **Local network layer.** `arp -a` / `ip neigh` — look for ARP anomalies (one MAC for many IPs = possible spoofing).
6. **Conclude.** Write a short verdict: compromised / clean / inconclusive, plus the **IOCs** found — IPs, ports, PIDs, file paths, hashes, timestamps.

## Verification

- Every suspicious socket from step 1 has been traced to a known process and explained, or flagged as an IOC.
- The timeline is consistent: connection start, process start, and log entries line up.
- The conclusion is evidence-backed — each claim cites a command output, not a hunch.

## Anti-cases

- Killing the process / deleting files before collecting evidence → destroys the investigation; if it may be needed legally, `acquire-forensic-evidence` first.
- Calling a connection malicious with no baseline → backups, monitoring agents, and updates all open "unusual" connections; compare against normal.
- Trusting `ps`/`ss` blindly on a host suspected of a rootkit → a kernel rootkit can hide processes; corroborate with `/proc` listing and external network capture.
