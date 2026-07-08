---
title: "ELK, Wazuh, or Security Onion: Which SIEM"
date: 2026-07-08
tags:
  - cyberstorm
  - siem
  - architecture
  - adr
draft: false
---

> **Why not Splunk?** Splunk licenses on ingested data volume, not seats or servers, and even entry-level SIEM deployments run into real money fast — list pricing generally starts around $150–$225 per GB/day for base platform access, before the Enterprise Security add-on that most genuine SIEM deployments require, which adds another 30–60% on top. Independent trackers peg all-in Splunk SIEM cost at roughly $2,000–$3,500 per GB/year once ES is included, with total year-one cost of ownership running 2–3x the headline license line once storage, integration, and staffing are added. CyberStorm SIEM is a volunteer-run project on a sponsored AWS budget capped at a t3.medium instance, 50GB of EBS, and under 10GB of S3 — there is no ingest budget line at all, so any tool with per-GB or per-day licensing is off the table by design, not preference. That leaves us choosing between open-source, self-hosted options: **ELK**, **Wazuh**, and **Security Onion**.

## TL;DR

These three aren't all competing for the same job. ELK is a data platform — storage, search, visualization — with no opinion about security. Wazuh is a security product that happens to ship its own indexer and dashboard (a fork of the OpenSearch/Elastic lineage), built host/endpoint-first. Security Onion is a turnkey network-security-monitoring distribution that bundles network IDS (Suricata), host detection, and a unified dashboard into one pre-wired stack. The real decision is really three framings: "build our own detection layer on ELK," "adopt Wazuh's endpoint-first detection layer," or "adopt Security Onion's network-first, batteries-included stack."

## What each one actually is

**ELK (Elasticsearch, Logstash, Kibana)**
In summary, ELK is known for its scalability, flexibility, and log management capabilities — it's designed to handle a massive amount of data and is commonly used in IT operations, DevOps, and business intelligence, not specifically security. It provides a powerful search and visualization toolset but does not natively provide built-in alerting and notification capabilities — you build detection logic, correlation, and MITRE mapping yourself, which is exactly the work our OCSF normalization and Sigma-to-ATT&CK mapping in Logstash is already doing.

**Wazuh**
Wazuh is a free, open source and enterprise-ready security monitoring solution for threat detection, integrity monitoring, incident response and compliance, focused specifically on security monitoring and intrusion detection rather than general log management. It ships 3,000+ pre-built detection rules mapped to the MITRE ATT&CK framework, built-in file integrity monitoring, and agent-based log collection for Windows, Linux, and macOS, with agents enrolled directly from the dashboard UI. In practice it can handle 30,000+ events/second with proper hardware, and users get compliance dashboards populated automatically without extra configuration work. The tradeoff is that Wazuh is primarily designed for smaller and mid-sized environments and may have limitations when it comes to scaling compared to a purpose-built Elastic deployment.

**Security Onion**
Security Onion is a free, open-source platform that combines SIEM, network security monitoring, and threat-hunting capabilities, built to help security teams detect intrusions, analyze logs, and manage security operations from one unified dashboard. It's less "single SIEM product" and more turnkey SOC distribution: it wraps Suricata for network intrusion detection alongside host-based monitoring, correlates events across both, and ships with pre-configured dashboards so a small team gets network-plus-host visibility without assembling the pieces themselves. The natural fit is an environment with real network traffic worth watching — it stands out specifically when the environment is network-heavy. The tradeoff is that it's the heaviest of the three to run: it's a full NSM distribution (its own sensor, search backend, and dashboarding), so it isn't something you bolt lightly onto an existing pipeline the way a single Wazuh agent can be.

## Comparison table

