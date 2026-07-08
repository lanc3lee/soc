---
title: "ELK or Wazuh: Which SIEM"
date: 2026-07-08
tags:
  - cyberstorm
  - siem
  - architecture
  - adr
draft: false
---

> **Why not Splunk?** Splunk licenses on ingested data volume, not seats or servers, and even entry-level SIEM deployments run into real money fast — list pricing generally starts around $150–$225 per GB/day for base platform access, before the Enterprise Security add-on that most genuine SIEM deployments require, which adds another 30–60% on top. Independent trackers peg all-in Splunk SIEM cost at roughly $2,000–$3,500 per GB/year once ES is included, with total year-one cost of ownership running 2–3x the headline license line once storage, integration, and staffing are added. CyberStorm SIEM is a volunteer-run project on a sponsored AWS budget capped at a t3.medium instance, 50GB of EBS, and under 10GB of S3 — there is no ingest budget line at all, so any tool with per-GB or per-day licensing is off the table by design, not preference. That leaves us choosing between open-source, self-hosted options: **ELK** and **Wazuh**.

## TL;DR

They aren't really competing for the same job. ELK is a data platform — storage, search, visualization — with no opinion about security. Wazuh is a security product that happens to ship its own indexer and dashboard (a fork of the OpenSearch/Elastic lineage). The real decision isn't "ELK vs Wazuh" so much as "build our own detection layer on ELK" vs "adopt Wazuh's detection layer and point it at our own ELK."

## What each one actually is

**ELK (Elasticsearch, Logstash, Kibana)**
In summary, ELK is known for its scalability, flexibility, and log management capabilities — it's designed to handle a massive amount of data and is commonly used in IT operations, DevOps, and business intelligence, not specifically security. It provides a powerful search and visualization toolset but does not natively provide built-in alerting and notification capabilities — you build detection logic, correlation, and MITRE mapping yourself, which is exactly the work our OCSF normalization and Sigma-to-ATT&CK mapping in Logstash is already doing.

**Wazuh**
Wazuh is a free, open source and enterprise-ready security monitoring solution for threat detection, integrity monitoring, incident response and compliance, focused specifically on security monitoring and intrusion detection rather than general log management. It ships 3,000+ pre-built detection rules mapped to the MITRE ATT&CK framework, built-in file integrity monitoring, and agent-based log collection for Windows, Linux, and macOS, with agents enrolled directly from the dashboard UI. In practice it can handle 30,000+ events/second with proper hardware, and users get compliance dashboards populated automatically without extra configuration work. The tradeoff is that Wazuh is primarily designed for smaller and mid-sized environments and may have limitations when it comes to scaling compared to a purpose-built Elastic deployment.

## Comparison table

| | ELK (self-built) | Wazuh |
|---|---|---|
| License | Elastic License 2.0 / SSPL (core free tier sufficient for a lab) | Apache 2.0, fully open source |
| Detection content | None out of the box — you write it | 3,000+ prebuilt rules, MITRE-mapped |
| Log collection | Beats / Logstash / manual pipelines | Built-in agent, enrolled from dashboard |
| File integrity monitoring | Not native | Built-in |
| Time-to-first-alert | Slow — weeks of rule-building | Fast — minutes after agent enrollment |
| Query flexibility | Full control via KQL/Logstash pipelines | Less flexible, opinionated rule engine |
| Resource footprint | Scales to petabytes, resource-hungry | Lighter, designed for smaller/mid environments |
| Fit for our AWS budget (t3.medium, 50GB EBS) | Tight, but it's what we've already architected around | Adds a second indexer/dashboard if run standalone — doubles footprint unless piped into our existing stack |

## Where this leaves CyberStorm SIEM

Our pipeline already does the job Wazuh would otherwise do for us: Logstash normalizes CloudTrail, VPC Flow Logs, GuardDuty, and honeypot telemetry into OCSF, and we map Sigma rules to MITRE ATT&CK ourselves. Rebuilding that on top of Wazuh's own indexer would mean running two data platforms in parallel on a 50GB EBS cap — not viable.

The option worth keeping open: run Wazuh **agents only** on our EC2 hosts for host-level telemetry (FIM, syscall monitoring, vulnerability detection) and redirect their output through our existing Logstash pipeline into our own indices, instead of standing up Wazuh's separate manager/indexer/dashboard stack. That gets us prebuilt host-level detection content without doubling our storage footprint or diverging from the architecture we're presenting for the BSides submission.

**Decision:** Continue with self-hosted ELK as the system of record. Evaluate Wazuh agents as a host-telemetry source into the existing pipeline, not as a parallel platform.

## Open questions for the team

- Do we have headroom under the 50GB EBS cap to trial a single Wazuh agent on one honeypot host, forwarding to Logstash, before committing?
- Does Dennis want Wazuh agent traffic counted against the same IaC/Terraform-managed footprint, or as a separate module?
