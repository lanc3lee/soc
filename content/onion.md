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


------

## Security Onion Console (SOC)

The Security Onion Console is the **primary web-based interface** where all the analyst work happens. It's the single pane of glass that ties together all the data from Suricata, Zeek, Elastic Agent, Strelka, and Wazuh into one place you can investigate from.

Access it from a browser at `https:// ... ( to be updated) >` after setup.


### The main sections and what analysts use them for

**Alerts** is your starting point for most shifts. This is where Suricata IDS alerts surface — every rule match appears here with severity, source/destination IPs, port, and the rule that triggered it. In a real SOC this is the queue analysts work through during triage. Your job is to determine: is this a true positive or false positive, and does it need escalation?

**Hunt** is the threat hunting interface. Rather than waiting for an alert to fire, you proactively query the data to look for suspicious patterns. You might hunt for all PowerShell executions across endpoints, or all connections to a specific country, or processes spawning from unusual parent processes. This is where you spend time during quieter periods or when investigating an ongoing incident.

**Dashboards** gives you visual overviews built on Kibana — top alerting IPs, login failures over time, geographic traffic distribution, endpoint activity summaries. Useful for situational awareness at the start of a shift and for building detection metrics.

**PCAP** lets you pull the raw packet capture for any network alert. If Suricata fires on suspicious traffic, you can right-click the alert and pull the full PCAP to inspect the actual bytes — what was in the payload, was there a file transfer, what did the HTTP request look like. This is critical for alert validation.

**Cases** is the incident management component. When you determine an alert is worth investigating further, you escalate it into a case, document your findings, attach evidence, and track the investigation through to closure. This mirrors how ticketing works in real SOCs (think ServiceNow, TheHive, or Jira for security).

---

### The analyst workflow in practice

A typical shift workflow through the console looks like this:

1. Open **Alerts** — sort by severity, start with critical and high
2. For each alert, check source/dest IPs, look up the triggering rule, assess context
3. If suspicious, pivot to **Hunt** — query for other activity from the same IP or host around the same timeframe
4. Pull **PCAP** to inspect the actual traffic if it's network-based
5. Cross-reference with **Dashboards** — is this IP a repeat offender? Is this a spike in a pattern?
6. If confirmed suspicious, open a **Case** and document your investigation chain
   
   
------


## Elasticsearch in Security Onion

Elasticsearch is the **data backbone** of Security Onion — it's the engine that stores, indexes, and makes searchable every single event that flows through the platform. Everything you see in the SOC console — alerts, logs, network flows, endpoint events — lives in Elasticsearch underneath.

---

### The core concept

Elasticsearch is a distributed search and analytics engine built on Apache Lucene. In simple terms, it's a database optimised not for rows and columns like SQL, but for **fast full-text search across massive volumes of semi-structured JSON documents**. Every log event in Security Onion gets stored as a JSON document in Elasticsearch.

A Suricata alert looks something like this when stored:

```
{
  "@timestamp": "2026-04-06T14:32:11Z",
  "source.ip": "192.168.1.45",
  "destination.ip": "10.0.0.1",
  "destination.port": 4444,
  "rule.name": "ET MALWARE Meterpreter Reverse TCP",
  "event.severity": 1,
  "network.protocol": "tcp"
}

```

Every event — whether from Suricata, Zeek, Elastic Agent, or Wazuh — gets normalised into a consistent structure like this and indexed so it can be queried instantly.

---

### What Elasticsearch enables

**Fast querying across huge datasets** — When you type a query in the Hunt interface or search in Dashboards, you're querying Elasticsearch. It can search millions of events in milliseconds because everything is pre-indexed. Without it, searching raw log files would take minutes or hours.

**Index-based organisation** — Security Onion stores different data types in separate indices. Network alerts go into one index, Zeek logs into another, endpoint events into another. This keeps data organised and lets you query a specific data source without scanning everything.

**Aggregations for dashboards** — The visualisations you see in Kibana dashboards are powered by Elasticsearch aggregations. When you see "top 10 alerting source IPs" or "login failures per hour", Elasticsearch is grouping and counting documents in real time to produce those numbers.

