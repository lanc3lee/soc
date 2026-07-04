
As a team lead for setting up a SIEM for Cyber Storm Center, I wondered if how this exercise can be a transition opportunity to become an architect. 

Here's documenting the thought process


**1. Turn Pull Request (PR) review role into an architecture review board — on paper.**  
Instead of just reviewing code correctness per-module, add an explicit "architecture decision record" (ADR) practice: 
Every time a workstream makes a non-trivial choice (e.g., Logstash vs. Vector for OCSF normalization, Sigma rule storage format), 
require a short ADR — problem, options considered, decision, tradeoffs.
Review and merge those too. 
This is a very recognizable senior/staff/architect artifact.

**2. Own the cross-cutting concerns nobody else naturally owns.**  
Individual workstreams optimize locally. 
The things that fall through the cracks between Infra, ELK, Data Pipeline, Detection, and Observability are exactly where an architect earns their title:

- Cost model across all five workstreams combined, not just infra  
  — model what happens if Detection adds heavier Sigma correlation or Observability adds more OTel exporters)
  
- A shared data contract/schema (OCSF field consistency) that Detection and Observability both depend on
  
- Failure domains — what happens if the EC2 instance dies, is state/data recoverable, what's your actual RTO/RPO story
  
- Security of the SIEM itself (who can see PR merges = who can touch PROD, secrets management for API keys)

**3. Push the AI security sub-team's OWASP LLM Top 10 threat model further — this is your most differentiated angle.**  
Threat-modeling our own team's collective-AI tool (DynamoDB + Lambda + API Gateway + per-volunteer API keys) is genuinely architect-level work already. Widen it: write a proper threat model document with attack trees, not just a mapping to OWASP categories. That document alone is a strong BSides submission on its own, separate from the SIEM release.

**4. Introduce a formal non-functional requirements (NFR) matrix.**  
Scalability triggers, availability targets, cost ceilings, security baselines, observability SLOs — get these written down as a single doc that every workstream's design has to satisfy. 

This is the artifact that separates "person who reviews PRs" from "person who sets the technical direction."

**5. Do a build-vs-reference-architecture comparison.**  
Before/alongside our custom build, write a short comparative analysis against how this would look on a managed alternative (Security Lake + OpenSearch, Elastic Cloud, Wazuh) — not to switch, but because being able to articulate _why_ self-hosted Docker Compose ELK on EC2 t3.medium is the right call for a volunteer/budget-constrained context, with tradeoffs named explicitly, is a hallmark of architectural thinking versus just execution.

**6. Present the architecture, not just review it.**  
Volunteer to be the one who presents the overall system design to Div0 leadership (Dennis) or at BSides — the end-to-end diagram, the decision rationale, the tradeoffs. Architects are often defined less by depth in one area and more by being the person who can explain the whole system coherently to someone who wasn't in the room for any of it.

### Qualities that matter here

- **Judgment under constraint** — we already have a good instinct for this (accepting the EC2/EBS/S3 downsizing from leadership rather than fighting it). Architects are valued for good tradeoffs under real constraints, not for building the most elegant possible system in a vacuum.
  
- **Systems thinking** — seeing how a decision in Data Pipeline ripples into Detection's rule performance or Observability's alert volume.
  
- **Written communication** — ADRs, threat models, NFR docs. 
  Architecture lives in documents people can review asynchronously, not in our head.
  
- **Comfort saying no** — part of the job is stopping scope creep or premature optimization from individual workstreams when it doesn't serve the whole system.
  
- **Technical breadth over depth in any one area** — ywe don't need to be the best Terraform person or the best Sigma rule writer; we need to understand enough of each to ask the right integration questions.
  
- **Stakeholder translation** — being able to talk to leadership in cost-and-risk terms and to our volunteers in technical terms, without those being two different stories.

--------

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


