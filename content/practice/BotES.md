---
title: "Practice: Add BOTES (Boss of the Elastic SOC) Data"
slug: practice/BotS/02-add-botes
date: 2026-07-13
tags: [soc, practice, elk, botes, botsv1, ingest]
status: draft
---

# Add BOTES Data to ELK

BOTES ("Boss of the Elastic SOC") is a community project that converted the Splunk-native BOTS datasets into JSON, with matching Elasticsearch index templates and Logstash pipeline configs — so we don't have to hand-write CIM-to-ECS field mappings from scratch. Do this after completing [Install ELK](./01-install-elk.md).

## Prerequisites

- Elasticsearch, Kibana, and Logstash installed and healthy (previous doc)
- Enough free disk for whichever dataset tier you're loading — attack-only (~135MB compressed) for onboarding, full BOTSv1 (~6.1GB compressed / ~120GB uncompressed) for the graded workshop exercise
- `sudo` access on the Logstash/Elasticsearch host

## 1. Pull the BOTES resources

BOTES publishes its converted dataset, Elasticsearch index template, and Logstash configs at `https://botes.gitbook.io/`. You'll need three things from there:

1. The converted BOTSv1 dataset (JSON, already ECS-shaped rather than Splunk CIM)
2. `template.json` — the Elasticsearch index template
3. The Logstash `.conf` files matching the sourcetypes you're loading

Download and stage these on the Logstash host:

```bash
mkdir -p ~/botes-staging
cd ~/botes-staging
# Pull the dataset, template.json, and .conf files from botes.gitbook.io into this directory
```

> Check file sizes and the BOTES site's current instructions before scripting a bulk download — the gitbook has been reorganized before, so confirm paths rather than assuming a fixed URL structure.

## 2. Load the Elasticsearch index template

```bash
curl -X PUT "localhost:9200/_index_template/botes-bots" \
  -H "Content-Type: application/json" \
  -d @~/botes-staging/template.json
```

Verify it registered:

```bash
curl -X GET "localhost:9200/_index_template/botes-bots?pretty"
```

## 3. Place the Logstash pipeline configs

Copy the BOTES `.conf` files into the pipeline directory created in the ELK install doc:

```bash
sudo cp ~/botes-staging/*.conf /etc/logstash/conf.d/bots/
```

Each `.conf` file typically defines:
- an **input** block pointing at the staged JSON files
- a **filter** block doing any remaining field cleanup
- an **output** block pointing at Elasticsearch, indexing into a `botsv1-*` (or similar) index pattern matching the template from step 2

Adjust the `input { file { path => ... } }` stanza in each conf to point at wherever you staged the dataset in step 1.

## 4. Adapt field names to OCSF (recommended, not optional for CyberStorm use)

BOTES ships with its own ECS-flavored schema. Since the CyberStorm SIEM Platform standardizes on OCSF via the Logstash/OCSF normalization pipeline (ADR-0001), don't just adopt BOTES' field names wholesale if this data is meant to flow through the same detection rules as production-shaped data:

- Add a `mutate`/`rename` filter stage in the BOTES conf files mapping ECS field names to the OCSF fields your Sigma rules expect
- Keep this mapping as its own documented step (or its own conf file) so it's easy to diff against the "as shipped by BOTES" version later
- If you're only using this for manual Kibana investigation (not rule validation), you can skip this step and use BOTES' schema as-is

## 5. Run the pipeline

```bash
sudo systemctl restart logstash
```

Watch the Logstash log for ingestion progress and errors:

```bash
sudo journalctl -u logstash -f
```

## 6. Verify in Kibana

1. **Stack Management → Data Views** — create a data view matching your BOTES index pattern (e.g. `botsv1-*`)
2. **Discover** — confirm events are present and timestamps parse correctly
3. Run a smoke-test query equivalent to the classic BOTS starter query (`index=botsv1 earliest=0` in Splunk terms) to confirm full-range data is visible, not just a partial load

## 7. Scope by dataset tier

- **Attack-only onboarding round:** load as-is, small enough to keep resident indefinitely on a lab box
- **Full dataset workshop:** consider loading only the sourcetypes relevant to that session's scenario at full volume, rather than the entire dataset, if disk is constrained — see the tiering discussion in the main [BOTS practice page](../BotS.md)

## Known gaps / open questions

- [ ] Confirm current BOTES gitbook download paths still match this doc (community projects drift)
- [ ] Decide whether the OCSF field-mapping conf (step 4) becomes a shared, versioned artifact in the `iac-practice` repo or stays workshop-local
- [ ] Document expected Elasticsearch storage footprint once the full dataset is loaded, to inform future workshop sizing
