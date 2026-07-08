---
tags: [nfr, architecture, template]
---

# Non-Functional Requirements Matrix — Template

**Last updated:** YYYY-MM-DD
**Owner:** [Integration/PR Lead]
**Applies to:** All workstreams — every design/PR should be checked against this before merge

---

## How to use this

Each row is a quality attribute the *whole system* must satisfy, not any single feature. When a workstream proposes a design (new log source, new detection rule, new dashboard, new alert channel), check it against the relevant rows before merging. If a design would violate a target, that's a decision point — either the design changes, or this matrix changes (via a new ADR, not silently).

---

## Cost

| Attribute | Target | Current status | Owner | Notes |
|---|---|---|---|---|
| Total monthly infra cost | ≤ SGD [X] | | | |
| EC2 instance size ceiling | | | | |
| EBS storage ceiling | | | | |
| S3 storage ceiling | | | | |
| Cost alerting threshold | | | | |

## Availability / Reliability

| Attribute | Target | Current status | Owner | Notes |
|---|---|---|---|---|
| Target uptime (SIEM ingestion) | | | | |
| Recovery Time Objective (RTO) — how fast we can restore after failure | | | | |
| Recovery Point Objective (RPO) — how much data loss is acceptable | | | | |
| Backup frequency | | | | |
| Backup verification (do we ever test restores?) | | | | |

## Performance

| Attribute | Target | Current status | Owner | Notes |
|---|---|---|---|---|
| Log ingestion rate (events/sec) | | | | |
| Detection rule alert latency (time from event to alert) | | | | |
| Dashboard query response time | | | | |
| Max sustained CPU/memory utilization before scaling trigger fires | | | | |

## Scalability

| Attribute | Target | Current status | Owner | Notes |
|---|---|---|---|---|
| Defined scaling trigger(s) | | | | |
| What happens when a trigger fires (manual/automated) | | | | |
| Log volume growth assumption (per month) | | | | |

## Security

| Attribute | Target | Current status | Owner | Notes |
|---|---|---|---|---|
| Secrets management approach (API keys, credentials) | | | | |
| Least-privilege IAM review cadence | | | | |
| Network exposure (what's public vs private) | | | | |
| Patch/update cadence for EC2 host + Docker images | | | | |
| Audit logging of who can merge/apply infra changes | | | | |

## Observability (of the SIEM itself)

| Attribute | Target | Current status | Owner | Notes |
|---|---|---|---|---|
| Alerting if ingestion pipeline stalls | | | | |
| Alerting if disk/storage approaches cap | | | | |
| Health check / heartbeat for core services | | | | |

## Maintainability

| Attribute | Target | Current status | Owner | Notes |
|---|---|---|---|---|
| Documentation requirement (ADR for major decisions) | Yes — see ADR process | | | |
| Volunteer onboarding time target (new contributor to first PR) | | | | |
| Bus factor — is there more than one person who understands each module? | | | | |

---

## Change log

| Date | Change | Related ADR |
|---|---|---|
| | | |
