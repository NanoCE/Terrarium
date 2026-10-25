# Phase A — MVP

**Status:** Pre-implementation  
**Last updated:** 2026-10-25  
**Depends on:** Nothing — this is the foundation  
**Unlocks:** Phase B (sensors), all subsequent phases  
**Walkthrough:** [PHASE-A-GUIDE.md](./PHASE-A-GUIDE.md)

---

## Scope

Phase A produces the minimum working system. No voice, no AI, no Thread hardware. Just the data foundation running correctly and proven — including proof that the data survives a reinstall and that a backup can actually be restored.

### In scope

- PostgreSQL running as a Docker container with a persistent volume
- Three database roles created at init: the `postgres` superuser (admin only), `hass` (owner of the `homeassistant` database), and `hass_readonly` (read-only, used by Grafana)
- An empty `terrarium` schema created at init, so Terrarium's own tables never share a schema with Home Assistant's recorder tables
- Home Assistant Container running via Docker Compose
- HA recorder writing state history to PostgreSQL from first boot (no SQLite at any point)
- Both Shelly Plug S 4th Gen units integrated via the local Shelly integration
- Grafana with a provisioned PostgreSQL datasource and a starter dashboard
- Backup script that dumps Postgres and archives HA's config directory — run manually, not yet scheduled
- A tested restore procedure

### Explicitly out of scope

- ALPSTUGA, Thread, OTBR, Matter server (Phase B)
- Voice pipeline (Phase C)
- Ollama, curation worker, HA observer, Alembic-managed tables (Phase D)
- Remote access, read API (Phase E)
- Presence and MQTT (Phase F)
- Backup scheduling and off-box targets

---

## Success Criteria

Phase A is complete when all of the following are true:

- [ ] `docker compose up -d` starts all three services (`postgres`, `homeassistant`, `grafana`) and `docker compose ps` shows them healthy
- [ ] HA is reachable at `http://<server-ip>:8123`
- [ ] HA recorder is writing to PostgreSQL — `states` has rows, and the HA logs show no recorder errors
- [ ] Both Shelly plugs appear in HA with live wattage readings
- [ ] Grafana is reachable at `http://<server-ip>:3000` and its PostgreSQL datasource test passes
- [ ] `hass_readonly` can `SELECT` from `states` (proves the default-privilege grants work) and cannot `INSERT`
- [ ] The MVP dashboard shows Shelly power data and recorder health without editing any query
- [ ] `scripts/backup.sh` produces a Postgres dump and an HA config archive
- [ ] **Restore test passes:** the dump restores into a throwaway Postgres container and its `states` row count matches
- [ ] **Reinstall test passes:** containers removed and the repo re-cloned; after setup and `up`, HA comes back with the same entities and users, and history continues unbroken
- [ ] **Reboot test passes:** after a server reboot, all three services come back on their own

---

## Implementation Order

Follow this order. Each step depends on the previous one.

1. **Shelly firmware** — update both plugs to current firmware before anything else (see Known Constraints)
2. **`.env.example` and `.gitignore`** — document every variable; ignore `.env`
3. **`db/init.sh`** — roles, database ownership, `terrarium` schema, read-only grants
4. **`docker-compose.yml`** — `postgres`, `homeassistant`, `grafana`, with pinned image tags
5. **`ha-config/configuration.yaml`** — template for the recorder and basic HA config
6. **`scripts/setup.sh`** — writes `.env`, creates the data directories, links `.env` into the repo, seeds HA config and `secrets.yaml`
7. **Grafana provisioning** — datasource YAML, dashboard provider YAML, dashboard JSON
8. **`scripts/backup.sh` and `scripts/restore-test.sh`**
9. **Verify** — every success criterion above

---

## Files Produced in Phase A

```
terrarium/
├── .env.example
├── .gitignore                       # ignores .env (the symlink) and any local artifacts
├── docker-compose.yml
├── db/
│   └── init.sh                      # .sh, not .sql — it reads passwords from the environment
├── ha-config/
│   └── configuration.yaml           # TEMPLATE — copied into the data root by setup.sh
├── grafana/
│   └── provisioning/
│       ├── datasources/
│       │   └── postgres.yaml
│       └── dashboards/
│           ├── dashboard.yaml
│           └── terrarium-mvp.json
└── scripts/
    ├── setup.sh
    ├── backup.sh
    └── restore-test.sh
```

