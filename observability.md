
## Observability & Threat Intel Lead — Setup Guide

**Owns:** OpenTelemetry Collector, infra health dashboards, AlienVault OTX integration  
**Goal:** Build the monitoring for the monitoring system — answer "is the SIEM itself healthy?" — plus enrich security events with threat intelligence.

---

### Before You Start

You're mostly independent. You don't need Role 3's OCSF pipelines or Role 4's Sigma rules to get started — you can begin in Week 1, right alongside the Infra & IaC lead. Your job is watching the _infrastructure_, not the security events flowing through it.

One important security note up front, since you'll be giving the OTel Collector access to the Docker socket (`/var/run/docker.sock`) to read container stats: this is a sensitive permission — anything with Docker socket access effectively has root-equivalent control over the host. Run the collector container as a non-root user with explicit Docker group access rather than `--privileged` or root, and don't expose the collector's OTLP ports beyond what's needed.

---

### Phase 1 — Get the Collector Running Standalone (Day 1–2)

Don't wire it into Elasticsearch yet. First, just get the OpenTelemetry Collector running in Docker and confirm it starts cleanly.

yaml

```yaml
# otel-collector-config.yaml — minimal starting point
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

exporters:
  debug:
    verbosity: detailed

service:
  pipelines:
    metrics:
      receivers: [otlp]
      exporters: [debug]
```

yaml

```yaml
# docker-compose.yml snippet
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    container_name: cyberstorm-otel-collector
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml:ro
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
    deploy:
      resources:
        limits:
          memory: 100m
```

**Verify:** `docker compose logs -f otel-collector` shows the collector starting and listening on both ports, with no config errors.

---

### Phase 2 — Host Metrics (Day 2–3)

Add the `hostmetrics` receiver to pull CPU, memory, disk, and network stats from the EC2 host itself.

yaml

```yaml
receivers:
  hostmetrics:
    collection_interval: 30s
    scrapers:
      cpu:
      memory:
      disk:
      filesystem:
      network:
```

**If running the collector itself inside a container**, it needs access to the host's filesystem to read real host-level stats rather than the container's isolated view. Mount the host root and set `root_path: /hostfs`:

yaml

```yaml
receivers:
  hostmetrics:
    root_path: /hostfs
    collection_interval: 30s
    scrapers:
      cpu:
      memory:
      disk:
      filesystem:
        exclude:
          mount_points: ["/boot", "/tmp/*", "/dev/*", "/run/*", "/sys/*"]
          match_type: regexp
      network:
```

yaml

```yaml
# docker-compose.yml addition
services:
  otel-collector:
    volumes:
      - /:/hostfs:ro
```

**Verify:** point the `metrics` pipeline at the `debug` exporter temporarily and confirm CPU/memory/disk numbers in the logs roughly match what `docker stats` and `top` show you directly on the host.

---

### Phase 3 — Docker Container Metrics (Day 3–4)

Add `docker_stats` to get per-container CPU, memory, network, and disk I/O — this is what lets you answer "which specific ELK container is struggling" rather than just "the host is under load."

yaml

```yaml
receivers:
  docker_stats:
    endpoint: unix:///var/run/docker.sock
    collection_interval: 30s
    timeout: 10s
    container_labels_to_metric_labels:
      com.docker.compose.service: service.name
```

**Docker socket permission gotcha:** official OTel Collector images run as non-root by default, which conflicts with reading the Docker socket. Don't reach for `user: root` as the fix — instead add the host's docker group ID as a supplementary group:

bash

```bash
# find the docker group ID on the host
getent group docker
```

yaml

```yaml
# docker-compose.yml
services:
  otel-collector:
    group_add:
      - "999"   # replace with your actual docker GID from the command above
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
```

**Verify:** logs show `docker_stats receiver` starting without permission errors, and you can see metrics tagged with `service.name` for each ELK container (elasticsearch, logstash, kibana, nginx).

---

### Phase 4 — Wire Into Elasticsearch (Day 4–5)

Now replace the `debug` exporter with the real `elasticsearch` exporter, so all this telemetry actually lands somewhere queryable in Kibana.

yaml

