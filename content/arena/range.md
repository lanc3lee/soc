---
title: "From SIEM to Cyber Range: Gamifying the Learning Experience"
tags: [siem, cyber-range, training, detection-engineering, cyberstorm]
date: 2026-07-04
---

# From SIEM to Cyber Range: Gamifying the Learning Experience

Cyber Range: a controlled environment where malware detonates safely, adversary behavior actually happens, and our own SIEM is the instrument our volunteers use to find it. 

Instead of reading about MITRE ATT&CK techniques, we hunt for them.

## Why bolt a range onto a SIEM project rather than build them separately

Most cyber ranges are built backwards from a training perspective — pick scenarios, then figure out what telemetry to show. 

We have the opposite advantage: 
**we already have the detection and observability stack.** 

The range doesn't need to be built from scratch; it needs to be built as a *feed source* into infrastructure that already exists.

This also means the more scenarios we build, the more our actual detection engineering matures — the range and the SIEM improve each other.

It also solves a real onboarding problem. New volunteers currently learn our SIEM by reading Sigma rules and OCSF schemas. 

That's necessary but dry. 

A range lets them learn by watching an attack unfold in near-real-time and asking "why did — or didn't — that fire an alert?"

## Core architecture

The design mirrors what we already have, with one addition: a disposable, isolated "detonation" environment that feeds the same pipeline.

```
┌─────────────────────┐
│   Detonation Zone   │   Isolated VPC / subnet
│  (sandboxed victims)│   No egress to real internet
│  malware, exploits, │   Snapshot-and-revert per round
│  red team tooling   │
└──────────┬──────────┘
           │  telemetry only (one-way)
           ▼
┌─────────────────────┐
│  Existing SIEM stack   │   CloudTrail, VPC Flow Logs,
│  (ELK + OCSF + Sigma)  │   GuardDuty, OTel Collector
└──────────┬───────────┘
           │
           ▼
┌─────────────────────┐
│   Scoring / Game Layer │   Tracks who found what, how fast
└─────────────────────┘
```

### 1. The detonation zone

This is a separate, tightly isolated VPC — never the same network as our production SIEM infrastructure. A few non-negotiables:

- **No outbound internet access.** Malware that can call home is malware that can escape. Egress goes only to an internal telemetry collector.
- **Snapshot-and-revert.** Every round starts from a known-clean AMI/snapshot. Nothing persists between scenarios.
- **Time-boxed and disposable.** Spin up for the exercise, tear down after. This also keeps cost bounded — a concern we already take seriously given our SGD 500 ceiling.
- **Segregated from the detection stack.** The SIEM should observe the detonation zone's telemetry, but never share credentials, IAM roles, or network paths with it. One-way glass, not a shared room.

A honeypot instance was already scoped as an optional line item in our original SIEM design — this is the natural evolution of that idea, built out deliberately rather than left as an afterthought.

### 2. Feeding telemetry into the existing SIEM

The detonation zone doesn't need its own SIEM — it needs to speak the same language as the one we've already built. VPC Flow Logs, process and network telemetry from the sandboxed hosts, and any EDR-style agent output all get normalized into OCSF just like our other log sources, then flow into the same Logstash pipeline and Elasticsearch indices.

This is the payoff of the architecture decisions we've already made: because we standardized on OCSF early, adding a new log source (malware sandbox telemetry) is an extension of the existing Data Pipeline workstream's work, not a parallel system.

### 3. The game layer

This is what turns "watch logs" into an actual exercise:

- **Scenarios, not just samples.** Each round is a mini attack chain — initial access, execution, persistence, lateral movement — mapped to MITRE ATT&CK, mirroring how our Sigma rules are already organized.
  
- **Objectives per role.** Blue team: find the alert, identify the technique, write the containment note. Red team (if we run one): achieve the objective without tripping a detection.
  
- **Scoring tied to detection quality, not guessing.** Points for correctly identifying the technique from the alert, bonus points for writing a Sigma rule that *would* have caught a variant, penalty for false-positive callouts. This keeps the game honest with real detection engineering skill rather than trivia.
  
- **Leaderboard and after-action review.** After each round, walk through what fired, what didn't, and why — this is where the actual learning compounds. The "why didn't this fire" conversation is often more valuable than the win.

## Where this fits our existing workstreams

| Workstream | Extension for the range |
|---|---|
| Infrastructure/IaC | Provision the isolated detonation VPC via Terraform — a new, clearly separated module, e.g. `modules/range/` |
| Data Pipeline/OCSF | Normalize sandbox telemetry (process execution, network connections) into the same OCSF schema |
| Detection | Write and validate Sigma rules against real detonations rather than synthetic test data — the range becomes the rule test bench |
| Observability/Threat Intel | Track range health, scenario timing, and feed IOCs discovered in exercises back into internal threat intel |
| ELK Platform | Build a dedicated dashboard for live-round visibility and the after-action review |

The AI security sub-team's threat modeling work also has an obvious application here: before we ever let real malware detonate, our own range infrastructure needs its own threat model — what happens if a sample *does* break out of isolation, who has access to spin up scenarios, how do we prevent the range itself from becoming an attack surface into the SIEM stack it feeds.

## Safety is the actual hard part

The interesting engineering problem here isn't the SIEM integration — it's containment discipline. Some ground rules worth writing into an ADR before any real sample runs:

- Malware samples are only sourced from established, controlled malware repositories intended for research use, never arbitrary internet downloads.
- Every scenario runs in a network with no path to production infrastructure or the public internet — verified, not assumed.
- Snapshots are wiped, not just "restarted," between rounds.
- A kill switch exists to immediately isolate or terminate the detonation zone if something behaves outside expected bounds.
- Access to trigger detonations is restricted and logged — this itself becomes a great real-world exercise in the audit logging principles from our NFR matrix.

## Starting small

This doesn't need to launch as a fully gamified platform. A reasonable path:

1. **Phase 1** — One isolated VM, one known malware family, manual detonation, observe in existing ELK stack. Prove the telemetry pipeline works end to end.
   
2. **Phase 2** — Formalize 3–5 scenarios mapped to ATT&CK techniques our Sigma rules already claim to cover. Validate the rules actually fire.
   
3. **Phase 3** — Add scoring and a leaderboard, open it up as a team exercise rather than a solo build-and-test loop.
   
4. **Phase 4** — Rotate in red-team-style challenges where one volunteer tries to evade detection and others hunt them down live.

Each phase produces something we can point to independently — working telemetry pipeline, validated detection coverage, a functioning training exercise — which also makes this a strong complement to the SIEM release itself, rather than a distraction from it.

## Why this is worth the team's time

Beyond the training value, a SIEM alone is a solid engineering project. 

A SIEM that includes a working, safely-contained cyber range that demonstrably validates its own detection coverage against real malware behavior tells a much stronger story 

— it shows the detections aren't theoretical, they've been tested against the thing they claim to catch.
