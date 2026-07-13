---
title: "Practice: Install ELK (Elasticsearch, Logstash, Kibana) then BotES next"
slug: practice/ELK-BotES
date: 2026-07-13
tags:
  - soc
  - practice
  - elk
  - elasticsearch
  - logstash
  - kibana
  - install
status: draft
---

# Install ELK for BOTS Practice

Single-node manual install of Elasticsearch, Logstash, and Kibana, sized for a training/practice lab (not production). 

Target OS is Ubuntu 22.04/24.04. If you're standing this up as a temporary workshop environment, treat this as the "day 1" build script and pair it with an OpenTofu teardown once the exercise is done — see the cleanup note at the bottom.

## Prerequisites

- Ubuntu 22.04 or 24.04 (bare metal, VM, or Proxmox instance)
- Minimum 4 vCPU / 8GB RAM for a single-node dev cluster; more if loading the full BOTSv1 dataset rather than attack-only
- 50GB+ free disk if loading full BOTSv1 (120GB uncompressed) — size the volume before you start, not after
- `sudo` access
- Outbound HTTPS access to `artifacts.elastic.co`

## 1. Add the Elastic APT repository

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates gnupg curl

curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | \
  sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | \
  sudo tee /etc/apt/sources.list.d/elastic-8.x.list

sudo apt-get update
```

> Pin to whatever 8.x minor version the rest of the CyberStorm SIEM Platform is standardized on — check the existing ELK manual install guide / ADRs before assuming latest.

## 2. Install Elasticsearch

```bash
sudo apt-get install -y elasticsearch
```

Edit `/etc/elasticsearch/elasticsearch.yml` for a single-node lab:

sudo nano /etc/elasticsearch/elasticsearch.yml

```yaml
cluster.name: cyberstorm-bots-lab
node.name: node-1
network.host: 0.0.0.0
discovery.type: single-node
```

For a **lab environment only**, disable the security bootstrap prompts to simplify setup:

```yaml
xpack.security.enabled: false
```

> Do not carry `xpack.security.enabled: false` into anything reachable outside your lab network or into the shared CyberStorm AWS environment — this is for an isolated practice box only. Production/shared infra should follow the existing KMS-backed security posture (ADR-0003).

Start and enable the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now elasticsearch
```

Verify:

```bash
curl -X GET "localhost:9200/?pretty"
```

You should get back a JSON blob with `cluster_name: cyberstorm-bots-lab`.

## 3. Install Kibana

```bash
sudo apt-get install -y kibana
```

Edit `/etc/kibana/kibana.yml`:

```yaml
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://localhost:9200"]
```

Start and enable:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now kibana
```

Kibana will be reachable at `http://<host>:5601`. Confirm it comes up and can see the Elasticsearch cluster under **Stack Management → Data → Index Management**.

## 4. Install Logstash

```bash
sudo apt-get install -y logstash
```

Create a working pipeline directory for BOTS-specific configs (kept separate from any existing CyberStorm SIEM Platform pipelines):

```bash
sudo mkdir -p /etc/logstash/conf.d/bots
```

You won't populate this directory yet — the BOTES ingestion guide (next doc) supplies the actual pipeline `.conf` files and index template.

Enable the service (it will sit idle with no active pipeline until conf files are added):

```bash
sudo systemctl daemon-reload
sudo systemctl enable logstash
```

## 5. Sanity check

- `curl localhost:9200/_cat/health?v` — cluster status should be `green` or `yellow` (yellow is expected/fine on a single node)
- Kibana UI loads at port 5601
- `systemctl status elasticsearch kibana logstash` — all three should show `active` (Logstash may show idle/no pipeline yet, that's expected)

## Teardown / cleanup note

If this box was provisioned via OpenTofu as an ephemeral workshop environment rather than a persistent lab:

- Snapshot or export any Kibana saved objects (dashboards, index patterns) you want to keep **before** tearing down — `Stack Management → Saved Objects → Export`
- Confirm no state or data volumes are left orphaned outside the OpenTofu-managed resource graph (matches the existing ELK manual install guide's cleanup convention)
- Destroy via `tofu destroy` rather than manually deleting instances, to keep KMS-backed state consistent

Next: [Add BOTES (Boss of the Elastic SOC) data →](./02-add-botes.md)
