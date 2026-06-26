
## Detection Engineer — Setup Guide

**Owns:** Sigma rule selection/tuning, MITRE ATT&CK mapping, Kibana dashboards and Watcher alerting  
**Goal:** Take the OCSF-normalized data from the Data Pipeline lead and turn it into actual detection content — rules that fire correctly, dashboards analysts actually use, and alerts that go somewhere.

---

### Before You Start

This is the SOC analyst seat — closest to what HTB's SOC Analyst path trains for directly. Explore what every analyst eventually faces: **"what does suspicious actually look like in this data, and how do I write that down precisely enough that a machine can check it?"**

**You're blocked until Role 3 (Data Pipeline & OCSF) has real normalized data flowing.** 
Don't try to write rules against raw CloudTrail/VPC Flow Log/GuardDuty fields — your rules should query the OCSF fields they produce (`src_endpoint.ip`, `actor.user.uid`, `severity_id`, etc.), not the original AWS field names. Writing against the wrong field names now just means rewriting everything once their pipeline lands.

**One important tooling note before you start:** the old Sigma conversion tool, `sigmac`, is deprecated. Use **`sigma-cli`** (built on `pySigma`) instead — it's the actively maintained replacement and has a proper Elasticsearch backend.

---

### Phase 1 — Tooling Setup (Day 1)

bash

```bash
pip install sigma-cli
sigma plugin install elasticsearch
```

Confirm it's working:

bash

```bash
sigma --help
sigma plugin list
```

Clone the community rule repository — you'll be adapting existing rules, not writing everything from scratch:

bash

```bash
git clone https://github.com/SigmaHQ/sigma.git
```

This repo has 3,000+ community-maintained rules. **Your job is mostly curation and adaptation, not invention** — find rules relevant to AWS/cloud log sources, then tune the field names to match your OCSF schema.

---

### Phase 2 — Understand the Sigma Rule Anatomy (Day 1–2)

Before writing anything, read 3–4 existing rules from the cloned repo to see the pattern. Every Sigma rule has the same skeleton:

yaml

```yaml
title: <human-readable, specific — not "Suspicious Activity">
id: <UUID>
status: experimental | test | stable
description: <one paragraph: what this detects and why it matters>
references:
  - <URL to the ATT&CK technique or threat report this is based on>
author: <your name>
date: <YYYY-MM-DD>
tags:
  - attack.<tactic>
  - attack.t<technique-id>
logsource:
  category: <e.g. process_creation, network_connection>
  product: <e.g. aws, windows>
detection:
  selection:
    <field>: <value>
  condition: selection
falsepositives:
  - <known legitimate activity that could also match>
level: low | medium | high | critical
```

**The `tags` field is what connects a rule to MITRE ATT&CK** — `attack.t1003.001` means "this maps to Technique 1003.001 (OS Credential Dumping: LSASS Memory)." This tagging is what lets you later generate an ATT&CK coverage heatmap from your whole rule set — a genuinely useful deliverable to show leadership (see Phase 5).

**Status field meanings — use these honestly, not optimistically:**

- `experimental` — new, not yet validated against real data
- `test` — validated against historical logs, not yet run live for 30 days
- `stable` — production-ready, well-tested

---

### Phase 3 — Write Your First Rule Against Real OCSF Data (Day 2–4)

Start with something you can actually test against data that exists right now in your pipeline — e.g., **a CloudTrail-based detection for a suspicious IAM action** (role assumption from an unusual source, repeated failed console logins, etc.), since CloudTrail is the first source Role 3 normalizes.

