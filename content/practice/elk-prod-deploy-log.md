---
title: "Session Log: Production ELK Deployment on AWS"
slug: practice/BotS/elk-prod-deploy-log
date: 2026-07-24
tags: [soc, elk, aws, opentofu, docker, lessons-learned, runbook]
status: draft
---

# Session Log: Production ELK Deployment on AWS

Log of standing up the CyberStorm production ELK stack on AWS EC2 via OpenTofu + Docker Compose, and onboarding the first real dataset. Written primarily for the lessons learned — the mechanical steps are already captured in the [Docker install guide](/practice/BotS/elk-docker) and the [Kibana account provisioning runbook](/runbook/kibana-user-provisioning).

## What got built

- EC2 instance provisioned via OpenTofu (security group, key pair, instance, S3 snapshot bucket)
- Docker Compose ELK stack (Elasticsearch, Kibana, Logstash) with `xpack.security.enabled=true`
- S3 snapshot repository registered and verified with a real test snapshot
- Kibana encryption keys generated and set; telemetry/APM reporting disabled
- Tailscale configured for teammate remote access, layered on top of a still-locked-down security group
- First real dataset ingested: an OTRF/Security-Datasets compound campaign (LSASS credential dumping via Metasploit), ~53,700 events

## Lessons learned

### 1. Container memory limits don't tell the process inside what to do
Setting `mem_limit` in Docker Compose caps a container from the outside, but doesn't tell a JVM or Node.js process running inside it to size its own memory usage accordingly. Both Elasticsearch and Kibana defaulted to sizing themselves based on the **host's** total memory, not the container's actual limit — this caused repeated out-of-memory kills until heap sizes were set explicitly (`ES_JAVA_OPTS`, `NODE_OPTIONS=--max-old-space-size=...`) to match the container budget, with real headroom left over for the OS and Docker daemon itself.

**Takeaway:** on a memory-constrained host, always explicitly cap the language runtime's own heap — don't rely on the container boundary alone.

### 2. A burstable instance type's usable memory is less than its nominal spec
A `t3.medium`'s "4GB" is closer to ~3.7GB of actually usable memory once the OS and kernel take their share. Budgeting container memory limits against the nominal spec rather than the real available number left no margin at all on the first attempt.

### 3. IAM least-privilege lockdowns can be tested safely with `--dry-run`
When working under a deliberately scoped-down IAM user, `--dry-run` on supported AWS CLI actions (`ec2:RunInstances`, `ec2:CreateSecurityGroup`, etc.) confirms whether a permission is granted without actually creating anything — useful for mapping out what a locked-down credential can do without guessing or over-asking for access. Not all actions support it (S3's `CreateBucket` notably doesn't), so it's not a complete substitute for a real attempt, but it narrows the guesswork a lot.

### 4. Some IAM actions have no safe check at all
Actions without dry-run support (e.g., S3 bucket creation) can only be confirmed by actually running them. Worth knowing which category a given action falls into before assuming a check is possible.

### 5. Third-party tooling written against older library versions breaks quietly
The OTRF dataset-ingestion script (`Mordor-Elastic.py`) was written against an older version of the Elasticsearch Python client and needed patching (positional → keyword arguments) to work with a current client version. A reminder that community tooling tied to a specific library version doesn't always age gracefully, and errors from a version mismatch can look unrelated to the actual cause at first glance.

### 6. Shell session state doesn't persist across reconnects
Environment variables loaded via `source .env` only exist for that shell session — reconnecting via SSH starts a fresh shell with none of it loaded. This caused a confusing authentication failure that had nothing to do with the actual credentials being wrong.


## Still open

- Index Lifecycle Management (ILM) policy against the storage budget — not yet configured
- `elastic` superuser password rotation — deferred
- Broader teammate onboarding — first account creation confirmed, wider rollout pending
