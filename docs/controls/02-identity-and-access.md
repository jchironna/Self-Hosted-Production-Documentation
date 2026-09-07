# 02 — Identity and Access

**Threat addressed:** unauthenticated access to published applications, and credential
compromise granting external access.

## Decision

Authentication is enforced at the edge, before a request reaches the application. Every
published route carries a Cloudflare Access policy:

| Client type | Method | Notes |
|---|---|---|
| Browser | Google SSO, explicit account allowlist | Not "any Google account" — an enumerated list |
| Programmatic / mobile app | Service token | Required where the client cannot complete a browser auth flow |
| All | Non-US traffic rejected by default | Noise reduction, not a boundary |

The significant consequence: a vulnerability in a published application is not reachable by an
unauthenticated caller. Application authentication becomes a second layer rather than the only
one, which is what makes this zero trust rather than merely closed ports.

## Concentration of risk

Every published route authenticates against one identity provider account. That account is a
single point of total failure for external access — phishing or session theft against it
bypasses the entire ingress architecture. Accordingly:

- Hardware-key MFA on the SSO account. **Not SMS**, which is defeated by SIM swap, and the
  weakest link determines the strength of everything downstream.
- Account allowlist reviewed on change, not left to accumulate.

## Service tokens

Tokens are long-lived by default and live in client configuration files, which makes them the
quietest failure mode in this design — a token issued once and forgotten remains valid
indefinitely.

- Scoped to the single route that requires them
- Rotation interval defined and recorded, because "rotate when I remember" is not an interval
- Inventory maintained: which token, which route, which client, issued when

## Other practices

- Distinct credentials per service; no reuse
- Containers run unprivileged where the image permits
- Secrets in environment files excluded from version control, never inline in compose files
- Docker socket not exposed to containers that do not require it

## Known gaps

- No centralized identity provider for the *internal* services, so account state is per
  application and revocation is a manual multi-step process.
- Access protects the route; it does not patch the application behind it. Published services
  still require timely updates.

## Verification

- [ ] Every published route confirmed to carry an Access policy — no unprotected hostnames
- [ ] Unauthenticated request to each route rejected at the edge
- [ ] Hardware-key MFA active on the SSO account
- [ ] Service token inventory current; none past its rotation interval
