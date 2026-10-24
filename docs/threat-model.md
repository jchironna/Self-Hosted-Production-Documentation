# Threat Model

A control is only meaningful against a stated threat. This defines what is protected, from
whom, and what risk is knowingly accepted.

## Assets, by consequence

| Asset | Loss | Disclosure | Unavailability |
|---|---|---|---|
| Personal photo library | **Severe — irreplaceable** | High | Low |
| Personal notes / knowledge base | High | **High** | Medium |
| Media library | Low (re-acquirable) | Low | Low |
| Host and service credentials | — | **Severe — enables all of the above** | — |
| Identity provider account (SSO) | — | **Severe — fronts every published service** | Medium |
| Household network availability | — | — | Medium |
| Domain and mailbox | High — public identity | Medium | **High — published on resume** |

The photo library and the SSO account drive most decisions. Media is explicitly low-value and
does not get to consume controls that matter elsewhere.

## Adversaries

**1. Opportunistic internet-wide scanning** — *primary, continuous.*
Automated discovery of exposed services and known-vulnerable software. Untargeted, high volume,
and the most likely source of compromise for any residential environment. Defeated
architecturally: there is no persistent listener on the WAN edge to discover.

**2. Credential compromise of the identity provider** — *primary.*
Because every published route authenticates against one Google account whitelist, that account
is a single point of total failure for external access. Phishing or session theft against it
bypasses the entire ingress design.

**3. Service token leakage** — *primary.*
Non-browser clients authenticate with long-lived service tokens held in client configuration.
Tokens do not expire by default and a token pasted into a config file two years ago is easy to
forget. Leakage grants unauthenticated-equivalent access to the route it covers.

**4. Malicious or compromised container image** — *secondary.*
Supply-chain compromise of a third-party image, or RCE in a published application, leading to
execution on the host.

**5. Ransomware reaching the array** — *secondary, high impact.*
Encryption of shares by a compromised client with write access. Parity provides zero defense —
it faithfully preserves the encrypted state.

**6. Compromise of an internet-exposed game server** — *situational.*
Certain games require a genuine inbound port. See accepted risk.

**7. Spoofing of the owned domain** — *primary, newly introduced.*
The domain now carries a published email address that people trust. Without enforced email
authentication, anyone can send mail appearing to come from it — to recruiters, contacts, or
the owner. Defeated by SPF, DKIM, and DMARC at enforcement, not by anything on the network.

**8. Domain registration lapse** — *low likelihood, high consequence.*
A lapsed registration can be re-registered by anyone, who then receives all mail to the domain.
Bounded deliberately: the address is not a recovery email for any other account, so a lapse
misdirects correspondence rather than enabling takeover of other accounts.

**9. Physical / environmental** — *low likelihood, total impact.*
Fire, theft, or failure exceeding single-parity tolerance.

Out of scope: targeted attack by a well-resourced adversary; insider threat. Neither is
realistic for a household, and defending against them would be disproportionate.

## Control coverage

| Adversary | Primary control | Residual risk |
|---|---|---|
| Internet scanning | Outbound-established tunnel; no persistent listener | Cloudflare edge compromise (accepted) |
| IdP credential compromise | Hardware-key MFA on the SSO account | Session token theft post-authentication |
| Service token leakage | Scoped tokens, defined rotation interval | **Rotation is manual; no automated expiry** |
| Container compromise | Per-stack network isolation, least-privilege runtime | Shared kernel; escape reaches flat LAN |
| Undetected compromise | Host-based monitoring, off-host alerting on high-severity events | Low-severity events exist only on the monitored host |
| Ransomware | Versioned offsite copy with immutability | *In progress — see roadmap* |
| Game server compromise | Source-IP allowlist, time-boxed exposure | **Flat LAN — no containment after access** |
| Physical loss | Encrypted offsite copy | *In progress — see roadmap* |
| Domain spoofing | SPF, DKIM, DMARC at `p=reject` | Policy set by provider; no visibility into attempts |
| Mailbox compromise | MFA on account, per-device app passwords | **IMAP cannot enforce MFA**; domain-based account reset may bypass MFA — under review |
| Registration lapse | Auto-renew, registrar MFA; not used for recovery | Misdirected correspondence |

## Accepted risk

Documented deliberately, with reasoning:

**Flat layer-2 network.** The mesh router and unmanaged switches cannot do VLANs. Every device
shares one broadcast domain; a compromised device reaches any other. Container network
isolation limits the blast radius of a *container* compromise but does nothing for a
compromised laptop, phone, IoT device, or game server. Resolving this requires replacing the
router and switching hardware — planned, not yet justified against current value at risk.

**Time-boxed inbound ports for specific games.** A small number of titles do not use relayed
matchmaking and require a real inbound port. When one is opened it is source-IP restricted to
known peers and closed afterward; Tailscale is used instead wherever the title permits it. The
allowlist protects the door and nothing beyond it — combined with the flat LAN above, a
compromise of the game server host has an unconstrained path to every other device. This is
the most concrete scenario justifying the eventual network hardware replacement.

**Geo-restriction is a filter, not a boundary.** Rejecting non-US traffic removes a large
fraction of automated scanning noise and is worth having for that reason alone. It stops no
adversary willing to use a VPN and is not counted as a security control anywhere in this
document.

**Cloudflare as a trusted intermediary.** TLS terminates at Cloudflare's edge, so Cloudflare
can observe plaintext for published routes — including the notes-sync database. Accepted:
Cloudflare's own compromise is outside the modelled adversary set, and the alternative
(mesh-only access for the notes database) meaningfully degrades mobile sync. Revisit if a
private-network path for the mobile client becomes practical.

**The monitor runs on the monitored host.** The SIEM is hosted on the machine it observes, so a
compromise of that host compromises the evidence. Dedicated monitoring hardware is not
proportionate at this scale. Mitigated by pushing high-severity alerts off-host at the moment
they fire, so suppression after the fact cannot erase them. Residual: anything generating only
low-severity events leaves evidence solely on the compromised host. See
[control 06](controls/06-monitoring-and-detection.md).

**Identity linked to the infrastructure domain.** The published email address sits on the same
domain as the published services, so anyone holding the resume can find the environment in two
steps. Certificate transparency already exposed the subdomains, so nothing secret is revealed.
Accepted because no route is reachable unauthenticated. See
[control 08](controls/08-email.md).

**DNS provider concentration, deepened.** The same account now controls ingress, access policy,
and mail routing. Its compromise could silently redirect mail as well as traffic. Mitigated by
MFA on that account; accepted as the cost of a single coherent configuration plane.

**Single DNS resolver.** Pi-hole is a single point of failure for name resolution. Accepted —
failure is an availability event, not a security one, and recovery is trivial.

**Non-redundant NVMe.** Cache and application-data devices are unmirrored. Accepted once the
offsite backup lands; until then, application data has one copy on non-redundant hardware.

## Review

Reviewed on every architectural change. Last reviewed: *[date]*