**Correlation across sources** — Because all data lands in Elasticsearch regardless of source, you can write a single query that spans Suricata alerts, Zeek connection logs, and Elastic Agent endpoint events simultaneously. This is what makes cross-source correlation possible — the pivot from a network alert to endpoint activity happens because both live in the same Elasticsearch cluster.

---

### The ELK stack context

Elasticsearch is the E in the ELK stack — a combination you'll hear constantly in SOC and SIEM engineering roles:

- **E**lasticsearch — stores and indexes the data
- **L**ogstash — ingestion pipeline that parses and normalises incoming logs before storing them
- **K**ibana — the visualisation layer that sits on top of Elasticsearch, which is what the SOC console's dashboards and Hunt interface are built on

Security Onion wraps all three together and pre-configures them for security use cases, but under the hood it's still the standard ELK stack. 

Skills practised in a Security Onion lab can transfer directly to environments running commercial ELK-based SIEMs like Elastic SIEM or even Splunk (which follows the same index/search/dashboard model).

### What can break and what to watch for

In a homelab, Elasticsearch is the most resource-hungry component. 

If your VM starts feeling sluggish or the SOC console times out on queries, Elasticsearch is usually the culprit. 

Check its health by SSH into Security Onion VM and run:

```
sudo so-elastic-status

```

Green means all indices healthy. Yellow means some replica shards are unassigned (normal in a single-node lab setup). Red means data loss risk — worth investigating.

Also check how much disk Elasticsearch is consuming:

```
sudo du -sh /nsm/elasticsearch/
```
In a lab environment, logs accumulate fast especially if you're running attack simulations. Security Onion has a built-in setting to control data retention — it will automatically purge old indices when disk usage crosses a threshold (default around 85%).


## DeepDive

Research
-concept of indexing and why it makes search fast, the difference between Elasticsearch and a traditional SQL database, what an index is, and how Kibana sits on top of it.

Explain
-the Elasticsearch model — JSON documents, indices, aggregations
-"how does your SIEM store and query data?"


------

## Suricata in Security Onion

Suricata is Security Onion's **network intrusion detection engine** — it's the component that watches all network traffic in real time and fires alerts when it sees something that matches a known threat pattern or suspicious behaviour.

If Elasticsearch is the brain that stores everything, Suricata is the eyes watching the wire.

---

### How it works

Suricata sits on our monitor NIC (the second network adapter we should configured during setup) in **promiscuous mode** — meaning it reads every packet traversing that interface, not just packets addressed to the Security Onion VM itself. It then compares that traffic against a ruleset of signatures and fires an alert when there's a match.

The detection model is signature-based, meaning each rule describes a specific pattern to look for. A rule looks like this:

```
alert tcp $HOME_NET any -> $EXTERNAL_NET 4444 (
  msg:"ET MALWARE Possible Meterpreter Reverse TCP";
  flow:established,to_server;
  sid:2000369;
  rev:1;
)

```

This rule fires whenever it sees an established TCP connection from our internal network going out to port 4444 on any external IP — a classic Metasploit Meterpreter reverse shell indicator.

---

### What Suricata detects

Suricata ships with rulesets from several sources that cover a wide range of threat categories. In our lab, we will see alerts for port scans and reconnaissance, exploitation attempts, malware command and control traffic, suspicious DNS queries, known bad IPs and domains, protocol anomalies (HTTP on non-standard ports, encrypted traffic on port 80), and data exfiltration patterns.

The primary rulesets bundled in Security Onion come from **Emerging Threats (ET)** — a community-maintained library of thousands of signatures covering current threats. Security Onion updates these automatically.

---

### Suricata's two modes

**IDS mode (default in Security Onion)** — Suricata observes and alerts but does not block. It's passive — traffic flows through regardless of what it sees, and alerts are written to Elasticsearch for analyst review. This is the standard mode for a monitoring deployment.

**IPS mode** — Suricata sits inline and can actively drop or reject packets matching rules. This requires a different network topology (traffic must physically pass through the sensor) and is more common in production deployments than home labs.

Run IDS mode in our lab, which is fine for practice — the alerting and investigation workflow is identical.

---

### Where alerts appear

Every Suricata alert lands in the **Alerts** section of the Security Onion Console with the following key fields we will use during triage:

