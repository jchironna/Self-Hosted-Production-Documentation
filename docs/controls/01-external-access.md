# 01 — External Access

**Threat addressed:** opportunistic internet-wide scanning and direct exploitation of exposed
services.

## Decision

No persistent inbound port is forwarded at the WAN edge. External reachability is provided by
two paths, chosen by who the consumer is.

| Path | Consumer | Mechanism |
|---|---|---|
| Cloudflare Tunnel | Services shared with other people | `cloudflared` establishes an **outbound** session to Cloudflare's edge; requests arrive over it |
| Tailscale (WireGuard) | Everything administrative | Encrypted mesh between enrolled devices only; nothing published |

## Why this over a reverse proxy behind a forwarded port

A conventional proxy on 443 places a residential IP in every internet-wide scan result and
makes the proxy itself the exposed surface — a single unpatched CVE away from host access. Under
the tunnel model there is no listener on the WAN address to find, so the class of attack that
compromises most self-hosted environments has no entry point. Compromise instead requires
defeating Cloudflare's edge or holding valid credentials, which is a materially different and
much harder problem.

## Published vs. private

Published through the tunnel, behind an Access policy (see [02](02-identity-and-access.md)):

- Media streaming
- Media request portal
- Photo library
- Notes-sync database (service-token authentication; non-browser client)

Never published — mesh-only:

- Unraid management interface
- SSH, Docker socket
- Container management and monitoring interfaces

## Exception: game server ports

A small number of games do not use relayed matchmaking and require a genuine inbound port.
Handling:

1. Tailscale first — used wherever the title tolerates it, which is most of them.
2. Where it does not, a port is opened **time-boxed** and **source-IP restricted** to known peers.
3. Closed when the session ends.

The allowlist is a real control at the perimeter and no control at all past it. Because the LAN
is flat (see [04](04-segmentation.md)), compromise of the game server host has an
unconstrained path onward. This is documented as accepted risk rather than presented as solved.

## Verification

- [ ] External scan of the WAN address returns no open ports in steady state
- [ ] Administrative interfaces unreachable with the mesh client disabled
- [ ] Tunnel re-establishes automatically after host reboot
- [ ] No forwarded port outlives its session
