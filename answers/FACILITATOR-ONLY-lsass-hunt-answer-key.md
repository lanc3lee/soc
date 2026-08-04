---
title: "FACILITATOR ONLY — Answer Key: Hunting Credential Dumping (LSASS)"
date: 2026-07-24
status: internal-only
---

# Facilitator Answer Key — LSASS Hunt

**Keep this off soc.lanc3.com.** This is for you to check teammates' work and unblock anyone stuck — publishing it turns the module into a lookup exercise instead of a hunt.

**Important caveat:** I don't have live query access to your actual Elasticsearch cluster, so I can't hand you the exact timestamps, process GUIDs, or literal field values from *your* specific ingested copy of this dataset. What follows is the expected pattern based on how this technique (Metasploit-driven LSASS memory access, consistent with the dataset's own naming) generally shows up in Sysmon/Security logs, and how OTRF's compound campaign datasets are typically structured. Treat this as "what you should find," and verify the exact values yourself by running the queries below against the real data before relying on it to check someone's work.

---

## Tier 2 — Finding the LSASS access

**Query to run:**
```
EventID: 10 and TargetImage: "*lsass.exe"
```

**What to expect:**
- One or a small handful of `ProcessAccess` events where `TargetImage` ends in `lsass.exe`
- The `SourceImage` is the process that requested the handle — given this campaign involves Metasploit/Mimikatz-style credential access, expect this to be either a Meterpreter-injected legitimate-looking process, or a directly visible tool binary depending on which specific technique variant this campaign used
- `GrantedAccess` is the field to scrutinize. Values like `0x1010` or `0x1fffff` are commonly associated with memory-read-capable access to LSASS and are considered high-signal for credential dumping — routine/benign processes touching LSASS (which does happen) typically request much more limited access rights
- `CallTrace` (if present) can further corroborate — a call trace touching `dbghelp.dll` or `dbgcore.dll` is a classic indicator of a memory-dump API being used (e.g., `MiniDumpWriteDump`)

**How to check a teammate's answer:** ask them to state the `SourceImage`, timestamp, and `GrantedAccess` value they found, and have them explain *why* that `GrantedAccess` value is suspicious rather than just citing it — the reasoning is the actual skill being checked here.

## Tier 3 — Timeline reconstruction

**Queries to run:**
```
Hostname: "MKT01.pandalab.com" and EventID: 1
```
(Process creation, in the window before the Tier 2 timestamp)

```
Hostname: "MKT01.pandalab.com" and EventID: 11
```
(File creation, in the window after)

**What to expect:**
- A process creation chain leading up to the access — potentially a parent process spawning the tool, or evidence of process injection/migration if this campaign used Meterpreter's `migrate` functionality
- A file-creation event shortly after the LSASS access is a strong signal of a memory dump being written to disk (a `.dmp` file or similarly named artifact) — this is the "what they got" piece of the scenario prompt

**How to check a teammate's answer:** the specific chain matters less than whether they correctly identified *that* a chain exists and can narrate it in order. Don't require them to match your exact process names if their reasoning about sequence and cause-effect is sound.

## Tier 4 — ATT&CK mapping

**Expected answer:** **T1003.001 — OS Credential Dumping: LSASS Memory**, under the **Credential Access** tactic.

**What a good justification looks like:** references the specific `GrantedAccess` value as indicating memory-read capability, and the direct handle-open to `lsass.exe` — not just "it touched LSASS therefore T1003.001," but tying the specific evidence to the specific technique definition (MITRE's T1003.001 page explicitly describes adversaries opening a handle to LSASS process memory to extract credentials).

## Tier 5 — Sigma rule

There's no single correct answer here — grade on structure and logic, not exact syntax. A reasonable rule should include, at minimum:

```yaml
detection:
  selection:
    EventID: 10
    TargetImage|endswith: '\lsass.exe'
    GrantedAccess:
      - '0x1010'
      - '0x1fffff'
  condition: selection
```

**Things to check for:**
- Did they scope `TargetImage` specifically to `lsass.exe` (not just any process)?
- Did they include a `GrantedAccess` filter, rather than alerting on every LSASS touch? (Without this, the rule would be extremely noisy in a real environment — antivirus and monitoring tools legitimately access LSASS with different access rights routinely.)
- Did they test it against Discover and confirm it returns the Tier 2 event and not a flood of unrelated ones? If their rule returns dozens of hits, that's a good teaching moment about false-positive tuning, not a failure — ask them what's different about the benign hits versus the true positive.

## Reflection question guidance (not "answers," just discussion prompts)

- **Harder to spot if more careful:** e.g., using a less common/rarer `GrantedAccess` value, avoiding writing the dump to disk at all (exfiltrating memory directly), or using a legitimate signed tool (LOLBin-style) instead of an obvious injected process.
- **What to check first on a live alert:** whether the source process is expected on that host (AV/EDR agents legitimately touch LSASS), then GrantedAccess, then whether any file was written afterward.
- **False positive risk:** security tooling, backup agents, and some legitimate diagnostic tools also access LSASS — a facilitator should confirm the teammate at least named one plausible benign source rather than assuming their rule is perfect.
