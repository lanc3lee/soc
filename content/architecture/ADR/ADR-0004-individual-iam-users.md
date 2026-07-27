---
tags: [adr, security, iam]
---

# ADR-0004: Individual IAM Users Instead of Shared AWS Credentials

**Status:** Proposed
**Date:** 2026-07-13
**Workstream:** Infrastructure/IaC
**Author(s):** Lance (Integration/PR Lead)
**Reviewer(s):**

---

## Context

Leadership approved an AWS budget and provided a single AWS access key / secret access key intended for the team (currently 1–2 active contributors, potentially more later) to use for provisioning practice and eventually production infrastructure. The natural default — sharing this one key pair across everyone — would satisfy immediate access needs but creates a real accountability gap.

CloudTrail logs the IAM identity, source IP, and action for every API call, and is already an ingested log source feeding our own SIEM. A shared credential means CloudTrail can only attribute actions to "the shared key," not to a specific person — source IP alone is not a reliable substitute, since contributors work from different, changing networks (home, office, mobile hotspots) and IPs can coincidentally overlap or be ambiguous.

## Decision

> Each active contributor will have their own individual IAM user with their own access key/secret, scoped to only the permissions they need. The originally provided shared credential will be used once to bootstrap these individual users, then retired from routine use.

## Options Considered

### Option 1: Shared single access key/secret for all contributors
- **Pros:** Zero setup — usable immediately as provided; simplest to distribute
- **Cons:** No real per-person accountability in CloudTrail (only shows "the shared key acted," not who); revocation is all-or-nothing (a departing contributor or a leaked credential forces rotating and redistributing to everyone); no ability to scope different permissions per person; goes against standard AWS security guidance

### Option 2: Individual IAM users per contributor
- **Pros:** CloudTrail attributes every action to a real identity, not an IP guess; clean, individual revocation; permissions can be scoped per person as needed; standard AWS best practice; consistent with the same accountability reasoning already applied in ADR-0003 (KMS access via IAM)
- **Cons:** Small one-time setup cost (creating users, attaching policies, each person generating their own key); slightly more to document for onboarding new volunteers

## Rationale

Given we already rely on CloudTrail-fed accountability elsewhere in the project (ADR-0003's reasoning for KMS state encryption), extending the same principle to general AWS access is a small, consistent step rather than a new pattern. The team is small today, but volunteer projects should expect turnover — individual, revocable access scales better than a shared secret as the team grows or changes, and avoids an awkward full-team key rotation the first time someone leaves or a credential is accidentally exposed (a real risk given the team already manages public practice repos).

## Consequences

- **Positive:** Real per-person audit trail via CloudTrail; clean individual offboarding; permissions can be scoped per contributor; consistent security posture with ADR-0003
- **Negative / tradeoffs accepted:** Small one-time setup effort; the originally issued shared key must be explicitly retired/rotated after bootstrap, or it remains a standing accountability gap alongside the individual users
- **Follow-up actions required:** Use the provided shared key once to create individual IAM users for each current contributor; attach a least-privilege policy scoped to actual project needs (EC2, VPC, S3, KMS per ADR-0003 — not full admin); each contributor generates their own access key under their own user; document the onboarding steps for future volunteers; rotate or deactivate the original shared key once individual users are confirmed working

## Related

- Related ADRs: ADR-0003 (AWS KMS for state encryption — same CloudTrail-attribution reasoning)
- Related docs/specs: NFR matrix (Security section — IAM least-privilege review cadence row)
- Supersedes / superseded by: none
