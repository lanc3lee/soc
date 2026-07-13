---
title: "Detection Engineering Practice Lab"
date: 2026-07-13
tags:
  - practice
  - detection-engineering
  - sigma
  - mitre-attack
  - wazuh
description: "Trigger real detections against a live Wazuh manager, read the MITRE ATT&CK mapping Wazuh already ships, and practice translating it into a portable Sigma rule."
---

# Detection Engineering Practice Lab

This is practice material for the **Detection Engineer** role on the CyberStorm SIEM project — Sigma rule selection/tuning, MITRE ATT&CK mapping, and (eventually) Kibana dashboards and Watcher alerting.

**A note on sequencing:** the team role-split doc lists this role as blocked on the OCSF Pipeline Lead producing normalised data. That's true for the *final* pipeline — Sigma rules running against OCSF-normalised events in ELK. But you don't need to wait for that to exist to start learning the actual skill, which is **translating a detection idea into a portable Sigma rule**. This lab gets you there using Wazuh's own alerts as the source, since Wazuh already ships detections pre-tagged with MITRE technique IDs — you're reading its logic and re-expressing it in Sigma, not inventing detections from scratch.

If you haven't already, complete the [Wazuh Installation Practice Lab](https://soc.lanc3.com/practice/wazuh) first — this guide assumes you have a running Wazuh manager and dashboard.

## What you're practicing

| Skill | Where it shows up |
|---|---|
| Reading a SIEM's native detection logic | Wazuh ruleset (`/var/ossec/ruleset/rules/`) |
| Recognizing MITRE ATT&CK mappings | `<mitre><id>` tags already present in Wazuh rules |
| Writing portable detection content | Hand-written Sigma rules |
| Validating a detection actually fires | Deliberately triggering the underlying event |

## Prerequisites

- A running Wazuh manager + dashboard (from the Wazuh install lab)
- At least one enrolled agent — can be the manager host itself
- `sudo` access on both the manager and the agent

### Enroll an agent, if you haven't

Easiest path is the dashboard wizard: **Wazuh dashboard → Agents → Deploy new agent**, pick your OS, and it hands you a ready-to-run enrollment command for that agent host. Alternatively, from the agent host:

```bash
sudo /var/ossec/bin/agent-auth -m <manager-ip>
sudo systemctl restart wazuh-agent
```

Confirm it's connected from the manager:

```bash
sudo /var/ossec/bin/agent_control -l
```

---

## Step 1 — Generate a real detection event

SSH brute-force is the easiest thing to trigger deliberately and reliably. From a *different* machine than the agent (or even localhost), throw a handful of bad password attempts at the agent host:

```bash
for i in {1..6}; do ssh baduser@<agent-ip> -o PreferredAuthentications=password -o PubkeyAuthentication=no; done
```

You'll get password prompts — just fail them (or let them time out). Six or so attempts in quick succession is usually enough to cross Wazuh's default brute-force threshold.

## Step 2 — Confirm the alert fired

On the manager:

```bash
sudo tail -50 /var/ossec/logs/alerts/alerts.json | grep -i "authentication_failures\|multiple attempts"
```

Or check the dashboard: **Threat Hunting → Events**, filter for `rule.description` containing "brute force" or "multiple authentication failures." You should see a match with a `rule.id` in the 5710–5760 range (SSHD rules) or similar, depending on your Wazuh version.

Note the `rule.id` — you'll need it for the next step.

## Step 3 — Read Wazuh's own detection logic

Find the rule definition on the manager:

```bash
sudo grep -rl "<id>5720</id>" /var/ossec/ruleset/rules/   # substitute your actual rule.id
```

Open the matching file and look at the rule block. You're looking for three things:

```xml
<rule id="5720" level="10" frequency="8" timeframe="120">
  <if_matched_sid>5716</if_matched_sid>
  <description>sshd: Multiple authentication failures.</description>
  <mitre>
    <id>T1110</id>
  </mitre>
  <group>authentication_failures,pci_dss_10.2.4,pci_dss_10.2.5,...</group>
</rule>
```

- **The trigger condition** — `frequency`/`timeframe`/`if_matched_sid` tell you this rule fires on repeated hits of a *lower-level* rule (single failed logins) within a time window, not on a single event. This "low-level events aggregate into a high-level detection" pattern is common and worth recognizing.
- **The MITRE tag** — `<mitre><id>T1110</id></mitre>`. That's [ATT&CK T1110 — Brute Force](https://attack.mitre.org/techniques/T1110/). Wazuh has already done the mapping work for you here; your job is to understand *why* this event maps to *this* technique, not just copy the ID.
- **The group tags** — these show what compliance/framework mappings ride along (PCI DSS clauses in this example). Worth noting for later when the team needs to justify detections against a framework.

