# 08 — Email on the Owned Domain

**Threats addressed:** spoofing of the domain to impersonate its owner; compromise of the mail
account; loss of the domain itself.

**State:** partial. Domain email authentication is complete and enforcing at `p=reject`. Remaining
gaps are on the account and client side.

## Why this needed its own control

Adding email to a domain changes what the domain *is*. Before, it was ingress for services.
Now it is also an identity: an address published on a resume and professional profiles, which
people will trust when a message arrives from it. That creates three new things to defend — the
domain's reputation as a sender, the mailbox, and continuity of the registration itself.

## Implementation

| | |
|---|---|
| Mail hosting | Third-party hosted provider; mail is not self-hosted |
| DNS | Managed at the same provider as ingress and access control |
| Clients | One mailbox, accessed over IMAP from a mobile client |
| Account login | MFA enabled (TOTP or security key) |

Self-hosting mail was not considered. Deliverability from a residential address is poor,
reputation is hard to establish, and an exposed SMTP listener would contradict the
no-inbound-ports design in [control 01](01-external-access.md). Paying a provider is the
proportionate choice.

## Domain email authentication

Three DNS records determine whether someone else can send mail that convincingly appears to come
from this domain. Verified against a public resolver, bypassing the local cache:

| Record | Purpose | Observed | Status |
|---|---|---|---|
| SPF | Declares which servers may send | Provider include, `~all` | ✅ |
| DKIM | Cryptographic signature on outbound mail | All three provider keys published | ✅ |
| DMARC | Tells receivers what to do on failure | Provider-managed, **`p=reject`** | ✅ |

A sent message was confirmed passing both SPF and DKIM from its delivered headers — checking that
records *exist* is not the same as confirming that real mail *passes*.

**DMARC is enforcing at the strongest policy.** A message claiming this domain that fails
authentication is rejected outright by receivers honoring DMARC, not quarantined. With no `sp=`
tag, subdomains inherit the same policy — correct, since none of the service subdomains send
mail, and they are now protected from spoofing too.

**Initial verification failed, and the reason is worth recording.** The first lookup of the DMARC
record returned nothing. The record is a CNAME to a provider-hosted policy, and the provider's
own guidance requires that record be set **DNS-only** at the DNS host: proxied, the host answers
with its own proxy addresses instead of following the CNAME, and the domain has no functional
DMARC policy despite the record appearing configured in the dashboard. On recheck the chain
resolved correctly. "Configured" and "working" were different facts, and only a lookup
distinguished them.

**SPF softfail is deliberate, not a gap.** With DMARC enforcing, DMARC makes the delivery decision
and the SPF qualifier stops mattering much, while softfail avoids breaking some legitimately
forwarded mail.

### Trade-off: the policy is set by a third party

The CNAME delegates the policy to the provider. Two consequences:

- **The domain's security policy can change without any action here.** If the provider altered
  the record, this domain's DMARC policy would change silently. Mitigated by re-verifying the
  resolved policy during periodic review.
- **Reports go to the provider, not the domain owner.** The policy carries a failure-report
  address at the provider and no aggregate-report address, so attempts to spoof this domain are
  invisible here. Low practical impact — most large receivers do not send failure reports — but
  it means rejection is happening without any visibility into whether anyone is trying.

Publishing a self-managed record at `p=reject` with an aggregate-report address would recover
both control and visibility, at the cost of maintaining it. Not currently justified.

## Account security

**The limit of MFA here, stated plainly.** IMAP has no mechanism for a second factor. MFA protects
the web login and account settings — including the ability to change MFA itself — but **not the
mail**. Anyone holding a credential that works over IMAP can read every message, regardless of
whether MFA is enabled.

That shifts what matters:

- **Mail clients should use per-device app passwords, not the main account password.** App
  passwords grant the same mail access, so they are no weaker as a target. The value is that each
  is individually revocable: a lost phone means revoking one credential, not rotating the account
  password and reconfiguring every device. And the main password — which controls the account's
  security settings — never sits on a device carried around daily.
- **The mobile device is now a mail credential store.** Its own lock and encryption are part of
  this control.

## Deliberate decision: not a recovery address

This address is **not** used as the recovery email for any other account.

If it were, the domain registration, the DNS account, and the mail account would collectively
secure every account that resets through it. A lapsed registration would then be an account
takeover path: whoever re-registers the domain receives the password resets.

Kept off recovery duty, a registration lapse is bounded to what it actually affects — lost and
misdirected correspondence. Damaging during a job search, but not a compromise of other accounts.

## Accepted risk

**Identity publicly linked to the infrastructure domain.** The address appears on a resume and
professional profiles, on the same domain that hosts published services. Certificate transparency
already exposes every subdomain with a public certificate, so this reveals nothing that was
secret — but it reduces finding the environment to two steps for anyone holding the resume.
Accepted because every published route sits behind an identity-aware proxy
([control 02](02-identity-and-access.md)) and nothing is reachable unauthenticated.

**Third-party provider holds the mail.** Mail is stored by the provider and readable by it.
Accepted: the mailbox carries professional correspondence, not material whose confidentiality
justifies the operational cost of self-hosting.

**Concentration at the DNS provider deepens.** One account now controls ingress, access policy,
and mail routing. Compromise of it could redirect mail as well as traffic. Mitigated by MFA on
that account; recorded in the [threat model](../threat-model.md).

## Known gaps

- **Provider-side account reset via domain control** not yet reviewed. The provider offers a
  setting allowing the admin account to be recovered by proving control of the domain. If
  enabled, control of DNS — or a lapsed registration — becomes a path to the mail account that
  may bypass its MFA. Should be disabled once MFA backup codes are confirmed stored offline.
- Mobile client using the main account password rather than an app password — *to be confirmed,
  since the provider normally requires an app password once MFA is on*
- No visibility into spoofing attempts — reports route to the provider

## Verification

- [x] MFA enabled on the mail account
- [x] Address not used as a recovery email for any other account
- [x] MX points at the provider
- [x] SPF published with the provider include
- [x] DKIM — all three keys published
- [x] Outbound mail passes SPF and DKIM — confirmed from headers of a sent message
- [x] DMARC resolves end to end, enforcing at `p=reject`
- [x] DMARC record set DNS-only at the DNS host
- [ ] MFA backup codes stored offline
- [ ] Provider "account reset via domain" reviewed and disabled
- [ ] Mobile client switched to a per-device app password
- [ ] Domain auto-renew enabled and registrar account MFA confirmed
- [ ] Mail provider renewal on the calendar
