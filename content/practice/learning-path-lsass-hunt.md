---
title: "Learning Path: Hunting Credential Dumping (LSASS) — OTRF Campaign 01"
slug: practice/BotS/learning-path-lsass-hunt
date: 2026-07-24
tags: [soc, elk, kibana, threat-hunting, digital-forensics, credential-access, mitre-attack, learning-path]
status: draft
---

# Learning Path: Hunting Credential Dumping (LSASS)

Built around the OTRF/Security-Datasets "LSASS_campaign_01" dataset already loaded into our ELK stack (`otrf-lsass-campaign-01`, ~53,700 events). This module is self-paced — work through the tiers in order, since each builds on what the last one taught you.

## Scenario

You're a SOC analyst at a small manufacturing firm. A workstation named **`MKT01.pandalab.com`** in the marketing department has been flagged by an automated alert for unusual process activity. Your manager hands you access to the relevant logs and says: *"Something touched LSASS. Find out what happened, when, how, and what they got."*

You don't get a pre-written answer key — you get the raw logs, the way a real analyst would. Your job is to reconstruct the story.

## What you'll learn

- How credential dumping from LSASS memory actually looks in Windows event logs (Sysmon + Security Event Log)
- How to use Kibana Discover to filter, pivot, and build a timeline from raw events
- How to map what you find back to a specific MITRE ATT&CK technique
- How to turn an investigative finding into a reusable detection (a Sigma rule)

## Before you start

- Log into Kibana: `http://100.107.55.21:5601/` (over Tailscale)
- Go to **Discover**, select the data view covering `otrf-*`
- Filter to `Hostname: "MKT01.pandalab.com"` as your starting point for every task below

---

## Tier 1 — Orientation (do this first)

**Goal:** get comfortable navigating the data before you start hunting anything specific.

1. In Discover, expand a handful of individual events. Note which fields are consistently present across most events (`@timestamp`, `Hostname`, `EventID`, `Channel`) versus which only show up on certain event types.
2. Find the full time range this dataset covers. What's the earliest and latest `@timestamp`?
3. Get a rough sense of event volume: which `EventID` values show up most often? (Hint: use the field's "Visualize" quick action in Discover to see a breakdown.)

**You should walk away knowing:** the shape of the data — what's noise, what's signal, and roughly how much of each.

---

## Tier 2 — Find the needle

**Goal:** identify the specific event(s) where something accessed LSASS memory.

Sysmon Event ID **10** (`ProcessAccess`) logs whenever one process opens a handle to another. That's your starting point — a credential-dumping tool has to open a handle to `lsass.exe` before it can read its memory.

1. Filter for `EventID: 10` and look for events where the **target** process is `lsass.exe`.
2. Once you find candidate events, look at the **source** process — what opened the handle? Is it a process you'd expect to see touching LSASS, or something unusual?
3. Note the `GrantedAccess` value on that event. This is a Windows access-rights bitmask — some values are far more suspicious than others when the target is LSASS. (Worth a quick web search on what specific `GrantedAccess` values typically indicate memory-read access versus routine access.)

**You should walk away knowing:** the specific process name, timestamp, and access rights involved in the LSASS access.

---

## Tier 3 — Build the timeline

**Goal:** reconstruct what happened *before* and *after* the LSASS access — attacks rarely start at the moment of impact.

1. Using the timestamp from Tier 2, pivot backward: filter to the same `Hostname` in the 10–15 minutes *before* the LSASS access. Look for process creation events (`EventID: 1` in Sysmon) — what process chain led up to this moment? What launched the tool that eventually touched LSASS?
2. Pivot forward from the LSASS access: is there evidence of a file being written afterward (a memory dump artifact)? Look for file-creation events (`EventID: 11`) shortly after.
3. Put it together as a short timeline, in your own words: process A launched process B, which did X, which led to LSASS access at time T, which produced artifact Y.

**You should walk away knowing:** how to reconstruct an attack chain from disconnected log events, not just spot one suspicious line.

---

## Tier 4 — Map it to ATT&CK

**Goal:** connect what you found to the standard framework the industry uses to describe adversary behavior.

1. Based on what you reconstructed, look up which MITRE ATT&CK technique (and sub-technique, if applicable) this matches under the **Credential Access** tactic.
2. Write one or two sentences describing why this specific evidence maps to that specific technique — not just the technique name, but *what in the logs* justifies it.

**You should walk away knowing:** how to translate a raw investigative finding into the shared vocabulary analysts use to communicate and compare detections.

---

## Tier 5 — Turn it into a detection (stretch goal)

**Goal:** go from "I found this once" to "I can catch this again."

1. Draft a **Sigma rule** that would detect the specific process-access pattern you found in Tier 2 (source process, target = `lsass.exe`, and the suspicious `GrantedAccess` value or range). Sigma's own documentation and existing public Sigma rules for LSASS access are fair game to reference — this is how real detection engineers work, not a closed-book exercise.
2. Test your rule's logic manually against the dataset: does filtering Discover by your rule's exact conditions return the event(s) you found in Tier 2, and *only* those (not a flood of unrelated benign events)?
3. Optional: save your query as a Kibana **Discover session** or build it into a simple dashboard panel, so a teammate could reuse it.

**You should walk away knowing:** the difference between finding an incident once and building something that catches the next one automatically.

---

## Reflection questions (discuss with the team)

- What would have made this attack harder to spot if the attacker had been more careful about their tooling choices?
- If this were a live alert instead of a static dataset, what would you check *first*, and why?
- Is your Tier 5 Sigma rule likely to have false positives in a real environment? What benign activity might also match your logic?

## Notes for whoever picks this dataset next

If you get stuck on Tier 2 or 3, it's fine to compare notes with a teammate working the same module — talk through your reasoning rather than just sharing the answer. The goal is the hunting process, not a specific timestamp.
