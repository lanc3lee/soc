---
tags: [nfr, architecture, siem]
---

# Non-Functional Requirements Matrix — CyberStorm SIEM

**Last updated:** 2026-07-08
**Owner:** Lance (Integration/PR Lead)
**Applies to:** All workstreams — every design/PR should be checked against this before merge

> Filled in based on constraints already established for the project. Blanks marked [CONFIRM] need input from you or workstream leads — don't take the filled values as final without checking against the current provisioning doc.

---

## Cost

| Attribute | Target | Current status | Owner | Notes |
|---|---|---|---|---|
| Total monthly infra cost | ≤ SGD 500 | ~SGD 116 (PoC) | Lance | Well within ceiling currently |
| EC2 instance size ceiling | t3.medium | Met | Infra/IaC lead | Downsized per leadership requirement |
| EBS storage ceiling | 50GB | [CONFIRM current usage] | Infra/IaC lead | |
| S3 storage ceiling | <10GB | [CONFIRM current usage] | Data Pipeline lead | |
| Cost alerting threshold | [CONFIRM — e.g. 80% of SGD 500] | Not yet set up | Lance | Consider AWS Budgets alert |

## Availability / Reliability

| Attribute | Target | Current status | Owner | Notes |
|---|---|---|---|---|
| Target uptime (SIEM ingestion) | [CONFIRM] | Not formally defined | Infra/IaC lead | Volunteer project — likely best-effort, not 99.9% |
| RTO (recovery time) | [CONFIRM] | Not formally defined | Infra/IaC lead | |
| RPO (data loss tolerance) | [CONFIRM] | Not formally defined | Infra/IaC lead | |
| Backup frequency | [CONFIRM] | Not yet implemented | Infra/IaC lead | Single EC2 instance = single point of failure currently |
| Backup restore tested? | [CONFIRM] | Unknown | Infra/IaC lead | Worth an ADR once decided |

## Performance

| Attribute | Target | Current status | Owner | Notes |
|---|---|---|---|---|
| Log ingestion rate | [CONFIRM expected events/sec] | Unknown/untested | Data Pipeline lead | Needs a load test once sources are wired up |
| Detection alert latency | [CONFIRM] | Not measured | Detection lead | |
| Dashboard query response time | [CONFIRM] | Not measured | ELK Platform lead | |
| Max sustained CPU/mem before scaling trigger | Per scaling trigger framework | Defined in provisioning doc | Infra/IaC lead | Reference the scaling trigger framework directly |

## Scalability

| Attribute | Target | Current status | Owner | Notes |
|---|---|---|---|---|
| Scaling trigger(s) defined | Yes | Formal framework exists | Lance | Reference provisioning doc |
| Action when trigger fires | [CONFIRM — manual review or automated?] | Likely manual (volunteer team) | Infra/IaC lead | |
| Log volume growth assumption | [CONFIRM] | Not modeled | Data Pipeline lead | |

## Security

| Attribute | Target | Current status | Owner | Notes |
|---|---|---|---|---|
| Secrets management (API keys, incl. Anthropic API keys) | OpenTofu state: AWS KMS-based encryption. Anthropic API keys / other app secrets: [CONFIRM — Secrets Manager / SSM Parameter Store?] | State encryption resolved via ADR-0003 (AWS KMS). Application-level secrets (per-volunteer Anthropic API keys) storage method still [CONFIRM] | AI Security sub-team | State encryption no longer open — see ADR-0003. Remaining gap is app-level secrets, not IaC state |
| IAM least-privilege review cadence | [CONFIRM] | Not scheduled | Infra/IaC lead | |
| Network exposure | [CONFIRM public vs private subnets] | [CONFIRM] | Infra/IaC lead | |
| Patch cadence (EC2 host, Docker images) | [CONFIRM] | Not scheduled | Infra/IaC lead | |
| Audit log of who can apply infra changes | Git PR history | In place via PR flow | Lance | Reinforced by ADR process |
| LLM-specific threat coverage | OWASP LLM Top 10 | Threat model in progress | AI Security sub-team | Feeds BSides submission |

## Observability (of the SIEM itself)

| Attribute | Target | Current status | Owner | Notes |
|---|---|---|---|---|
| Alert if ingestion pipeline stalls | [CONFIRM] | Not yet implemented | Observability lead | |
| Alert if storage nears cap (50GB EBS / 10GB S3) | [CONFIRM] | Not yet implemented | Observability lead | Directly tied to cost ceiling above |
| Health check for core services (ELK, Logstash) | [CONFIRM] | OTel Collector in place; alerting rules [CONFIRM] | Observability lead | |

## Maintainability

| Attribute | Target | Current status | Owner | Notes |
|---|---|---|---|---|
| ADR required for major decisions | Yes | Process just introduced | Lance | New — see ADR template |
| Volunteer onboarding time target | [CONFIRM] | Not measured | Lance | Worth tracking once ADRs + docs are in place |
| Bus factor per module | [CONFIRM] | Likely 1 person per workstream currently | Lance | Risk — five workstreams, five single points of knowledge |

---

## Change log

| Date | Change | Related ADR |
|---|---|---|
| 2026-07-04 | Initial matrix drafted | — |
| 2026-07-08 | Secrets management row updated — OpenTofu state encryption resolved via AWS KMS; app-level secrets (Anthropic API keys) still open | ADR-0002, ADR-0003 |