- `rule.name` — the human-readable rule description
- `source.ip` / `destination.ip` and ports
- `event.severity` — 1 (high) to 3 (low)
- `network.protocol` — tcp, udp, icmp, http, dns, tls etc.
- `rule.sid` — the unique rule ID, useful for looking up context on the Emerging Threats website

From any alert, pivot to the full PCAP to inspect the actual packets, or cross-reference with Zeek logs to see the broader connection context.

---

### Generating alerts to test

The fastest way to see Suricata in action is to run an Nmap scan from Windows VM targeting the Security Onion IP:

```
# On Windows VM
nmap -sS -A 192.168.x.x

```

Within seconds we should see reconnaissance-related alerts fire in the Security Onion console. From there practice the triage workflow — read the rule name, identify the source and destination, pull the PCAP, determine whether it's expected behaviour or something worth escalating.

For more realistic detections, the **Atomic Red Team** framework simulate specific MITRE ATT&CK techniques on our Windows VM.
Watch the corresponding Suricata (and Elastic Agent) alerts appear:

```

# Install Atomic Red Team on Windows VM
Install-Module -Name invoke-atomicredteam
# Run a specific technique - T1046 is network service scanning
Invoke-AtomicTest T1046

```

### Tuning rules — a key SIEM engineering skill

In a real SOC, raw Suricata without tuning generates enormous amounts of noise — alerts that fire constantly on legitimate traffic. 

**Rule tuning** is one of the most valued SIEM engineering skills because it's what makes a detection platform actually usable.

In Security Onion, learn to suppress noisy rules, modify thresholds, or whitelist known-good IPs. Do this by editing the local rules configuration:

```
sudo so-rule-update
# Or suppress a specific SID from alerting
sudo vi /opt/so/saltstack/local/salt/suricata/local.rules

```

TLDR:
Detection is not just about turning rules on, but maintaining signal quality over time.


Question:
-how would you detect a port scan on your network?
-example answer:
Suricata with an ET reconnaissance rule

---------

## Zeek in Security Onion

Zeek (formerly Bro) is Security Onion's **network traffic analyser and logger** — but it approaches network visibility completely differently from Suricata. Where Suricata asks _"does this traffic match a known bad pattern?"_, Zeek asks _"what is this traffic actually doing?"_

Zeek doesn't alert. It **narrates**.

---

### The core difference from Suricata

Suricata is signature-based — it fires when something looks malicious. Zeek is behaviour-based — it logs everything that happens on the network in structured, human-readable logs regardless of whether it looks suspicious or not.

Think of it this way: Suricata is the alarm system, Zeek is the CCTV footage. When the alarm goes off, we go back to the CCTV to understand exactly what happened before, during, and after.

This makes Zeek indispensable for **threat hunting and incident investigation** — we can reconstruct network activity even for attacks that had no matching Suricata signature.


------

### What Zeek logs

Zeek breaks network traffic down by protocol and writes a separate structured log for each type. The key logs you'll use constantly are:

**conn.log** — every single network connection, regardless of protocol. Source IP, destination IP, ports, bytes transferred, duration, connection state. This is your starting point for almost every investigation.

```
ts           src_ip         src_port  dst_ip        dst_port  proto  duration  bytes
1712345678   192.168.1.45   54821     185.220.101.5  443       tcp    302s      1.2MB

```

**dns.log** — every DNS query and response. Domain queried, query type, response IPs, TTL. Invaluable for detecting DNS-based C2, domain generation algorithms (DGA), and data exfiltration over DNS.

**http.log** — full HTTP transaction details. URI, user agent, referrer, response code, file transfers. A malware dropper downloading a payload over HTTP shows up here in full detail even if Suricata missed it.

**ssl.log / x509.log** — TLS connection metadata and certificate details. Since most modern malware communicates over HTTPS, Zeek's ability to log certificate subjects, issuers, validity periods, and JA3 fingerprints is critical for detecting malicious encrypted traffic without decrypting it.

**files.log** — metadata about every file transferred over the network. File type, MD5/SHA1 hash, size, whether it was extracted. This feeds directly into Strelka for deeper file analysis.

**weird.log** — protocol violations and anomalies that don't fit expected behaviour. Things like malformed packets, unexpected protocol usage, or connections with unusual characteristics. Often surfaces novel attack techniques that signature rules don't yet cover.

------

