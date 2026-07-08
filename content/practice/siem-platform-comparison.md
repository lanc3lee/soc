---
title: "Practice: SIEM Platform Comparison — ELK vs Wazuh"
tags: [practice, siem-platform, elk, wazuh, adr-prep]
date: 2026-07-08
---

# Practice: SIEM Platform Comparison — ELK vs Wazuh

**Purpose:** hands-on comparison to inform the still-open ELK-vs-Wazuh ADR — not a final recommendation on its own, but real evidence rather than documentation-only research.

**Method:** same OpenTofu config (`/practice/IaC`), switching `siem_platform` between `"elk"` and `"wazuh"`, applied and torn down same day to control cost on the free-tier account.

> Note: this replaces what might have been called "ELK Platform practice" — kept platform-agnostic since the tool choice isn't decided yet. Once ADR-000X settles it, the real project workstream can be named for the chosen tool if useful; this comparison doc stays as the historical record either way.

---

## Day 1 — ELK

**Date:**
**`tofu apply` time:**
**`tofu destroy` time:**

### Install process
- Install method used (manual step-by-step / script / Docker):
- Time from EC2 up → dashboard reachable:
- Number of manual steps required:
- Any errors/friction encountered:

### Resource usage on t2.micro
- Did it run acceptably on 1 vCPU / 1GB RAM, or did you need to size up?
- CPU/memory observations:
- Disk usage after install:

### Dashboard / UX
- First impressions of Kibana:
- How intuitive was getting a log source flowing in?

### Detection content out of the box
- Any pre-built rules, dashboards, or detections included by default?
- How much custom work would Sigma rule integration require?

### Notes / surprises


---

## Day 2 — Wazuh

**Date:**
**`tofu apply` time:**
**`tofu destroy` time:**

### Install process
- Install method used (all-in-one script / manual / Docker):
- Time from EC2 up → dashboard reachable:
- Number of manual steps required:
- Any errors/friction encountered:

### Resource usage on t2.micro
- Did it run acceptably on 1 vCPU / 1GB RAM, or did you need to size up?
- CPU/memory observations:
- Disk usage after install:

### Dashboard / UX
- First impressions of the Wazuh dashboard:
- How intuitive was agent enrollment / getting a log source flowing in?

### Detection content out of the box
- Any pre-built rules/decoders included by default (Wazuh ships with a fair amount out of the box — worth comparing directly against ELK's blank slate)?
- How much custom work would Sigma/MITRE ATT&CK mapping require, given what's already in our Detection workstream?

### Notes / surprises


---

## Side-by-side summary (fill in after both days)

| Dimension | ELK | Wazuh |
|---|---|---|
| Time to working dashboard | | |
| Resource usage on t2.micro | | |
| Out-of-box detection content | | |
| Agent/log-shipping model | | |
| Fit with existing OCSF/Sigma pipeline | | |
| Learning curve for volunteers | | |
| Licensing/cost at our scale | | |

## Feeds into

This comparison should be the primary evidence base for **ADR-000X: ELK vs Wazuh for CyberStorm SIEM** — once both days are done, the ADR's "Options Considered" section should draw directly from the side-by-side summary above, not be written independently of this practice.
