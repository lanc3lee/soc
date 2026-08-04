---
title: "Workshop: Case File MORDORDC-01 — The LSASS Break-In"
slug: practice/workshop-lsass-case-file-01
date: 2026-08-04
tags:
  - soc
  - elk
  - kibana
  - threat-hunting
  - workshop
  - beginner
  - credential-access
  - mitre-attack
status: draft
---

# Case File MORDORDC-01: The LSASS Break-In

**Difficulty:**  Beginner
**Tools needed:** A web browser and a Kibana login. Nothing else.
**Estimated time:** 30–45 minutes

---

## Your Briefing

You're a SOC analyst at **The Shire Domain** (`theshire.local`). 
This automated alert lands in your queue, start investigating:

> **Ticket #MORDORDC-01**
> **Priority:** High
> **Reported by:** Automated EDR alert
> **Summary:** Unusual PowerShell activity detected on domain controller `MORDORDC`. Preliminary alert suggests possible credential access attempt. No further detail provided.
>
> *"Find out what happened. Was this real? What did they touch? How do you know?"*
> — Your SOC Lead

You don't get a pre-written answer. 
You get log access and your own judgment. 
Everything you need is already sitting in Kibana 
— your job is to find it, understand it, and prove your case.

This case revolves around one of the most classic attacker moves there is: 
stealing passwords directly out of a critical Windows process called **LSASS**. 

By the end of this case file, you'll know exactly what that looks like in real logs, and why it matters.

---

## Mission Objectives

By solving this case, you will:
- [ ] Navigate Kibana Discover like an analyst
- [ ] Understand what LSASS is and why attackers target it
- [ ] Decode a Windows `AccessMask` value and explain why it matters
- [ ] Cross-reference two independent log sources to confirm a single incident
- [ ] Recognize the signature of fileless, in-memory attack tooling

---

## Before You Start

1. Log into Kibana.
2. Go to **Discover** (left navigation).
3. Select the data view for the `empire-mimikatz-*` index.
4. Set your time range to cover **August 2020** 
   (the incident is old, but the trail is exactly as it was left — nothing about hunting changes based on how recent the data is).

![[kibana-workshop-lsass-case-file-01-15-years.png]]
to keep it simple, you can just pull down the submenu near top right, change it from minutes to years (so it will search past 15 years of data)

---

## 🔍 Investigation Part 1: The Access Attempt

**Background — what is LSASS, and why should you care?**

LSASS (Local Security Authority Subsystem Service) is a core Windows process. Every time someone logs in, Windows stores their credentials in LSASS's memory — this is what lets you unlock your laptop once and not get asked for your password again every five minutes. That convenience is exactly what makes LSASS such a high-value target: if an attacker can read LSASS's memory, they can potentially extract passwords, password hashes, and authentication tickets for every account that has recently logged into that machine.

Under normal conditions, only Windows itself (running as the `SYSTEM` account) has good reason to touch LSASS's memory. A regular human user's account doing so is already a strong signal something's wrong.

**Your task:** Windows keeps a native audit trail of processes accessing other processes. Paste this into the Discover search bar:

```
Channel: "Security" and (EventID: 4663 or EventID: 4656) and ObjectName: *lsass.exe* and not SubjectUserName: *$
```

![[kibana-lsass-4656-4663.png]]

