
if you are new to git, this is a page that helps you practise setting up your own repo on github
### 1. Create the repo on GitHub

Go to [github.com/new](https://github.com/new) and set:

- **Repository name:** `iac-practice`
- **Visibility:** Public
- **Do NOT** check "Add a README" or "Add .gitignore" — you'll add both yourself in the right order below

Click **Create repository**.
![[github-new.png]]


### 2. Clone it locally

bash

```bash
git clone https://github.com/lanc3lee/iac-practice.git
cd iac-practice
```

example:
```
lance@LANC3 ~ % git clone https://github.com/lanc3lee/iac-practice.git

Cloning into 'iac-practice'...

...

lance@LANC3 ~ % cd iac-practice

lance@LANC3 iac-practice %
```

### 3. Create `.gitignore` FIRST — before adding any `.tf` files

bash

```bash
cat > .gitignore << 'EOF'
# Terraform/OpenTofu state — never commit, may contain sensitive data
*.tfstate
*.tfstate.backup
*.tfstate.*.backup

# Real variable values — contains your actual IP, key name, etc.
terraform.tfvars
*.auto.tfvars

# Local provider plugin cache
.terraform/

# SSH keys if ever generated/downloaded into this folder
*.pem

# OS/editor cruft
.DS_Store
EOF
```

### 4. Commit the `.gitignore` alone, first

bash

```bash
git add .gitignore
git commit -m "Add .gitignore before any config files"
git push
```

This is the important ordering — the ignore rules exist in git history _before_ any file that could match them ever gets created in the folder.


### 5. Add a basic README

bash

```bash
cat > README.md << 'EOF'
# iac-practice

Personal OpenTofu practice repo — provisioning AWS infrastructure as part of learning IaC for the CyberStorm SIEM project.

Not the production CyberStorm repo — this is a personal, disposable, free-tier practice environment. See the writeup at soc.lanc3.com/practice/IaC for context.

## Structure

- `tofu/` — OpenTofu config provisioning an EC2 host sized/networked for either ELK or Wazuh practice, controlled by the `siem_platform` variable.
EOF

git add README.md
git commit -m "Add README"
git push
```

### 6. Now add your actual `.tf` files

bash

```bash
mkdir tofu
# copy in main.tf, variables.tf, outputs.tf, terraform.tfvars.example
git add tofu/
git commit -m "Add OpenTofu config for SIEM practice host (ELK/Wazuh)"
git push
```

At this point, double-check before pushing:

bash

```bash
git status
```

```
lance@LANC3 iac-practice % git status

On branch main

Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
```

Confirm `terraform.tfvars` and any `.tfstate` file do **not** appear as staged/untracked-to-be-added — they should be silently ignored. If you ever see them listed as trackable, stop and fix `.gitignore` before committing.

![[github-demo.png]]

### 7. Locally, create your real `terraform.tfvars` (never pushed)

bash

```bash
cd tofu/
cp terraform.tfvars.example terraform.tfvars
# edit terraform.tfvars with your real IP, key pair name, siem_platform choice
```

This file stays local only — `git status` should show it as ignored, not untracked.