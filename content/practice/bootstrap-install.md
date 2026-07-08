---
title: "Practice: Automating ELK/Wazuh Install via Bootstrap Script"
tags: [practice, iac, opentofu, automation, elk, wazuh]
date: 2026-07-08
---

# Practice: Automating ELK/Wazuh Install via Bootstrap Script

Builds on [/practice/IaC](https://soc.lanc3.com/practice/IaC), which provisions a bare EC2 host sized/networked for either ELK or Wazuh, but leaves software install as a manual step. This segment automates that install so the whole cycle — provision → install → ready dashboard — happens in one `tofu apply`, no manual SSH steps required.

Code lives in the same repo: [github.com/lanc3lee/iac-practice](https://github.com/lanc3lee/iac-practice).

## Two ways to automate this — and which one to use

OpenTofu (and Terraform) offer two different mechanisms for running install commands on a freshly created instance:

| | `user_data` (cloud-init) | `remote-exec` provisioner |
|---|---|---|
| **How it runs** | Passed to AWS at instance launch; the OS runs it automatically at first boot via cloud-init | OpenTofu SSHes into the instance *after* creation and runs commands directly |
| **Reliability** | Runs even if OpenTofu loses connection, is interrupted, or your laptop sleeps mid-apply | Fails if SSH isn't reachable yet, network hiccups, or OpenTofu is interrupted — has to be re-run manually |
| **Official guidance** | Recommended approach | OpenTofu/Terraform's own docs explicitly discourage provisioners as a "last resort" |

**We're using `user_data`.** It's simpler to reason about, doesn't depend on OpenTofu maintaining an SSH session throughout the install, and is the pattern you'll see in most real-world infra code — including the SANS DShield reference repo studied in Phase 1, and Wazuh's own official quick-start scripts.

## What gets added to the existing config

One new resource attribute on the EC2 instance, plus two new template files — one bootstrap script per platform, selected using the same `siem_platform` variable already driving the security group logic.

```
iac-practice/
└── tofu/
    ├── main.tf                    # add user_data to aws_instance
    ├── variables.tf               # unchanged
    ├── outputs.tf                 # unchanged
    ├── terraform.tfvars.example   # unchanged
    └── templates/
        ├── elk-install.sh.tpl     # new
        └── wazuh-install.sh.tpl   # new
```

### `main.tf` addition

```hcl
resource "aws_instance" "siem_practice" {
  ami                    = data.aws_ami.ubuntu.id
  instance_type          = var.instance_type
  key_name               = var.key_pair_name
  vpc_security_group_ids = [aws_security_group.siem_practice.id]

  root_block_device {
    volume_size = var.ebs_volume_size_gb
    volume_type = "gp3"
    encrypted   = true
  }

  # Bootstrap script chosen based on siem_platform — same variable that drives
  # the security group ports, so the whole config stays consistent off one switch.
  user_data = var.siem_platform == "elk" ? file("${path.module}/templates/elk-install.sh.tpl") : file("${path.module}/templates/wazuh-install.sh.tpl")

  tags = {
    Name = "${var.project_tag}-${var.siem_platform}"
  }
}
```

### `templates/elk-install.sh.tpl`

```bash
#!/bin/bash
# Runs automatically at first boot via cloud-init — check progress with:
#   cloud-init status --wait
# and full output in /var/log/cloud-init-output.log on the instance

set -euo pipefail

apt-get update
apt-get install -y apt-transport-https ca-certificates gnupg curl

# Elastic's official APT repo
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | gpg --dearmor -o /usr/share/keyrings/elastic.gpg
echo "deb [signed-by=/usr/share/keyrings/elastic.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" \
  > /etc/apt/sources.list.d/elastic-8.x.list

apt-get update
apt-get install -y elasticsearch kibana

# Bind to all interfaces so it's reachable from your IP (security group already restricts who can reach it)
sed -i 's/#network.host: 192.168.0.1/network.host: 0.0.0.0/' /etc/elasticsearch/elasticsearch.yml
sed -i 's/#server.host: "localhost"/server.host: "0.0.0.0"/' /etc/kibana/kibana.yml

systemctl enable elasticsearch kibana
systemctl start elasticsearch kibana

echo "ELK bootstrap complete" > /var/log/siem-bootstrap-done.log
```

### `templates/wazuh-install.sh.tpl`

```bash
#!/bin/bash
# Runs automatically at first boot via cloud-init — check progress with:
#   cloud-init status --wait
# and full output in /var/log/cloud-init-output.log on the instance

set -euo pipefail

apt-get update
apt-get install -y curl

# Wazuh's official all-in-one install script (installs manager + indexer + dashboard)
curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh
bash wazuh-install.sh -a

echo "Wazuh bootstrap complete" > /var/log/siem-bootstrap-done.log
```

> Both scripts are a starting point, not production-hardened — no TLS certificate customization, no credential rotation, default settings throughout. That's intentional for a practice exercise; treat hardening as a separate, later practice step if this ever informs real production config.

## How to run it

```bash
git clone https://github.com/lanc3lee/iac-practice.git
cd iac-practice/tofu/
cp terraform.tfvars.example terraform.tfvars
# edit terraform.tfvars — set my_ip_cidr, key_pair_name, and siem_platform

tofu init
tofu plan
tofu apply
```

`tofu apply` now returns as soon as the *instance* exists — it does **not** wait for the bootstrap script to finish running inside it. Install still takes a few minutes in the background after `apply` completes.

## Confirming the install actually finished

SSH in using the `ssh_command` output, then check:

```bash
# Cloud-init's own status — blocks until bootstrap is done or failed
cloud-init status --wait

# Full bootstrap output/errors, useful for debugging
sudo cat /var/log/cloud-init-output.log

# Our own marker file, only appears if the script reached its last line
cat /var/log/siem-bootstrap-done.log
```

If `siem-bootstrap-done.log` doesn't appear after a few minutes, something failed partway — `cloud-init-output.log` will show exactly which command errored.

## Why this matters for the Day 1/Day 2 ELK-vs-Wazuh comparison

This directly speeds up the comparison exercise from earlier: instead of manually walking through install steps for each platform, `tofu apply` with `siem_platform = "elk"` (then destroy, then `siem_platform = "wazuh"`, apply again) gets you to a working dashboard on both platforms with minimal manual intervention — freeing up the actual practice time for evaluating the platforms themselves (dashboard UX, detection content, resource usage) rather than fighting through install steps by hand each time.

Update the comparison capture doc's "install process" section to note **automated via bootstrap script** and record how long `cloud-init status --wait` took on each platform — that's now a genuinely comparable, repeatable number instead of a rough manual-install time estimate.

## Tear down

```bash
tofu destroy
```

Since `user_data` only runs at first boot, if you ever needed to re-run the bootstrap on the *same* instance (not recommended — better to destroy/recreate for a clean state), you'd need to either taint the resource (`tofu taint aws_instance.siem_practice`) to force replacement, or SSH in and re-run the script manually. Destroy-and-reapply is the cleaner habit for practice purposes.

## Next practice step

Two natural extensions from here, not covered in this segment:

- **Idempotent bootstrap** — making the script safe to re-run without erroring on things already installed (relevant if you move toward `remote-exec` for iterative development instead of always destroying/recreating)
- **Config management tool** (Ansible, or a proper Packer-built AMI) for anything beyond a simple bootstrap script — once the install logic gets more complex than a shell script comfortably handles, this is the natural next tool to reach for, and worth its own ADR when that point is reached.
