---
title: "Wazuh Installation Practice Lab"
date: 2026-07-13
tags:
  - practice
  - wazuh
  - siem
  - homelab
description: "Two-part practice lab: install Wazuh with the official quickstart assistant, then walk through exactly what it did under the hood."
---

# Wazuh Installation Practice Lab

This lab has two parts, run in order:

1. **Install with the quickstart assistant.** Get a working Wazuh stack up in minutes, the way Wazuh itself recommends. No cert-path typos, no half-configured services — everyone ends this part with a working dashboard.
   
2. **Inspect what the assistant did.** Go back through the same host and find every cert, config file, and systemd unit the script touched. This is the part that matters for CyberStorm — it's the same architecture we'll eventually declare in OpenTofu, so you need to know what's actually there before we can automate it.

Don't skip straight to Part 2. Trying to build the trust chain by hand *before* you've seen a working reference install is how you burn an afternoon chasing a typo instead of learning anything.

## What you're installing

| Component | Role | Default port |
|---|---|---|
| Wazuh indexer | Stores and indexes alert/event data (OpenSearch-based) | 9200 |
| Wazuh server (manager) | Analyzes data from agents, runs rules/decoders | 1514, 1515, 55000 |
| Wazuh dashboard | Web UI (OpenSearch Dashboards-based) | 443 |

All three land on a single host for this lab. In production these get split across hosts — that's a later exercise.

## Prerequisites