---

## Where State Lives

This is the most important section of Phase A. The repo holds only templates and code. Everything HA, Postgres, or Grafana writes at runtime lives under one data root.

```
~/.config/terrarium/
└── .env                             # secrets + settings; the repo's .env is a symlink to this

~/.local/share/terrarium/            # TERRARIUM_DATA — always written as an absolute path in .env
├── postgres-data/                   # Postgres cluster
├── ha-config/                       # HA's /config: configuration.yaml, secrets.yaml, .storage/
└── grafana-data/                    # Grafana's internal DB (dashboards are provisioned, so this is low-value)
```

**What must survive a reinstall:**

| Path | Why it matters |
|---|---|
| `postgres-data/` | All recorded history and long-term statistics |
| `ha-config/.storage/` | HA's entity and device registries, users, auth, and integration entries. Lose this and every device gets re-added — often with new entity IDs (`_2` suffixes), which breaks dashboard queries and splits history |
| `ha-config/secrets.yaml` | Recorder connection string and any other HA secrets |

`grafana-data/` can be lost safely because the datasource and dashboard are provisioned from the repo.

**Why not mount `./ha-config` from the repo directly:** HA writes `.storage/`, logs, and its own database files into its config directory. Mounting the repo there would put runtime state and secrets inside the git working tree and make "re-clone the repo" destroy HA's registries.

**Why `TERRARIUM_DATA` must be absolute:** Docker Compose expands `~` relative to whichever user runs it. Run it once as root (via `sudo` or a system service) and every volume silently resolves under `/root` — Postgres starts empty, with no error.

---

## Services

### PostgreSQL

- Image: `postgres:17` — pin to a major version; a major-version upgrade requires a dump and restore
- Volume: `${TERRARIUM_DATA}/postgres-data` → `/var/lib/postgresql/data`
- Published port: `127.0.0.1:5432:5432` — loopback only, so HA (host network) can reach it and nothing off the box can
- Init: `db/init.sh` mounted into `/docker-entrypoint-initdb.d/`; runs only on the first start of an empty data directory
- Health check: `pg_isready -U postgres`
- Database: `homeassistant`, created by the image via `POSTGRES_DB`

**Roles and grants created by `db/init.sh`:**

```sql
-- hass: owns the database, so it owns the public schema (Postgres 15+) and can create recorder tables
CREATE ROLE hass WITH LOGIN PASSWORD :'hass_password';
ALTER DATABASE homeassistant OWNER TO hass;

-- Terrarium's own schema, empty until Phase D
CREATE SCHEMA terrarium AUTHORIZATION hass;

-- hass_readonly: SELECT only, now and for every table hass creates later
CREATE ROLE hass_readonly WITH LOGIN PASSWORD :'readonly_password';
GRANT CONNECT ON DATABASE homeassistant TO hass_readonly;
GRANT USAGE ON SCHEMA public, terrarium TO hass_readonly;
ALTER DEFAULT PRIVILEGES FOR ROLE hass IN SCHEMA public    GRANT SELECT ON TABLES TO hass_readonly;
ALTER DEFAULT PRIVILEGES FOR ROLE hass IN SCHEMA terrarium GRANT SELECT ON TABLES TO hass_readonly;
```

The `FOR ROLE hass` clause is required. Default privileges apply only to objects created by the named role, and HA's recorder creates its tables as `hass`. Without it, `hass_readonly` gets "permission denied" on every recorder table and Grafana shows nothing. A plain `GRANT SELECT ON ALL TABLES` at init time grants nothing, because no tables exist yet.

The passwords come from the environment (`HASS_DB_PASSWORD`, `HASS_READONLY_PASSWORD`) and are passed to `psql` as variables. No password is ever written into a file in the repo.

### Home Assistant Container

