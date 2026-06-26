## Data Pipeline & OCSF Lead — Setup Guide

**Owns:** Logstash filter pipelines, OCSF schema mapping for CloudTrail, VPC Flow Logs, and GuardDuty  
**Goal:** Take raw AWS log formats and transform them into OCSF-compliant events that Elasticsearch can index and Kibana/Sigma rules can query consistently.

---

### Before You Start

This is the role where "it works" and "it's correct" are two different things. Anyone can write a Logstash filter that produces valid JSON. The actual job is making sure that JSON means the same thing every other source's JSON means — so a Sigma rule written once can match across CloudTrail, VPC Flow Logs, and GuardDuty without three separate versions of the same logic.

Three things to internalize before writing any filter code:

- **OCSF organizes events into numbered "classes."** Authentication is class 3001, Network Activity is 4001, Detection Finding is 2004, and so on. Every event you produce needs a `class_uid` telling Elasticsearch which shape to expect.
- **Field names in OCSF are fixed.** You don't get to call something `src_ip` because it's shorter — it's `src_endpoint.ip`, every time, regardless of source.
- **Real fields rarely map 1:1.** CloudTrail's `userIdentity.arn`, GuardDuty's `finding.id`, and VPC Flow Logs' `srcAddr` all need different transformation logic to land in OCSF's common shape. This is the actual skill — judgment calls on what to do when a source doesn't cleanly fit.

---

### Phase 1 — Learn the Schema Before Writing Any Filters (Day 1–2)

Don't open Logstash yet. Start at the schema browser and actually read the event classes relevant to your three sources:

