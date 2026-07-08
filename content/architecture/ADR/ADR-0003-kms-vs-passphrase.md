---
tags: [adr, infrastructure-iac, security]
---

# ADR-0003: Use AWS KMS as the Key Provider for OpenTofu State Encryption

**Status:** Accepted
**Date:** 2026-07-08
**Workstream:** Infrastructure/IaC
**Author(s):** Lance (Integration/PR Lead)
**Reviewer(s):**

---

## Context

ADR-0002 established that we're using OpenTofu instead of Terraform, primarily to gain native client-side state encryption — closing an open security gap around our state file (which contains sensitive references, including material tied to the collective AI system's API keys and DynamoDB context store). ADR-0002 flagged the choice of *key provider* for that encryption as an open follow-up, without deciding it.

OpenTofu's state encryption supports multiple key providers, the two realistic options for us being AWS KMS and a static passphrase. This ADR decides between them.

Relevant project context: every volunteer already configures AWS credentials locally (`aws configure`) as part of onboarding. CloudTrail is already one of our ingested log sources feeding the SIEM. The team is five volunteers with expected turnover over time, given the nature of a community project.

## Decision

> We will use AWS KMS as the key provider for OpenTofu's state encryption, rather than a static passphrase.

## Options Considered

### Option 1: AWS KMS
- **Pros:** Access controlled via existing per-volunteer IAM permissions — no separate secret to distribute; revocation is a single IAM change when a volunteer leaves or rotates off the project; every decrypt operation is logged via CloudTrail, which we already ingest into our own SIEM; supports native key rotation; negligible cost (~$1/month per key plus trivial per-request cost) against our budget ceiling
- **Cons:** Requires AWS connectivity to decrypt/encrypt (not usable fully offline); slightly more setup than a passphrase (one-time KMS key creation + IAM policy)

### Option 2: Static passphrase
- **Pros:** Simple to set up initially; works without any AWS API dependency
- **Cons:** The passphrase itself becomes a shared secret that must be distributed to all volunteers through some channel (Slack, Discord, etc.), which is itself a security weakness; no clean revocation — if a volunteer leaves or the passphrase leaks, state must be re-encrypted with a new passphrase and redistributed to everyone; no audit trail of who decrypted state or when; no native rotation support

## Rationale

Given that every volunteer already has AWS credentials configured as a prerequisite for using OpenTofu against our AWS account, KMS introduces no new operational dependency — it uses infrastructure we already require. The deciding factors are revocation and auditability: a volunteer-run project should expect contributor turnover, and KMS lets us cut off access cleanly via IAM without a disruptive re-encryption cycle across the whole team. The audit trail is a genuine bonus specific to us — KMS decrypt events land in CloudTrail, which we already pipe into our own SIEM, so key usage is visible through the same detection stack we're building. A shared passphrase creates exactly the kind of single-point-of-failure secret that our AI security sub-team's OWASP LLM Top 10 threat modeling work would flag if it applied to a different part of the system — no reason to introduce that pattern here.

## Consequences

- **Positive:** Clean, IAM-based revocation for volunteer turnover; decrypt/encrypt operations auditable via CloudTrail; no shared secret to distribute or rotate manually; consistent with least-privilege principles already noted as an open item in the NFR matrix (Security section)
- **Negative / tradeoffs accepted:** OpenTofu operations now require valid AWS credentials and KMS permissions to run, not just AWS credentials in general — one more IAM policy to set up and maintain; state decryption isn't possible without AWS connectivity (acceptable, since applying infrastructure changes already requires AWS connectivity regardless)
- **Follow-up actions required:** Create the KMS key (Infrastructure/IaC lead); write the IAM policy granting decrypt/encrypt to volunteer roles; document the KMS key ARN and setup steps in the onboarding doc; update the NFR matrix's "Secrets management approach" row to reference this ADR

## Related

- Related ADRs: ADR-0002 (adopt OpenTofu over Terraform — established the need for this decision)
- Related docs/specs: NFR matrix (Security section — "Secrets management approach" row); OpenTofu state encryption docs
- Supersedes / superseded by: none
