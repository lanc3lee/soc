---
title: "Wazuh Manual Installation (Single-Node Practice Lab)"
date: 2026-07-09
tags:
  - practice
  - wazuh
  - siem
  - homelab
description: "Step-by-step manual installation of Wazuh indexer, server, and dashboard on a single Ubuntu host, without the all-in-one quick-start script."
---

# Wazuh Manual Installation (Single-Node Practice Lab)

This guide walks through installing Wazuh **component-by-component** rather than using the `wazuh-install.sh` all-in-one quick-start script. The goal is to understand what each piece does — indexer, server, and dashboard — since that's exactly what you'll need to reason about when we move CyberStorm's stack into OpenTofu.

All three components are installed on a **single node** here for practice. In a production layout you'd split these across hosts and adjust the config files / certs accordingly.

## What you're installing

| Component | Role | Default port |
|---|---|---|
| Wazuh indexer | Stores and indexes alert/event data (OpenSearch-based) | 9200 |
| Wazuh server (manager) | Analyzes data from agents, runs rules/decoders | 1514, 1515, 55000 |
| Wazuh dashboard | Web UI (OpenSearch Dashboards-based) | 443 |

## Prerequisites

- Ubuntu 22.04 or 24.04, x86_64
- Minimum 4 vCPU / 4 GB RAM for a practice single-node (Wazuh recommends more for production — see [sizing guide](https://documentation.wazuh.com/current/getting-started/requirements.html))
- Root or sudo access
- Ports open on the host/security group: 443 (dashboard), 1514/1515 (agent comms), 55000 (API), 9200 (indexer, only if you need external access)
- A stable hostname (avoid changing it after cert generation)

```bash
sudo apt update && sudo apt upgrade -y
sudo hostnamectl set-hostname wazuh-node
```

## Step 1 — Import the Wazuh GPG key and repository

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring \
  --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | \
  sudo tee /etc/apt/sources.list.d/wazuh.list

sudo apt update
```

## Step 2 — Generate certificates

Wazuh components authenticate to each other over TLS. Download the cert tool and a config file describing your node names/IPs.

```bash
curl -sO https://packages.wazuh.com/4.x/wazuh-certs-tool.sh
curl -sO https://packages.wazuh.com/4.x/config.yml
```

Edit `config.yml` so each node entry (`wazuh-indexer`, `wazuh-server`, `wazuh-dashboard`) points at your host's IP. For a single-node lab, that's the same private IP for all three.

```bash
nano config.yml
```

Then generate the certs:

```bash
chmod +x wazuh-certs-tool.sh
sudo ./wazuh-certs-tool.sh -A
```

This produces a `wazuh-certificates/` directory. Keep it — you'll copy files out of it into each component's config directory in the steps below.

## Step 3 — Install and configure the Wazuh indexer

```bash
sudo apt install -y wazuh-indexer
```

Copy the certs for the indexer:

```bash
NODE_NAME=wazuh-indexer   # match the name you used in config.yml
sudo mkdir -p /etc/wazuh-indexer/certs
sudo cp wazuh-certificates/$NODE_NAME.pem /etc/wazuh-indexer/certs/indexer.pem
sudo cp wazuh-certificates/$NODE_NAME-key.pem /etc/wazuh-indexer/certs/indexer-key.pem
sudo cp wazuh-certificates/root-ca.pem /etc/wazuh-indexer/certs/root-ca.pem
sudo cp wazuh-certificates/admin.pem /etc/wazuh-indexer/certs/admin.pem
sudo cp wazuh-certificates/admin-key.pem /etc/wazuh-indexer/certs/admin-key.pem
sudo chmod 500 /etc/wazuh-indexer/certs
sudo chmod 400 /etc/wazuh-indexer/certs/*
sudo chown -R wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/certs
```

Edit `/etc/wazuh-indexer/opensearch.yml` and confirm `network.host`, `node.name`, and the cluster/discovery settings match your hostname/IP (the package installer usually pre-fills a sane default for single-node; check `discovery.type: single-node` is present for a lab setup).

Start and enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-indexer
sudo systemctl start wazuh-indexer
```

Initialize the security index (only needs to run once, from the indexer node):

```bash
sudo /usr/share/wazuh-indexer/bin/indexer-security-init.sh
```

Verify it's up:

```bash
curl -k -u admin:admin https://localhost:9200
```

You should get a JSON response with cluster name and version info. (Default `admin:admin` — you'll change this password in Step 6.)

## Step 4 — Install and configure the Wazuh server (manager)

```bash
sudo apt install -y wazuh-manager
```

Copy certs the same way:

```bash
NODE_NAME=wazuh-server
sudo mkdir -p /etc/wazuh-server/certs
sudo cp wazuh-certificates/$NODE_NAME.pem /etc/wazuh-server/certs/server.pem
sudo cp wazuh-certificates/$NODE_NAME-key.pem /etc/wazuh-server/certs/server-key.pem
sudo cp wazuh-certificates/root-ca.pem /etc/wazuh-server/certs/root-ca.pem
sudo chmod 500 /etc/wazuh-server/certs
sudo chmod 400 /etc/wazuh-server/certs/*
sudo chown -R wazuh-server:wazuh-server /etc/wazuh-server/certs
```

Point the manager at your indexer in `/etc/wazuh-indexer/opensearch.yml`-adjacent config — specifically `/usr/share/wazuh-server/apps/wazuh-server-management-apis/config.yml` or the connection block in the manager's `ossec.conf` (`<indexer>` section), setting the host to your indexer's IP and port 9200.

Start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-manager
sudo systemctl start wazuh-manager
sudo systemctl status wazuh-manager
```

Check that it's talking to the indexer without errors:

```bash
sudo tail -f /var/ossec/logs/ossec.log
```

## Step 5 — Install and configure the Wazuh dashboard

```bash
sudo apt install -y wazuh-dashboard
```

Copy certs:

```bash
NODE_NAME=wazuh-dashboard
sudo mkdir -p /etc/wazuh-dashboard/certs
sudo cp wazuh-certificates/$NODE_NAME.pem /etc/wazuh-dashboard/certs/dashboard.pem
sudo cp wazuh-certificates/$NODE_NAME-key.pem /etc/wazuh-dashboard/certs/dashboard-key.pem
sudo cp wazuh-certificates/root-ca.pem /etc/wazuh-dashboard/certs/root-ca.pem
sudo chmod 500 /etc/wazuh-dashboard/certs
sudo chmod 400 /etc/wazuh-dashboard/certs/*
sudo chown -R wazuh-dashboard:wazuh-dashboard /etc/wazuh-dashboard/certs
```

Edit `/etc/wazuh-dashboard/opensearch_dashboards.yml`:

- `server.host` → `0.0.0.0` (or your private IP)
- `opensearch.hosts` → `["https://<indexer-ip>:9200"]`
- confirm the `server.ssl.*` and `opensearch_security.*` cert paths point at the files you just copied

Start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-dashboard
sudo systemctl start wazuh-dashboard
```

## Step 6 — Change default passwords

Don't skip this, even in a practice lab — bad habits carry over.

```bash
cd wazuh-certificates
sudo /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh -a
```

This regenerates random passwords for the internal users (`admin`, `kibanaserver`, etc.) and prints them out — save them somewhere safe (a password manager, not a note in the repo).

## Step 7 — Log in

Browse to `https://<your-server-ip>` and log in with the `admin` user and the password from Step 6.

You should land on the Wazuh dashboard home screen with no agents connected yet — that's expected, since this guide only covers the server-side stack.

## Verifying everything is healthy

```bash
sudo systemctl status wazuh-indexer wazuh-manager wazuh-dashboard
curl -k -u admin:<new-password> https://localhost:9200/_cat/nodes?v
sudo /var/ossec/bin/wazuh-control status
```

## Common issues

- **Dashboard shows a blank/500 page** — usually a mismatch between the cert CN and the hostname in `opensearch_dashboards.yml`. Re-check `config.yml` from Step 2 matched your actual hostname before you generated certs.
- **Indexer won't start** — check `vm.max_map_count`: `sudo sysctl -w vm.max_map_count=262144` and persist it in `/etc/sysctl.conf`.
- **Manager can't reach indexer** — confirm the security group / firewall allows port 9200 between the two (or `localhost` if same box), and that the indexer's security init (Step 3) actually completed.
- **Certs rejected after re-running the tool** — `wazuh-certs-tool.sh` is destructive if re-run against the same `config.yml`; regenerate and redistribute to *all* components together, not just one.

## Next step for the team

Once you've done this manually once and understand what each cert/config file does, compare it against how CyberStorm's OpenTofu module provisions the same stack — same components, same trust relationships, just declared instead of typed by hand.

---
*Practice guide — CyberStorm SIEM project. Questions/corrections → open a PR against the `soc` repo.*