- Ubuntu 22.04 or 24.04, x86_64
- Minimum 4 vCPU / 4 GB RAM (check current sizing against the [Wazuh quickstart requirements](https://documentation.wazuh.com/current/quickstart.html) before you provision, since recommendations shift between versions)
- Root or sudo access
- Ports open on the host/security group: 443 (dashboard), 1514/1515 (agent comms), 55000 (API)
- A stable hostname — don't rename the box mid-lab

```bash
sudo apt update && sudo apt upgrade -y
sudo hostnamectl set-hostname wazuh-node
```

---

## Part 1 — Install with the quickstart assistant

This is the officially recommended path for a single-node install. Run it, verify it works, then move to Part 2.

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

Installation takes several minutes — the script pulls all three components, generates certificates, wires up the config, and starts the services. Watch the terminal output; you'll see it move through indexer → manager → dashboard.

When it finishes, you'll get a summary block like this:

```
INFO: --- Summary ---
INFO: You can access the web interface https://<WAZUH_DASHBOARD_IP_ADDRESS>
    User: admin
    Password: <ADMIN_PASSWORD>
INFO: Installation finished.
```

**Save that password now** — either copy it out of your terminal scrollback or pull it from the passwords file (see Part 2, Step 4).

### Verify the install

```bash
sudo systemctl status wazuh-indexer wazuh-manager wazuh-dashboard
```

All three should show `active (running)`.

Browse to `https://<your-server-ip>` and log in with `admin` and the password from the summary block. Accept the browser's self-signed cert warning — that's expected, and you'll see exactly why in Part 2.

You should land on the dashboard home screen with zero agents connected. That's fine — this lab only covers the server-side stack.

### Optional: disable auto-updates

Recommended so a package update doesn't quietly break your lab mid-session:

```bash
sudo sed -i "s/^deb /#deb /" /etc/apt/sources.list.d/wazuh.list
sudo apt update
```
----

Whole installation process using quickstart assistant takes about 9 mins. 

```
**ubuntu@ip-172-31-33-159**:**~**$ curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh

**ubuntu@ip-172-31-33-159**:**~**$ sudo bash wazuh-install.sh -a

13/07/2026 03:35:58 INFO: Starting Wazuh installation assistant. Wazuh version: 4.9.2

...

13/07/2026 03:43:49 INFO: You can access the web interface https://<wazuh-dashboard-ip>:443

    User: admin

    Password: L5****************ED

13/07/2026 03:43:49 INFO: Installation finished.

**ubuntu@ip-172-31-33-159**:**~**$


```

wazuh-dashboard-ip stated above is your AWS EC2 instance IP, 

to find that, run 

```
curl -s http://169.254.169.254/latest/meta-data/public-ipv4

```


---

## Part 2 — What the installer actually did

You now have a working stack. This part doesn't install anything new — it's a guided tour through the files and services the assistant created, so you understand the pieces well enough to reason about them (and eventually declare them in OpenTofu).

### Step 1 — Find the artifacts the installer left behind

The assistant generates a tarball of certs and a passwords file in the directory you ran it from:

```bash
ls -la wazuh-install-files.tar
tar -tf wazuh-install-files.tar
```

You should see `config.yml` (the node-name/IP definitions used to generate certs) and the certificate set inside. This is the same `config.yml` structure you'd hand-edit in a fully manual install — the assistant just filled it in with sane single-node defaults and ran the cert tool against it for you.

Extract it somewhere so you can look:

```bash
mkdir -p ~/wazuh-install-inspect
tar -xf wazuh-install-files.tar -C ~/wazuh-install-inspect
ls ~/wazuh-install-inspect
cat ~/wazuh-install-inspect/config.yml
```

### Step 2 — Trace the trust chain (certs)

Each component gets its own cert/key pair signed by the same root CA, which is *why* they trust each other. Find where each one actually landed:

```bash
sudo ls -l /etc/wazuh-indexer/certs/
sudo ls -l /etc/wazuh-server/certs/     # or /etc/filebeat/certs/ depending on version
sudo ls -l /etc/wazuh-dashboard/certs/
```

For each directory, note:
- the node's own `.pem` / `-key.pem` pair
- a copy of `root-ca.pem` — same file, present in all three directories, which is what lets indexer, manager, and dashboard verify each other
- permissions — you should see `500` on the certs directory and `400` on the individual files, owned by that component's service user (`wazuh-indexer`, `wazuh-server`/`wazuh-manager`, `wazuh-dashboard`)

This is the self-signed cert your browser warned you about in Part 1 — the root CA the assistant generated isn't in your browser's trust store, which is expected for a lab.

### Step 3 — Read the config that wires them together

The indexer doesn't know about the manager or dashboard by magic — it's config:

```bash
sudo cat /etc/wazuh-indexer/opensearch.yml | grep -E "network.host|node.name|discovery.type"
sudo cat /etc/wazuh-dashboard/opensearch_dashboards.yml | grep -E "server.host|opensearch.hosts"
```

Confirm you can see:
- the indexer's `discovery.type: single-node` (this is what tells it not to look for cluster peers)
- the dashboard's `opensearch.hosts` pointing at the indexer's address on port 9200

For the manager, check the `<indexer>` block in its config for the connection details:

```bash
sudo grep -A5 "<indexer>" /var/ossec/etc/ossec.conf
```

### Step 4 — Find the credentials

```bash
cat ~/wazuh-install-inspect/wazuh-passwords.txt 2>/dev/null || \
  find ~/wazuh-install-inspect -iname "*password*"
```

This is where `admin`, `kibanaserver`, and the other internal service accounts' passwords live — the same output the assistant printed to your terminal at the end of Part 1. Move this file somewhere safe (a password manager, not a repo) and don't leave it sitting in a home directory.

### Step 5 — Map the systemd units

```bash
systemctl list-unit-files | grep wazuh
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-dashboard
```

Each service was enabled independently. Note the start order dependency isn't enforced by systemd here — it's enforced by the installer running them in sequence during setup (indexer must be healthy before the manager and dashboard can talk to it). That ordering constraint is one of the things you'll need to model explicitly once this becomes an OpenTofu/provisioning module — systemd won't do it for you on a reboot unless you add that dependency yourself.

### Step 6 — Watch it work end to end

```bash
curl -k -u admin:<password-from-step-4> https://localhost:9200/_cat/nodes?v
sudo tail -f /var/ossec/logs/ossec.log
```

The `_cat/nodes` call proves the indexer's security layer and your admin credentials work. The manager log will show it flushing alerts to the indexer if everything's wired correctly.

---

## Common issues

- **Browser cert warning on first login** — expected; see Part 2 Step 2. Only worry if it's a *different* error (name mismatch, expired) than the standard self-signed warning.
- **Indexer won't start** — check `vm.max_map_count`: `sudo sysctl -w vm.max_map_count=262144`, and persist it in `/etc/sysctl.conf`.
- **Dashboard can't reach indexer** — re-check the `opensearch.hosts` value from Part 2 Step 3 against the indexer's actual listening address.
- **Lost the admin password and didn't save it** — it's in `wazuh-passwords.txt` inside `wazuh-install-files.tar`, per Part 2 Step 4, as long as you kept that file.

## Next step for the team

Now that you've seen both the automated path and what it actually configures, compare this against how CyberStorm's OpenTofu module will provision the same stack: same components, same trust relationships, same systemd units — just declared instead of assistant-generated. Part 2 is effectively the spec for that module.

---
*Practice guide — CyberStorm SIEM project. Questions/corrections → open a PR against the `soc` repo.*