## Step 4 — Write the Sigma equivalent by hand

Sigma is a generic, backend-agnostic rule format — the same rule can later compile to an Elasticsearch query, a Splunk search, or other targets, which is why the team standardized on it as the portable format for the eventual OCSF-normalised pipeline. Structure:

```yaml
title: SSH Brute Force - Multiple Authentication Failures
id: <generate-a-uuid-here>
status: experimental
description: Detects repeated SSH authentication failures from a single source within a short window, consistent with a brute-force attempt.
author: <your name>
date: 2026-07-13
references:
  - https://attack.mitre.org/techniques/T1110/
logsource:
  product: linux
  service: sshd
detection:
  selection:
    logsource: sshd
    event_type: authentication_failure
  timeframe: 120s
  condition: selection | count() by src_ip >= 8
falsepositives:
  - Legitimate users repeatedly mistyping credentials
  - Automated scanners performing authorized security testing
level: medium
tags:
  - attack.credential_access
  - attack.t1110
```

A few judgment calls worth noticing as you write this — these are the "real-world fields rarely map cleanly" moments the role is meant to build intuition for:

- **Sigma's `logsource` block is generic** ("linux"/"sshd"), while Wazuh's rule is specific to its own decoder output. When Role 3's OCSF pipeline exists, `logsource` will eventually point at OCSF's Authentication class fields instead — you're previewing that translation gap now.
- **The `timeframe`/`count()` aggregation pattern** mirrors Wazuh's `frequency`/`timeframe`, but Sigma's correlation syntax varies by backend — flag this as a note-to-self rather than assuming it'll compile as-is everywhere.
- **`falsepositives` is not optional busywork.** A brute-force rule with no thought given to legit-but-noisy users (password managers retrying, scheduled vuln scans) is how analysts get alert fatigue. Write real ones, not placeholders.

Save this as `sigma/T1110-ssh-brute-force.yml` in your working area (or the repo, once it's approved).

## Step 5 — Repeat with a second technique

One rule doesn't build the muscle. Pick a second Wazuh default rule and repeat Steps 2–4. Good second candidates, since they're easy to trigger deliberately and map to distinct techniques:

| Trigger | Likely Wazuh rule group | MITRE technique |
|---|---|---|
| `sudo` as an unexpected/unauthorized user | `authentication_success`, PAM rules | T1548 — Abuse Elevation Control Mechanism |
| Editing `/etc/passwd` or `/etc/shadow` directly | Wazuh FIM (`syscheck`) rules, if enabled | T1098 — Account Manipulation |
| Repeated `sudo` failures | `authentication_failures` | T1110 (variant), or T1078 depending on context |

For FIM-based triggers, you'll need File Integrity Monitoring enabled on the agent first — check `<syscheck>` in `/var/ossec/etc/ossec.conf` and confirm the paths you're testing against are being watched.

## Step 6 — Build a small personal library

By the end of this lab you should have 2–3 hand-written Sigma rules, each with:

- a Wazuh rule you traced it back to
- the MITRE technique it maps to, and a one-sentence justification for *why* (not just the ID)
- realistic false-positive notes

This is the seed of the detection library Role 4 owns once the full pipeline exists. When Role 3's OCSF mapping is live, the next exercise is re-pointing these same rules' `logsource` blocks at OCSF fields instead of Wazuh-native ones — same detection logic, different data shape underneath.

## Common issues

- **Brute-force rule doesn't fire** — check the threshold in the rule block (`frequency`) actually matches how many attempts you sent; defaults vary by Wazuh version. Increase attempts or check `timeframe` isn't too short for how fast you're sending them.
- **Can't find the rule file** — `grep -r` across `/var/ossec/ruleset/rules/` for the `rule.description` text from the dashboard alert instead of the ID; descriptions are easier to full-text search.
- **FIM events not appearing** — `syscheck` runs on a scan interval, not real-time by default; check `<frequency>` in the syscheck config, or trigger a manual scan.

---
*Practice guide — CyberStorm SIEM project. Questions/corrections → open a PR against the `soc` repo.*
