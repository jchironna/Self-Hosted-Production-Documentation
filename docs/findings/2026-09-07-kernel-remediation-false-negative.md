# Finding: Remediation Reported Complete While Vulnerable Kernel Still Running

**Date:** 2026-09-07
**Host:** resolver (Raspberry Pi OS, arm64)
**Type:** tooling limitation — false negative in remediation status
**Impact:** low actual risk; high confidence risk. The exposure window was short. The
misplaced confidence would not have been.

---

## Summary

After remediating a large batch of vulnerability findings, the monitoring platform's
vulnerability module would have reported the host's kernel CVEs as resolved while the
vulnerable kernel was still the running kernel. The platform derives remediation status from
**installed package version**, not from the version actually executing. The operating system's
own reboot-required indicator did not fire, so nothing in the environment signalled the
discrepancy.

Detected by manually comparing running state against installed state rather than accepting the
dashboard's count.

---

## Timeline

| Step | Observation |
|---|---|
| Initial state | 567 findings on the host: 20 critical, 233 high, 261 medium, 53 low |
| Action | Applied all outstanding package updates |
| Verification | Compared `uname -r` against installed kernel packages |
| **Discrepancy found** | Running kernel two minor versions behind the newest installed kernel |
| Cross-check | OS reboot-required flag **absent** — no signal that a reboot was pending |
| Action | Rebooted; confirmed running version matched newest installed |
| Final state | 79 findings: 6 critical, 32 high, 34 medium, 7 low |

Between the package upgrade and the reboot, the host presented as remediated while executing
the superseded kernel.

---

## Root cause

Vulnerability discovery here is **inventory matching**, not scanning. An agent module
inventories installed packages and the manager matches those versions against CVE feed data.
Installing a newer kernel package changes the inventory immediately; it does not change what is
executing until the next boot. The two facts diverge for as long as the reboot is deferred, and
the platform only observes one of them.

Compounding factor specific to this platform: the vendor kernel packaging does not reliably
create the distribution's `reboot-required` marker. On a stock Debian system that flag would
have been the visible cue. Its absence was therefore not evidence of being current — it was no
evidence at all, which is a meaningfully different thing.

---

## Why the reduction was so large, and what it means

The 86% drop came almost entirely from applying available updates, not from dismissing false
positives. That reframes the initial number: it was a **patch cadence gap**, not scanner noise.

This is worth stating because the instinct on seeing 567 findings is to assume the tool is
noisy and begin discounting it. Had the findings been triaged individually before the update was
applied, the effort would have gone into analyzing issues that one command eliminated. Cheap,
wholesale remediation first; analysis of the remainder second.

The 79 findings that survived patching are the ones that merit individual validation, and their
false-positive rate is currently unmeasured.

---

## Lessons

**Verify remediation against running state, not reported state.** For kernels, services, and
shared libraries, "package upgraded" and "vulnerable code no longer executing" are different
claims. Restarts and reboots are part of remediation, not cleanup afterward.

**Absence of a signal is not a negative signal.** The reboot-required flag not appearing was
treated as information when it carried none. Knowing which indicators a platform actually
populates is a prerequisite for reading them.

**Remediate cheaply before triaging expensively.** Bulk patching is minutes; per-finding
validation is hours. Establishing the residual set first makes the expensive work
proportionate.

**A dashboard count is a starting point.** Both directions of error were present in one pass:
false positives from version-string matching against a backported distribution, and a false
negative from running-versus-installed state.

---

## Follow-up

| Action | Status |
|---|---|
| Reboot and confirm running version | Done |
| Remove superseded kernel packages (after confirming the new kernel boots) | Open |
| Sample-validate remaining findings against the upstream security tracker | Open |
| Define a recurring patch and review interval | Open |
| Extend vulnerability coverage to the server host | Open |
| Add container image scanning | Open |

Tracked in [control 07](../controls/07-vulnerability-management.md) and the
[roadmap](../roadmap.md).
