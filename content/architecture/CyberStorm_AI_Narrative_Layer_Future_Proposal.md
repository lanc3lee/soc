# Future Enhancement: AI-Assisted Detection Narrative Layer

**Status:** Documented for future budget cycle — not currently proposed for approval
**Author:** Lance Lee — Infrastructure & Solutions Architect, CyberStorm (Div0)
**Date:** July 2026
**Depends on:** Sensor-to-Detection-to-Narrative Reference Architecture (see companion proposal)

---

## BLUF

SANS ISC's handler diaries are written entirely by hand, no AI involved. CyberStorm can differentiate further — beyond the reference architecture already proposed — by adding an **AI-assisted drafting layer** on top of it: Sigma detections and OCSF context get turned into a draft narrative automatically, which a human handler then reviews and publishes. This is not being proposed for approval right now because it requires ongoing AI API spend that Div0 isn't budgeting for at this time. This document exists so the idea is preserved, scoped, and ready to pitch once budget opens up — either through a future Div0 allocation or an external sponsor/grant.

---

## 1. The Idea

Once the reference architecture (sensor → OCSF → SIEM → Sigma/MITRE) is in place and producing real detections, the natural next step is to reduce the manual burden of writing them up. Rather than replacing the human handler, the AI layer drafts a first pass:

```
Sigma alert fires --> OCSF-normalized event context --> Lambda calls Claude API
                                                              |
                                                              v
                                              Draft narrative (ATT&CK technique,
                                              observed behavior, plain-language summary)
                                                              |
                                                              v
                                              Human handler reviews, edits, publishes
                                              to soc.lanc3.com
```

This reuses architecture already designed for CyberStorm's broader collective AI layer: DynamoDB as a shared context store, Lambda routing between Haiku (cheap, high-volume triage) and Sonnet (higher-quality drafting for publish-worthy write-ups), API Gateway with per-volunteer API keys, and spend alerting via SNS/Slack. Nothing here requires new architectural invention — it's an extension of work already scoped for the collective AI layer, applied specifically to the narrative/handler-diary use case.

---

## 2. Why This Differentiates Further

| | SANS ISC | CyberStorm + reference architecture | CyberStorm + AI narrative layer |
|---|---|---|---|
| Narrative | Manual only | Manual, but detection-grounded | AI-drafted, human-reviewed |
| Volunteer effort per write-up | High (full manual write) | High | Lower (edit vs. write from scratch) |
| Consistency | Varies by handler's writing style | Varies by handler | Consistent structure, faster turnaround |

Nobody in the ISC-adjacent space is publicly doing AI-assisted detection write-ups yet. This is a genuinely novel angle, and pairs well as a BSides SG talk topic once it's real rather than theoretical.

---

## 3. Why It's Deferred, Not Dropped

This requires **ongoing Claude API spend** — every Sigma alert that gets drafted costs tokens, and detection volume (especially false positives during tuning) makes usage cost harder to predict than a one-off project cost. Div0 isn't allocating budget for additional AI usage right now, so pushing this alongside the reference architecture risks stalling both. Sequencing them separately means the reference architecture (zero incremental cost) can move forward immediately while this stays shovel-ready for whenever funding exists.

---

## 4. Cost Control Design (already thought through)

- **Haiku-first routing**: cheap model handles initial triage/classification; only alerts worth publishing escalate to Sonnet for the actual narrative draft — keeps volume costs low.
- **Per-volunteer API keys via API Gateway**: makes usage attributable, not just aggregate — useful for justifying or capping spend by workstream.
- **SNS/Slack spend alerting**: already designed as part of the collective AI layer, so cost overruns get caught early rather than discovered at month-end.
- **OWASP LLM Top 10 mapping**: security risks for this layer have already been assessed, so this isn't a from-scratch risk exercise when it's time to build.

---

## 5. Trigger Conditions to Revisit

Bring this back for approval when any of the following happen:
- Div0 opens a budget cycle or grant that could cover modest, capped AI API usage.
- An external sponsor (e.g. a security vendor, or a grant tied to the BSides SG talk) is willing to cover API costs directly.
- The reference architecture has been running long enough to produce real detection volume, which gives a concrete, data-backed cost estimate instead of a guess — makes the budget ask much easier to defend.

---

## 6. What to Reuse When the Time Comes

This document + the existing collective AI layer design (DynamoDB context store, Lambda/Haiku/Sonnet routing, API Gateway, OWASP LLM Top 10 risk mapping) are the starting point — no redesign needed, just budget approval and integration work against whatever detection volume the reference architecture is producing by then.
