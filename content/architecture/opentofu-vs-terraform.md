---
title: "OpenTofu vs Terraform: Which one for CyberStorm SIEM 's IaC ?"
tags:
  - iac
  - terraform
  - opentofu
  - cyberstorm
  - decision
date: 2026-07-08
---
## Executive Summary

OpenTofu and Terraform are essentially the same tool at the syntax level

The decision of which to use for our Infrastructure as Code (IaC) really comes down to licensing posture and which side has shipped the specific feature we need. 

For us — Cyber Storm Center by Div0 — one of those features, native state encryption, is directly relevant to a real gap in our current design.

## Where they came from

Terraform was open source (MPL 2.0) from 2014 until August 2023, when HashiCorp relicensed it under the **Business Source License (BSL 1.1)** — a source-available license that restricts building competing commercial products on top of it, but doesn't affect a team like ours just running `terraform apply` against our own AWS account. 

In response, the community forked the last MPL-licensed version and formed **OpenTofu**, now governed by the Linux Foundation rather than a single vendor. IBM completed its acquisition of HashiCorp in early 2025, which added a further wrinkle: Terraform's roadmap is now shaped by IBM's commercial priorities, not just HashiCorp's.

By 2026, this is no longer "Terraform with a different name on the binary." The two have genuinely diverged — same core HCL language and workflow, different feature sets and governance.

## What actually matters for a project like ours

### 1. Licensing — probably not a blocker, but worth naming

The BSL doesn't stop us from running Terraform internally for our own infrastructure — it restricts competing hosted/managed service offerings, which isn't remotely relevant to a volunteer SIEM project. 

So on pure legality, either tool works fine for us.

Where it's *not* nothing: 
OpenTofu's MPL 2.0 license is OSI-approved open source with zero ambiguity, governed by a multi-vendor steering committee rather than one company (now inside IBM). 

If we ever want to say "this project is built entirely on open-source tooling" — relevant framing for a BSides submission, or just a values alignment with how Div0 and the broader community operate 

— OpenTofu is the cleaner story.

### 2. State encryption — this one is directly relevant to us

This is the feature gap that actually matters for our specific project. **OpenTofu shipped native, client-side state encryption in v1.7** — Terraform's open-source CLI still doesn't have an equivalent. 

Our Terraform state will contain real sensitive data: resource IDs, IPs, and depending on how we wire things up, potentially credentials or references tied to the collective AI system's API keys and DynamoDB context store.

We already flagged secrets management as an open item in the NFR matrix (security section, "Secrets management approach" — currently unconfirmed as of July 2026). 

If we go with OpenTofu, state encryption becomes a checkbox instead of a problem we have to solve with an external KMS workflow bolted on top of vanilla Terraform.

### 3. Provider ecosystem — a wash for what we're doing

Both tools pull from the same provider ecosystem (AWS provider works identically either way), and OpenTofu's registry has full parity for the mainstream providers we'll actually use — AWS, and whatever we eventually add for observability/alerting integrations. 

This isn't a differentiator for a project scoped to a single-cloud AWS setup.

### 4. HCP Terraform / enterprise features — not relevant to us

Terraform's exclusive features (Stacks, Sentinel policy-as-code, deeper HCP Terraform integration) live behind HashiCorp's commercial platform, and HCP Terraform's free tier ended in March 2026 — remote state and execution now require a paid plan there. 

We were never going to use HCP Terraform as a volunteer project with a hard cost ceiling; a free remote backend (S3 + DynamoDB for locking, which we already planned) works identically for both tools. 

This whole category of Terraform's advantages doesn't apply to us.

### 5. Governance — a soft factor, but consistent with how we operate

OpenTofu's Linux Foundation, multi-vendor governance means no single company can unilaterally change the license or roadmap direction under us again — which is literally the reason OpenTofu exists. 

As a community/volunteer project ourselves, that governance model is philosophically closer to how CyberStorm operates than a single-vendor-controlled tool now sitting inside IBM's portfolio.

## Practical compatibility — low risk either way

If we ever needed to switch direction, the migration cost is low:

- Same HCL syntax, same CLI commands (`init`, `plan`, `apply`, `state` — just `tofu` instead of `terraform`)
- State files are compatible in both directions for the version range we'd be starting on, with one caveat:
   **once we enable OpenTofu's state encryption, that state becomes unreadable to Terraform** — a one-way door worth noting, not a blocker
- Since we're greenfield (no existing Terraform investment, no HCP Terraform usage, no Sentinel policies), there's no migration debt to pay down regardless of which we pick first

## Recommendation

For CyberStorm SIEM specifically:

- We have **no existing HashiCorp platform investment** (no HCP Terraform, no Sentinel)
- We have **no vendor relationship pressure** in either direction
- We have a **real open item** (secrets/state security) that OpenTofu's native encryption directly addresses
- We're a **volunteer, community-governed project** ourselves — OpenTofu's governance model fits that posture

**OpenTofu is the better default for us**, and adopting it now — before any `.tf` files exist — costs nothing. Switching later would mean touching every environment's backend config and validating state compatibility across whatever we'd built by then.

This is exactly the kind of decision that should get its own ADR rather than being settled informally in a discord message

---

## ADR entry

**Title:** Use OpenTofu instead of Terraform for CyberStorm SIEM infrastructure
**Status:** Proposed
**Context:** 
Original requirement was "adopt Terraform/IaC" generally, not the HashiCorp binary specifically. 
No existing HCP Terraform or Sentinel investment exists on this project.

**Decision:** Use OpenTofu as our IaC engine, targeting the same HCL configuration approach already planned.

**Rationale:** Native state encryption directly addresses an open NFR security gap; MPL 2.0 licensing removes any BSL ambiguity; Linux Foundation governance aligns with our volunteer/community model; no lost functionality since we don't use any Terraform-exclusive enterprise features.

**Consequences:** Minor terminology shift in docs/scripts (`tofu` instead of `terraform`); once state encryption is enabled, state becomes OpenTofu-specific going forward — acceptable since we have no reason to move back.