- Image: `ghcr.io/home-assistant/home-assistant` pinned to an exact release tag at implementation time — never `:stable` or `:latest`
- Config volume: `${TERRARIUM_DATA}/ha-config` → `/config`
- Network mode: `host` — required for mDNS discovery of the Shelly plugs (and later, Matter and Thread)
- Depends on: `postgres` with `condition: service_healthy`
- `TZ=UTC`
- Recorder connection string: `postgresql://hass:<password>@127.0.0.1:5432/homeassistant`, stored in `secrets.yaml` and referenced as `!secret recorder_db_url`

Note the `127.0.0.1`. A host-network container is not on the Compose network, so it cannot resolve the `postgres` service name.

### Grafana

- Image: `grafana/grafana` pinned to an exact release tag at implementation time
- Network: the Compose default bridge network; reaches Postgres at `postgres:5432` by service name
- Published port: `3000`
- Volumes: `./grafana/provisioning` → `/etc/grafana/provisioning` (read-only), `${TERRARIUM_DATA}/grafana-data` → `/var/lib/grafana`
- Admin credentials from `.env` (`GF_SECURITY_ADMIN_USER`, `GF_SECURITY_ADMIN_PASSWORD`)
- Datasource password injected through provisioning: `${HASS_READONLY_PASSWORD}`
- Default timezone for viewers: browser (data is stored in UTC)

All three services use `restart: unless-stopped`. With the Docker daemon enabled at boot (the default on Ubuntu), that is enough to bring the stack back after a reboot. No systemd unit is needed.

### Backup Script

- `scripts/backup.sh` produces two files in `$BACKUP_TARGET`:
  - `terrarium-db-YYYY-MM-DD.sql.gz` — via `docker compose exec -T postgres pg_dump -U hass homeassistant | gzip` (no Postgres client needed on the host)
  - `terrarium-ha-config-YYYY-MM-DD.tar.gz` — the HA config directory, including `.storage/` and `secrets.yaml`
- Rotation keeps the last 30 of each
- Exits with an error if `BACKUP_TARGET` is unset or not writable
- The archive contains secrets. `BACKUP_TARGET` must be a location you trust as much as the server itself
- Not scheduled in Phase A

### Restore Test

`scripts/restore-test.sh` starts a throwaway `postgres:17` container with no volume, loads the latest dump into it, compares the `states` row count with the live database, and removes the container. A backup counts as working only once this passes.

---

## HA Configuration Template

`ha-config/configuration.yaml` in the repo is copied into the data root once by `setup.sh`. After that, edits happen in the data-root copy.

```yaml
default_config:

recorder:
  db_url: !secret recorder_db_url
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

- `secrets.yaml` is generated by `setup.sh` in the data root, never in the repo
- `730` days of state history. Long-term statistics (hourly mean/min/max, and cumulative `sum` for energy) are kept indefinitely regardless of this setting
- `commit_interval: 5` — batches writes every 5 seconds

---

## Grafana Dashboard — MVP

### HA recorder schema notes

The dashboard queries HA's tables directly, so they have to match the current recorder schema:

- `states` has no `entity_id` column. Entities live in `states_meta`; join on `metadata_id`
- The timestamp is `last_updated_ts`, a Unix epoch in seconds (double precision). `last_updated` is no longer populated
- `state` is text. Cast it, and exclude `unavailable` and `unknown`
- Long-term statistics live in `statistics`, keyed by `metadata_id` → `statistics_meta.statistic_id` (the entity ID), timestamped by `start_ts`
- For energy sensors, `statistics.sum` is cumulative. A day's energy is the change in `sum` across the day, not a sum of rows

### No hardcoded entity IDs

A dashboard variable `plug_power` lists the Shelly power sensors by querying `states_meta`:

```sql
SELECT entity_id FROM states_meta
WHERE entity_id LIKE 'sensor.%' AND entity_id LIKE '%power'
ORDER BY entity_id;
```

Panels reference `$plug_power` instead of literal entity IDs, so the provisioned dashboard works on any install without patching the JSON. If the pattern catches unrelated sensors, narrow it to the Shelly naming prefix.

### Panels

| Panel | Type | Query basis |
|---|---|---|
| Current watts (one per plug, repeated by variable) | Stat | Latest non-null `state::numeric` per entity, by `last_updated_ts` |
| Power over time | Time series | `states` ⋈ `states_meta`, `to_timestamp(last_updated_ts)`, time-filtered |
| Daily energy — last 7 days | Bar chart | `statistics` ⋈ `statistics_meta`, per-day change in `sum` |
| Combined draw | Time series | Both plugs summed per time bucket |
| State changes today | Stat | `COUNT(*)` from `states` where `last_updated_ts >= extract(epoch from date_trunc('day', now()))` |
| Total rows in `states` | Stat | `COUNT(*)` from `states` |
| Most recent state change | Stat | `to_timestamp(MAX(last_updated_ts))` |

Example — power over time:

```sql
SELECT
  to_timestamp(s.last_updated_ts) AS time,
  sm.entity_id AS metric,
  s.state::numeric AS value
