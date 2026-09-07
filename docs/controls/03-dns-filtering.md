# 03 — DNS Filtering

**Threat addressed:** malvertising, tracker telemetry, and resolution of known-malicious domains
by any device on the network — including devices that cannot run an endpoint agent.

## Decision

Pi-hole runs on dedicated hardware and is the sole resolver issued by DHCP. **No secondary
upstream resolver is configured.**

Omitting the fallback is deliberate. A secondary public resolver silently defeats the control:
clients race both, the unfiltered answer often wins, and filtering appears to work while doing
nothing. A hard failure is the correct trade for a security control — a broken resolver is
noticed within minutes, a bypassed one is never noticed at all.

## Why the resolver layer

It is the only enforcement point covering unmanaged devices. Televisions, consoles, and IoT
hardware cannot run an agent and will not respect any policy they are not forced through — but
they must all resolve names.

## Limitations, stated plainly

- Applications using DNS-over-HTTPS to a hardcoded resolver bypass this entirely
- Hardcoded IP addresses skip resolution altogether
- Blocklists are reactive; a novel domain resolves until it is listed
- Single point of failure for name resolution (accepted — see threat model)

This is a noise-reduction and telemetry control. It is not a security boundary and is not
counted as one anywhere in this documentation.

## Verification

- [ ] Query log shows traffic from wireless and IoT clients, not just wired
- [ ] A known-blocked domain fails to resolve from a wireless client
- [ ] Blocklists updating on schedule
- [ ] No client observed using an alternate resolver
