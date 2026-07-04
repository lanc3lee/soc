### What an ADR is

An **Architecture Decision Record (ADR)** is a short, standalone document that captures one significant technical decision — what problem it solves, what options were considered, what was chosen, and why. 

The key idea: instead of a decision living only in someone's head or buried in a Discord/ Superthread post, it becomes a permanent, searchable artifact in our repo/vault.

Why it matters for our team specifically:

- **New volunteers can ask "why is it built this way?" and get an answer**, instead of us having to remember and re-explain
  
- **Stop relitigating settled decisions** — if someone questions why Logstash was chosen over Vector three months from now, the ADR is the answer
  
- **It's evidence of architectural thinking**, not just working code — this is exactly the kind of artifact that shows up in senior/staff/architect portfolios and would strengthen a BSides submission
  
- **Decisions become reviewable** — since we are doing PR review anyway, an ADR can go through the same PR flow as code

Each ADR is one small file, numbered sequentially, and — critically — **immutable once accepted**. 

If circumstances change, we don't edit the old one; we write a new ADR that supersedes it. 

That way the history of _how the thinking evolved_ is preserved, not erased.

A few notes on writing ADR:

- **Numbering**: keep a single `decisions/` folder in your Obsidian vault (or repo) with sequential filenames — `ADR-0001-...`, `ADR-0002-...` — so the order of decisions is visible at a glance.
  
- **When to require one**: not every PR needs an ADR — reserve it for decisions that are hard to reverse or affect more than one workstream (tool choices, schema decisions, security boundaries). A config tweak doesn't need one; choosing Logstash over Vector does.
  
- **PR integration**: have workstream leads open the ADR as part of the same PR as the implementation, so you review the reasoning alongside the code, not after the fact.
