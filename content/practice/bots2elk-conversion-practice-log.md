# BOTS-to-ECS Conversion Script — Practice Log
**Date:** July 14, 2026
**Environment:** MacBook (macOS, zsh), working dir `/Users/lance/Documents/BotS`
**Goal:** Build a Python pipeline to convert Splunk BOTS v1 JSON exports into ECS-compliant, chunked, gzip-compressed NDJSON — suitable for local Elasticsearch ingestion and eventual public distribution (GitHub Releases + Zenodo mirror).

---

## 1. Environment Setup Attempt — Git LFS (dead end)

Initial approach assumed BOTS data might be tracked via Git LFS inside Splunk's own repo:

```bash
brew install git-lfs
git lfs install
cd /Users/lance/Documents/BotS
GIT_LFS_SKIP_SMUDGE=1 git clone https://github.com/splunk/botsv1.git splunk_repo
cd splunk_repo
git lfs pull --include="json-by-sourcetype/botsv1.suricata.json.gz"
cp json-by-sourcetype/botsv1.suricata.json.gz /Users/lance/Documents/BotS/data/raw/
```

**Result:** `cp` failed — `No such file or directory`. Confirmed with:
```bash
find . -name "*suricata*" -o -name "*.json.gz"
```
— returned nothing. The clone only pulled 87 objects (26KB), meaning the `splunk/botsv1` repo contains only docs/scripts, **not** the actual dataset via Git LFS. The data was never stored in the repo itself.

