---
tags: [adr, infrastructure-iac]
---

# ADR-0002: Use OpenTofu Instead of Terraform for CyberStorm SIEM Infrastructure

**Status:** Accepted
**Date:** 2026-07-08
**Workstream:** Infrastructure/IaC
**Author(s):** Lance (Integration/PR Lead)
**Reviewer(s):**

---

## Context

Dennis's original requirement was to adopt Terraform/IaC as our infrastructure management approach — this was scoped as a category of tooling (declarative, version-controlled infra config), not necessarily the HashiCorp binary by name. We have no existing Terraform investment, no HCP Terraform usage, and no dependency on any HashiCorp-exclusive enterprise features (Stacks, Sentinel).

JS raised OpenTofu as an option worth considering in Discord ([message link](https://discord.com/channels/699993820787638302/1451426167591669770/1523738488665804954)), prompting a closer look at whether OpenTofu or Terraform is the better fit before any `.tf`/`.tofu` configuration is written for real.

Separately, our NFR matrix already flags an open security gap: secrets management approach for state/credentials is currently unconfirmed, and our state file will contain sensitive data (resource IDs, IPs, and potentially references tied to the collective AI system's API keys and DynamoDB context store).

## Decision

> We will use OpenTofu instead of Terraform as our IaC engine for CyberStorm SIEM infrastructure.

## Options Considered

### Option 1: Terraform (HashiCorp/IBM)
- **Pros:** Largest provider ecosystem and community documentation; industry-default name recognition; HashiCorp-owned providers (Vault, Consul, Nomad) receive updates first — not relevant to our AWS-only scope
- **Cons:** Open-source CLI lacks native state encryption; licensed under BSL 1.1 (source-available, not OSI-approved); roadmap controlled by a single vendor, now IBM; HCP Terraform's free tier ended March 2026, and we have no reason to pay for it

### Option 2: OpenTofu
- **Pros:** Native client-side state encryption (v1.7+) — directly closes our open NFR gap on state/secrets security; MPL 2.0 license, OSI-approved, zero commercial ambiguity; governed by the Linux Foundation via a multi-vendor steering committee rather than a single company; same HCL syntax, same CLI workflow, same provider ecosystem as Terraform, so no functional loss for our AWS-only use case
- **Cons:** Slightly smaller adoption footprint than Terraform overall (not a practical concern for a single-cloud AWS project); once state encryption is enabled, that state becomes unreadable to Terraform — a one-way door, though irrelevant since we have no reason to move back

## Rationale

Both tools are functionally equivalent for what we're building — same HCL, same AWS provider, same `plan`/`apply` workflow, just `tofu` instead of `terraform` on the command line. Given that equivalence, the deciding factor is what each side has actually shipped: OpenTofu's native state encryption gives us a real, concrete answer to a security gap we already identified ourselves, without needing to bolt on an external KMS workflow. Since we have zero existing investment in Terraform-exclusive enterprise tooling (HCP Terraform, Sentinel), there's no cost to choosing OpenTofu now — the switch, if made later instead, would only get more expensive as more infrastructure gets built.

OpenTofu's Linux Foundation governance model is also a closer philosophical fit for a volunteer, community-run project like ours, though this was a secondary consideration relative to the state encryption gap.

## Consequences

- **Positive:** State encryption closes the open NFR security item on secrets/state management; no BSL licensing ambiguity to reason about going forward; governance model consistent with our community project posture
- **Negative / tradeoffs accepted:** Minor terminology shift across docs and scripts (`tofu` instead of `terraform`); once state encryption is enabled, our state becomes OpenTofu-specific — acceptable, since reverting to Terraform is not a scenario we're planning for
- **Follow-up actions required:** Update onboarding docs/scripts to reference `tofu` instead of `terraform`; configure state encryption (key provider TBD — AWS KMS vs. passphrase) as part of the Infrastructure/IaC workstream's first module; update the NFR matrix's "Secrets management approach" row to reflect this decision

## Related

- Related ADRs: none yet
- Related docs/specs: NFR matrix (Security section — "Secrets management approach" row)
- Supersedes / superseded by: none
- Credit: OpenTofu was originally suggested by JS in Discord — [message link](https://discord.com/channels/699993820787638302/1451426167591669770/1523738488665804954)
