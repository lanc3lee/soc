---
title: "Runbook: Provisioning Individual Kibana Accounts"
slug: practice/BotS/kibana-user-provisioning
date: 2026-07-24
tags: [soc, elk, kibana, elasticsearch, iam, runbook]
status: draft
---

# Provisioning Individual Kibana Accounts

Internal checklist for creating a new teammate's Kibana account on the CyberStorm ELK stack. Every teammate gets their own account — nobody uses the `elastic` superuser or `kibana_system` service account to log in.

## Why individual accounts

- **Least privilege** — scope each person to only what their role needs
- **Audit trail** — Kibana logs actions per user, useful for tracing changes to detections/dashboards later
- **Clean revocation** — remove one account when someone leaves, instead of rotating a password everyone shares

## Prerequisites

- Admin (`elastic`) credentials for the cluster, kept out of this document
- SSH access to the ELK host
- The new teammate's preferred username

## Checklist

- [ ] **Pick a role** for the new account based on what they'll actually do:
  - `viewer` — read-only, dashboards and Discover only
  - `editor` — can build/edit dashboards and visualizations
  - `kibana_admin` — full Kibana administration (reserve for a small number of trusted leads)
  - Custom roles can be defined in Kibana under **Stack Management → Roles** if none of the built-ins fit

- [ ] **Create the user** via the Elasticsearch security API, run from the ELK host:

  ```bash
  curl -u elastic:$ELASTIC_PASSWORD -X POST http://localhost:9200/_security/user/<username> \
    -H "Content-Type: application/json" \
    -d '{
      "password": "<temporary-password>",
      "roles": ["<role-from-above>"]
    }'
  ```

  A successful response returns `{"created":true}`.

- [ ] **Send credentials to the teammate over a private channel** (DM, not a shared channel) — never paste credentials into group chat or commit them anywhere.

- [ ] **Ask them to log in and change their password immediately** — under their user menu in Kibana, top right → **Profile** → change password.

- [ ] **Record the account** in the internal team roster (name, username, role, date added) — kept separately from this public runbook.

- [ ] **Confirm access** — have them load the Kibana URL over Tailscale and confirm login succeeds and their role's permissions look correct (e.g. a `viewer` shouldn't be able to edit a dashboard).

## Offboarding

When someone leaves the team, remove their account rather than just letting the password lapse:

```bash
curl -u elastic:$ELASTIC_PASSWORD -X DELETE http://localhost:9200/_security/user/<username>
```

## Notes

- Never share the `elastic` or `kibana_system` credentials with teammates — these are cluster-admin and internal service accounts respectively, not meant for individual login.
- Rotate the `elastic` superuser password periodically, and immediately if it's ever been exposed (e.g. pasted somewhere it shouldn't have been).
