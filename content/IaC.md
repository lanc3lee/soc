### What is IaC?

Infrastructure as Code (IaC) means defining and managing infrastructure through config files instead of manually clicking through a console/GUI. Because configs are declarative and version-controlled, changes to infrastructure become repeatable, auditable, and easy to collaborate on.
#### Phase 0 — Foundations (Day 1–2, before touching AWS)

**Learn Terraform basics first.** Don't skip this even if eager to jump into AWS — HashiCorp's official tutorial is free and the right starting point:

- [https://developer.hashicorp.com/terraform/tutorials/aws-get-started](https://developer.hashicorp.com/terraform/tutorials/aws-get-started)

This walks through: install Terraform, write a basic `.tf` file, `terraform init`, `terraform plan`, `terraform apply`, `terraform destroy`. 

Do this exact tutorial first, end to end, with a throwaway AWS resource (their tutorial uses a single EC2 instance) — don't touch your real project resources yet.

**Core concepts to understand before writing your project's code:**

- Providers (the `aws` provider block)
- Resources (`resource "aws_instance" "x" {...}`)
- Variables (`variable` blocks + `terraform.tfvars`)
- Outputs
- State (and why it should live in S3, not locally, for a team)

#### Phase 1 — Reference SANS's Pattern (Day 2–3)

Browse the actual SANS DShield-SC Terraform code structure 
(referenced in doc 17 — the repo is `DShield-ISC/dshield`, though the AWS Terraform specifically may be in a sibling repo or branch; worth checking `github.com/DShield-ISC` org for a `dshield-sc` or similar Terraform-focused repo). 
The educational value here isn't copying their code — it's seeing:

- How they structure variables vs. hardcoded values
- How they use `null_resource` + provisioners to bootstrap install scripts onto the instance (this maps directly to what you'll do for installing Docker + ELK)
- What mistakes to avoid — the Azure guest diary (doc 17) is genuinely useful reading here, since the author documents _exactly_ what broke when their Terraform got stale (old provider versions, broken provisioners) — a good "here's what not to do" case study.

#### Phase 2 — Write Your Project's Terraform (Week 1)

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

Each `.tf` file maps to a numbered resource in your existing provisioning doc — that doc essentially _is_ the spec we coding against.

#### Phase 3 — Remote State (Week 1, once basics work)

Once a local `terraform apply` works, move state to S3 + DynamoDB locking 
(refer to section 6.1 — "State stored in S3 with DynamoDB locking"). HashiCorp's guide:

- [https://developer.hashicorp.com/terraform/language/backend/s3](https://developer.hashicorp.com/terraform/language/backend/s3)

This is a small but important step — without it, only one person can ever safely run `terraform apply`, which defeats the point of a team project.

#### Phase 4 — GitHub Actions CI/CD (Week 1, parallel)

Once Terraform works locally, wire up the pipeline: `terraform plan` on PR, `terraform apply` on merge to main. HashiCorp has an official guide for this exact pattern:

- [https://developer.hashicorp.com/terraform/tutorials/automation/github-actions](https://developer.hashicorp.com/terraform/tutorials/automation/github-actions)

#### Phase 5 — CloudWatch Scaling Alarms (Week 1–2)

Once base infra is up, add the CloudWatch alarms from your Section 7 scaling triggers (EBS >70%, CPU sustained >70%, etc.) as Terraform resources too (`aws_cloudwatch_metric_alarm`) — this keeps the alarms themselves under IaC rather than manually clicked into the console, consistent with your immutability principle.

----------

Suggested First Exercise/Practice

a **single, small, real win first**: write Terraform for just the S3 bucket (Resource #3 in your doc — simplest, lowest-risk resource). Get that one resource's full lifecycle working (`init` → `plan` → `apply` → `destroy`) before moving to EC2/VPC, which are more complex and interdependent.

-----

**real, runnable example, not pseudocode.** It includes encryption, public-access blocking, versioning, and the Glacier lifecycle rule mentioned throughout your provisioning doc — so it's a faithful implementation of the spec, not a toy.

**Heavy comments by design.** Since this is the IaC lead's first Terraform task, nearly every block has a comment explaining _why_, not just _what_ — e.g. why the bucket name includes the account ID, why state is local for now but won't stay that way, why `.tfvars` is gitignored but `.tfvars.example` isn't.

**The README has a "what to actually understand" checklist** — deliberately framed so they're not just running commands successfully but can explain _why_ each step matters. That's the difference between "it worked" and actual learning.

**Next step is pre-seeded.** The README points them toward CloudTrail (Resource #4) as the next task, and flags that it'll consume this file's `log_bucket_arn` output — setting them up to learn how Terraform resources reference each other, which is the next real concept after the basic apply/destroy loop.

To get this into your `siem` repo, they'd drop this into `terraform/` per the structure I outlined earlier, then:

bash

```bash
git add terraform/
git commit -m "Add S3 log-landing bucket Terraform (Resource #3)"
git push
```

refer to 
IaC-1st-exercise.zip


-------

Further Reading

What is Terraform?

Terraform is an Infrastructure as Code (IaC) tool created by HashiCorp (later acquired by IBM)
Because Terraform is platform-agnostic, you can use the exact same tool to manage your AWS cloud alongside other platforms like Azure, GCP, or on-premise systems

### Why Terraform specifically

- **Multi-cloud**: works across AWS, Azure, GCP, and other platforms/services through a single tool
- **Readable syntax**: HCL (HashiCorp Configuration Language) is designed to be quick to write and understand
- **State tracking**: Terraform keeps a state file that maps your config to real-world resources, so it knows what's changed
- **Version control friendly**: configs live in git, enabling PR review, rollback, and shared ownership — same workflow we're already using for the SIEM repo

### How it connects to real APIs: Providers

Terraform itself is API-agnostic — **providers** are plugins that translate your HCL into actual API calls for a given platform (e.g., the AWS provider, GitHub provider). Most common ones are in the [Terraform Registry](https://registry.terraform.io/browse/providers); if something's missing, you can write a custom provider.

### Declarative, not procedural

You describe the _end state_ you want ("I need one EC2 instance, one S3 bucket, this IAM role"), not the steps to get there. Terraform figures out resource dependencies on its own and sequences creation/destruction in the right order.

### The core workflow

| Step                              | What happens                                                 |
| --------------------------------- | ------------------------------------------------------------ |
| **Scope**                         | Decide what infrastructure the project actually needs        |
| **Author**                        | Write the `.tf` config files                                 |
| **Initialize** (`terraform init`) | Download the required provider plugins                       |
| **Plan** (`terraform plan`)       | Dry-run showing exactly what will be added/changed/destroyed |
| **Apply** (`terraform apply`)     | Execute the plan against real infrastructure                 |


### In our case (Cyber Storm SIEM project)

- **Provider**: AWS provider (single-account setup)
- **Compute**: EC2 t3.medium — sized down deliberately to stay within budget/leadership constraints
- **Storage**: EBS volume capped at 50GB attached to the EC2 instance; S3 bucket kept under 10GB (used for log archival / cold storage)
- **Networking**: VPC with Flow Logs enabled as one of our log sources feeding the ELK stack
- **Log sources tied to this infra**: CloudTrail, VPC Flow Logs, GuardDuty findings, honeypot telemetry — all provisioned/wired up via Terraform rather than manually clicked together in the console
- **Why this matters for us specifically**: since infra sizing has been revised before and will likely be revised again, having it in Terraform means a change (e.g., bumping instance size, adjusting EBS) is a one-line diff in a PR (pull request)  — reviewable, trackable in git history, and reversible — rather than someone remembering to manually resize things in the AWS console and nobody else knowing it happened
- **Team workflow implication**: since we have five volunteers across different workstreams (Infra/IaC, ELK Platform, Data Pipeline/OCSF, Detection, Observability/Threat Intel), Terraform state + PR (pull request) review is what lets the Infra/IaC lead's changes stay visible and reviewable by the rest of us instead of infra drifting silently

-------

### Terraform repo structure

```
siem-infra/
├── environments/
│   └── prod/
│       ├── main.tf              # wires modules together for this env
│       ├── variables.tf
│       ├── outputs.tf
│       ├── terraform.tfvars     # actual sizing values (t3.medium, 50GB, etc.)
│       └── backend.tf           # remote state config (e.g., S3 + DynamoDB lock)
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
| `logging/`                               | Data Pipeline & OCSF Lead                              |
| `observability/`                         | Observability & Threat Intel Lead                      |
| `honeypot/`                              | Detection Engineer                                     |
| `environments/prod/` (composition layer) | Team Lead — integration & PR review across all modules |

#### conventions in README

- **State**: use a remote backend (S3 + DynamoDB for locking) rather than local `.tfstate` — critical with five people touching the same infra
- **PR flow**: every change goes through `terraform plan` output pasted into the PR before merge, so reviewers see the diff before `apply`
- **Variables not hardcoded**: sizing (instance type, EBS size, S3 thresholds) lives in `terraform.tfvars`, not baked into module code — makes future revisions a one-file change
- **No one applies directly**: `terraform apply` only runs after PR approval, ideally via CI rather than a volunteer's laptop, so there's always a record of who approved what



-----

Here I (Lance) am documenting my own practice with Terraform
Since I'm using MacOS (macbook), the steps here will be irrelevant to your practice as you are likely to use a PC. 
Refer to https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli for your practice

If you are using MacOS like I do, in your terminal, run

brew tap hashicorp/tap
brew install hashicorp/tap/terraform

```
lance@Lances-MacBook-Air ~ % brew tap hashicorp/tap

==> Tapping hashicorp/tap

Cloning into '/opt/homebrew/Library/Taps/hashicorp/homebrew-tap'...

...

lance@Lances-MacBook-Air ~ % brew install hashicorp/tap/terraform

==> Trusted formula hashicorp/tap/terraform

==> Would install 1 formula:

terraform

==> Fetching downloads for: terraform

✔︎ Formula terraform (1.15.7)                                                    Verified     32.8MB/ 32.8MB

==> Installing terraform from hashicorp/tap

...
```

To confirm it's working 

terraform -version

```
lance@Lances-MacBook-Air ~ % terraform -version

Terraform v1.15.7

on darwin_arm64
```

-------

next is to install AWS CLI on your PC/Mac 


```
lance@Lances-MacBook-Air ~ % brew install awscli
...
==> Would install 1 formula:

awscli

==> Downloading https://ghcr.io/v2/homebrew/core/awscli/manifests/2.35.15

#################################################################################################### 100.0%

==> Would install 8 dependencies for awscli:

...

==> Installing dependencies for awscli: openssl@3 ↑, mpdecimal, readline, sqlite, xz, lz4, zstd and python@3.14
...


lance@Lances-MacBook-Air ~ % aws --version

aws-cli/2.35.15 Python/3.14.6 Darwin/24.6.0 source/arm64

lance@Lances-MacBook-Air ~ %
```

-----

Next step is to set up credentials using 

```
aws configure

```

For credentials, have your own AWS practice account ready, or the team's AWS account (paid for with sponsorship from CSA / Div0)

