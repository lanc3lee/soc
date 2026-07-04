
### What an NFR matrix is

Most of what our workstreams design gets discussed in terms of **functional requirements** — "the pipeline should normalize CloudTrail logs into OCSF," "the detection engine should fire on failed login patterns." These describe _what_ the system does.

A **Non-Functional Requirements (NFR) matrix** captures the _qualities_ the system must have regardless of feature 
— things like how much it can cost, 
how available it needs to be, 
how fast it needs to respond, 
how secure it needs to be, and 
how much load it can take before it breaks. 

These are easy to skip because no single feature "needs" them — but every workstream's design either satisfies them or silently violates them.

### Why our team needs this specifically

Right now, constraints like "EC2 capped at t3.medium," "EBS capped at 50GB," "S3 under 10GB," "PoC cost under SGD 500/month" exist as scattered facts 
— some in our head, 
some in leadership's revisions, 
some in individual PR discussions. 

An NFR matrix pulls all of that into **one document every workstream lead checks their design against before writing code**, not after.

Concretely, this prevents situations like:

- Detection adding a heavy correlation rule that blows past CPU/memory budget on the t3.medium
- Observability's OTel exporters pushing cost past the ceiling nobody notices until the AWS bill arrives
- A workstream assuming "someone else" owns backup/recovery, and nobody actually does

It's also one of the clearest signals of architectural ownership 
— a good architect isn't the person who writes the most code, 
they're the person who makes sure the _whole system_ still holds together under real-world constraints (cost, security, uptime) 
as five different people build five different pieces of it in parallel.