```yaml
processors:
  batch:
    timeout: 10s
    send_batch_size: 1024
  resourcedetection:
    detectors: [env, system, docker]
    override: false

exporters:
  elasticsearch:
    endpoint: "http://elasticsearch:9200"
    user: elastic
    password: ${ELASTIC_PASSWORD}

service:
  pipelines:
    metrics:
      receivers: [hostmetrics, docker_stats]
      processors: [resourcedetection, batch]
      exporters: [elasticsearch]
```

**Add a memory limiter processor** — important on a 4GB box, this protects the collector itself from ballooning and taking memory away from Elasticsearch/Logstash:

yaml

```yaml
processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 100
    spike_limit_mib: 30
```

Put `memory_limiter` first in the processor list — order matters, it needs to run before `batch`.

**Verify:** query Elasticsearch directly and confirm metrics are landing:

bash

```bash
curl -u elastic:changeme "http://localhost:9200/otel-metrics-*/_search?pretty"
```

---

### Phase 5 — Infra Health Dashboard in Kibana (Day 5–6)

Build the dashboard that answers "is the SIEM itself healthy" at a glance. At minimum:

- **Host-level:** CPU %, memory %, disk usage — the three numbers tied directly to your project's scaling triggers (Section 7 of the provisioning doc)
- **Per-container:** memory usage per ELK container, so you can see which one is closest to its Docker memory limit before it gets OOM-killed
- **Logstash pipeline health:** events in/out, queue size — if you have time, query Logstash's own metrics API and feed that in too

Keep this simple and boring on purpose — this dashboard's job is to be glanced at, not studied. Alert thresholds worth marking visually:

- CPU > 70% sustained → matches your EC2 scaling trigger
- Memory > 80% → matches your t3.medium RAM-constraint risk
- Disk/EBS > 70% → matches your EBS scaling trigger

This dashboard is the **actual mechanism** by which the team knows a scaling trigger has fired — not a nice-to-have, it's load-bearing for the whole PoC-to-production plan.

---

### Phase 6 — AlienVault OTX Threat Intel Integration (Day 6–8)

Separate workstream from the OTel pipeline above — this enriches _security_ events (once Role 3's OCSF data is flowing) with threat intelligence, rather than monitoring infrastructure health.

1. Create a free OTX account: [https://otx.alienvault.com](https://otx.alienvault.com)
2. Generate an API key from your account settings
3. Store it in Secrets Manager (Resource #9 in the provisioning doc) — never hardcode it
4. Build a Logstash filter (or a small script) that queries OTX for IoC matches against IPs/domains seen in your normalized events, and appends results as an OCSF enrichment array: `{ name, value, type, provider }`

**This phase depends on Role 3's normalized data existing** — unlike Phases 1–5, which are fully independent, this is where your work and the Data Pipeline lead's work intersect. Coordinate before building this so you're enriching the right fields.

---

### Quick Reference

|Resource|Use for|
|---|---|
|[OTel Collector Docker guide](https://opentelemetry.io/docs/collector/configuration/)|Core config syntax, receivers/processors/exporters/pipelines|
|[Docker Stats receiver docs](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/dockerstatsreceiver/README.md)|Per-container metrics, socket permission handling|
|[Host Metrics receiver guide](https://uptrace.dev/opentelemetry/collector/host-metrics)|CPU/memory/disk/network scraper config|
|[Elastic OTel Collector integration](https://oneuptime.com/blog/post/2026-02-06-elastic-distribution-opentelemetry-collector/view)|Sending OTLP data directly to Elasticsearch|
|[AlienVault OTX](https://otx.alienvault.com)|Free threat intel API, account + key setup|

---

### Knowledge check

You should be able to explain, not just have configured:

- Why the collector needs `root_path: /hostfs` to read accurate host metrics when running inside a container
- Why you used `group_add` with the Docker group GID instead of running the container as root to access the Docker socket
- Why `memory_limiter` has to be the first processor in the pipeline, not just somewhere in the list
- Which three metrics on your dashboard map directly to the project's CloudWatch scaling triggers, and what threshold each one watches for
- How OTX enrichment attaches to an already-normalized OCSF event, rather than being its own separate event class