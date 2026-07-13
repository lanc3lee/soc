---
title: "Practice: Install ELK via Docker Compose (fast path)"
slug: practice/BotS/elk-docker
date: 2026-07-13
tags: [soc, practice, elk, docker, elasticsearch, logstash, kibana, install]
status: draft
---

# Install ELK via Docker Compose (fast path)

This is a faster alternative to the [native package install](./01-install-elk.md). Full stack up in under a minute, with no manual `elasticsearch.yml` editing — which also means none of the config landmines documented in that guide (missing-space YAML errors, orphaned SSL keys, the `discovery.type`/`initial_master_nodes` conflict).

**Use this when the goal is reps in Kibana** — hunting through BOTS data, validating Sigma rules, running a workshop. **Use the native install instead when the goal is also teaching how to actually operate Elasticsearch/Kibana as services** (systemd, native config files, production-shaped ops). This doc is optimized for the former.

## Why this avoids today's errors

The official Elastic Docker images configure the basics via environment variables rather than a hand-edited YAML template, and they skip the security auto-configuration step that generates the SSL keystore secrets, self-signed certs, and enrollment tokens — the exact things that caused four rounds of debugging in the native install. There's no file to typo, no orphaned nested key, no conflicting bootstrap settings.

## Prerequisites

- Ubuntu 22.04/24.04 (same target as the native guide)
- Docker Engine + Docker Compose plugin installed
- 4GB+ RAM available (Elasticsearch's JVM heap is capped explicitly below, so this is more predictable than the native install's automatic ~50%-of-RAM sizing)
- Outbound HTTPS access to `docker.elastic.co`

### Install Docker (if not already present)

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
```

Log out and back in (or `newgrp docker`) for the group change to take effect, then confirm:

```bash
docker --version
docker compose version
```

## 1. Create the project directory

```bash
mkdir -p ~/elk-docker/pipeline
cd ~/elk-docker
```

The `pipeline/` subdirectory is where Logstash `.conf` files (including the BOTES ones from the next guide) will be mounted in.

## 2. Write the compose file

```bash
cat > docker-compose.yml <<'EOF'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.19.18
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - cluster.name=cyberstorm-bots-lab
      - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
    ports:
      - "9200:9200"
    volumes:
      - es-data:/usr/share/elasticsearch/data
    restart: unless-stopped

  kibana:
    image: docker.elastic.co/kibana/kibana:8.19.18
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
    restart: unless-stopped

  logstash:
    image: docker.elastic.co/logstash/logstash:8.19.18
    container_name: logstash
    volumes:
      - ./pipeline:/usr/share/logstash/pipeline
    depends_on:
      - elasticsearch
    restart: unless-stopped

volumes:
  es-data:
EOF
```

> Pin the image version (`8.19.18` above) to match whatever the rest of the CyberStorm SIEM Platform is standardized on — check the existing ELK manual install guide before assuming this is current.

Notes on the choices baked into this file:
- `ES_JAVA_OPTS` explicitly caps the heap at 2g rather than letting Elasticsearch auto-size — predictable memory usage on a shared/small lab box.
- `xpack.security.enabled=false` is the Docker-native equivalent of the setting you fought with manually — one environment variable, no YAML file to edit.
- No SSL/keystore settings at all — the Docker image simply doesn't run the security auto-config step unless you explicitly enable it.

## 3. Start the stack

```bash
docker compose up -d
```

Watch startup:

```bash
docker compose logs -f
```

Look for Elasticsearch logging `started` and Kibana logging `Kibana is now available` (or `http server running`). Ctrl+C to stop following once both are up — this doesn't stop the containers.

## 4. Verify

```bash
curl -X GET "localhost:9200/?pretty"
```

Should return the same kind of JSON blob as the native install, with `"cluster_name" : "cyberstorm-bots-lab"`.

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:5601
```

`302` or `200` means Kibana's serving requests. Then load `http://<your-ec2-public-ip>:5601` in a browser (same security-group port 5601 rule as the native install applies here too).

## 5. Common commands

```bash
docker compose ps              # status of all three containers
docker compose logs elasticsearch   # just one service's logs
docker compose restart kibana       # restart a single service
docker compose down                 # stop everything, keep data volume
docker compose down -v              # stop everything AND delete data — full reset
```

## Teardown

Since this is meant for ephemeral workshop use:

```bash
docker compose down -v
```

This removes the containers and the `es-data` volume in one command — no orphaned state to hunt for afterward, unlike a manual install where you'd need to separately purge `/var/lib/elasticsearch`.

## Next step

Continue to the BOTES ingestion guide — the Logstash `.conf` files and Elasticsearch index template drop into `~/elk-docker/pipeline/`, and `docker compose restart logstash` picks them up. See the main [BOTS practice page](../BotS.md) for dataset tiering guidance (attack-only vs. full).
