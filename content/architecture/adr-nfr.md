---
title: "ADRs vs NFR Matrix: What Each One Is For"
tags: [architecture, adr, nfr, documentation]
date: 2026-07-08
---

# ADRs vs NFR Matrix: What Each One Is For

We use two different documentation artifacts for architecture in our CyberStorm SIEM project. They're related, but they answer different questions and are structured differently on purpose.

## What the acronyms mean

- **ADR = Architectural Decision Record.** A short document capturing one significant technical decision — the problem, the options considered, what was chosen, and why.
- **NFR = Non-Functional Requirement.** A requirement that describes a *quality* the system must have (cost, availability, security, performance) rather than a *feature* it must do. The **NFR matrix** is the single table that tracks all of these requirements and their current targets in one place.

## The core distinction

> **ADRs are the "why." The NFR matrix is the "current state."**

| | ADR | NFR Matrix |
|---|---|---|
| **Answers** | Why did we decide this? | What must the system satisfy, right now? |
| **Format** | One document per decision, numbered sequentially (ADR-0001, ADR-0002...) | One single living document |
| **Mutability** | Immutable once accepted — never edited after the fact | Edited in place as targets are confirmed or revised |
| **When it's created** | At the moment of a significant, hard-to-reverse decision | Once, then continuously maintained |
| **What changes** | Nothing — a new ADR supersedes an old one, the old one stays as history | Rows get updated whenever a target changes |

## Why the same numbering scheme doesn't work for both

ADRs are a **chronological decision log** — like commit history. Numbering them makes sense because each one is a frozen snapshot: "here's what we decided, on this date, and why."

The NFR matrix is not a sequence of decisions — it's **one current source of truth**. Numbering it the way we number ADRs (NFR-0001, NFR-0002...) would wrongly imply each version is a separate, frozen artifact — which undermines its actual job: being the single place anyone can check "what's our target for X right now?" without wondering which version is current.

## How they connect

A single ADR can update multiple rows in the NFR matrix at once. The ADR explains the reasoning; the matrix reflects the resulting target.

**Example:** ADR-0002 (adopt OpenTofu over Terraform) directly resolves the previously-unconfirmed "Secrets management approach" row in the NFR matrix's Security section. The ADR is the record of *why* we made that call. The matrix's Security row is the record of *what's true now* as a result.

## How to keep them in sync

The NFR matrix has two built-in mechanisms for this — don't skip them:

1. **"Last updated" field** at the top of the matrix — keep it current
2. **Change log table** at the bottom — every time a row changes, log it:

   ```
   | Date       | Change                                          | Related ADR |
   |------------|--------------------------------------------------|-------------|
   | 2026-07-08 | Secrets management row updated (OpenTofu state encryption) | ADR-0002    |
   ```

If a change to the matrix came from a real architectural decision, link the ADR. If it's just a routine confirmation of a previously-blank field (e.g., someone finally measured actual EBS usage), the change log entry doesn't need an ADR reference — not every NFR update is decision-worthy.

## File structure

```
decisions/
├── ADR-template.md
├── ADR-0001-log-normalization.md
├── ADR-0002-opentofu-vs-terraform.md
└── ADR-000N-...md          ← keep numbering sequentially

architecture/
└── NFR-matrix.md           ← one file, always current, edited over time
```

## The one-line rule to remember

**If you're recording why something was decided, write an ADR. If you're recording what's currently true about the system, update the NFR matrix.** When in doubt: decisions get numbered and frozen, current state gets one file and stays editable.
