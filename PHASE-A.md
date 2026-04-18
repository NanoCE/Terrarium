# Phase A — MVP

**Status:** Pre-implementation  
**Depends on:** Nothing — this is the foundation  
**Unlocks:** Phase B (sensors), all subsequent phases  

---

## Scope

Phase A produces the minimum working system. No voice, no AI, no Thread hardware. Just the data foundation running correctly and proven.

### In scope

- PostgreSQL running as a Docker container with persistent volume
- `hass` read/write role and `hass_readonly` read-only role created at init
- Home Assistant Container running via Docker Compose
- HA recorder writing state history to PostgreSQL (not SQLite)
- Both Shelly Plug S 4th Gen units integrated via local Shelly integration
- Grafana running with a provisioned PostgreSQL datasource and starter dashboard
- Backup script exists, is documented, target is configurable via `.env` — not yet scheduled

### Explicitly out of scope

- ALPSTUGA / Thread / OTBR / Matter server (Phase B)
- Zigbee dongle (Phase C)
- Voice pipeline — Whisper, Piper, Wyoming satellites (Phase C)
- Intent router, curation worker, Ollama (Phase D)
- Remote access, VPN, read API (Phase E)
- Alembic migrations — custom Terrarium tables are not created yet; only HA recorder tables exist at this stage

---

## Success Criteria

Phase A is complete when all of the following are true:

- [ ] `docker compose up` starts all four services with no errors
- [ ] HA is reachable at `http://<server-ip>:8123`
- [ ] HA recorder is writing to PostgreSQL — confirmed by querying the `states` table directly
- [ ] Both Shelly plugs appear as entities in HA with live wattage readings
- [ ] Grafana is reachable at `http://<server-ip>:3000`
- [ ] Grafana PostgreSQL datasource connects successfully (green checkmark on test)
- [ ] Grafana MVP dashboard loads and displays Shelly power data
- [ ] `scripts/backup.sh` runs without error and produces a `.sql.gz` file
- [ ] Reinstall test passes: bring stack down, delete HA config volume, bring back up, recorder resumes writing to same Postgres data

---

## Implementation Order

Follow this order exactly. Each step depends on the previous one.

1. **`.env` and directory structure** — establish config paths, write `.env.example`
2. **`db/init.sql`** — create roles, set permissions; runs automatically on first Postgres start
3. **`docker-compose.yml`** — Postgres, HA, Grafana services only
4. **`ha-config/configuration.yaml`** — recorder pointed at Postgres, exclusions, retention
5. **`scripts/setup.sh`** — interactive first-run script; writes `.env`, validates prereqs
6. **Grafana provisioning** — datasource YAML, dashboard YAML, dashboard JSON
7. **`scripts/backup.sh`** — `pg_dump` wrapper, compress, rotate
8. **Verify** — run through all success criteria

---

## Files Produced in Phase A

```
terrarium/
├── .env.example
├── docker-compose.yml
├── db/
│   └── init.sql
├── ha-config/
│   └── configuration.yaml
├── grafana/
│   └── provisioning/
│       ├── datasources/
│       │   └── postgres.yaml
│       └── dashboards/
│           ├── dashboard.yaml
│           └── terrarium-mvp.json
└── scripts/
    ├── setup.sh
    └── backup.sh
```

---

## Services

### PostgreSQL

- Image: `postgres:16`
- Volume: `~/.local/share/terrarium/postgres-data/` — the only stateful artifact that must survive a reinstall
- Exposed port: `5432` on localhost only — not accessible from outside the host
- Init: `db/init.sql` runs automatically on first container start via the `docker-entrypoint-initdb.d` mechanism
- Database name: `homeassistant`
- Roles created at init:
  - `hass` — read/write owner of the `homeassistant` database
  - `hass_readonly` — `SELECT` only on all tables; used by Grafana

### Home Assistant Container

- Image: `ghcr.io/home-assistant/home-assistant:stable`
- Config volume: `./ha-config/` mapped to `/config` inside the container
- Network mode: `host` — required for HA device discovery (mDNS, Shelly local API)
- Depends on: `postgres` (health check)
- Recorder configured in `ha-config/configuration.yaml` to write to Postgres from first boot — avoids SQLite entirely

### Grafana

- Image: `grafana/grafana:latest`
- Provisioning volume: `./grafana/provisioning/` mapped to `/etc/grafana/provisioning/`
- Data volume: `~/.local/share/terrarium/grafana-data/`
- Port: `3000`
- Default credentials: set via `.env` (`GF_SECURITY_ADMIN_USER`, `GF_SECURITY_ADMIN_PASSWORD`)
- Datasource and dashboard loaded automatically on first start — no manual UI setup required

