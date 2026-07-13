## CyberStorm SIEM: Team Roles & Learning Plan

_A community-built, IaC-driven SIEM pipeline for Div0's Cyber Storm Centre_

**Repo:** [github.com/lanc3lee/siem](https://github.com/lanc3lee/siem)  
**Team size:** 5 volunteers  
**Timeline:** 3-4 weeks  

### Why Split It This Way

One person could build this entire SIEM in a focused weekend. 
That's exactly why we're _not_ doing it that way. 

The goal here isn't just to ship a working SIEM — it's to give five volunteers five genuinely different, resume-worthy skill sets by the end of 2-3 weeks. 

Each role below is a **vertical slice** with real decisions to make, not a chunk of someone else's homework.


---

### Role Split

#### 1. [Infrastructure & IaC Lead](https://soc.lanc3.com/IaC)

**Owns:** OpenTofu modules, GitHub Actions CI/CD, AWS resource provisioning

Provisions every AWS resource in the project — EC2, EBS, S3, VPC, GuardDuty, CloudTrail — as code, end to end. Also owns the CloudWatch scaling-trigger alarms.

- **Why it matters:** IaC from scratch, on a real multi-resource project, using OpenTofu (per ADR-0002) rather than Terraform — directly transferable to almost any cloud role while building experience with the open-source fork.
- **Stretch skill:** Remote state management leveraging OpenTofu's native client-side state encryption (ADR-0002) with AWS KMS as the key provider (ADR-0003), plus PR-gated infrastructure changes.
- **Sequencing:** Goes first — everyone else is blocked until this exists.

---

#### 2. ELK Platform Engineer

**Owns:** Docker Compose stack, Elasticsearch/Logstash/Kibana config, JVM tuning, Nginx reverse proxy, Wazuh agent deployment

Makes the ELK stack actually run — and not fall over — inside a 4GB RAM constraint. Per the architecture decision, ELK is the system of record; Wazuh runs agents-only, feeding into the OCSF/Logstash pipeline rather than serving as a parallel stack.

- **Why it matters:** Forces real understanding of how Elasticsearch eats memory, how Logstash pipelines behave, and how to set container resource limits. Maps directly to HTB's ELK training.
- **Stretch skill:** Performance tuning under hard resource constraints — a transferable skill, since most real environments are constrained too.

---

#### 3. [Data Pipeline & OCSF Lead](https://soc.lanc3.com/ocsf)

**Owns:** Logstash filter pipelines, OCSF schema mapping (CloudTrail, VPC Flow Logs, GuardDuty)

Reads raw AWS log formats and writes the transformation logic that normalises everything into OCSF.

- **Why it matters:** The most intellectually demanding role — "data engineering meets security." Not commonly taught, increasingly valuable.
- **Stretch skill:** Schema design judgment calls — real-world fields rarely map cleanly.

---

#### 4. [Detection Engineer](https://soc.lanc3.com/detection)

**Owns:** Sigma rule selection/tuning, MITRE ATT&CK mapping, Kibana dashboards and Watcher alerting

Takes the normalised data from Role 3 and turns it into actual detection content analysts use daily.

- **Why it matters:** This is the SOC analyst seat — closest to HTB's SOC Analyst path.
- **Stretch skill:** Translating abstract ATT&CK tactics into concrete, testable detection logic.
- **Sequencing:** Blocked on Role 3 producing real normalised data.

---

#### 5. [Observability & Threat Intel Lead](https://soc.lanc3.com/observability)

**Owns:** OpenTelemetry Collector, infra health dashboards, AlienVault OTX integration

Builds the monitoring for the monitoring system, plus threat intel enrichment.

- **Why it matters:** OTel is a hot, in-demand skill (CNCF standard). This person becomes the team's go-to for "is the SIEM itself healthy."
- **Stretch skill:** Meta-observability — matters a lot in production SRE/security roles.
- **Sequencing:** Mostly independent — can start in Week 1 alongside Role 1.

---

### Lance as team lead

Lead doesn't take a sixth "do everything" role. The job is different:

- **Week 1:** Pair with the Infra/IaC lead if they're new to OpenTofu, then step back.
- **Throughout:** Review PRs across all five workstreams — this is how you stay close to everything without doing the work for anyone.
- **Integration:** Own the seams between roles — e.g. making sure Role 3's OCSF output actually matches what Role 4's Sigma rules expect.
- **Unblocking:** Be the escalation point when someone's stuck (e.g. JVM OOM errors), not the one quietly fixing it for them.

---

### Sequencing at a Glance

```
Week 1   Role 1 (Infra)  ──────────────┐
         Role 5 (Observability) ───────┤  (can start independently)
                                       ▼
Week 1–2 Role 2 (ELK Platform + Wazuh agents) ─┐
         Role 3 (OCSF Pipeline) ───────┤  (need infra to exist)
                                       ▼
Week 2–3 Role 4 (Detection) ───────────┘  (needs OCSF data flowing)

Week 3-4   Integration, handover, analyst walkthrough — all roles
```

---

### Notes for Next Cycle

- **Rotate roles next phase.** Whoever did OCSF mapping this round should do detection engineering next round, and so on — avoids five people each calcifying into a single skill silo.
- **Make each deliverable visible.** "I designed the OCSF normalisation layer for our SIEM" is a far stronger portfolio line than "I helped with the SIEM project." Each role should leave the project with something they can point to and explain.