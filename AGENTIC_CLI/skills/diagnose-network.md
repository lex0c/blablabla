---
name:        diagnose-network
description: Locate the failure point in a network systematically — physical, config, logs, segment isolation.
version:     1
trigger_keywords: [network, connectivity, ping, timeout, offline, dns, unreachable]
tools:       [bash]
source:      project_shared
created_at:  2026-05-21
updated_at:  2026-05-21
expires:     null
---

## When to use

Goal-shape: "diagnose this network failure", "X can't reach Y", "the connection dropped / is slow". Use when the symptom is about **connectivity or reachability**, not application logic.

Not a use case: an application error that still returns an HTTP response (the network worked — check app logs); database query latency (use `pg-blocked-sessions`); suspected malicious traffic on an established connection (that's monitoring, not failure diagnosis).

## Prerequisites

- Shell access on the affected host (ideally also a healthy host for comparison).
- A concrete target that should be reachable (IP, hostname, port).

## Procedure

1. **Define the problem.** Confirm what fails (whole internet? one host? one port?) and the scope (one user or many). Broad scope points to shared infra; narrow scope points to the affected host.
2. **Physical / link layer.** `ip link` — is the interface `UP`? `ip addr` — does it have the expected IP? If not, the problem is local and the later steps are moot.
3. **Active diagnosis, near to far:**
   - `ping <gateway>` → is the local link ok?
   - `ping 1.1.1.1` → is routing/egress ok? (raw IP isolates DNS)
   - `ping <host>` by name → if IP works but name doesn't, it's **DNS**.
   - `traceroute <host>` → where the path dies.
   - `ss -tunap` / `nc -vz <host> <port>` → does the target port accept connections?
4. **Check configuration.** IP, netmask, gateway (`ip route`), DNS (`resolvectl status` or `/etc/resolv.conf`). Check the firewall (`iptables -L -n` / `nft list ruleset` / `ufw status`).
5. **Examine logs.** `journalctl -u NetworkManager`, router/firewall logs — errors or drops correlated with when the symptom started.
6. **Isolate by segment.** Test each hop separately, from the affected user toward the resource, until you find the first point where it breaks.
7. **Fix** the isolated point (hardware, config, or firewall rule) — one change at a time.

## Verification

- Re-run the step 3 test that was failing: it now completes.
- `ping`/`traceroute` reach the target; `nc -vz` on the port succeeds.
- Confirm with the originally affected user, not just from the test host.

## Anti-cases

- Intermittent symptom under load → not a fixed-point failure; it's capacity/saturation — instrument before "fixing".
- "Slowness" with no packet loss → likely an application issue or slow DNS, not routing.
