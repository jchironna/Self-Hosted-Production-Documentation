# 06 — Monitoring and Detection

**Threat addressed:** compromise that succeeds despite preventive controls and would otherwise
continue indefinitely undetected.

## Current state

**Not implemented.** This is the largest gap in the environment.

Every control documented so far is preventive. There is no aggregation of authentication events,
no alerting on anomalous activity, and no ability to reconstruct what happened after the fact.
A successful compromise would leave traces scattered across individual container logs with no
retention guarantee and nothing watching them.

Prevention without detection means the posture depends on every preventive control holding
permanently, with no feedback if one fails. The ingress design is strong; the assumption that it
will never fail is not one worth making silently.

## Planned

Host-based monitoring with log aggregation across:

| Source | Value |
|---|---|
| Access authentication events | Failed and unusual SSO attempts, service token use, source geography |
| Tunnel request logs | Which routes are being probed, and from where |
| Application authentication | Brute force, credential stuffing, first-seen accounts |
| Host authentication and privilege escalation | Unexpected administrative sessions |
| Container lifecycle | Unexpected starts, image changes, config drift |
| Resolver query log | Beaconing intervals, newly registered domains, DNS exfiltration patterns |

Intended outcomes: file integrity monitoring on configuration paths, alerting on authentication
anomalies, and a written investigation note for every alert that fires — including the ones that
turn out to be nothing, since the reasoning is the point.

## Verification

- [ ] Agent reporting from host and resolver
- [ ] Each log source confirmed ingesting
- [ ] A deliberately generated test event triggers the expected detection
- [ ] Retention sufficient to investigate an incident discovered days later
