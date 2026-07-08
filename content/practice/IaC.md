---
title: "Practice: IaC — Provisioning a SIEM Host with OpenTofu"
tags: [practice, iac, opentofu, elk, wazuh]
date: 2026-07-08
---

# Practice: IaC — Provisioning a SIEM Host with OpenTofu

Prerequisite: AWS CLI configured with credentials — see [/practice/aws](https://soc.lanc3.com/practice/aws).

This goes beyond the basic "single S3 bucket" tutorial exercise — it's a working OpenTofu config that provisions an actual EC2 host sized and networked for running **either ELK or Wazuh**, since that platform decision is still open (see the note at the bottom — it deserves its own ADR).

Full code lives in a separate repo: [github.com/lanc3lee/iac-practice](https://github.com/lanc3lee/iac-practice) — kept out of this vault since it's runnable infrastructure code, not documentation (see the note on separating docs from code). 

Below is what it does and why, not a line-by-line walkthrough — read the files directly in that repo for the details, they're commented.
## What this provisions

- **One EC2 instance** — `t2.micro`, free-tier eligible. Deliberately smaller than our real production spec (`t3.medium` per the NFR matrix) — this is a practice/PoC host, not the team's shared infrastructure.
- **One EBS volume** — 20GB, `gp3`, encrypted at rest. Free tier covers up to 30GB total, so this leaves headroom.
- **One security group** — SSH restricted to your own IP, plus platform-specific ports (see below). Nothing open to `0.0.0.0/0` except outbound.
- **Outputs** — public IP, instance ID, security group ID, and a ready-to-paste SSH command.

## The ELK-vs-Wazuh port logic

Rather than hardcoding one platform's ports, `siem_platform` is a variable (`"elk"` or `"wazuh"`) that drives which security group rules actually get created, using a `dynamic` block:

| Platform | Ports opened | Purpose |
|---|---|---|
| **ELK** | 9200, 5601, 5044 | Elasticsearch REST API, Kibana UI, Logstash Beats input |
| **Wazuh** | 1514, 1515, 55000, 443 | Agent event data, agent enrollment, Wazuh API, dashboard (HTTPS) |

This means the same config can stand up a practice host for either platform just by changing one variable — useful right now since we haven't committed to one yet, and useful later as a template regardless of which way that decision goes.

## Why this is a step up from the basics exercise

- **Variables with validation** — `siem_platform` uses a `validation` block, so a typo (`"wazhu"`) fails fast with a clear error instead of silently doing nothing
- **A `dynamic` block** — generates security group rules conditionally based on a variable, rather than every rule being hardcoded
- **A data source** (`data "aws_ami"`) — looks up the latest Ubuntu 22.04 AMI at apply time instead of hardcoding an AMI ID that goes stale
- **Sensible defaults with override capability** — works out of the box for practice, but every value is overridable via `terraform.tfvars`
- **Encrypted EBS by default** — a habit worth carrying into the real project regardless of what this instance ends up running

## Deliberately different from our real project setup

- **Local state, not remote S3 + DynamoDB.** This is a solo practice exercise — remote state and locking only earn their keep once multiple people touch the same state file. Our real project state uses S3 + DynamoDB + KMS encryption per ADR-0002 and ADR-0003 — that pattern isn't repeated here on purpose, to keep this exercise fast to run and tear down.
- **`t2.micro`, not `t3.medium`.** Sized for the AWS free tier, not for running a real production SIEM workload under load.

## How to prep it (edit terraform.tfvars)

```bash
cd tofu/
cp terraform.tfvars.example terraform.tfvars
# edit terraform.tfvars — set my_ip_cidr, key_pair_name, and siem_platform

```

refer to https://soc.lanc3.com/practice/aws for details on how to get:
a. aws_region (in case you didn't define it right during initial aws setup)
b. my_ip_cidr
c. key_pair_name

![[terraform-tfvars-region.png]]



### Running 


```
tofu init
tofu plan
tofu apply
```


-------


Once up, SSH in using the `ssh_command` output, then install ELK or Wazuh manually for now — automating the software install itself (via provisioners or a bootstrap script) is a good next practice step, not covered here.

Tear down when done to avoid leaving resources running on the free-tier account:

```bash
tofu destroy
```
-------

## Windows setup notes

If you're on Windows rather than linux or Mac, the OpenTofu/AWS CLI workflow itself is identical — only key pair creation and SSH key file handling differ slightly.

**Key pair location:** use `%USERPROFILE%\.ssh\` (typically `C:\Users\<username>\.ssh\`) instead of `~/.ssh/`.

**Creating the key pair (PowerShell):**

powershell

```powershell
mkdir -Force $env:USERPROFILE\.ssh

aws ec2 create-key-pair `
  --key-name siem-practice `
  --query 'KeyMaterial' `
  --output text | Out-File -Encoding ascii "$env:USERPROFILE\.ssh\siem-practice.pem"
```

Note the `-Encoding ascii` — PowerShell defaults to UTF-16 for redirected output, which corrupts `.pem` files. Using `Out-File -Encoding ascii` avoids that; plain `>` redirection in PowerShell can silently break the key format.

**Setting file permissions** (Windows equivalent of `chmod 400`):

powershell

```powershell
icacls "$env:USERPROFILE\.ssh\siem-practice.pem" /inheritance:r
icacls "$env:USERPROFILE\.ssh\siem-practice.pem" /grant:r "$($env:USERNAME):(R)"
```

This restricts the key file to read-only access for your user only — without it, some SSH clients will refuse to use the key.

**Connecting via SSH** — Windows 10/11 ships OpenSSH client built in, so the command is otherwise identical to Mac/Linux:

powershell

```powershell
ssh -i "$env:USERPROFILE\.ssh\siem-practice.pem" ubuntu@<public-ip>
```








## Open decision this practice exercise doesn't resolve

Whether we actually use **ELK or Wazuh** for the real CyberStorm SIEM is still undecided — this config supports practicing with either, but doesn't make that call. That decision — cost, maintenance burden, out-of-the-box detection content, community support, fit with our existing OCSF/Sigma pipeline — deserves its own ADR before it affects the real project. Flagging it here so it doesn't get decided implicitly just because one platform got practiced with first.
