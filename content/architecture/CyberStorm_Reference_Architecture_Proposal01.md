# CyberStorm Reference Architecture: Sensor-to-Detection-to-Narrative Pipeline

**Status:** Proposed
**Author:** Lance Lee — Infrastructure & Solutions Architect, CyberStorm (Div0)
**Date:** July 2026

---

## BLUF

Propose formalizing CyberStorm's existing sensor → OCSF → SIEM → Sigma/MITRE detection stack as a **documented, deployable, open-source reference architecture** — a full pipeline that any Div0-adjacent or regional nonprofit SOC can stand up from a single OpenTofu apply. No new budget required: this packages infrastructure we already run. Deliverable is code + ADRs + a public-facing architecture writeup on soc.lanc3.com.

---

## 1. Why This, Why Now

SANS Internet Storm Center (ISC/DShield) is the closest well-known analogue to what CyberStorm does, and Div0 leadership wants CyberStorm to be recognized as its own thing, not a copy. The clearest differentiation isn't scale — ISC has thousands of sensors globally and we can't compete there — it's **completeness of the pipeline**:

| | SANS ISC / DShield | CyberStorm (proposed) |
|---|---|---|
| Sensors | Crowdsourced, global, low-interaction honeypots | Regional, purpose-deployed, same low-interaction model |
| Normalization | Ad hoc, feed-specific | OCSF via Logstash — vendor-neutral, standard schema |
| Detection | None — ISC reports raw trend data | Sigma rules mapped to MITRE ATT&CK, running in Wazuh/ELK |
| Output | Manually written handler diary (trend commentary) | Documented detections with ATT&CK context (narrative layer starts human-authored; see companion doc for AI-assisted future phase) |
| Reusability | Not designed to be re-deployed by others | OpenTofu-codified, ADR-documented, meant to be forked and stood up elsewhere |

ISC collects and reports. CyberStorm can be the one that also **detects and explains**, and — critically — makes that whole stack something a small regional nonprofit SOC could actually stand up themselves, which is a genuinely novel offering in this space.

---

## 2. Proposed Architecture

```
[Sensor Layer]          [Normalization]        [SIEM / Storage]        [Detection]           [Narrative]
Low-interaction   -->    Logstash          -->   Wazuh / ELK       -->  Sigma rules     -->   Handler
honeypots, regional      OCSF mapping            (self-hosted,          mapped to              write-up
firewall/IDS feeds                               AWS, state via         MITRE ATT&CK           (human-authored
                                                  OpenTofu + KMS)                               in this phase)
```

**Infrastructure layer**
- Sensor deployment: DShield-style low-interaction honeypot pattern (AWS free-tier-sized instances), but tuned for regional relevance rather than duplicating global DShield participation.
- IaC: OpenTofu modules (per ADR-0002), state encrypted via AWS KMS (per ADR-0003) — this is already CyberStorm convention, so the reference architecture just needs to be modularized and documented for external reuse rather than built from scratch.

**Data pipeline layer**
- Logstash performs OCSF normalization so sensor output is schema-consistent regardless of source device — this is what lets Sigma rules run against normalized events instead of per-vendor log formats.

**Detection layer**
- Sigma rules mapped to MITRE ATT&CK technique IDs, version-controlled alongside the IaC so detection logic ships with the infrastructure that runs it.

**Narrative layer (this phase)**
- Human-authored write-ups, same as ISC's model, but grounded in actual detections with ATT&CK context rather than raw traffic trend commentary. (An AI-assisted drafting layer is designed and ready as a follow-on — see the companion "AI-Assisted Detection Narrative Layer" document — but is explicitly out of scope here due to budget.)

---

## 3. Deliverables

1. **OpenTofu module set** — sensor deployment, Logstash/OCSF pipeline, Wazuh/ELK stack, wired together as reusable modules with sane defaults and a `terraform.tfvars.example`.
2. **ADRs** documenting every architectural decision (sensor choice, OCSF over raw parsing, Sigma/MITRE mapping approach, state encryption) — published to soc.lanc3.com per existing convention.
3. **Deployment guide** — practice-lab style doc (same format as the ELK/Wazuh install guides already at soc.lanc3.com/practice) so another org's volunteer can stand this up without needing to ask us questions.
4. **Public architecture writeup** — the differentiation narrative itself, framed for external audiences (Div0 community, potential BSides SG talk material).

---

## 4. Role & Ownership

I would own this as **infrastructure and solutions architect**: designing the module boundaries, writing the ADRs, and reviewing PRs from other CyberStorm volunteers who implement individual pieces (Infrastructure/IaC and Data Pipeline workstreams already report through me for cross-workstream integration). This is a natural extension of my current CyberStorm role, not a new mandate — I'm asking to formalize and publish work that's largely already been decided, not to start from zero.

---

## 5. Budget / Resource Ask

**None beyond current CyberStorm AWS spend.** This is packaging and documentation work on top of infrastructure we already run and pay for. No new AI/LLM usage, no new paid services. The only ask is time allocation across existing volunteers for the modularization and documentation effort.

---

## 6. Rough Phasing

| Phase | Scope | Owner |
|---|---|---|
| 1 | Modularize existing OpenTofu into reusable, documented modules | Lance (architecture), IaC workstream volunteers (implementation) |
| 2 | Write ADRs for each major decision, backfilling where undocumented | Lance |
| 3 | Deployment guide + practice docs | Lance + SIEM Platform workstream |
| 4 | Public writeup + soc.lanc3.com publication | Lance |
| 5 (stretch) | BSides SG talk pitch based on this writeup | Lance |

---

## 7. Success Metrics

- Reference architecture can be stood up end-to-end by someone outside the core team following only the published docs.
- At least one ADR explicitly contrasts our approach with ISC's model (the differentiation story, in writing).
- Architecture writeup published and shareable as a standalone portfolio artifact.

---

## 8. Risks

- **Documentation debt**: some current decisions weren't captured as ADRs at the time; backfilling requires reconstructing rationale from memory/Slack history. Mitigate by starting ADR backfill early and in parallel with modularization.
- **Volunteer bandwidth**: modularizing existing infra without breaking the live stack needs careful sequencing — treat as a parallel branch, not a live rewrite.
