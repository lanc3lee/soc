---
title: "Manual ELK Install — SIEM Practice Lab"
slug: elk-install-manual
category: practice
tags: [siem, elk, elasticsearch, logstash, kibana, homelab]
---

# Manual ELK Install — SIEM Practice Lab

This guide walks through a manual, from-scratch installation of the ELK Stack (Elasticsearch, Logstash, Kibana) as a self-hosted SIEM. It's written as a hands-on practice exercise for the CyberStorm team — the goal is to understand what each component actually does under the hood, not just run a docker-compose file.

**Before you start:** have you tried the [Wazuh practice lab](https://soc.lanc3.com/practice/wazuh) yet? If not, we'd recommend doing that one first — see below for why.

## Why Wazuh first?

ELK on its own is just a data platform — Elasticsearch stores and indexes data, Logstash moves and transforms it, and Kibana visualizes it. None of that is SIEM-specific out of the box. To turn it into an actual security monitoring tool, you'd need to bring your own agents, log parsers, detection rules, and correlation logic — which is a significant amount of work on top of the stack itself.

Wazuh is built on top of the same Elastic foundation, but ships with the security-specific layer already in place: host agents, file integrity monitoring, log decoders, out-of-the-box detection rules, and a manager that correlates events into alerts. Practicing with Wazuh first gives you a working SIEM end-to-end much faster, and a clearer picture of what a "finished" SIEM is supposed to look like — decoders, rules, alerts, dashboards — before you try to assemble the equivalent from raw ELK components.

Once you've seen that full picture, coming back to a manual ELK build makes a lot more sense: you'll recognize which piece you're building at each step (ingestion vs. storage vs. visualization vs. detection) instead of just following commands.

With that said — here's the manual ELK install.

## Lab Overview

| Item | Value |
|---|---|
| OS | Ubuntu 22.04 LTS |
| Components | Elasticsearch 8.x, Logstash 8.x, Kibana 8.x, Filebeat |
| Sizing | 1 vCPU / 2GB RAM minimum for a lab; 4GB+ recommended |
| Network | Single host, all services local (adjust for multi-node) |

> This is a single-node lab setup for learning purposes — not a production hardening guide. For a production build, see the CyberStorm IaC repo (OpenTofu) instead.

## Step 1 — Prerequisites

Update the system and install dependencies:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl gnupg2 apt-transport-https
```

Check available memory — Elasticsearch will need at least 2GB, ideally with swap disabled:

```bash
free -h
sudo swapoff -a
```

## Step 2 — Add the Elastic APT Repository

Import the Elastic GPG key and add the repo:

```bash
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elastic.gpg

echo "deb [signed-by=/usr/share/keyrings/elastic.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list

sudo apt update
```

## Step 3 — Install and Configure Elasticsearch

```bash
sudo apt install -y elasticsearch
```

Note the auto-generated `elastic` superuser password printed at the end of install — save it somewhere safe (you'll need it for Kibana enrollment).

Edit the config:

```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

Set the following for a single-node lab:

```yaml
cluster.name: cyberstorm-lab
node.name: node-1
network.host: 0.0.0.0
discovery.type: single-node
```

Start and enable the service:

```bash
sudo systemctl enable --now elasticsearch
```

Verify it's up (replace `<password>` with the elastic user password):

```bash
curl -k -u elastic:<password> https://localhost:9200
```

You should get back a JSON response with the cluster name and version.

## Step 4 — Install and Configure Kibana

```bash
sudo apt install -y kibana
```

Edit the config:

```bash
sudo nano /etc/kibana/kibana.yml
```

Set:

```yaml
server.host: "0.0.0.0"
server.port: 5601
elasticsearch.hosts: ["https://localhost:9200"]
```

Generate an enrollment token from the Elasticsearch node and use it to set up Kibana:

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
```

Start Kibana:

```bash
sudo systemctl enable --now kibana
```

Then browse to `http://<your-host-ip>:5601`, paste the enrollment token, and log in with the `elastic` user credentials from Step 3.

## Step 5 — Install and Configure Logstash

```bash
sudo apt install -y logstash
```

Create a basic pipeline config to receive syslog and forward to Elasticsearch:

```bash
sudo nano /etc/logstash/conf.d/syslog.conf
```

```
input {
  beats {
    port => 5044
  }
}

filter {
  grok {
    match => { "message" => "%{SYSLOGTIMESTAMP:timestamp} %{SYSLOGHOST:host} %{DATA:program}(?:\[%{POSINT:pid}\])?: %{GREEDYDATA:log_message}" }
  }
}

output {
  elasticsearch {
    hosts => ["https://localhost:9200"]
    index => "syslog-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "<password>"
    ssl_verification_mode => "none"
  }
}
```

Test the config syntax before starting:

```bash
sudo /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/syslog.conf --config.test_and_exit
```

Start the service:

```bash
sudo systemctl enable --now logstash
```

## Step 6 — Ship Logs with Filebeat

On the machine you want to monitor (can be the same host for this lab):

```bash
sudo apt install -y filebeat
sudo nano /etc/filebeat/filebeat.yml
```

Point Filebeat at Logstash instead of Elasticsearch directly:

```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/syslog
      - /var/log/auth.log

output.logstash:
  hosts: ["localhost:5044"]
```

Start Filebeat:

```bash
sudo systemctl enable --now filebeat
```

## Step 7 — Verify Data Is Flowing

In Kibana, go to **Stack Management → Index Management** and confirm a `syslog-*` index has appeared. Then create a data view (**Stack Management → Data Views**) matching `syslog-*`, and check **Discover** to see live events coming in.

## Step 8 — Next Steps (Turning This Into an Actual SIEM)

At this point you have a working log pipeline, but not yet a SIEM. Some things worth exploring from here:

- **Elastic Security app** — Kibana ships with a Security solution that adds detection rules, cases, and a basic alerting engine on top of your indices.
- **Ingest pipelines / additional Beats** — Winlogbeat for Windows event logs, Packetbeat for network data, Auditbeat for host auditing.
- **Index lifecycle management (ILM)** — set retention policies so indices roll over and age out automatically.
- **Comparing back to Wazuh** — go back through the [Wazuh lab](https://soc.lanc3.com/practice/wazuh) and note how much of Steps 1–7 above Wazuh's agent + manager architecture already handles for you.

## Cleanup

How you tear this down depends on how the box was provisioned:

**If your lab instance was spun up via OpenTofu** (e.g. an ephemeral EC2 instance just for this exercise), the instance itself is disposable — just destroy it:

```bash
tofu destroy
```

This removes the whole instance, so there's nothing left to individually disable or purge. Don't bother with the manual steps below in this case — they're redundant once the instance is gone.

**If you're on a persistent or shared host** (a long-lived box, a homelab VM you reuse, or anything not managed by the CyberStorm IaC repo), `tofu destroy` isn't an option — there's no instance to tear down, and even if there were, you might be sharing it with other labs. In that case, manually stop and remove just the ELK components instead:

```bash
sudo systemctl disable --now elasticsearch kibana logstash filebeat
sudo apt purge -y elasticsearch kibana logstash filebeat
```

Rule of thumb: if you provisioned it, destroy it. If you're borrowing it, clean up after yourself instead of deleting it.

---

*Part of the CyberStorm SIEM practice series — [soc.lanc3.com/practice](https://soc.lanc3.com/practice)*
