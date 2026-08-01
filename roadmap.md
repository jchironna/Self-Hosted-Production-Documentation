# Roadmap

Open gaps, prioritized by risk rather than by how interesting they are to build.

## 1. Encrypted offsite backup — *highest priority*

**Gap:** no offsite or immutable copy of any data, including the irreplaceable photo library.
Against ransomware and physical loss, both rated in the threat model, the environment currently
has no defense.

**Plan:** encrypted, versioned replication to object storage with immutability enabled. Photo
library first, then application data and configuration. Write-scoped credential that cannot
delete history. Restore drill on completion, from a different host.

**Done when:** every checkbox in [control 05](controls/05-backup-and-recovery.md) passes,
including the credential-cannot-delete test.

## 2. Monitoring and detection

**Gap:** no detective controls whatsoever. See [control 06](controls/06-monitoring-and-detection.md).

**Plan:** host-based monitoring with agents on the server and resolver, ingesting Access
authentication events, host authentication, container lifecycle, and DNS query logs. Start with
authentication anomalies and file integrity on configuration paths; expand once baseline noise
is understood.

**Done when:** a deliberately generated test event produces the expected alert, and retention
supports investigating something discovered a week late.

## 3. Service token rotation discipline

**Gap:** long-lived tokens with no automated expiry and no enforced rotation.

**Plan:** inventory every issued token against route, client, and issue date. Define a rotation
interval and set a calendar mechanism, since an interval without a trigger is a preference.

## 4. Layer-3 network segmentation

**Gap:** flat layer 2 — see [control 04](controls/04-segmentation.md). A compromised client,
IoT device, or game server host reaches everything.

**Plan:** replace router and switching with VLAN-capable hardware. Separate trusted clients,
IoT, and server workloads with default-deny between segments. Deferred on cost, not on
disagreement about the risk.

## 5. NVMe redundancy

**Gap:** cache and application-data devices unmirrored. Partially mitigated once item 1 lands.

## 6. MFA coverage audit

**Gap:** MFA status per service is not recorded anywhere, so coverage is assumed rather than
known.

**Plan:** enumerate every account, record MFA state and method, and flag anything on SMS for
migration to an authenticator or hardware key.