FROM states s
JOIN states_meta sm ON sm.metadata_id = s.metadata_id
WHERE sm.entity_id IN ($plug_power)
  AND s.state ~ '^-?[0-9.]+$'
  AND s.last_updated_ts BETWEEN $__unixEpochFrom() AND $__unixEpochTo()
ORDER BY 1;
```

---

## Environment Variables

All secrets and settings live in `~/.config/terrarium/.env`. The repo's `.env` is a symlink to it (gitignored), so Docker Compose finds it for `${VAR}` substitution and services load it with `env_file: .env`. `.env.example` documents every variable.

| Variable | Used by | Description |
|---|---|---|
| `TERRARIUM_DATA` | Compose volumes, scripts | Absolute path to the data root, e.g. `/home/<you>/.local/share/terrarium` |
| `POSTGRES_PASSWORD` | Postgres | Superuser password — admin use only; no service connects as `postgres` |
| `HASS_DB_PASSWORD` | init.sh, setup.sh | Password for `hass`; setup.sh writes it into HA's `secrets.yaml` |
| `HASS_READONLY_PASSWORD` | init.sh, Grafana | Password for `hass_readonly` |
| `GF_SECURITY_ADMIN_USER` | Grafana | Grafana admin username |
| `GF_SECURITY_ADMIN_PASSWORD` | Grafana | Grafana admin password |
| `BACKUP_TARGET` | backup.sh | Absolute path to the backup directory |
| `TZ` | All containers | `UTC` |
| `SERVER_IP` | setup.sh | The server's static LAN IP |

---

## Network

| Service | Network | Reaches Postgres at | Host exposure |
|---|---|---|---|
| `postgres` | Compose bridge | — | `127.0.0.1:5432` only |
| `homeassistant` | Host | `127.0.0.1:5432` | `8123` on the LAN |
| `grafana` | Compose bridge | `postgres:5432` | `3000` on the LAN |

Nothing is exposed beyond the LAN. Remote access is Phase E.

---

## Known Constraints and Gotchas

**Shelly stability.** One of the plugs has already had spontaneous reboots and been replaced once. Update both to current firmware before Phase A. Reboots show up as gaps and `unavailable` states in Grafana that look like recorder bugs. If they continue, note the timestamps — the recorder will capture them, which is useful evidence for a support ticket.

**Init only runs once.** `db/init.sh` runs only against an empty data directory. Changing roles later means running SQL by hand through `docker compose exec postgres psql -U postgres -d homeassistant`.

**Host network and service names.** HA cannot use Compose service names. Anything HA connects to on the Compose network must be published to the host and addressed as `127.0.0.1`.

**`~` in paths.** Never use `~` in `docker-compose.yml`. Use `${TERRARIUM_DATA}`.

**Compose project name.** Container names depend on the clone's directory name. Use `docker compose exec <service>` rather than `docker exec <container-name>` in scripts and docs.

**`.storage` is state, not config.** Back it up with the database. Never commit it.

---

## Definition of Done

1. Every success criterion is checked off
2. The restore, reinstall, and reboot tests have all passed on the real server
3. `README.md` marks Phase A complete
4. A git tag `phase-a` is applied to the commit