| | ELK (self-built) | Wazuh | Security Onion |
|---|---|---|---|
| License | Elastic License 2.0 / SSPL (core free tier sufficient for a lab) | Apache 2.0, fully open source | Free, open source |
| Primary focus | General data platform, no security opinion | Endpoint/host-first (agents, FIM, compliance) | Network-first (NIDS via Suricata) + host detection |
| Detection content | None out of the box — you write it | 3,000+ prebuilt rules, MITRE-mapped | Prebuilt network + host detection, correlated |
| Log collection | Beats / Logstash / manual pipelines | Built-in agent, enrolled from dashboard | Sensors + host agents, pre-wired |
| File integrity monitoring | Not native | Built-in | Available via host component |
| Time-to-first-alert | Slow — weeks of rule-building | Fast — minutes after agent enrollment | Fast, but requires network tap/span port setup |
| Query flexibility | Full control via KQL/Logstash pipelines | Less flexible, opinionated rule engine | Less flexible; tuned for NSM workflows |
| Resource footprint | Scales to petabytes, resource-hungry | Lighter, designed for smaller/mid environments | Heaviest — full sensor + search + dashboard stack |
| Fit for our AWS budget (t3.medium, 50GB EBS) | Tight, but it's what we've already architected around | Adds a second indexer/dashboard if run standalone — doubles footprint unless piped into our existing stack | Not a fit as a standalone platform on our current instance sizing; would need its own dedicated sensor host |
| Fit for our log sources today (CloudTrail, VPC Flow Logs, GuardDuty, honeypot) | Native — this is what we built OCSF/Sigma mapping around | Adds host-level telemetry we don't currently collect | Strong on network/honeypot traffic, weak on our cloud-API-centric sources (CloudTrail, GuardDuty) |

## Where this leaves CyberStorm SIEM

Our pipeline already does the job Wazuh would otherwise do for us: Logstash normalizes CloudTrail, VPC Flow Logs, GuardDuty, and honeypot telemetry into OCSF, and we map Sigma rules to MITRE ATT&CK ourselves. Rebuilding that on top of Wazuh's own indexer would mean running two data platforms in parallel on a 50GB EBS cap — not viable.

The option worth keeping open: run Wazuh **agents only** on our EC2 hosts for host-level telemetry (FIM, syscall monitoring, vulnerability detection) and redirect their output through our existing Logstash pipeline into our own indices, instead of standing up Wazuh's separate manager/indexer/dashboard stack. That gets us prebuilt host-level detection content without doubling our storage footprint or diverging from the architecture we're presenting for the BSides submission.

Security Onion doesn't fit as a standalone platform given our sources are mostly cloud-API-centric (CloudTrail, VPC Flow Logs, GuardDuty) rather than raw network traffic — its strength is Suricata-driven packet inspection, which needs a network tap or span port we don't have on our current EC2-only footprint. It stays relevant only if the honeypot workstream grows into something with real network traffic worth instrumenting directly; at that point it's worth revisiting as a dedicated sensor rather than a wholesale platform swap.

**Decision:** Continue with self-hosted ELK as the system of record. Evaluate Wazuh agents as a host-telemetry source into the existing pipeline, not as a parallel platform. Table Security Onion until/unless the honeypot workstream needs dedicated network-traffic instrumentation.

## Open questions for the team

- Do we have headroom under the 50GB EBS cap to trial a single Wazuh agent on one honeypot host, forwarding to Logstash, before committing?
- Does Dennis want Wazuh agent traffic counted against the same IaC/Terraform-managed footprint, or as a separate module?
- If the honeypot workstream expands to real network capture, does Security Onion get its own budget line/instance, or do we replicate just its Suricata component into our existing pipeline?

---

**Footnote — other alternatives considered, not pursued:**

- **OpenSearch** — an AWS-led license-fork of Elasticsearch/Kibana, not a distinct architecture. Same pipeline shape as our current ELK setup; a license hedge, not a different decision.
- **Graylog Open** — free and lighter to operate than raw Kibana, but log-management-first with thin detection content. Doesn't solve the problem Wazuh or Security Onion solve.
- **OSSIM (AlienVault)** — correlation, asset discovery, and vuln scanning in one box, but dated and doesn't scale well. Noted for completeness only.
- **Suricata, Sagan, SEC** — detection/correlation components, not platforms. Any of these could sit *inside* whichever stack we pick rather than replace it.
- **Splunk, Microsoft Sentinel, Google Chronicle, Elastic Security (paid tier)** — all consumption/ingest-priced like Splunk (Sentinel ~$2.96–4.30/GB, Elastic paid tier ~$0.55–1.10/GB). Different numbers, same problem: no ingest budget line exists for CyberStorm SIEM, so these don't clear the first filter regardless of feature fit.
