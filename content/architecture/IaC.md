### Why OpenTofu, not Terraform

We've decided to use **OpenTofu** instead of Terraform for CyberStorm SIEM's infrastructure. In short: the two tools are nearly identical at the syntax level, but OpenTofu ships native client-side state encryption — which directly closes an open security gap in our design (our state file will contain sensitive data tied to the collective AI system's API keys and DynamoDB context store) — and it carries a permissive, unambiguous open-source license (MPL 2.0) with Linux Foundation governance, fitting a volunteer/community project better than a single-vendor-controlled tool. Since we have no existing Terraform/HCP investment to lose, there's no downside to making this call now, before any real infra exists.

Full reasoning and the formal ADR are here: **[soc.lanc3.com/architecture/opentofu-vs-terraform](https://soc.lanc3.com/architecture/opentofu-vs-terraform)**

Everything below has been updated accordingly — commands, install steps, and terminology now reference OpenTofu (`tofu`) rather than Terraform (`terraform`). The good news: because OpenTofu is a compatible fork, almost everything about *learning* IaC concepts is identical — HCL syntax, `.tf` files, the plan/apply workflow. Only the binary name and a couple of installer commands change.

---

### What is IaC?

Infrastructure as Code (IaC) means defining and managing infrastructure through config files instead of manually clicking through a console/GUI. Because configs are declarative and version-controlled, changes to infrastructure become repeatable, auditable, and easy to collaborate on.

#### Phase 0 — Foundations (Day 1–2, before touching AWS)

**Learn OpenTofu/HCL basics first.** Don't skip this even if eager to jump into AWS. Two good starting points:

- OpenTofu's own tutorial (same workflow, same syntax, uses the `tofu` binary): [https://opentofu.org/docs/intro/](https://opentofu.org/docs/intro/)
- HashiCorp's official Terraform AWS tutorial is still fine to follow for *learning the concepts* — the config language and workflow are the same. Just run every command as `tofu` instead of `terraform` once you get to the CLI steps: [https://developer.hashicorp.com/terraform/tutorials/aws-get-started](https://developer.hashicorp.com/terraform/tutorials/aws-get-started)

This walks through: install OpenTofu, write a basic `.tf` file, `tofu init`, `tofu plan`, `tofu apply`, `tofu destroy`.

Do this exact tutorial first, end to end, with a throwaway AWS resource (a single EC2 instance) — don't touch your real project resources yet.

**Core concepts to understand before writing your project's code:**

- Providers (the `aws` provider block — identical in OpenTofu and Terraform, same registry)
- Resources (`resource "aws_instance" "x" {...}`)
- Variables (`variable` blocks + `terraform.tfvars` — yes, the file is still called `terraform.tfvars` even in OpenTofu; the name stuck)
- Outputs
- State (and why it should live in S3, not locally, for a team)
- State encryption (OpenTofu-specific — covered in Phase 3 below)

#### Phase 1 — Study an Established Open-Source Example (Day 2–3)

Before writing our own config from scratch, it's worth studying how a mature, real-world honeypot/telemetry project has structured its Terraform — not to copy it, but to learn from patterns that have already been battle-tested. One useful reference is the SANS DShield sensor project's Terraform code
(referenced in doc 17 — the repo is `DShield-ISC/dshield`, though the AWS Terraform specifically may be in a sibling repo or branch; worth checking `github.com/DShield-ISC` org for a `dshield-sc` or similar Terraform-focused repo).

We're building our own architecture, workflows, and tooling decisions (OpenTofu, our own module structure, our own workstream ownership model) — this step is purely about learning transferable IaC patterns, the same way you'd read someone else's open-source code to pick up good habits, not modeling our project's identity on theirs. The value here is seeing:

- How they structure variables vs. hardcoded values
- How they use `null_resource` + provisioners to bootstrap install scripts onto the instance (this maps directly to what you'll do for installing Docker + ELK)
- What mistakes to avoid — the Azure guest diary (doc 17) is genuinely useful reading here, since the author documents _exactly_ what broke when their Terraform got stale (old provider versions, broken provisioners) — a good "here's what not to do" case study.

Since this reference repo is written in Terraform (not OpenTofu), the code will still run fine through `tofu` — just mentally substitute the binary name as you follow along.

#### Phase 2 — Write Your Project's Code (Week 1)

Structure it as one resource block per line item:

```
terraform/
├── main.tf          # provider config, backend config
├── variables.tf     # instance_type, region, etc.
├── ec2.tf           # the SIEM host
├── ebs.tf           # the 50GB volume
├── s3.tf            # log landing bucket + lifecycle policy
├── vpc.tf           # VPC, subnets, security groups, endpoints
├── cloudtrail.tf
├── guardduty.tf
├── outputs.tf       # e.g. EC2 public IP, S3 bucket name
└── terraform.tfvars.example
```

> Note: we're keeping the folder named `terraform/` and files with the `.tf` extension — OpenTofu reads `.tf` files natively, no renaming needed. (`.tofu` extension exists as an option but isn't required, and sticking with `.tf` keeps the reference material above directly copy-paste-able.)

Each `.tf` file maps to a numbered resource in your existing provisioning doc — that doc essentially _is_ the spec we're coding against.

#### Phase 3 — Remote State + State Encryption (Week 1, once basics work)

Once a local `tofu apply` works, move state to S3 + DynamoDB locking
(refer to section 6.1 — "State stored in S3 with DynamoDB locking"). HashiCorp's backend guide is still accurate for the S3 backend block syntax:

- [https://developer.hashicorp.com/terraform/language/backend/s3](https://developer.hashicorp.com/terraform/language/backend/s3)

This is a small but important step — without it, only one person can ever safely run `tofu apply`, which defeats the point of a team project.

**This is also where OpenTofu's state encryption comes in** — since our state will hold sensitive references (API keys, DynamoDB context store details), enable client-side encryption on top of the S3 backend:

- [https://opentofu.org/docs/language/state/encryption/](https://opentofu.org/docs/language/state/encryption/)

Key provider choice (AWS KMS vs. passphrase) still needs deciding. This was flagged as a follow-up action in ADR-0002 (the OpenTofu adoption decision) — but ADR-0002 itself only covers the choice of OpenTofu over Terraform, not this specific question. Once we decide, it should get its own ADR (e.g. ADR-0003) so the reasoning is recorded, not just the conclusion.

#### Phase 4 — GitHub Actions CI/CD (Week 1, parallel)

Once OpenTofu works locally, wire up the pipeline: `tofu plan` on PR, `tofu apply` on merge to main. OpenTofu maintains its own official GitHub Action, built for this exact pattern:

- [https://github.com/opentofu/setup-opentofu](https://github.com/opentofu/setup-opentofu)

(HashiCorp's Terraform GitHub Actions guide describes the same PR-plan / merge-apply pattern if you want the conceptual walkthrough — just swap the action and CLI calls for the OpenTofu equivalents above.)

#### Phase 5 — CloudWatch Scaling Alarms (Week 1–2)

Once base infra is up, add the CloudWatch alarms from your Section 7 scaling triggers (EBS >70%, CPU sustained >70%, etc.) as OpenTofu resources too (`aws_cloudwatch_metric_alarm`) — this keeps the alarms themselves under IaC rather than manually clicked into the console, consistent with your immutability principle.

----------

### Suggested First Exercise/Practice

Start with a **single, small, real win first**: write the config for just the S3 bucket (Resource #3 in your doc — simplest, lowest-risk resource). Get that one resource's full lifecycle working (`tofu init` → `tofu plan` → `tofu apply` → `tofu destroy`) before moving to EC2/VPC, which are more complex and interdependent.

-----

**Real, runnable example, not pseudocode.** It includes encryption, public-access blocking, versioning, and the Glacier lifecycle rule mentioned throughout your provisioning doc — so it's a faithful implementation of the spec, not a toy.

**Heavy comments by design.** Since this is the IaC lead's first task, nearly every block has a comment explaining _why_, not just _what_ — e.g. why the bucket name includes the account ID, why state is local for now but won't stay that way, why `.tfvars` is gitignored but `.tfvars.example` isn't.

**The README has a "what to actually understand" checklist** — deliberately framed so they're not just running commands successfully but can explain _why_ each step matters. That's the difference between "it worked" and actual learning.

**Next step is pre-seeded.** The README points them toward CloudTrail (Resource #4) as the next task, and flags that it'll consume this file's `log_bucket_arn` output — setting them up to learn how resources reference each other, which is the next real concept after the basic apply/destroy loop.

To get this into your `siem` repo, they'd drop this into `terraform/` per the structure outlined earlier, then:

```bash
git add terraform/
git commit -m "Add S3 log-landing bucket config (Resource #3)"
git push
```

refer to
IaC-1st-exercise.zip

-------

### Further Reading

#### What is OpenTofu?

OpenTofu is an Infrastructure as Code (IaC) tool, originally forked from Terraform (created by HashiCorp, later acquired by IBM) after Terraform's 2023 license change. OpenTofu remains fully open source under MPL 2.0 and is governed by the Linux Foundation rather than a single company. It's platform-agnostic, so you can use the exact same tool to manage your AWS cloud alongside other platforms like Azure, GCP, or on-premise systems.

#### Why OpenTofu specifically

- **Multi-cloud**: works across AWS, Azure, GCP, and other platforms/services through a single tool
- **Readable syntax**: HCL (HashiCorp Configuration Language) is designed to be quick to write and understand — identical syntax to Terraform
- **State tracking**: keeps a state file that maps your config to real-world resources, so it knows what's changed
- **Native state encryption**: client-side encryption of the state file — closes a real security gap for us, without needing an external KMS workflow bolted on
- **Open governance**: MPL 2.0 license, Linux Foundation stewardship, no single-vendor control over the roadmap
- **Version control friendly**: configs live in git, enabling PR review, rollback, and shared ownership — same workflow we're already using for the SIEM repo

#### How it connects to real APIs: Providers

OpenTofu itself is API-agnostic — **providers** are plugins that translate your HCL into actual API calls for a given platform (e.g., the AWS provider, GitHub provider). Most common ones are in the [OpenTofu Registry](https://registry.opentofu.org/) (mirrors the same providers as the Terraform Registry — AWS, Azure, GCP, GitHub, etc. all work identically); if something's missing, you can write a custom provider.

#### Declarative, not procedural

You describe the _end state_ you want ("I need one EC2 instance, one S3 bucket, this IAM role"), not the steps to get there. OpenTofu figures out resource dependencies on its own and sequences creation/destruction in the right order.

#### The core workflow

| Step                        | What happens                                                  |
| ---------------------------- | -------------------------------------------------------------- |
| **Scope**                    | Decide what infrastructure the project actually needs         |
| **Author**                   | Write the `.tf` config files                                   |
| **Initialize** (`tofu init`) | Download the required provider plugins                         |
| **Plan** (`tofu plan`)       | Dry-run showing exactly what will be added/changed/destroyed   |
| **Apply** (`tofu apply`)     | Execute the plan against real infrastructure                   |

### In our case (Cyber Storm SIEM project)

- **Provider**: AWS provider (single-account setup)
- **Compute**: EC2 t3.medium — sized down deliberately to stay within budget/leadership constraints
- **Storage**: EBS volume capped at 50GB attached to the EC2 instance; S3 bucket kept under 10GB (used for log archival / cold storage)
- **Networking**: VPC with Flow Logs enabled as one of our log sources feeding the ELK stack
- **Log sources tied to this infra**: CloudTrail, VPC Flow Logs, GuardDuty findings, honeypot telemetry — all provisioned/wired up via OpenTofu rather than manually clicked together in the console
- **State security**: state file encrypted client-side via OpenTofu's native encryption (see Phase 3) — protects sensitive references (API keys, DynamoDB context store details) even if the S3 backend were ever misconfigured
- **Why this matters for us specifically**: since infra sizing has been revised before and will likely be revised again, having it in OpenTofu means a change (e.g., bumping instance size, adjusting EBS) is a one-line diff in a PR (pull request) — reviewable, trackable in git history, and reversible — rather than someone remembering to manually resize things in the AWS console and nobody else knowing it happened
- **Team workflow implication**: since we have five volunteers across different workstreams (Infra/IaC, ELK Platform, Data Pipeline/OCSF, Detection, Observability/Threat Intel), state + PR (pull request) review is what lets the Infra/IaC lead's changes stay visible and reviewable by the rest of us instead of infra drifting silently

-------

### Repo structure

```
siem-infra/
├── environments/
│   └── prod/
│       ├── main.tf              # wires modules together for this env
│       ├── variables.tf
│       ├── outputs.tf
│       ├── terraform.tfvars     # actual sizing values (t3.medium, 50GB, etc.)
│       └── backend.tf           # remote state config (S3 + DynamoDB lock + encryption)
│
├── modules/
│   ├── network/                 # VPC, subnets, flow logs, security groups
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   ├── compute/                 # EC2 instance, EBS volume, IAM instance role
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   ├── logging/                 # CloudTrail, S3 log bucket, GuardDuty
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   ├── observability/           # OpenTelemetry Collector config/infra, alerting hooks
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   └── honeypot/                # DShield-inspired honeypot instance + telemetry pipe
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
│
├── .gitignore                   # exclude .tfstate, .terraform/, *.tfvars secrets
└── README.md                    # setup instructions, module ownership map
```

#### Rough mapping to volunteer roles

| Module                                   | Owner (workstream)                                     |
| ---------------------------------------- | ------------------------------------------------------ |
| `network/`                               | Infrastructure & IaC Lead                              |
| `compute/`                               | Infrastructure & IaC Lead                              |
| `logging/`                               | Data Pipeline & OCSF Lead                               |
| `observability/`                         | Observability & Threat Intel Lead                       |
| `honeypot/`                              | Detection Engineer                                      |
| `environments/prod/` (composition layer) | Team Lead — integration & PR review across all modules  |

#### Conventions in README

- **State**: use a remote backend (S3 + DynamoDB for locking, plus client-side encryption) rather than local `.tfstate` — critical with five people touching the same infra
- **PR flow**: every change goes through `tofu plan` output pasted into the PR before merge, so reviewers see the diff before `apply`
- **Variables not hardcoded**: sizing (instance type, EBS size, S3 thresholds) lives in `terraform.tfvars`, not baked into module code — makes future revisions a one-file change
- **No one applies directly**: `tofu apply` only runs after PR approval, ideally via CI rather than a volunteer's laptop, so there's always a record of who approved what

-----

### Practical practice steps — installing and running OpenTofu

Here I (Lance) am documenting my own practice with OpenTofu.
Since I'm using macOS (MacBook), the steps here will be irrelevant to your practice as you are likely to use a PC.
Refer to OpenTofu's official install guide for your platform: [https://opentofu.org/docs/intro/install/](https://opentofu.org/docs/intro/install/)

If you are using macOS like I do, in your terminal, run:

```bash
brew install opentofu
```

(Note: this replaces the earlier `brew tap hashicorp/tap` / `brew install hashicorp/tap/terraform` steps used for Terraform — OpenTofu has its own official Homebrew formula, no separate tap needed.)

To confirm it's working:

```bash
tofu -version
```

Expected output will look like:

```
OpenTofu v1.11.x
on darwin_arm64
```

-------

### Next: install AWS CLI on your PC/Mac

Same as before — this step is unaffected by the OpenTofu vs Terraform choice, since the AWS CLI is just used for credentials, not for running IaC itself.

```bash
brew install awscli
```

Confirm with:

```bash
aws --version
```

-----

### Set up credentials

```bash
aws configure
```

For credentials, have your own AWS practice account ready, or the team's AWS account (paid for with sponsorship from CSA / Div0).

-----

### Quick command cheat-sheet: Terraform → OpenTofu

If you've used Terraform before, or are following older reference material (including our own earlier drafts of this doc), here's the direct mapping:

| Terraform command   | OpenTofu equivalent |
| --------------------| --------------------|
| `terraform init`    | `tofu init`         |
| `terraform plan`    | `tofu plan`         |
| `terraform apply`   | `tofu apply`        |
| `terraform destroy` | `tofu destroy`      |
| `terraform state`   | `tofu state`        |
| `terraform -version`| `tofu -version`     |

Everything else — file names, HCL syntax, provider blocks, module structure — stays the same.
