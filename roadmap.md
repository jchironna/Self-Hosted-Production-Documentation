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

## 2. Heartbeat monitoring of the detection platform — *demonstrated failure*

**Gap:** nothing verifies the monitoring platform is alive. A scheduled container update
restarted the stack, the analysis daemon failed to start, and the platform ran half-initialized
for ten hours before being found by accident. Container health is not service health — the
runtime reported everything up throughout.

**Plan:** external heartbeat check. A scheduled job on the host reports liveness to a
third-party dead-man's-switch service, which alerts when the check stops arriving. The check
must probe something meaningful — daemon status, or recent event ingestion — not merely whether
a container is running.

Same mechanism covers the backup job's "no success in 36 hours" case, so build it once.

**Done when:** stopping the analysis daemon produces an external alert within the check
interval, and the alert path does not traverse the host being monitored.

## 3. Deliberate patch cadence for the detection stack

**Gap:** automatic container updates are now disabled for this stack, because an unattended
update both restarted the platform unexpectedly and risked pulling a version past the pinned
agent — or a beta major release. The consequence is that these containers no longer update at
all unless acted on.

Pinned versions are a decision to update *on purpose*, not a decision never to update. A
published service with no patch path is a slower-moving version of the same risk.

**Plan:** recurring calendar task to review upstream releases and update deliberately, agent
version raised in step with the manager. Same class of problem as token rotation below.

## 4. Detection tuning and remaining log sources

**Partially addressed.** Custom decoders and rules for the resolver query log are written and
verified — see [control 06](controls/06-monitoring-and-detection.md).

**Gap:** aggregate thresholds are estimates and carry no information until baselined. Four of
five intended log sources are still not reporting, including external authentication events,
which are the only record of activity originating outside the network.

**Plan, in order:**

1. Baseline a week of traffic; set thresholds above observed normal peaks rather than at guessed
   round numbers. Add a targeted exclusion for the known randomized-subdomain tracker instead of
   loosening the rule globally
2. Identify the device generating that tracker traffic and write it up as an investigation
3. Agent on the Unraid host (awkward: non-persistent root filesystem, needs install at array start)
4. External authentication events via API polling into the pipeline
5. Workstation agent for authentication and process telemetry
6. File integrity monitoring on configuration paths
7. Size retention against realistic detection lag

**Done when:** thresholds are derived from observed data, and at least one detection exists for a
threat specific to this environment rather than inherited from a default ruleset.

## 5. Service token rotation discipline

**Gap:** long-lived tokens with no automated expiry and no enforced rotation.

**Plan:** inventory every issued token against route, client, and issue date. Define a rotation
interval and set a calendar mechanism, since an interval without a trigger is a preference.

## 6. Layer-3 network segmentation

**Gap:** flat layer 2 — see [control 04](controls/04-segmentation.md). A compromised client,
IoT device, or game server host reaches everything.

**Plan:** replace router and switching with VLAN-capable hardware. Separate trusted clients,
IoT, and server workloads with default-deny between segments. Deferred on cost, not on
disagreement about the risk.

## 7. NVMe redundancy

**Gap:** cache and application-data devices unmirrored. Partially mitigated once item 1 lands.

## 8. MFA coverage audit

**Gap:** MFA status per service is not recorded anywhere, so coverage is assumed rather than
known.

**Plan:** enumerate every account, record MFA state and method, and flag anything on SMS for
migration to an authenticator or hardware key.
