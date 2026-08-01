# 04 — Segmentation

**Threat addressed:** lateral movement from one compromised workload to another, or to the host.

## Container-level segmentation — implemented

Each application stack is bound to its own Docker bridge network. Containers resolve and reach
only peers on their own network; there is no default path between stacks.

| Network | Contents | Rationale |
|---|---|---|
| `media-net` | Download client, indexers, automation, streaming server | Highest-risk group — ingests untrusted external content by design |
| `photos-net` | Application server, database, cache, ML worker | Isolates the most sensitive data store |
| `sync-net` | Notes database | Single service; no peer access required |
| `tunnel-net` | Tunnel daemon | Bridges only to the specific services it publishes |

The media stack is the most isolated group despite holding the least valuable data, because
value at risk and likelihood of compromise are different questions. It downloads arbitrary
files from the internet as its normal function, which makes it the probable initial access
vector regardless of what it stores.

## Host-level segmentation — not implemented

The network is flat layer 2. The mesh router and unmanaged switches do not support VLANs, so
every device shares one broadcast domain and a compromised device can reach any other.

This is the largest structural weakness in the environment. It is accepted rather than solved,
because closing it requires replacing the router and switching hardware.

What container isolation does **not** cover:

- A compromised laptop, phone, or IoT device
- A compromised game server host during a time-boxed port exposure
- Anything that escapes a container to the host — the host itself sits on the flat LAN

Intended end state: separate segments for trusted clients, IoT, and server workloads, with
inter-segment traffic default-deny.

## Verification

- [ ] Cross-stack connectivity test fails from each container network
- [ ] No container attached to more networks than its function requires
- [ ] Published services reachable only via the tunnel daemon's network
