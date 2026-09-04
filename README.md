# TheHive Home Lab 🐝

> TheHive 5 + Cortex 4 SOAR/case-management stack deployed with Docker Compose for home lab and learning purposes. Sibling repo to [wazuh-homelab](https://github.com/D4rkDr4gon/wazuh-homelab) — Wazuh forwards its alerts here.

## What is this?

- **TheHive** — Security Incident Response Platform (SIRP). Case management, alerts, tasks, observables. Alerts land here from Wazuh via a custom integration and get triaged/escalated into cases.
- **Cortex** — Observable analysis and active response engine. Runs analyzers (VirusTotal, etc.) and responders against observables attached to TheHive cases/alerts.

Together they form a lightweight SOAR: SIEM (Wazuh) detects → alert forwarded to TheHive → analyst triages/escalates to case → Cortex enriches observables.

## Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│                          Docker Host                              │
│                                                                   │
│  ┌────────────┐   ┌────────────┐                                  │
│  │   nginx    │──▶│  thehive   │──┐                               │
│  │  (reverse  │   │   :9000    │  │                               │
│  │   proxy)   │   │  5.7.1     │  │   ┌──────────────┐            │
│  │  :8444     │   └────────────┘  ├──▶│ elasticsearch│            │
│  │   (HTTPS)  │   ┌────────────┐  │   │    :9200     │            │
│  │            │──▶│   cortex   │──┘   │   8.19.15    │            │
│  └────────────┘   │   :9001    │      └──────────────┘            │
│                    │   4.0.1    │                                 │
│                    └────────────┘      ┌──────────────┐           │
│                    ┌────────────┐      │  cassandra   │           │
│                    │  thehive   │─────▶│   4.1.11     │           │
│                    │            │      └──────────────┘           │
│                    └────────────┘                                 │
│                                                                   │
│                    thehive-cortex-network (bridge)                │
└───────────────────────────────────────────────────────────────────┘
```

| Service | Container port | Published port | Description |
|---------|-----------------|-----------------|-------------|
| **thehive** | 9000 | 9000 (also proxied via nginx `/thehive`) | Case management app — alerts, cases, tasks, observables |
| **cortex** | 9001 | 9001 (also proxied via nginx `/cortex`) | Analyzer/responder engine — runs jobs as sibling Docker containers (mounts `docker.sock`) |
| **cassandra** | 9042 | — (internal only) | JanusGraph storage backend for TheHive (graph DB) |
| **elasticsearch** | 9200 | — (internal only) | Full-text index for both TheHive and Cortex |
| **nginx** | 443 | 8444 (HTTPS) | TLS-terminating reverse proxy in front of `thehive`/`cortex` |

All services communicate over a single Docker bridge network, `thehive-cortex-network`.

## Prerequisites

- **Docker Engine** (not Docker Desktop) with the Compose plugin
- **At least 6 GB RAM** allocated to Docker (Cassandra + Elasticsearch + TheHive + Cortex each reserve up to 2 GB)
- `openssl` (for self-signed certs, if not providing your own)

## Deploy

### 1. Clone the repository

```bash
git clone https://github.com/D4rkDr4gon/thehive-homelab.git
cd thehive-homelab
```

### 2. Initialize the environment

The `scripts/init.sh` helper generates a fresh `.env`, a random ElasticSearch password, random Play `secret.key` values for both TheHive and Cortex, and self-signed (or custom) TLS certs — all in one shot:

```bash
bash scripts/init.sh
```

This will:
- Create `thehive/config/index.conf` and `cortex/config/index.conf` from their `.template` counterparts, injecting a random ElasticSearch password
- Create `thehive/config/secret.conf` and `cortex/config/secret.conf` with random `play.http.secret.key` values (won't overwrite if they already exist)
- Create `.env` from `dot.env.template` + `versions.env`, plus `UID`/`GID`/`nginx_server_name`
- Prompt for a hostname and generate a self-signed cert in `nginx/certs/` (or use custom certs dropped into `certificates/` — see `scripts/generate_certs.sh`)

If you'd rather do it manually, copy `.env.example` to `.env` and fill in the placeholders, then copy each `*.conf.template` to its real `.conf` name and set the values yourself.

### 3. Start the stack

```bash
docker compose up -d
```

Cassandra takes the longest to become healthy (up to ~2 min). `thehive` and `cortex` wait on `elasticsearch`/`cassandra` healthchecks via `depends_on`.

### 4. Access the UI

- **TheHive**: https://localhost:8444/thehive (or `http://localhost:9000/thehive` direct, bypassing nginx)
- **Cortex**: https://localhost:8444/cortex (or `http://localhost:9001/cortex` direct)

> ⚠️ Self-signed certificate warning is expected unless you dropped your own CA-signed cert into `certificates/`.

First-run setup (create the initial organisation/admin user, link Cortex to TheHive, etc.) follows the official StrangeBee docs:
- [Deploy TheHive with Docker Compose](https://docs.strangebee.com/thehive/installation/docker/)
- [Run Cortex with Docker](https://docs.strangebee.com/cortex/installation-and-configuration/run-cortex-with-docker/)

### Other scripts

| Script | Purpose |
|--------|---------|
| `scripts/init.sh` | First-time setup — generates secrets, certs, `.env` |
| `scripts/check_permissions.sh` | Verifies bind-mount ownership matches `UID`/`GID` before init |
| `scripts/generate_certs.sh` | Generates self-signed certs or picks up custom ones from `certificates/` (sourced by `init.sh`) |
| `scripts/backup.sh` / `scripts/restore.sh` | Backup/restore Cassandra + Elasticsearch + TheHive file storage |
| `scripts/reset.sh` | Wipes generated secrets (`secret.conf`) and index config so you can re-run `init.sh` clean |
| `scripts/test_init_*.sh` | Smoke tests against a fresh install (create org, users, custom fields, link Cortex) — uses well-known StrangeBee testing defaults, **not** this lab's real credentials |

## Integrations

### Wazuh

This lab receives alerts from [wazuh-homelab](https://github.com/D4rkDr4gon/wazuh-homelab) via a custom Wazuh integration (`custom-thehive`). Alerts with level **≥ 3** are converted into TheHive alerts, which can then be escalated into cases.

- **Service account in TheHive**: `wazuh-final@thehive.local`
- **Profile**: `analyst`
- **Auth**: API key, configured on the Wazuh side via `THEHIVE_URL`/`THEHIVE_API_KEY` env vars

> **⚠️ Known limitation:** TheHive 5's `testing` Docker profile (the one this stack labels its services with, see `com.strangebee.environment: "testing"` in `docker-compose.yml`) has a bug where Organisation-type profiles (`analyst`, `org-admin`) return correct permissions on read queries but reject mutation operations (create alert/case) via the REST API. Workaround: use the `prod1-thehive` profile instead of `analyst`/`org-admin` when creating the service user. See `MAN-001244` in the wazuh-homelab notes for the full writeup.

### Cortex analyzers

Cortex ships with the ability to pull public analyzers/responders from `https://download.thehive-project.org/{analyzers,responders}.json`, or load private ones dropped into `cortex/neurons/{analyzers,responders}`. None are pre-configured in this lab — add API keys for individual analyzers (e.g. VirusTotal) through the Cortex UI, they're stored in Cortex's own Elasticsearch index, not in this repo.

## Secrets management

This repo is **public**. Nothing with a real secret in it is tracked:

| What | Where it lives | Tracked? |
|------|-----------------|----------|
| ElasticSearch password, image versions, nginx hostname | `.env` | ❌ gitignored — see `.env.example` / `dot.env.template` |
| TheHive/Cortex `play.http.secret.key` | `thehive/config/secret.conf`, `cortex/config/secret.conf` | ❌ gitignored — see `*.secret.conf.template` |
| ElasticSearch password baked into TheHive/Cortex HOCON config | `thehive/config/index.conf`, `cortex/config/index.conf` | ❌ gitignored (generated by `init.sh` from `*.index.conf.template`, which **is** tracked) |
| TLS cert/key | `nginx/certs/`, `certificates/` | ❌ gitignored |
| Cassandra/Elasticsearch/TheHive data + logs | `*/data/`, `*/logs/` | ❌ gitignored (bind-mounted volumes) |
| Cortex analyzer API keys (e.g. VirusTotal) | Cortex's own internal Elasticsearch index (set via UI) | not a file in this repo at all |

All `*.template` files (`dot.env.template`, `thehive/config/*.conf.template`, `cortex/config/*.conf.template`) contain only placeholders and are safe to commit. `scripts/test_init_*.sh` use hardcoded credentials (`admin@thehive.local:secret`, `thehive1234`) — these are StrangeBee's well-known upstream testing defaults for a throwaway first-run smoke test, not this lab's real secrets.

If you fork this repo, run `bash scripts/init.sh` to generate your own secrets before starting the stack — never reuse the values that may appear in someone else's screenshots/logs.

## Troubleshooting

### Cassandra takes forever / thehive stays "starting"

Cassandra's healthcheck has a 120s `start_period`. `thehive` won't even attempt to start until Cassandra and Elasticsearch both report healthy (`depends_on: condition: service_healthy`). Check with:

```bash
docker ps --filter "name=cassandra"
docker logs cassandra --tail 50
```

### `thehive` / `cortex` unhealthy

```bash
docker logs thehive --tail 100
docker logs cortex --tail 100

# Check status endpoint directly
docker exec thehive curl -s -f http://thehive:9000/thehive/api/status
docker exec cortex curl -s -f http://cortex:9001/cortex/api/status
```

Common causes: `index.conf`/`secret.conf` missing (re-run `scripts/init.sh` or `scripts/check_permissions.sh`), or the ElasticSearch password in `.env` doesn't match the one baked into `thehive/config/index.conf` / `cortex/config/index.conf` (they must all be generated together by `init.sh`, not mixed and matched by hand).

### Permission errors on bind mounts

`UID`/`GID` in `.env` must match the host user that owns `cassandra/data`, `elasticsearch/data`, `thehive/data`, etc. (the containers run as that user, not root). Run:

```bash
bash scripts/check_permissions.sh
```

### Reset everything and start clean

```bash
bash scripts/reset.sh   # removes generated secret.conf / index.conf
bash scripts/init.sh    # regenerates them
```

> This does **not** wipe `*/data/` — remove those directories manually if you want a fully clean Cassandra/Elasticsearch state.

## Project Structure

```
thehive/
├── docker-compose.yml
├── .env                        # Real secrets (gitignored)
├── .env.example                # Placeholder template
├── dot.env.template            # Upstream StrangeBee template (used by scripts/init.sh)
├── versions.env                # Pinned image versions
├── thehive/config/              # application.conf, index.conf(.template), secret.conf(.template), logback.xml
├── cortex/config/               # same pattern as thehive/config/
├── cortex/neurons/              # optional private analyzers/responders
├── nginx/templates/             # default.conf.template (nginx reverse proxy config)
├── nginx/certs/                 # TLS cert/key (gitignored, generated)
├── certificates/                # drop custom TLS cert/key/CA here (gitignored)
└── scripts/                     # init, backup, restore, reset, cert generation, smoke tests
```

## License

This project is for educational purposes. TheHive and Cortex are licensed under AGPL-3.0 by StrangeBee.