### Backup Script

- Location: `scripts/backup.sh`
- Produces: compressed `pg_dump` at `$BACKUP_TARGET/terrarium-YYYY-MM-DD.sql.gz`
- Rotation: keeps last 30 dumps, deletes older ones
- Target: configured via `BACKUP_TARGET` in `.env` — any writable local path
- Not scheduled in Phase A — run manually to verify, wire up cron in a later phase

---

## HA Recorder Configuration

Key decisions in `ha-config/configuration.yaml`:

```yaml
recorder:
  db_url: !env_var HA_DB_URL
  purge_keep_days: 730
  commit_interval: 5
  exclude:
    domains:
      - update
    entity_globs:
      - sensor.*_linkquality
      - sensor.*_lqi
      - sensor.*_rssi
```

- `HA_DB_URL` is injected from `.env` — no credentials in the config file
- `730` days retention — two years of history from day one
- `commit_interval: 5` — writes to Postgres every 5 seconds, reducing I/O without losing data
- Exclusions remove high-frequency noise that doesn't have analytical value

---

## Grafana Dashboard — MVP

One dashboard, two sections.

### Shelly Power Section

| Panel | Type | Query basis |
|---|---|---|
| Plug 1 — current watts | Stat | Latest `state` for Shelly plug 1 power entity |
| Plug 2 — current watts | Stat | Latest `state` for Shelly plug 2 power entity |
| Power over time — both plugs | Time series | `states` joined to `states_meta` by entity_id, last 24h |
| Daily energy totals — last 7 days | Bar chart | `statistics` table, daily sum per plug |
| Combined household draw | Time series | Sum of both plugs, last 24h |

### Recorder Health Section

| Panel | Type | Query basis |
|---|---|---|
| State changes today | Stat | `COUNT(*)` from `states` where `last_updated > today` |
| Total rows in states table | Stat | `COUNT(*)` from `states` |
| Most recent state change | Stat | `MAX(last_updated)` from `states` |

---

## Environment Variables

All secrets and environment-specific config live in `~/.config/terrarium/.env`. Never committed to the repo. `.env.example` in the repo root documents every variable with a description and safe default.

| Variable | Used by | Description |
|---|---|---|
| `POSTGRES_PASSWORD` | Postgres, HA, Grafana | Password for the `hass` user |
| `POSTGRES_READONLY_PASSWORD` | Grafana | Password for the `hass_readonly` user |
| `HA_DB_URL` | HA recorder | Full Postgres connection string for HA |
| `GF_SECURITY_ADMIN_USER` | Grafana | Grafana admin username |
| `GF_SECURITY_ADMIN_PASSWORD` | Grafana | Grafana admin password |
| `BACKUP_TARGET` | backup.sh | Absolute path to backup destination directory |
| `TZ` | All containers | Timezone — e.g. `America/Los_Angeles` |
| `SERVER_IP` | setup.sh | LAN IP of the server |

---

## Network

All services communicate over a single Docker bridge network (`terrarium`). No services are exposed to the internet. Ports exposed to the host:

| Service | Host port | Purpose |
|---|---|---|
| `homeassistant` | host network | Required for mDNS / Shelly discovery |
| `grafana` | `3000` | Dashboard UI |
| `postgres` | `5432` (localhost only) | Direct DB access for debugging |

---

## Known Constraints and Gotchas

**HA must use host network mode.** Without it, HA cannot perform mDNS discovery to find Shelly devices. This is a known requirement for HA Container on Linux.

**Postgres init script only runs on first start.** If you need to re-run `db/init.sql` (e.g. to add a role), you must either delete the data volume and restart, or run the SQL manually via `docker exec`.

**Grafana entity IDs in queries are environment-specific.** The dashboard JSON uses Shelly entity IDs that will be specific to your HA instance. The setup script should prompt for these and patch the dashboard JSON at install time, or the dashboard should use HA-agnostic queries where possible. This is a known rough edge for Phase A.

**SQLite migration is not needed.** Because HA is pointed at Postgres from first boot, there is no SQLite data to migrate. If HA has previously been run on this machine with SQLite, the old database is left untouched and ignored.

---

## Definition of Done

Phase A is done when:

1. All success criteria are checked off
2. A reinstall test has been performed and passed
3. `scripts/backup.sh` has been run manually and output verified
4. Both files — `DESIGN.md` and `README.md` — reflect Phase A as complete
5. A git tag `phase-a` is applied to the commit