> **What this filter actually says, in plain English:** 
> "Show me Windows security-audit events where a process either requested or completed access to something named `lsass.exe` 
> — but only if the account doing it was a real human user, 
> not a computer/service account." 
> (Computer and service accounts always end in `$` in Windows 
> — that's why `not SubjectUserName: *$` filters those routine ones out.)

Add these as columns for a cleaner view (hover over each field in the left sidebar → click the `+` to add as a column):
- `SubjectUserName`
- `ProcessName`
- `ObjectName`
- `AccessMask`


![[kibana-lsass-add-as-column.png]]

After adding the columns, here's the view

![[kibana-columns-added.png]]

### Case Clue #1

You should find **two** events, same timestamp, same user, same process — but with **different `AccessMask` values**. This isn't a mistake or a duplicate. 

Windows logs the *request* for access (event 4656) and the *actual use* of that access (event 4663) as two separate log entries for the same real-world action.

**Decode the evidence:**

`AccessMask` is a number that packs several yes/no permissions into one value — think of it like a set of checkboxes, where each possible permission is represented by a specific bit, and the final number is all the checked boxes added together.

- **`0x10`** = permission to **read the process's memory**. On its own, this is the *only* right actually required to pull credentials out of LSASS.
- **`0x1010`** = the same memory-read permission, **plus** an additional permission (`0x1000`, "query limited information"). This specific combined value, `0x1010`, is well-documented as the exact access mask that **Mimikatz's `sekurlsa::logonpasswords` command** requests when it runs.

**Case Clue #1 unlocked:** You've found a human user account, running PowerShell, requesting the *exact* access-rights combination associated with a known credential-theft tool, directly against LSASS. 

That's not proof yet on its own — but it's a very strong lead.

**Write it down** (seriously — real analysts keep case notes): 
timestamp, username, process name, and both `AccessMask` values. 

You'll need this to compare against your next piece of evidence.

---

## Investigation Part 2: The Second Witness

One log source giving you a lead is a start. 

A **second, independent** log source confirming the same story is how real investigations get built. 

This is where **Sysmon** comes in — a monitoring tool that logs much richer technical detail than Windows' native auditing does.

**Your task:** paste this into Discover:

```
Channel: "Microsoft-Windows-Sysmon/Operational" and EventID: 10 and TargetImage: *lsass.exe* and CallTrace: *UNKNOWN*
```

![[kibana-lsass-sysmon.png]]


> **What this filter says:** "Show me Sysmon's record of one process accessing another process's memory, where the target was `lsass.exe`, and where part of the technical call trail couldn't be matched to any known file on disk."

Add these columns:
- `SourceImage`
- `TargetImage`
- `GrantedAccess`
- `SourceProcessGUID`
- `CallTrace`


![[kibana-lsass-sysmon-columns-added.png]]


### Case Clue #2

You should find exactly **one** event. Look closely at three things:

1. **Does the timestamp match** what you wrote down from Part 1? (Should be within a second or two.)
2. **Does `GrantedAccess` match** the `0x1010` value from before?
3. **What's going on in `CallTrace`?**

`CallTrace` is Sysmon showing you the technical breadcrumb trail of code that led to this action — which DLL (a shared code library) each step of the process passed through. Most of it will look like normal Windows internals (`ntdll.dll`, `KERNELBASE.dll`) — every process passes through these, attacker or not.

The part that matters is the last entry: **`UNKNOWN(...)`**. This means that step of the code isn't associated with *any* recognizable file on disk. Code that exists purely in memory, with nothing on disk to point to, is the signature of **fileless attack tooling** — malicious code that was loaded and run directly in memory, specifically to avoid leaving a file behind for antivirus to catch.

**Case Clue #2 unlocked:** Same process, same target, same suspicious access level, seen independently through a second logging tool — and now with direct evidence the code behind it wasn't a normal, on-disk program at all.

---

## Close the Case

Based on what you found in Parts 1 and 2, write a short incident summary — 3–5 sentences, like you're handing it to your SOC Lead. Cover:

- What process was involved, and what account was it running as?
- What did it access, and with what specific permission level?
- What evidence suggests this wasn't a normal, legitimate program?
- Was the initial alert justified? Why?

**No answer key here on purpose** — compare your write-up with a teammate who worked the same case and see where your reasoning matches or differs. That comparison is often where the most learning happens.

---

## What You Actually Learned

If you made it through both parts, you now know how to:
- Pull a specific, targeted filter out of a sea of log data using Kibana Discover
- Read and interpret a Windows `AccessMask` instead of just seeing a scary-looking hex number
- Corroborate one piece of evidence with a second, independent source
- Recognize what "fileless" or "in-memory" attack activity looks like in a call trace

This exact technique has a name in the security industry: **MITRE ATT&CK T1003.001 — OS Credential Dumping: LSASS Memory**. Every large organization's SOC watches for this pattern. You just hunted it by hand.

---

## Next Case File

Coming in a future session: visualizing this kind of evidence at scale using **Kibana Lens** — instead of finding one suspicious process by hand, you'll learn to spot it automatically across thousands of events.
