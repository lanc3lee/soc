---
tags: [adr, data-pipeline]
---

# ADR-0001: Use Logstash for OCSF Log Normalization

**Status:** Accepted
**Date:** 2026-05-12
**Workstream:** Data Pipeline/OCSF
**Author(s):** [volunteer name]
**Reviewer(s):** Lance (integration/PR lead)

---

## Context

Raw logs from CloudTrail, VPC Flow Logs, and GuardDuty arrive in inconsistent formats. Detection rules (Sigma, mapped to MITRE ATT&CK) need a consistent schema to query against. We need a normalization layer that converts raw log sources into OCSF (Open Cybersecurity Schema Framework) before ingestion into the ELK stack. The team has mixed experience levels, the stack runs self-hosted on a single t3.medium EC2 instance, and we need something maintainable by rotating volunteers rather than one specialist.

## Decision

> We will use Logstash as the OCSF normalization layer between raw log sources and Elasticsearch.

## Options Considered

### Option 1: Logstash
- **Pros:** Native ELK ecosystem fit, large plugin library for AWS log sources, well-documented, most volunteers already familiar with it from HackTheBox SOC Analyst training
- **Cons:** Heavier resource footprint (JVM-based), slower throughput than newer alternatives

### Option 2: Vector
- **Pros:** Lower resource usage, faster throughput, modern config syntax
- **Cons:** Smaller community/plugin ecosystem for OCSF-specific transforms, steeper learning curve for volunteers coming from Splunk/ELK backgrounds, less prior art to copy from for this use case

### Option 3: Custom Lambda-based normalization
- **Pros:** Serverless, no persistent compute cost, full control over transform logic
- **Cons:** More custom code to write and maintain, no volunteer currently has bandwidth to own this long-term, harder to debug/observe than a standard pipeline tool

## Rationale

Given the resource ceiling (t3.medium, capped EBS/S3) and volunteer turnover risk, we prioritized ecosystem maturity and volunteer familiarity over raw performance. Logstash's resource overhead is a real cost, but it's a known, bounded cost — we've validated it fits within budget in the PoC. Vector and the custom Lambda approach both trade near-term maintainability for hypothetical future efficiency we don't yet need at this log volume.

## Consequences

- **Positive:** Faster onboarding for new volunteers already exposed to Logstash via training; large body of existing OCSF/Logstash config examples to reference
- **Negative / tradeoffs accepted:** Locked into JVM resource overhead on a constrained EC2 instance; may need to revisit if log volume grows past current scaling triggers
- **Follow-up actions required:** Data Pipeline lead to document the specific Logstash filter configs used for CloudTrail/VPC Flow Logs/GuardDuty in this repo for future reference

## Related

- Related ADRs: none yet
- Related docs/specs: current provisioning spec (scaling trigger framework)
- Supersedes / superseded by: none
