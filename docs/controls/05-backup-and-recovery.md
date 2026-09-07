# 05 — Backup and Recovery

**Threat addressed:** hardware failure, accidental deletion, ransomware encryption, and physical
loss of the site.

## Current state

| Layer | Mechanism | Protects against | State |
|---|---|---|---|
| Array parity | Single parity disk | Loss of one data disk | Implemented |
| Application data | Scheduled appdata backup to array | Container corruption, bad update | Implemented |
| Offsite | Encrypted, versioned, immutable object storage | Ransomware, fire, theft | **In progress** |

## Parity is not backup

Parity reconstructs a failed disk. It does not protect against deletion, corruption, or
encryption — those propagate to parity instantly and are preserved faithfully. Treating parity
as backup is the most common failure in self-hosted environments.

Application data additionally sits on non-redundant NVMe; the scheduled array backup is the only
copy protecting it against device failure.

## The requirement that actually matters

An offsite copy written with a credential that can also delete or overwrite history is not a
ransomware control. Malware on the host encrypts the array, then uses the same credential to
walk out and encrypt or delete the remote copy. The result is paying monthly for a second copy
of the damage.

Requirements for the offsite layer:

1. **Append-only or immutable** — object lock, or a write-scoped key that cannot delete versions
2. **Versioned**, with retention exceeding realistic detection lag for silent corruption
3. **Encrypted before leaving the host**, so the provider is not in the trust boundary
4. **Restore-tested**, because an untested backup is an assumption, not a control

Priority order for what goes offsite: photo library first (irreplaceable), application data and
configuration second, notes third, media never — it is re-acquirable and would dominate cost
for no benefit.

## Verification

- [ ] Offsite copy exists and is current
- [ ] Single-file restore tested from each layer
- [ ] Full recovery drill completed; time-to-recover recorded
- [ ] Backup credential demonstrably **cannot** delete or overwrite existing versions
- [ ] Restore tested from a host other than the one that wrote the backup
