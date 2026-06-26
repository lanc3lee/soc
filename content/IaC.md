
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