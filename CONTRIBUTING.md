# Documentation Conventions

Rules this repository holds itself to.

1. **Every control names the threat it addresses.** A control without a stated threat is a
   preference. If the threat cannot be articulated, the control does not belong here.

2. **Incomplete is written as incomplete.** No aspirational present tense. If something is
   planned, it appears in the roadmap with the gap described, not in a control document as
   though it exists.

3. **Weak controls are labelled weak.** Geo-restriction reduces noise; it stops nobody with a
   VPN. Saying so is more credible than the alternative and prevents the documentation from
   becoming a source of false confidence.

4. **Accepted risk is written down with reasoning.** An undocumented accepted risk is
   indistinguishable from an oversight.

5. **Verification steps are checkboxes, not prose.** A control that cannot be tested cannot be
   trusted to still be working six months from now.

6. **No real hostnames, addresses, tokens, or versions.** Sanitize. Service-level detail belongs
   in the private repository.