**Example skeleton, adapted to OCSF fields** (illustrative — adjust field names once you confirm exactly what Role 3's pipeline outputs):

yaml

```yaml
title: Multiple Failed Console Logins Followed by Success
id: <generate-a-uuid>
status: experimental
description: Detects a pattern of repeated failed AWS console login attempts followed by a successful login from the same source IP — a classic brute-force/credential-stuffing signature.
references:
  - https://attack.mitre.org/techniques/T1110/
author: <your name>
date: 2026-06-26
tags:
  - attack.credential_access
  - attack.t1110
logsource:
  category: authentication
  product: aws
detection:
  selection_failed:
    class_uid: 3001
    status: Failure
  selection_success:
    class_uid: 3001
    status: Success
  condition: selection_failed and selection_success
falsepositives:
  - Legitimate user mistyping their password multiple times
level: medium
```

**Convert it and check the actual query Elasticsearch will run:**

bash

```bash
sigma convert -t elasticsearch rules/your-rule.yml
```

**Critical step — always review the converted output manually.** Field name mapping between Sigma's generic names and your actual OCSF schema is handled by pySigma "pipelines," and the default pipeline won't know about your custom OCSF field structure automatically. You will likely need to either pass a custom pipeline or hand-verify the converted query references the right fields before trusting it.

---

### Phase 4 — Validate Against Real Data, Not Just Syntax (Day 4–5)

A rule that converts cleanly can still be wrong. Validate two ways:

**True positive test:** Find or generate a sample event that _should_ trigger the rule. Run the converted query against your Elasticsearch index and confirm it matches.

**False positive test:** Find legitimate activity that looks superficially similar and confirm it does _not_ match. This is where the `falsepositives` field in your rule's metadata earns its keep — write down what you tested, not just what you assume.

**A rough industry bar worth aiming for, even informally:** rules with very high false-positive rates are noise, not detection. If your rule fires constantly on benign activity, it's not ready for `stable` status — tune the selection logic further before promoting it.

---

### Phase 5 — MITRE ATT&CK Mapping & Coverage View (Day 5–6)

Once you have a handful of rules tagged with `attack.t****` IDs, you can visualize your actual detection coverage against the ATT&CK matrix — this is a genuinely useful artifact for the team and for any conference/portfolio writeup later.

Tools to look at:

- **MITRE ATT&CK Navigator** ([https://mitre-attack.github.io/attack-navigator/](https://mitre-attack.github.io/attack-navigator/)) — manually build a layer showing which techniques you have rules for
- **S2AN** (referenced in the Sigma tooling ecosystem) — generates ATT&CK Navigator layers automatically from a directory of Sigma rules, rather than building the heatmap by hand

**Deliverable to aim for:** a simple visual (even a basic heatmap) showing which ATT&CK tactics/techniques your current rule set covers, and — just as importantly — which obvious ones it doesn't yet. Gaps are useful information, not embarrassment.

---

### Phase 6 — Kibana Dashboards (Day 6–8)

Build the dashboards analysts will actually look at day to day. At minimum:

- **Alert overview** — rules that fired, when, severity
- **Per-source view** — CloudTrail activity, VPC Flow Log activity, GuardDuty findings, each queryable independently
- **ATT&CK coverage view** — if feasible, a simple breakdown by tactic

Use Kibana's Lens or dashboard builder against the OCSF-normalized indices Role 3 produces. Keep early dashboards simple — a working, boring dashboard beats an ambitious broken one.

---

### Phase 7 — Watcher Alerting (Day 8–10, stretch)

Once a few rules are validated and converted to Elasticsearch queries, wire up Kibana **Watcher** so matches actually notify someone, rather than just sitting queryable in an index. Start with one rule, one notification channel (email or webhook), end to end — confirm the full loop works before adding more rules to the alerting pipeline.

---

### Quick Reference

|Resource|Use for|
|---|---|
|[SigmaHQ rule repository](https://github.com/SigmaHQ/sigma)|3,000+ community rules to study and adapt — don't write from scratch|
|[pySigma / sigma-cli](https://github.com/SigmaHQ/pySigma)|The current, maintained conversion tooling (NOT the deprecated `sigmac`)|
|[Sigma Elasticsearch backend docs](https://sigmahq.io/docs/digging-deeper/backends)|How field mapping & pipelines work for the ES backend specifically|
|[MITRE ATT&CK Navigator](https://mitre-attack.github.io/attack-navigator/)|Build/visualize your rule set's technique coverage|

---

### Knowledge Check

You should be able to explain, not just have configured:

- Why a rule's `status` field should reflect honest validation state, not optimism
- At least one rule where you found a false positive during testing, and how you tuned the selection logic to fix it
- Why field mapping (Sigma's generic names → your actual OCSF schema) needs manual review even after a clean conversion
- Which 2–3 ATT&CK techniques your current rule set does NOT cover, and why those gaps exist (not enough data yet? not relevant to your environment? just not written yet?)
- The full path from "rule fires" to "analyst gets notified" — not just that a query matches in Elasticsearch