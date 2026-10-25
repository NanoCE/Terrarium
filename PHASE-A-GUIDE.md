# Phase A — Implementation Guide

**Read first:** [PHASE-A.md](./PHASE-A.md) — scope, success criteria, where state lives, and definition of done  
**Last updated:** 2026-10-25  
**Time estimate:** 3–5 hours for a first Docker install, including the restore and reinstall tests  
**Difficulty:** Moderate — no prior Docker experience assumed  

This guide assumes the Phase A files (compose file, `db/init.sh`, scripts, Grafana provisioning) exist in the repo. If you're reading this before they're written, read it as the plan they're written against.

---

## Before You Start

Complete this checklist before running a single command.

- [ ] The server has a static LAN IP or a DHCP reservation on your router, and you know it
- [ ] Ubuntu is up to date (`sudo apt update && sudo apt upgrade -y`)
- [ ] You can SSH into the server
- [ ] The repo is cloned onto the server, in a directory you'll keep (this guide uses `~/terrarium`)
- [ ] Both Shelly plugs are on the same LAN and subnet as the server, and you know their IPs
- [ ] Both Shelly plugs are on current firmware (update through each plug's web page at `http://<plug-ip>` or the Shelly app)
- [ ] You've picked a `BACKUP_TARGET` directory for testing — any writable path on the server is fine for Phase A

---

## Step 1 — Install Docker

**Concept.** A container is an isolated process with its own filesystem and network, built from an *image*. Docker Compose starts several containers together from one file, `docker-compose.yml`, in the right order and with the right settings.

### Remove conflicting installs

Ubuntu can carry a snap-packaged Docker or old distro packages. Remove both before installing the official packages:

```bash
sudo snap remove docker 2>/dev/null
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt remove -y $pkg 2>/dev/null
done
```

### Install Docker Engine and the Compose plugin

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Run Docker as your user, and start it at boot

```bash
sudo usermod -aG docker $USER
sudo systemctl enable docker
```

Log out and back in for the group change to apply.

**Always run Terrarium's Docker commands as your own user, never with `sudo`.** Running Compose as root is the most common way to end up with a second, empty database (see PHASE-A.md → Where State Lives).

### Verify

```bash
docker --version          # Docker version 2x.x or later
docker compose version    # Docker Compose version v2.x or later
docker run --rm hello-world
```

---

## Step 2 — Run the Setup Script

```bash
cd ~/terrarium
./scripts/setup.sh
```

It asks for:

- The server's LAN IP
- A Postgres superuser password (admin only — you'll rarely use it)
- A password for `hass` (Home Assistant's database user)
- A password for `hass_readonly` (Grafana's user)
- A Grafana admin username and password
- Your `BACKUP_TARGET` path

Then it:

1. Writes `~/.config/terrarium/.env` with `TERRARIUM_DATA` set to an absolute path and `TZ=UTC`
2. Creates `postgres-data/`, `ha-config/`, and `grafana-data/` under the data root
3. Symlinks the repo's `.env` to `~/.config/terrarium/.env` (the symlink is gitignored)
4. Copies the repo's `ha-config/configuration.yaml` template into the data root — only if one isn't there already
5. Writes `secrets.yaml` into the data root's `ha-config/` with the recorder connection string

Check the result:

```bash
ls -la ~/terrarium/.env                     # should point at ~/.config/terrarium/.env
grep TERRARIUM_DATA ~/.config/terrarium/.env # should be an absolute path, no ~
ls ~/.local/share/terrarium/ha-config/      # configuration.yaml and secrets.yaml
```

Running `setup.sh` again later never overwrites existing data or HA config. It only fills in what's missing.

---

## Step 3 — Read the Compose File

Open `docker-compose.yml` before starting anything. What you're looking at:

**`services`** — one entry per container. Phase A has three: `postgres`, `homeassistant`, `grafana`.

**`image`** — the image and its tag. Tags are pinned to exact versions so a reinstall gets the same software you tested.

**`volumes`** — maps a host directory into the container, written `host_path:container_path`. This is how data outlives the container:

```yaml
volumes:
  - ${TERRARIUM_DATA}/postgres-data:/var/lib/postgresql/data
```

Removing the container doesn't touch the host directory.

**`${VAR}`** — substituted from the repo's `.env` file (your symlink) when Compose reads the file. That's why the symlink matters: `env_file:` only passes variables *into* a container; it doesn't make them available for substitution in the compose file itself.

**`depends_on` with `condition: service_healthy`** — HA waits until Postgres passes its health check before starting.

**Networks** — `postgres` and `grafana` share Compose's default bridge network, and Grafana reaches Postgres by its service name, `postgres`. HA is different: `network_mode: host` puts it directly on the server's network so it can discover devices over mDNS. That also means HA is *not* on the Compose network and can't resolve `postgres`. It connects to `127.0.0.1:5432` instead, which works because Postgres publishes its port on the loopback interface only.

**`restart: unless-stopped`** — Docker restarts the container after a crash or a reboot, unless you stopped it yourself.

---

## Step 4 — Start the Stack

### First run: attached

```bash
cd ~/terrarium
docker compose up
```

Watch for, in order:

1. Image downloads (first run only)
2. Postgres running `init.sh`, then `database system is ready to accept connections`
3. HA starting once Postgres is healthy — look for `Home Assistant initialized`
4. Grafana: `HTTP Server Listen` on port 3000

Any line with `ERROR` or `FATAL` from Postgres or the recorder: stop with `Ctrl+C` and go to Troubleshooting.

### Then: detached

```bash
docker compose up -d
docker compose ps        # all three should show "running" (postgres "healthy")
```

### Commands you'll use constantly

```bash
docker compose ps                         # status
docker compose logs -f                    # all logs, live
docker compose logs -f homeassistant      # one service
docker compose restart homeassistant      # after an HA config change
docker compose exec postgres psql -U hass -d homeassistant   # SQL shell as hass
docker compose down                       # stop and remove containers (data stays)
```

`docker compose down -v` removes *named* volumes. Phase A uses bind mounts, so it won't delete your data — but don't get in the habit.

---

## Step 5 — Set Up Home Assistant

Open `http://<server-ip>:8123` and create your HA account. Set your home location when asked; leave the timezone as is.

### Confirm the recorder is writing to Postgres

After HA has run for a minute:

```bash
docker compose exec postgres psql -U hass -d homeassistant -c \
  "SELECT count(*) FROM states;"

docker compose exec postgres psql -U hass -d homeassistant -c \
  "SELECT sm.entity_id, s.state, to_timestamp(s.last_updated_ts)
   FROM states s JOIN states_meta sm USING (metadata_id)
   ORDER BY s.last_updated_ts DESC LIMIT 5;"
```

The first should be above zero; the second shows the latest state changes with their entity IDs. If `states` is empty after a few minutes:

```bash
docker compose logs homeassistant | grep -iE "recorder|database|error"
```

### Confirm Grafana's read-only access works

```bash
docker compose exec postgres psql -U hass_readonly -d homeassistant -c \
  "SELECT count(*) FROM states;"
# should succeed

docker compose exec postgres psql -U hass_readonly -d homeassistant -c \
  "INSERT INTO states (state) VALUES ('test');"
# should fail with "permission denied"
```

If the `SELECT` fails with "permission denied", the default-privilege grants in `init.sh` are wrong — see Troubleshooting.

---

## Step 6 — Add the Shelly Plugs

Go to **Settings → Devices & Services**. If the plugs appear under *Discovered*, add them. If not: **Add Integration → Shelly**, enter each plug's IP.

Open each device and confirm you see:

- `sensor.<name>_power` — live watts
- `sensor.<name>_energy` — cumulative kWh
- `switch.<name>` — on/off

Plug a load into each to confirm the wattage moves.

**Rename devices now, before any history matters.** Entity IDs come from device names. Give each plug a stable, meaningful name (e.g. "Desk Plug", "Server Plug") and let HA update the entity IDs. Renaming later splits history across the old and new IDs.

---

## Step 7 — Check Grafana

Open `http://<server-ip>:3000` and log in with the admin credentials from setup.

1. **Connections → Data sources → PostgreSQL → Save & test** — expect "Database Connection OK"
2. **Dashboards → Terrarium MVP** — the plug selector at the top should list both power sensors, and every panel should populate without edits

If the selector also picks up non-Shelly power sensors, narrow the variable's query to your plugs' naming prefix and save. Energy-per-day bars appear once HA has compiled a few hours of statistics; the rest is immediate.

---

## Step 8 — Backup and Restore Test

```bash
./scripts/backup.sh
ls -lh "$(grep BACKUP_TARGET ~/.config/terrarium/.env | cut -d= -f2)"
```

You should see a `terrarium-db-<date>.sql.gz` and a `terrarium-ha-config-<date>.tar.gz`.

Now prove the dump is usable:

```bash
./scripts/restore-test.sh
```

It loads the newest dump into a temporary Postgres container, compares `states` row counts with the live database, prints PASS or FAIL, and cleans up. The counts won't match exactly — the live database keeps growing — so the script allows the restored count to be at most slightly lower.

Peek inside the HA archive to confirm `.storage/` is there:

```bash
tar -tzf <BACKUP_TARGET>/terrarium-ha-config-<date>.tar.gz | grep -m3 ".storage"
```

The archive contains `secrets.yaml`. Treat the backup directory like the server itself.

---

## Step 9 — Reinstall Test

This proves the repo really is disposable and the data root really holds everything that matters.

```bash
# Record a baseline
docker compose exec postgres psql -U hass -d homeassistant -tc "SELECT count(*) FROM states;"

# Tear down and throw the clone away
docker compose down
cd ~
mv terrarium terrarium.old

# Fresh clone, setup, start
git clone <your-repo-url> terrarium
cd terrarium
./scripts/setup.sh        # should detect existing .env and data, and change nothing
docker compose up -d
```

Then check:

- [ ] `states` row count is equal to or higher than the baseline
- [ ] You can log into HA with your existing account (no onboarding screen)
- [ ] Both Shelly plugs are present with the same entity IDs
- [ ] The Grafana power graph is continuous across the restart, apart from the minutes it was down

When it passes: `rm -rf ~/terrarium.old`.

---

## Step 10 — Reboot Test

```bash
sudo reboot
```

After it comes back (give it a couple of minutes):

```bash
cd ~/terrarium
docker compose ps
```

All three services should be running without you starting them. That's `restart: unless-stopped` plus Docker starting at boot — no systemd unit required.

---

## Troubleshooting

### Postgres won't start

```bash
docker compose logs postgres
```

- **Port 5432 in use** — a host Postgres is running. `sudo systemctl disable --now postgresql`
- **Permission errors on the data directory** — the directory exists but is owned by someone else, usually root from a `sudo` run. Check `ls -ld $TERRARIUM_DATA/postgres-data`

### Postgres starts but is empty, or HA shows onboarding again

Compose was almost certainly run as a different user, or `TERRARIUM_DATA` changed. Check:

```bash
grep TERRARIUM_DATA ~/.config/terrarium/.env
docker compose config | grep -A2 volumes
sudo ls /root/.local/share/terrarium 2>/dev/null   # if this exists, a root run happened
```

Stop the stack, fix the path, and start again as your own user. Your real data is still in the original directory.

### HA can't reach Postgres

```bash
docker compose logs homeassistant | grep -iE "recorder|database|error"
```

- `secrets.yaml` must use `127.0.0.1`, not `postgres`, as the host
- The `hass` password in `secrets.yaml` must match `HASS_DB_PASSWORD`
- If HA started before Postgres was ready: `docker compose restart homeassistant`

### Grafana: "permission denied for table states"

The default privileges weren't set for the `hass` role. Fix in place:

```bash
docker compose exec postgres psql -U postgres -d homeassistant -c "
  ALTER DEFAULT PRIVILEGES FOR ROLE hass IN SCHEMA public GRANT SELECT ON TABLES TO hass_readonly;
  GRANT SELECT ON ALL TABLES IN SCHEMA public TO hass_readonly;"
```

Then fix `db/init.sh` so a fresh install doesn't repeat it.

### Grafana panels empty but the datasource test passes

- Confirm the plug selector lists your sensors
- The queries use `last_updated_ts` and `states_meta`. A query written against `last_updated` or `states.entity_id` will return nothing on current HA versions

### Shelly plugs not discovered

- Same subnet/VLAN as the server?
- HA running with `network_mode: host`?
- Add manually by IP

### A plug keeps going unavailable

Check its uptime on the plug's web page. If it's rebooting on its own, update the firmware and note the times from Grafana for Shelly support.

---

## Phase A Complete

Check every box in [PHASE-A.md → Success Criteria](./PHASE-A.md#success-criteria), then:

```bash
git tag phase-a
git push origin phase-a
```

Mark Phase A done in `README.md` and commit.
