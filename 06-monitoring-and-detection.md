# 06 — Monitoring and Detection

**Threat addressed:** compromise that succeeds despite preventive controls and would otherwise
continue indefinitely undetected.

**State:** partial. Collection and alerting are live; detection content is minimal.

## Why this exists

Every other control in this repository is preventive. Without detection, the security posture
depends on every preventive control holding permanently, with no feedback if one fails. That is
an assumption, not a design.

## Current implementation

Wazuh 4.14.7, single-node, deployed on the Unraid host.

| Component | Placement |
|---|---|
| Manager, indexer, dashboard | Containerized on the Unraid host |
| Dashboard access | Bound to the WireGuard interface — never published, never LAN-reachable |
| Manager API (55000) | Unpublished; reached over the container network only |
| Agent enrollment (1514/1515) | LAN-reachable, required for agents |
| Index storage | Bind-mounted to the NVMe application pool, not the array |

### Agents and sources

| Endpoint | Sources | State |
|---|---|---|
| Raspberry Pi (resolver) | systemd journal (SSH, auth, service events), DNS query log | Active |
| Unraid host | Host authentication, container lifecycle | Planned |
| Cloudflare Access | External authentication events, request logs | Planned |
| Windows workstation | Authentication, Sysmon | Planned |

The resolver was deliberately the first agent. It is the endpoint whose logs carry the most
security signal per line — every device on the network resolves through it — and it was the
least likely to consume a build session in platform troubleshooting.

### Detection content

**Native ruleset.** SSH authentication failure and brute-force detection confirmed firing end to
end, from event on the endpoint through to off-host notification.

**Custom DNS content.** Seven decoders and twelve rules written for the resolver's query log,
which the native ruleset has no knowledge of. Query type, queried domain, client address, block
source, answer, and upstream resolver are all extracted and pivotable.

Design constraint worth stating, because it drove every decision: this source produces roughly
37 MB per day, a few hundred thousand events. **Every base rule is level 0** — decoded, indexed,
searchable, and silent. A rule firing per query would saturate the notification channel within
an hour and train the operator to ignore it, which is worse than having no rule at all. Only
aggregate and pattern rules alert.

| Detection | Approach | Alerts |
|---|---|---|
| CNAME-cloaked domain blocked | Per-event tag | Tagged, not paged |
| TXT record queried | Per-event tag | Tagged, not paged |
| Long, high-entropy DNS label | Per-event tag | Tagged, not paged |
| Commonly-abused TLD queried | Per-event tag | Tagged, not paged |
| Blocked-query volume spike | Aggregate over window | Yes |
| NXDOMAIN burst — DGA candidate | Aggregate over window | Yes |
| Sustained TXT volume from one client | Aggregate, per source | Yes |
| Sustained long-label queries from one client | Aggregate, per source | Yes |

The last three correspond to DNS tunneling, DNS-based exfiltration, and domain-generation-algorithm
beaconing.

### Limits of the DNS detections

Stated because a detection whose limits are not understood is a false sense of coverage.

**No client attribution on NXDOMAIN.** The resolver logs a query and its answer as separate
lines, so NXDOMAIN responses carry no source address. The DGA rule can establish that something
on the network is exhibiting the behaviour but not what. Attribution requires manual correlation
against query lines by timestamp.

**Thresholds are unbaselined.** The aggregate rules carry estimated thresholds. Until a week of
observation exists, neither an alert nor a silence from them carries information. Recorded rather
than quietly accepted.

**A known false positive is already present.** A device on this network queries randomized
29-character subdomains with embedded epoch timestamps, under a parent domain that also appears
CNAME-blocked — a tracker using per-query unique hostnames and CNAME cloaking to defeat blocklist
matching. It trips the long-label rule continuously. The correct response is a targeted exclusion
for that parent domain, **not** raising the global threshold: raising it to silence one known
source blinds the rule to what it exists to catch.

## Off-host alerting

High-severity alerts (level 10 and above) are pushed to an external notification service
immediately on fire.

This is not a convenience feature. It is the mitigation for the accepted risk below, and it was
built in the same session as the collection rather than deferred — an alert channel added later
is an alert channel that does not exist during the window when it is most needed.

Alert delivery uses certificate verification against an explicit CA bundle. The obvious
workaround for the trust-store problem in this environment is to disable verification, which
would mean the one control whose job is working when something else has failed would accept any
certificate presented to it.

## Accepted risk: the monitor runs on the monitored host

The SIEM is hosted on the same machine it observes. If the Unraid host is compromised, the
evidence is compromised with it — logs can be altered, the indexer stopped, alerts suppressed.

There is no clean fix at this scale. Dedicated hardware for the monitoring stack is not
proportionate here.

**Mitigation:** high-severity alerts leave the host at the moment they fire, so suppression
after the fact cannot erase them. An attacker who silences the SIEM still has to do it before
the first alert reaches an external service, and doing so leaves a gap in a stream that is
observable from outside.

**Residual risk:** alerts below the notification threshold exist only on the compromised host.
Anything an attacker does that generates only low-severity events is recoverable evidence in
theory and lost in practice.

## Known gaps

- **Nothing monitors the monitor.** This has already failed in practice: a scheduled container
  update restarted the stack, the manager's analysis daemon failed to start, and the platform ran
  half-initialized for ten hours. It was found by accident. Alerting exists for events *inside*
  the pipeline; nothing watches whether the pipeline is alive. An external heartbeat check is the
  fix and is the highest-priority remaining item here.

  The diagnostic lesson generalizes: **container health is not service health.** The runtime
  reported the container up the entire time. Only the platform's own daemon status reflected
  reality.

- **Detection thresholds not baselined.** See limits above.
- **Single agent.** Four of five intended sources are not yet reporting. The most valuable
  remaining one is external authentication, since that is the only log describing activity from
  outside the network.
- **No file integrity monitoring** on configuration paths.
- **Retention not yet sized** against a realistic detection lag. An incident found a week late
  should still be investigable.

## Verification

- [x] Manager, indexer, dashboard running and surviving a host reboot
- [x] `vm.max_map_count` persists across reboot
- [x] Dashboard unreachable from the LAN without the mesh client
- [x] No default upstream credential remaining
- [x] First agent Active, events arriving
- [x] Native rule fires end to end on a deliberately generated event
- [x] High-severity alert confirmed delivered off-host
- [x] Custom decoder and rules for resolver query data, field extraction verified per event type
- [ ] Aggregate thresholds baselined against a week of observed traffic
- [ ] External heartbeat check on platform liveness
- [ ] Remaining agents reporting
- [ ] File integrity monitoring on configuration paths
- [ ] Retention sized and documented
