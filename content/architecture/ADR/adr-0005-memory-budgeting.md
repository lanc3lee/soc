---
title: "ADR-0004: Explicit Memory Budgeting for Containerized ELK on Resource-Constrained EC2"
slug: practice/BotS/adr-0004-memory-budgeting
date: 2026-07-24
tags: [soc, elk, adr, architecture-decision, docker, aws]
status: draft
---

# ADR-0004: Explicit Memory Budgeting for Containerized ELK on Resource-Constrained EC2

## Status

Accepted

## Context

The CyberStorm SIEM Platform runs Elasticsearch, Kibana, and Logstash as Docker containers on a single, budget-constrained EC2 instance (`t3.medium`-class: 2 vCPU, ~4GB nominal / ~3.7GB usable RAM). Docker Compose's `mem_limit` setting was initially relied on as the sole memory control per service.

In practice, this was insufficient: both Elasticsearch (JVM) and Kibana (Node.js) size their default memory usage based on the **host's total detected memory**, not the cgroup/container limit imposed by Docker. This led to repeated out-of-memory kills — the Linux OOM killer terminated the Elasticsearch process directly, and separately, Kibana's own Node.js process crashed internally with a V8 heap allocation failure — both within the first hour of the stack running.

Additionally, the instance's real usable memory (~3.7GB) was found to be measurably less than its nominal "4GB" spec once the OS and kernel's own overhead is accounted for, leaving less real headroom than initial capacity planning assumed.

## Decision

For every containerized JVM- or Node.js-based service in this stack, memory will be constrained at **two layers**, not one:

1. **Container layer** — Docker Compose `mem_limit`, as before.
2. **Runtime layer** — the language runtime's own heap size explicitly set to a value comfortably under the container limit:
   - Elasticsearch: `ES_JAVA_OPTS=-Xms<N> -Xmx<N>` (fixed min/max heap, not left to auto-sizing)
   - Kibana: `NODE_OPTIONS=--max-old-space-size=<N>`
   - Logstash: `LS_JAVA_OPTS=-Xms<N> -Xmx<N>`

The sum of all container memory limits will be kept below the instance's **actual measured available memory** (confirmed via `free -h`), not its nominal advertised spec, with a deliberate margin reserved for the OS and Docker daemon itself.

A swap file is also configured as a secondary safety net, not as the primary mitigation — the explicit heap caps are the actual fix; swap only limits the blast radius if capacity planning is ever wrong again.

## Consequences

**Positive:**
- Eliminates the OOM-kill failure mode observed during initial deployment
- Makes memory budgeting an explicit, documented, reviewable number per service rather than an implicit assumption
- Establishes a repeatable pattern for future services added to this or other CyberStorm hosts

**Negative / trade-offs:**
- Requires manual recalculation of heap values any time a service is added, removed, or the instance is resized
- Conservative heap sizing may leave some performance on the table compared to letting the JVM/Node auto-size on a dedicated, non-shared host — an acceptable trade given this is a shared, budget-capped instance running three services at once, not a single-purpose production ES cluster

## Alternatives considered

- **Rely on `mem_limit` alone, tune reactively if OOM occurs again** — rejected; already observed to fail, and reactive tuning in production risks data loss / downtime versus catching it in initial sizing.
- **Move to a larger instance type instead of tuning heap sizes** — not rejected outright, but treated as a separate, budget-impacting decision rather than a substitute for correct memory configuration. Even on a larger instance, explicit heap sizing remains good practice; it just adds more margin for error.
- **Split each service onto its own smaller instance** — rejected for now on cost grounds; would multiply the number of billed instances for a workload this stack's current data volume doesn't yet justify.
