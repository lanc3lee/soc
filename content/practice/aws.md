Setting up an AWS account for practice is a great but if you accidentally leave the wrong resources running, it can rack up a bill.

For new practice/sandbox accounts, use **Free Plan** option that provides up to **$200 in promotional credits over a 6-month window** alongside traditional "Always Free" service caps. 

This makes it significantly safer to explore without surprise overages.

step-by-step walkthrough to get set up securely.

## Phase 1: Create the Account

### Step 1: Basic Information

1. Go to [aws.amazon.com/free](https://aws.amazon.com/free/) and click **Create a free account**.

2. Enter your email address and a preferred **AWS Account Name** (you can change this later).

3. Click **Verify email address**, grab the verification code from your inbox, and enter it.

4. Set a strong password. This credential controls your **Root User** account—protect it fiercely.

### Step 2: Plan Selection & Contact Info

1. When asked for your account type, choose **Personal** (this is optimized for learning and experimenting).

2. Fill in your name, phone number, and address.

3. **Crucial Choice:** Ensure you explicitly select the **Free account plan** option rather than the Paid plan when prompted.


### Step 3: Billing & Identity Verification

1. **Add a Payment Method:** AWS requires a credit or debit card. Even on the Free plan, they use this to verify your identity and prevent bot fraud.

2. **Temp Charge:** AWS will temporarily hold **$1 USD** to validate the card. This is automatically reversed within a few days.

3. **Phone Verification:** Choose SMS or a voice call, input your phone number, and enter the verification code sent to your device.


### Step 4: Support Plan

1. You will be asked to choose a Support Plan level.

2. Select **Basic Support (Free)**. Do not accidentally click Developer or Business support, as those carry immediate monthly subscription charges.


Once completed, wait a few minutes for AWS to activate your resources. You will receive an email confirmation when the account is fully ready.

## Phase 2: Post-Signup Safety Guardrails (Do These Immediately)

Before you spin up a single virtual machine or database, log in to your new console and set up these essential security and cost controls.

### 1. Lock Down Your Root Account

By default, you are logged in as the "Root User," which has absolute control over everything (including billing).

- Go to the **IAM Dashboard** (Identity and Access Management).

- Set up **Multi-Factor Authentication (MFA)** immediately on your Root account using an app like Google Authenticator or Bitwarden.

- _Best Practice:_ For day-to-day practice, create an **IAM User** or use AWS IAM Identity Center to give yourself admin privileges, and stop logging in with the Root email.


### 2. Set Up a Zero-Dollar Budget Alarm

AWS will not automatically shut down resources if you exceed your free tier limit or credit balance; it will just start billing your card. You need an alarm to catch this.

- Search for **AWS Budgets** in the top search bar.

- Click **Create budget**.

- Choose the ***Zero spend budget** template.

- Configure the alert to send an email to your personal inbox the absolute second your forecasted or actual spend hits that threshold.


## 3. Rules for Safe Practice

- **Stick to the Right Instance Sizes:** When launching virtual servers (EC2) or databases (RDS), look for the clear, green **"Free tier eligible"** text labels. Typically, this means sticking strictly to `t2.micro` or `t3.micro` sizes.

- **Stop vs. Terminate:** "Stopping" an EC2 instance turns it off, but you still pay a tiny fee for the attached EBS storage volume. If you are totally done with a lab exercise, choose **Instance State -> Terminate** to completely wipe it out and stop all associated charges.

- **Watch Your Regions:** Free tier allowances are global or per-region depending on the service. Stick to a single region close to you (like _ap-southeast-1_ for Singapore) for all your practice so you can easily track down everything you’ve built in one dashboard.

## 4. Set up an IAM user account

Using your root account for daily tasks, programming, or configuring tools is highly risky. Think of the root account like the master key to a building: if someone gets hold of your root access keys, they have complete control over your entire AWS account, billing, and resources, and you cannot easily restrict what they can do.

By creating an IAM user instead, you can give that specific user only the exact permissions they need to do their job, keeping your master root account safe and locked down with Multi-Factor Authentication (MFA).

Here is the exact path to create a user and get your keys safely:

### Step 1: Create the IAM User

1. Type **IAM** into the top search bar of the AWS Console and select it.
    
2. In the left navigation menu, click **IAM Users**, then click the **Create user** button on the right.
    
3. Give the user a name (e.g., `admin` or `dev-user`).
    
4. _Optional:_ If you want to log in with this account via a web browser later, check **Provide user access to the AWS Management Console**. Otherwise, leave it unchecked if this is strictly for command-line tools. Click **Next**.
    

### Step 2: Set Permissions

1. On the permissions page, choose **Attach policies directly**.
   Best practice is to attach policies to a group, and add user to that group
   but since this is for practice, let's just do "**Attach policies directly**"
    
2. If you are setting up this user for yourself to learn and build things, search for and check **AdministratorAccess** (or select a more restrictive policy like `PowerUserAccess` depending on your needs).
    
3. Click **Next**, review the details, and click **Create user**.
    

### Step 3: Generate the Access Keys

1. You will be taken back to the Users list. Click on the **Username** you just created.
    
2. Select the **Security credentials** tab.
    
3. Scroll down to the **Access keys** section and click **Create access key**.
    
4. AWS will ask you for your use case (e.g., Command Line Interface (CLI), Local code, etc.). Select the one that matches what you are doing, check the confirmation box at the bottom, and click **Next**.
    
5. Click **Create access key**.
    
6. **Important:** Copy the **Access key ID** and **Secret access key**, or click **Download .csv file** right away.
    

Once you have those keys configured in your local environment or tools, you can safely log out of your root account and use this new IAM user going forward!

-------

You will need your **Access key ID** and **Secret access key** as you set up your AWS CLI on your PC or Mac

for region, choose 
**`ap-southeast-1`** 
it stands for the **Asia Pacific (Singapore)** region.

Since we are physically located in Singapore, using this region gives you a couple of great advantages for practice:

- **Lowest Latency:** Because the data centers are right here on the island, your connections, API calls, and web traffic to your hosted resources will be blazing fast with minimal ping.
    
- **Default Availability:** It is enabled by default on all AWS accounts, so you don't have to manually go into your settings to "opt-in" or enable the region before spinning up resources.
    

### A Small Warning for Practice & Budgeting:

While `ap-southeast-1` is perfect for performance, it's worth keeping in mind that **Singapore is slightly more expensive** for certain services compared to the major US regions like `us-east-1` (N. Virginia) or `us-west-2` (Oregon).

If you are just spinning up small, Free Tier-eligible resources (like a `t2.micro` or `t3.micro` EC2 instance) to test things out and then tearing them down, the price difference will be pennies. However, if you plan to leave things running or start using heavier data processing/analytics services, keep an eye on the billing dashboard!

**Pro Tip:** No matter which region you build in, the very next thing you should do after setting up your IAM user is to search for **Billing** in the AWS console and set up a **Zero-Spend Budget alert**. That way, if you accidentally leave something running that exceeds the Free Tier, AWS will ping your email immediately before you get a surprise bill.

-----


for below, read up on https://soc.lanc3.com/practice/IaC , setup below is related to deploying AWS cloud resources using IaC Infrastructure as Code

### Getting your real IP address 

Easiest way — no AWS console needed for this part, just your terminal or browser:

bash

```bash
curl https://checkip.amazonaws.com
```

This returns your current public IP, e.g. `103.24.xxx.xxx`. 
For your `terraform.tfvars`, append `/32` to make it a valid CIDR block (meaning "exactly this one IP, nothing else"):

```
my_ip_cidr = "103.24.xxx.xxx/32"
```

**One thing worth knowing**: 
if you're on home broadband or mobile data, your IP can change over time (dynamic IP). 

If `tofu apply` was run today and SSH stops working in a few days, 
re-run `curl https://checkip.amazonaws.com`, 
update `terraform.tfvars`, and 
`tofu apply`
 again — it'll just update the security group rule, not recreate the whole instance.

### Getting/creating an EC2 key pair name

You have two options — console (visual) or CLI (faster, and consistent with the rest of your workflow).

#### Option A: AWS Console

1. Log into the AWS console → search for **EC2** in the top search bar
2. In the left sidebar, under **Network & Security**, click **Key Pairs**
3. Check if you already have one listed — if yes, its **Name** column is exactly what goes into `key_pair_name` in your `terraform.tfvars`
   
![[aws-network-security-keypairs.png]]   

4. If none exist, click **Create key pair**:
    - **Name:** `siem-practice`
    - **Key pair type:** RSA
    - **Private key file format:** `.pem` (works for Mac/Linux OpenSSH )
    - Click **Create key pair** — this will **immediately download** `siem-practice.pem` to your Downloads folder. This is the only time AWS lets you download it, so don't lose it.

![[aws-create-keypair.png]]

5. Move it somewhere sensible and lock down its permissions (SSH refuses to use a key with overly open permissions):

bash

```bash
mv ~/Downloads/siem-practice.pem ~/.ssh/siem-practice.pem
chmod 400 ~/.ssh/siem-practice.pem
```

-----

#### Option B: AWS CLI (faster, & recommended)

bash



```bash

mkdir -p ~/.ssh

aws ec2 create-key-pair \
  --key-name siem-practice \
  --query 'KeyMaterial' \
  --output text > ~/.ssh/siem-practice.pem

chmod 400 ~/.ssh/siem-practice.pem
aws ec2 create-key-pair \
  --key-name siem-practice \
  --query 'KeyMaterial' \
  --output text > ~/.ssh/siem-practice.pem

chmod 400 ~/.ssh/siem-practice.pem
```

example:
```
lance@LANC3 ~ % mkdir -p ~/.ssh

aws ec2 create-key-pair \

  --key-name siem-practice \

  --query 'KeyMaterial' \

  --output text > ~/.ssh/siem-practice.pem

  

chmod 400 ~/.ssh/siem-practice.pem



!!! if u r curious how RSA key looks like...

lance@LANC3 ~ % cat ~/.ssh/siem-practice.pem

-----BEGIN RSA PRIVATE KEY-----

...

-----END RSA PRIVATE KEY-----

lance@LANC3 ~ %
```


Either way, in `terraform.tfvars`:

```
key_pair_name = "siem-practice"
```

**Important — never commit the `.pem` file to your `iac-practice` repo.** Your `.gitignore` already has `*.pem` covered from the earlier setup, but worth double-checking with `git status` before your first push once the key exists on disk in that project folder (safer habit: keep it in `~/.ssh/` entirely, outside the repo folder, so there's no chance of it ever being in the working directory git tracks).


--------

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

------

### How to check where your CLI's default region actually is

bash

```bash
aws configure get region
```

To check if your key pair is in correct region

```
aws ec2 describe-key-pairs --key-names siem-practice --region ap-southeast-1
```

example

```
lance@LANC3 ~ % aws ec2 describe-key-pairs --key-names siem-practice --region ap-southeast-1

{

    "KeyPairs": [

        {

            "KeyPairId": "key-06********************c6",

            "KeyType": "rsa",

            "Tags": [],

            "CreateTime": "2026-07-08T06:39:38.115000+00:00",

            "KeyName": "siem-practice",

            "KeyFingerprint": "1e:******************:f3"

        }

    ]

}

lance@LANC3 ~ %
```

------


Errors I encountered when trying to launch AWS instances using OpenTofu / Terraform

in this case, I realized that ap-southeast-1 does not offer t2-micro in free tier, hence it was not deployed





![[tofu-apply-error-not-free.png]]


![[AWS-free-tier.png|356]]


![[AWS-SG-only-t3-micro.png]]