**OCSF Schema Browser:** [https://schema.ocsf.io](https://schema.ocsf.io)  
Look up these specific classes before doing anything else:

- **Authentication (3001)** and **API Activity (6003)** — for CloudTrail
- **Network Activity (4001)** — for VPC Flow Logs
- **Detection Finding (2004)** — for GuardDuty

For each class, note the **required fields** vs optional ones. Required fields are non-negotiable — if your filter doesn't populate them, the event isn't valid OCSF.

**Reference mapping example (CloudTrail → OCSF):** there's a published example showing exactly how raw CloudTrail JSON gets transformed — `eventTime` becomes `time` (converted to epoch), `userIdentity.arn` becomes `actor.user.uid`, `responseElements.ConsoleLogin` becomes `status`. This is worth reading closely before writing your own filters:  
[https://www.ocsf.ai/templates/3ce5663f-78e8-4021-87a2-91e36130c3b1](https://www.ocsf.ai/templates/3ce5663f-78e8-4021-87a2-91e36130c3b1)

**Pre-validated sample data:** AWS publishes a set of real OCSF-compliant sample events (CloudTrail, GuardDuty, and others) you can use as your "answer key" — compare your filter's output against these to sanity-check you're producing valid shapes:  
[https://github.com/aws-samples/amazon-security-lake-ocsf-validation](https://github.com/aws-samples/amazon-security-lake-ocsf-validation)  
(see the `AWSLogs_OCSF_1.0.0-rc2_samples` folder)

---

### Phase 2 — CloudTrail → OCSF (Day 2–4)

CloudTrail is the best starting source — it's the most heavily documented, and AWS Security Lake already does this exact mapping natively, so there's a lot to check your work against.

**Key field mappings to implement:**

|CloudTrail field|OCSF field|Notes|
|---|---|---|
|`eventTime`|`time`|Convert ISO 8601 string to epoch milliseconds|
|`sourceIPAddress`|`src_endpoint.ip`|Direct mapping|
|`userIdentity.arn`|`actor.user.uid`||
|`userIdentity.userName`|`actor.user.name`||
|`eventName`|`api.operation`||
|`eventSource`|`api.service.name`||
|`awsRegion`|`cloud.region`||
|`responseElements.ConsoleLogin`|`status`|Needs conditional logic — "Success"/"Failure" map differently depending on event type|

**What to actually build:** a Logstash filter block (using `mutate`, `date`, and conditional `if` statements) that takes raw CloudTrail JSON and outputs these OCSF fields. Test against the sample CloudTrail data already sitting in the ELK starter stack (`sample-data/sample-cloudtrail.log`) — that's your test fixture, already wired into a working pipeline.

**Verify:**

- Run your filter, then query Elasticsearch and confirm `class_uid`, `time`, `actor.user.uid`, and `src_endpoint.ip` are all populated correctly
- Cross-check at least 3 events against the AWS sample data structure linked above

---

### Phase 3 — VPC Flow Logs → OCSF (Day 4–5)

Network Activity (class 4001) is structurally simpler than CloudTrail but introduces a different judgment call: VPC Flow Logs are **space-delimited text**, not JSON, so you need a `grok` filter to parse them before any OCSF mapping happens.

**Key field mappings:**

|VPC Flow Log field|OCSF field|Notes|
|---|---|---|
|`srcaddr`|`src_endpoint.ip`||
|`dstaddr`|`dst_endpoint.ip`||
|`srcport`|`src_endpoint.port`||
|`dstport`|`dst_endpoint.port`||
|`action`|`disposition`|"ACCEPT"/"REJECT" → OCSF's `disposition` enum — not a direct string copy, needs a lookup|
|`protocol`|`connection_info.protocol_num`|Numeric protocol (6 = TCP, 17 = UDP)|

**Judgment call to make here:** OCSF's `disposition` field expects specific enum values, not arbitrary strings. You'll need to decide how "ACCEPT" and "REJECT" map to OCSF's allowed values — this is exactly the kind of "real fields don't map cleanly" decision the role is built around. Check the schema browser's allowed values list for the `disposition` field before guessing.

**Verify:**

- Confirm the grok pattern correctly parses raw VPC Flow Log lines into named fields before OCSF mapping runs
- Confirm `src_endpoint.ip`/`dst_endpoint.ip` enable cross-referencing the same IP across CloudTrail events — this cross-source correlation is the entire point of normalizing in the first place

---

### Phase 4 — GuardDuty → OCSF (Day 5–7)

Detection Finding (class 2004) is the most judgment-heavy of the three, because GuardDuty's own finding structure is already fairly rich and nested — you're not just renaming fields, you're deciding how GuardDuty's proprietary severity scale and finding types map onto OCSF's standardized equivalents.

**Key field mappings:**

|GuardDuty field|OCSF field|Notes|
|---|---|---|
|`id`|`metadata.uid`|For tracking/dedup|
|`type`|`category_name` (or `finding_info.title`)|GuardDuty's finding type string (e.g. `UnauthorizedAccess:EC2/SSHBruteForce`) needs a decision: store verbatim, or parse into OCSF's category taxonomy?|
|`severity`|`severity_id`|GuardDuty uses 0.1–8.9 float scale; OCSF uses a 0–6 integer enum. This conversion is NOT 1:1 — you have to define your own bucketing logic|
|`region`|`cloud.region`||

**The hardest call in this whole role lives here:** GuardDuty's severity is a float (e.g. `7.5`), OCSF's `severity_id` is a small integer enum (Informational, Low, Medium, High, Critical, Fatal). There's no official AWS-published bucketing for this conversion — you have to define defensible thresholds yourself (e.g. 0–3.9 = Low, 4.0–6.9 = Medium, 7.0–8.9 = High) and **document your reasoning**, because whoever builds Sigma rules against `severity_id` later depends on understanding why those buckets were chosen.

**Verify:**

- Confirm GuardDuty findings land in Elasticsearch with `class_uid` 2004
- Confirm your severity bucketing logic is written down somewhere (a comment in the filter file, or a short doc) — not just implemented silently

---

### Phase 5 — Threat Intel Enrichment Overlay (Stretch, Day 7+)

Once the three core sources are normalized, AlienVault OTX IoC matches get appended as an OCSF **enrichment array** — `{ name, value, type, provider }` — rather than as their own event class. This lets Kibana pivot from any normalized event straight into threat intel context, regardless of which of the three sources the original event came from.

This is explicitly a stretch goal — don't start here. It only makes sense once Phases 2–4 are solid, since enrichment attaches to events that already have a stable OCSF shape.

---

### Quick Reference

|Resource|Use for|
|---|---|
|[OCSF Schema Browser](https://schema.ocsf.io)|Look up event classes, required fields, enum values|
|[CloudTrail → OCSF mapping example](https://www.ocsf.ai/templates/3ce5663f-78e8-4021-87a2-91e36130c3b1)|Real field-by-field transformation reference|
|[AWS OCSF validation samples](https://github.com/aws-samples/amazon-security-lake-ocsf-validation)|Pre-validated "answer key" event data to check your output against|
|ELK starter stack — `sample-data/sample-cloudtrail.log`|Test fixture already wired into a working Logstash pipeline|

---

### Knowledge check

You should be able to explain, not just have implemented:

- Why OCSF assigns numbered classes to events, and which class number applies to each of your three sources
- At least one field from each source that did **not** map cleanly, and what judgment call you made to resolve it
- Why GuardDuty's severity conversion needed custom bucketing logic, and what your thresholds were
- How a Sigma rule written against `src_endpoint.ip` can match both a CloudTrail event and a VPC Flow Log event without source-specific logic — this is the actual payoff of doing the normalization work correctly