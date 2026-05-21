---
name:        ssh-tunnel
description: Forward a port over SSH — local (-L), remote (-R), or dynamic SOCKS (-D) — and lock forwarding down server-side.
version:     1
trigger_keywords: [ssh, tunnel, port forward, socks, -L, -R, -D, jump host]
tools:       [bash, edit]
source:      project_shared
created_at:  2026-05-21
updated_at:  2026-05-21
expires:     null
---

## When to use

Goal-shape: "reach an internal port from outside", "expose my local server through a server", "route traffic through an SSH host", "stop users from tunneling". Pick the direction first:

- Reach a service that only the SSH host can see → **local forward** (step 1).
- Let the SSH host (or its network) reach a service on your machine → **remote forward** (step 2).
- Route arbitrary app traffic through the SSH host → **dynamic SOCKS** (step 3).
- Harden the server against tunneling → **step 4**.

Not a use case: a persistent site-to-site link (use a VPN/WireGuard — `TUNNELING.md`); exposing a service to the public internet (use a reverse proxy).

## Prerequisites

- SSH access (`user@host`) to the intermediary, with a working key or password.
- The target host/port reachable *from the relevant side* of the tunnel.
- For step 4: root or sudo on the SSH server.

## Procedure

1. **Local forward** — local port → service reachable by the SSH host:
   ```bash
   ssh -L 9090:localhost:8080 user@sshserver
   ```
   Traffic to `localhost:9090` exits the SSH host toward `localhost:8080`. The `localhost` is resolved *on the SSH host*.
2. **Remote forward** — port on the SSH host → service on your machine:
   ```bash
   ssh -R 8080:localhost:3000 user@sshserver
   ```
   Connections to `sshserver:8080` reach your local `:3000`. To bind on all interfaces of the server, it needs `GatewayPorts yes`.
3. **Dynamic SOCKS proxy** — route any app through the SSH host:
   ```bash
   ssh -D 1080 user@sshserver
   ```
   Point the app (browser, etc.) at SOCKS5 `localhost:1080`.
   Add `-N` (no shell) and `-f` (background) for a tunnel-only session.
4. **Disable forwarding server-side.** Edit `/etc/ssh/sshd_config`:
   - All forwarding off: `AllowTcpForwarding no` and `AllowStreamLocalForwarding no`
   - SOCKS/dynamic only: `PermitOpen none`
   - Per user/group/address: scope it with a `Match User|Group|Address` block.
   Then `sudo systemctl restart sshd`.

## Verification

- Local/remote forward: `nc -vz localhost <forwarded-port>` succeeds, or the app connects.
- SOCKS: `curl --socks5 localhost:1080 https://ifconfig.me` returns the SSH host's IP, not yours.
- After step 4: a forwarding attempt is refused; confirm with `journalctl -u sshd` / `/var/log/auth.log` for "administratively prohibited".

## Anti-cases

- Tunnel drops on idle → not a config bug; add `ServerAliveInterval`/`ExitOnForwardFailure` or use `autossh`.
- Remote forward "works" but only from the server itself → that's default behavior without `GatewayPorts`; intended, not a fault.
