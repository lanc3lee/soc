## ELK Platform Engineer — Setup Guide

**Owns:** Docker Compose stack, Elasticsearch/Logstash/Kibana config, JVM tuning, Nginx reverse proxy  
**Goal:** Get ELK running stably on a t3.medium (4GB RAM) — and understand why each tuning decision matters.

----------


### Before You Start

4GB RAM is genuinely tight for a full ELK stack. Don't start from a heavy pre-built docker-compose bundle — start minimal (just Elasticsearch + Kibana) and add pieces one at a time, checking memory usage at each step.

---

### Phase 1 — Minimal Stack

Get Elasticsearch + Kibana running together, nothing else yet. No Logstash, no security features, no SSL.

A clean reference for a standard ELK docker-compose structure:  
[https://github.com/deviantony/docker-elk](https://github.com/deviantony/docker-elk)

Read it to understand the structure (services, environment variables, volumes, networks) — don't just clone and run it as-is.

**Verify before moving on:**

- `docker stats` shows headroom left in your 4GB budget
- Kibana loads at `http://<host-ip>:5601`
- Elasticsearch responds at `http://<host-ip>:9200`

---

### Phase 2 — JVM Heap Tuning

Target: **`-Xms1g -Xmx1g`** for Elasticsearch (1GB heap, min = max).

yaml

```yaml
environment:
  - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
```

**Why min = max:** if they differ, the JVM can resize the heap at runtime, causing pauses and memory spikes — not something we can afford on 4GB.

**Why 1GB specifically:** heap should generally be ≤50% of total RAM. On 4GB, 1GB (25%) leaves room for the OS, Docker overhead, Logstash, and Kibana, all sharing the same box.

Reference: [https://www.elastic.co/guide/en/elasticsearch/reference/current/important-settings.html#heap-size-settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/important-settings.html#heap-size-settings)

**Verify:**

- Elasticsearch starts cleanly with the heap setting applied (`docker compose logs elasticsearch | grep -i heap`)
- `docker stats` confirms memory use matches expectations at idle

---

### Phase 3 — Add Logstash

Limit pipeline workers to 1 (default tries to match CPU core count — too much for our RAM budget on a 2-vCPU box):

yaml

```yaml
pipeline.workers: 1
```

Set a separate, smaller JVM heap for Logstash:

yaml

```yaml
environment:
  - "LS_JAVA_OPTS=-Xms256m -Xmx256m"
```

**Verify:**

- Logstash starts and connects to Elasticsearch (`docker compose logs logstash`)
- Combined memory across all 3 containers still leaves headroom — if it's maxed at idle, there's no room left for actual log processing, and something needs dialing back further

---

### Phase 4 — Nginx Reverse Proxy

Kibana should sit behind Nginx rather than being exposed directly — gives us one controlled entry point where we can later add TLS or IP restrictions.

nginx

```nginx
server {
    listen 80;
    server_name your-domain-or-ip;

    location / {
        proxy_pass http://kibana:5601;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Reference: [https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)

**Verify:**

- Kibana is reachable through Nginx's port instead of needing `:5601` directly
- The Elastic IP routes through Nginx to Kibana

---

### Phase 5 — Container Memory Limits

JVM heap limits what's inside the JVM process. Docker memory limits are the hard ceiling on the whole container — this is what actually stops one container from starving the others.

yaml

```yaml
services:
  elasticsearch:
    deploy:
      resources:
        limits:
          memory: 1.5g
  logstash:
    deploy:
      resources:
        limits:
          memory: 512m
  kibana:
    deploy:
      resources:
        limits:
          memory: 512m
```

**Why container limit > JVM heap, never less:** the container needs room for JVM heap plus overhead (metaspace, thread stacks, native memory). If the container limit is lower than the heap, Docker will OOM-kill it even though the JVM thinks it has room.

**Verify:**

- `docker stats` shows each container respecting its limit under load
- Stress-test with a large sample log file and confirm nothing gets OOM-killed. If something does, that's useful — it tells you exactly where to tune next.

---

### Phase 6 — Real Load Test (stretch)

Feed the stack sample CloudTrail/VPC Flow Log JSON (doesn't need to be live AWS data yet) and watch `docker stats` during ingestion. This is where the tuning actually gets tested against something real, not just theory.

---

### Quick Reference

|Resource|Use for|
|---|---|
|[Elastic — Install Elasticsearch with Docker](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html)|Container basics|
|[Elastic — Heap size settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/important-settings.html#heap-size-settings)|`-Xms`/`-Xmx`, the 50% rule|
|[deviantony/docker-elk](https://github.com/deviantony/docker-elk)|Clean vanilla compose structure|
|[Nginx reverse proxy guide](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)|Kibana access pattern|

---

### Knowledge check

You should be able to explain, not just have configured:

- Why `Xms` should equal `Xmx`
- Why heap should be ~50% of available RAM, not more
- The difference between JVM heap limits and Docker container memory limits — and why container limit must exceed JVM heap
- What happens when a container gets OOM-killed, and how to spot it in `docker stats` or logs
- Why Logstash pipeline worker count affects memory, not just speed

--------

refer to 
ELK-engineering.zip

**All 5 tuning phases are pre-applied in the compose file, but commented to explain why** — `Xms`/`Xmx` heap settings, pipeline worker limits, container memory limits, all there with inline reasoning. The engineer can read the file top to bottom as a second-pass explanation of the rollout guide, not just copy-paste it blindly.

**It includes a working Logstash pipeline and 5 lines of fake CloudTrail-style JSON** — so there's something to actually ingest and watch flow through on day one, rather than an empty stack with nothing to test against. The pipeline has a placeholder filter block clearly marked as where the real OCSF normalisation logic will plug in later (that's Role 3's job).

**Nginx is wired in but Kibana's direct port is still exposed** — intentional, so they can hit Kibana directly during early phases before Nginx is confirmed working, then drop the direct port mapping once Phase 4 passes, exactly as the rollout guide describes.

**The README has phase-by-phase verification commands** (`docker stats`, `curl` health checks, log greps) — matches the structure of the IaC starter so the team has a consistent "read guide → run starter → verify" pattern across roles.

To get this into the repo, they'd drop it into a top-level `docker-compose/` or `elk/` directory per your repo structure, then `git add . && git commit && git push`.