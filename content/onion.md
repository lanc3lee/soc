Security Onion is a free Linux distro that bundles an entire SOC toolkit.
It includes network IDS, host IDS, network monitoring, and log management — all bundled into one platform, built on open source tools like Suricata, Zeek, and Elasticsearch

![[security-onion-architecture.png]]



## Step-by-step setup

### Step 1 — Hardware provisioning

Hardware requirements include a multicore CPU with at least 2 cores, a minimum of 16GB of memory (32GB works better), and a minimum of 80GB of disk space. 
Aim for:
- CPU: 4+ cores allocated to the VM
- RAM: 12–16GB for the Security Onion VM alone
- Disk: 100GB minimum

### Step 2 — Download the ISO

Go to [securityonionsolutions.com/software](https://securityonionsolutions.com/software) which redirects to the GitHub releases page. After downloading, verify the hash in PowerShell:

Get-FileHash .\securityonion-2.4.xxx.iso -Algorithm SHA256

Compare against the published checksum on GitHub.

### Step 3 — Create VM in VirtualBox

Enable two network cards in the settings — use Bridged Adapter on Adapter 1 and 2, or NAT depending on your setup. 

Key points:
- **Adapter 1** (management NIC): NAT — for internet access during setup
- **Adapter 2** (monitor NIC): Host-only — for sniffing traffic
- Allocate at least 4 CPU cores and 12GB RAM

### Step 4 — Install Security Onion

Start the VM and select "Install Security Onion" from the boot menu. It will ask you to type "yes" to confirm you're wiping the disk, then prompt you to set a username and password. Once installation completes, reboot. 

After reboot and login, run the setup wizard from terminal if it doesn't auto-launch:

sudo SecurityOnion/setup/so-setup iso


### Step 5 — Configure during setup wizard

Evaluation Mode is recommended for first-time users or standalone VMs. 

Key choices during the wizard:

- Install type: **Standalone**
- Management NIC: select your first adapter
- Monitor NIC: select your second adapter
- Set a static IP for your management interface
- Add your host PC's IP to the allowed access list when asked

If the Security Onion web interface is not accessible after setup, 
run `sudo so-allow` to configure the firewall to allow traffic from your IP.


### Step 6 — Install Elastic Agent on a Windows VM

Download the installer files from the Security Onion Console, then run the Elastic Agent `.exe` as an administrator on Windows VM. 

This ships Windows event logs, Sysmon logs, and more into Security Onion instance — providing real data to analyse.

-------

## What to practise once it's running

Generate test traffic using tools like Nmap or Metasploit to assess detection effectiveness, create custom dashboards, and design alerts for specific security events. 

Try these scenarios:

**Alert triage** — Run an Nmap scan from  Windows VM against the Security Onion IP and find the resulting Suricata alert in the console. 
Practice writing the alert narrative: 
what triggered it, source/dest IPs, severity, false positive or true positive?

**Log correlation** — With the latest rulesets, Security Onion can detect common attacks like Kerberoasting, pass-the-hash, and the Windows ZeroLogon vulnerability. Try simulating these with Atomic Red Team and trace the detection chain.

**Threat hunting** — Use the Hunt interface in the SOC console to pivot from an IP to all related events. 
Practice the analyst workflow: 
start with an alert 
→ expand to related network flows in Zeek 
→ check process events from the endpoint agent.

**Dashboard building** — Build a custom dashboard in Kibana showing failed logins over time, top alerting source IPs, and geographic distribution. Interviewers love hearing you've built something from scratch.