A follow-up attempt at a raw GitHub file path also failed for the same underlying reason (data isn't hosted in the repo):
```bash
curl -L -# "https://github.com/splunk/botsv1/raw/master/botsv1-attack-only.tgz" -o data/raw/botsv1-attack-only.tgz
```

**Conclusion:** the actual BOTS v1 data lives externally, on Splunk's own public S3 bucket — not inside the git repo at all.

---

## 2. Confirmed Working Data Source

Verified earlier in a separate AWS-based session (successfully downloaded and inspected 3.8GB of real data), so this URL pattern is confirmed live, not just found via search:

```
https://s3.amazonaws.com/botsdataset/botsv1/json-by-sourcetype/botsv1.<sourcetype>.json.gz   (per sourcetype)
https://s3.amazonaws.com/botsdataset/botsv1/botsv1-attack-only.tgz                            (135MB, small subset)
https://s3.amazonaws.com/botsdataset/botsv1/botsv1.json.gz                                    (11.3GB, full combined — not needed for per-sourcetype work)
```

**Step 1 — Download the raw sourcetype file(s) actually needed** (not the full 11GB combined file — per-sourcetype files are the right unit of work for incremental conversion):

```bash
cd /Users/lance/Documents/BotS
curl -L -O https://s3.amazonaws.com/botsdataset/botsv1/json-by-sourcetype/botsv1.suricata.json.gz
mv botsv1.suricata.json.gz data/raw/
```

Directory structure prepared ahead of time:
```bash
mkdir -p data/raw data/converted
```

---

## 3. Script Review — `bots2elk.py` (original, user-provided)

Reviewed an existing conversion script with the following design: streams Splunk JSON, maps a fixed set of fields to ECS, chunks output into gzip-compressed NDJSON files capped at a configurable size (default 1.5GB, to stay under GitHub Releases' 2GB per-asset limit).

### Bugs identified

1. **Critical — doesn't unwrap Splunk's export envelope.** Every line in the real dataset is wrapped as `{"preview": false, "offset": N, "result": {...actual event...}}`. The original script read fields directly off the outer object, meaning every field lookup (`_time`, `host`, etc.) would silently return `None` — producing near-empty ECS output with no error raised.
2. **Timestamp parsing didn't match BOTS' actual format.** Real `_time` values look like `"2016-08-28 17:59:00.000 MDT"` — a formatted string, not epoch. The original script's `float(splunk_time)` call would fail and fall back to passing the raw string through, which Elasticsearch cannot reliably parse as a date field.

### Other gaps identified

- Only ~10 fields were ever copied to output; everything else (an large majority of real fields, e.g. `EventCode`, `TaskCategory`, `Message`, `dvc`) was silently discarded.
- No handling for files that turn out to be a single JSON array instead of NDJSON — would silently produce zero output with no warning.
- No output directory creation — chunks would land in the current working directory regardless of the intended `data/converted/` structure.
- Positional-args-only CLI, no sample/limit flag for safe testing before a full run.

### Fixes applied (`bots2elk_v2.py`)

- Added `unwrap_envelope()` to detect and unwrap the `result` wrapper when present, while still supporting already-flat records.
- Added `first_if_list()` to normalize fields that appear as either a scalar or a duplicated list (e.g. `Account_Domain`) — avoids inconsistent field typing in Elasticsearch.
- Rewrote `parse_timestamp()` to handle both epoch and Splunk's string format properly, flagging any record where parsing genuinely failed (`event.timestamp_parse_failed`) instead of silently guessing.
- Preserved all unmapped fields under `labels.splunk_source` instead of discarding them — keeps full fidelity for later OCSF mapping work.
- Added a fallback path (`iter_records()`) to detect and handle whole-file JSON array input, with an explicit warning instead of silent zero-output.
- Added `argparse` CLI: `--chunk-size-gb`, `--sample` (for safe small-batch testing), automatic output directory creation.

### Second bug found via real sample output, then fixed

Running the fixed script against a real 200-record Suricata sample surfaced two further issues, caught by inspecting actual output rather than trusting the code alone:

1. **`@timestamp` was not valid ISO 8601.** Output looked like `"2016-08-28T23:58:59.931000 (MDT)"` — the trailing `" (MDT)"` is plain text, not a numeric UTC offset, so Elasticsearch would fail to map this as a proper `date` field (would likely fall back to `text`, breaking all time-based queries/Kibana timelines).
2. **Ports were strings, not integers.** ECS specifies `source.port`/`destination.port` as `long`. Output had `"port": "53"` instead of `53`.

**Fixes:**
- Added a fixed-offset lookup table for common US timezone abbreviations (`MST`, `MDT`, `PST`, `PDT`, `CST`, `CDT`, `EST`, `EDT`, `UTC`, `GMT`) and applied the real offset to produce valid ISO 8601 output, e.g. `2016-08-28T23:58:59.931000-06:00`.
- Cast `*.port` fields to `int` defensively (falls back to original value if genuinely non-numeric, rather than crashing the run).

**Verified fixed** via a re-run of the same 200-record sample:
```json
"@timestamp": "2016-08-28T23:58:59.931000-06:00"
"source": { "ip": "8.8.8.8", "port": 53 }
"destination": { "ip": "192.168.225.125", "port": 53338 }
```
Both timestamp validity and port typing confirmed correct on inspection.

---

## 4. Local Tooling Note — macOS `zcat`

```bash
zcat data/converted/suricata/botsv1_suricata_ecs.part_001.ndjson.gz | head -n 2 | jq .
```
failed with `can't stat ... .gz.Z: No such file or directory` — macOS's BSD `zcat` expects a `.Z` extension, unlike Linux's GNU `zcat`. Fixed with either:
```bash
gzcat data/converted/suricata/botsv1_suricata_ecs.part_001.ndjson.gz | head -n 2 | jq .
# or
zcat -f data/converted/suricata/botsv1_suricata_ecs.part_001.ndjson.gz | head -n 2 | jq .
```

---

## 5. BOTS Version Availability (carried over research)

| Version | Format | ECS/Elastic conversion available? |
|---|---|---|
| **v1** | Splunk pre-indexed **and** raw JSON-by-sourcetype (S3) | Partial — a 2019 community project (BOTES) attempted this but its hosting is now dead (`NoSuchBucket`). Raw JSON-by-sourcetype from Splunk's own S3 is still live and is the basis of this conversion work. |
| **v2** | Splunk pre-indexed **only** | No — no existing conversion; BOTES never covered v2 (incompatible export format) |
| **v3** | Splunk pre-indexed **only**, intentionally (avoids per-event licensing limits) | No — no known community conversion |

**Feasibility note for v2/v3 conversion (future scope):** would require standing up Splunk Enterprise (free trial license) to read the pre-indexed data, exporting per-sourcetype via Splunk search as JSON, then repeating this same cleaning/mapping process — plus net-new ECS mapping for cloud-native sourcetypes (AWS CloudTrail, Azure, GCP, Docker) introduced in v2/v3 that v1 didn't have. Realistically weeks of work, but a strong differentiated angle for a BSides Singapore submission: "extending community BOTS-to-ECS conversion beyond v1."

---

## 6. Step-by-Step Execution Plan (going forward)

**Step 1 — Download the raw sourcetype file(s)**
```bash
cd /Users/lance/Documents/BotS
curl -L -O https://s3.amazonaws.com/botsdataset/botsv1/json-by-sourcetype/botsv1.<sourcetype>.json.gz
mv botsv1.<sourcetype>.json.gz data/raw/
```

**Step 2 — Validate the conversion script on a small sample first**
```bash
python3 bots2elk_v2.py data/raw/botsv1.<sourcetype>.json.gz \
    data/converted/<sourcetype>/botsv1_<sourcetype>_ecs \
    --sample 200
```

**Step 3 — Inspect output carefully before trusting it**
```bash
gzcat data/converted/<sourcetype>/botsv1_<sourcetype>_ecs.part_001.ndjson.gz | head -n 2 | jq .
```
Check specifically: valid ISO 8601 `@timestamp` with real offset, correctly typed/populated mapped fields (e.g. `source.ip`, `destination.port` as integer), and unmapped fields present under `labels.splunk_source`.

**Step 4 — Load the sample into local ELK stack to confirm it's actually queryable (in progress / next session)**

Note: Elasticsearch's `_bulk` API requires an action/metadata line interleaved before each document — the script's current NDJSON output is one flat document per line, so a raw file can't be bulk-loaded as-is. Interim workaround to test with:
```python
import json
with open('data/converted/<sourcetype>/botsv1_<sourcetype>_ecs.part_001.ndjson') as f, open('bulk_ready.ndjson', 'w') as out:
    for line in f:
        out.write('{"index":{"_index":"bots-<sourcetype>-test"}}\n')
        out.write(line)
```
```bash
curl -X POST "http://localhost:9200/_bulk" -H "Content-Type: application/x-ndjson" --data-binary @bulk_ready.ndjson
curl "http://localhost:9200/bots-<sourcetype>-test/_mapping?pretty" | grep -A 3 '"@timestamp"'
```
Confirm ES actually assigned `"type": "date"` to `@timestamp` — the real confirmation that the fix worked, not just that it looks correct via `jq`.

**Step 5 — Run the full conversion on that sourcetype** (once sample is validated)
```bash
python3 bots2elk_v2.py data/raw/botsv1.<sourcetype>.json.gz data/converted/<sourcetype>/botsv1_<sourcetype>_ecs
```

**Step 6 — Repeat Steps 1–5 for each additional sourcetype**, one at a time rather than batch-downloading/converting all ~22 upfront.

**Step 7 — Publish once a validated set of chunks exists**
- Primary: GitHub Releases (chunks already sized to stay under the 2GB per-asset limit)
- Backup/mirror: Zenodo (free, permanent DOI) — worth doing given two separate instances of dead hosting links encountered this session (BOTES S3 bucket, wrong GitHub Releases URL) already proved single-source hosting is fragile

**Step 8 — Write a README** documenting the ECS mapping table, known limitations (unmapped fields under `labels.splunk_source`, any `timestamp_parse_failed` records), and how to regenerate the dataset from source.

---

## 7. Outstanding TODOs (next session)

- [ ] Add proper Elasticsearch `_bulk` API formatting (action/metadata + document line pairs) directly into the script, rather than the one-off Python snippet workaround
- [ ] Load Suricata sample into local ELK stack, confirm `@timestamp` mapped as `date` type
- [ ] Once validated, run full conversion on `botsv1.suricata.json.gz`
- [ ] Repeat for additional sourcetypes (Windows Security events already explored in a prior AWS session)
- [ ] Consider whether to prefer a sourcetype's own native timestamp (e.g. Suricata's `timestamp` field, which already carries a precise numeric offset) over Splunk's `_time` for network-derived sourcetypes
- [ ] Draft README with ECS mapping table + methodology, ahead of GitHub Releases + Zenodo publication
- [ ] Revisit v2/v3 conversion feasibility as a possible BSides Singapore submission angle once v1 pipeline is fully proven out
