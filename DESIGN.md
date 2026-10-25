# Terrarium — System Design Document

**Status:** Pre-implementation draft  
**Last updated:** 2026-10-25  
**Wake word:** "Hey Terra"  
**Repo:** terrarium  

---

## Table of Contents

1. [Design Principles](#1-design-principles)
2. [System Overview](#2-system-overview)
3. [Hardware](#3-hardware)
4. [Infrastructure](#4-infrastructure)
5. [Home Assistant](#5-home-assistant)
6. [Data Layer](#6-data-layer)
7. [Intent Router](#7-intent-router)
8. [Voice Pipeline](#8-voice-pipeline)
9. [Presence Detection](#9-presence-detection)
10. [Notification Worker](#10-notification-worker)
11. [AI Layer](#11-ai-layer)
12. [Lookup Table and Curation](#12-lookup-table-and-curation)
13. [Automation System](#13-automation-system)
14. [Explainability and Audit](#14-explainability-and-audit)
15. [Remote Access](#15-remote-access)
16. [Dashboard](#16-dashboard)
17. [Database Schema](#17-database-schema)
18. [Repository Structure](#18-repository-structure)
19. [Capability Tiers](#19-capability-tiers)
20. [Implementation Phases](#20-implementation-phases)
21. [Open Questions](#21-open-questions)

---

## 1. Design Principles

These are constraints, not goals. Every architectural decision is evaluated against all three.

### 1.1 Automation should be effortless

The system reduces friction to near zero for both capturing and acting on data. A user speaks a command, gets a fast response, and the system learns from that interaction. Automation setup is initiated by the user, never silently inferred. The LLM may propose automations based on explicit user requests or voice patterns, but always through an approval queue — it never commits anything without confirmation.

### 1.2 Data is fast, transparent, and integrable

Every action, response, and AI decision is logged with enough context to explain why it happened — days or weeks later. All state lives in structured PostgreSQL storage, queryable by any tool. There are no black boxes. The system can answer its own audit questions via the same voice interface used for everything else.

### 1.3 Local-first, cloud-never (for data)

All LLM inference runs locally via Ollama. All speech processing runs locally via Whisper and Piper. All data is stored locally in PostgreSQL. No data transits a cloud service in plaintext. Remote access is provided by a VPN tunnel — the network perimeter, not the application, handles external security. Cloud backup is opt-in and encrypted at rest before leaving the LAN.

---

## 2. System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        HARDWARE LAYER                           │
│                                                                 │
│  Voice satellite         Smart Devices         ALPSTUGA         │
│  (wake word + mic        (Shelly Plug S,        (Thread/Matter   │
│   + speaker)              Zigbee later)          air quality)    │
└────────────┬────────────────────┬──────────────────┬────────────┘
             │ ESPHome API        │ WiFi / Zigbee    │ Thread
             ▼                    ▼                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    HOME ASSISTANT CONTAINER                      │
│                                                                 │
│  Whisper (Wyoming)  Piper (Wyoming)  Assist Pipeline  Automations│
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    INTENT ROUTER                         │  │
│  │  (1) Lookup hit → template fill → HA state → respond    │  │
│  │  (2) HA local intent → respond                          │  │
│  │  (3) LLM fallback → Ollama → respond                    │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
             ┌───────────────┼───────────────┐
             ▼               ▼               ▼
        PostgreSQL        Ollama         Read API
        (all data)     (local LLM)    (FastAPI RO)
             │
    ┌────────┴────────┐
    │  HA Observer     │  (persists automation traces + context)
    │  Curation Worker │  (LLM curates lookup table, proposes automations)
    │  Notif Worker    │
    │  Presence Worker │
    └─────────────────┘
```

---

## 3. Hardware

### 3.1 Server

| Component | Spec |
|---|---|
| CPU | AMD Ryzen 5 7600X (6-core, 12-thread) |
| RAM | 32GB DDR5-6000 |
| Storage | 1TB Samsung 870 EVO SATA SSD |
| OS | Ubuntu (latest LTS) |
| GPU (future) | NVIDIA RTX 3070 Ti (8GB VRAM) — migrated from gaming rig |

**Ollama model sizing** (specific models are chosen in Phase D — local model releases move too fast to pin here):

| Hardware | Role | Size class | Notes |
|---|---|---|---|
| CPU-only (now) | Voice fallback | ~3–4B, Q4 | Small enough to answer within a few seconds on the 7600X |
| CPU-only (now) | Curation worker | ~7–9B, Q4 | Background job, no latency constraint |
| With 3070 Ti (8GB VRAM) | Both roles | ~7–9B, Q4 | Fits fully in VRAM; several times faster than CPU |

### 3.2 Voice Devices

| Device | Role | Notes |
|---|---|---|
| Home Assistant Voice Preview Edition | Primary room satellite | Purpose-built for HA Assist; ESPHome-based; good mics and speaker; first choice |
| ESP32-S3-BOX-3 | Alternate room satellite | ESPHome-based; weaker audio than Voice PE |
| ATOM Echo (M5Stack) | Command-only rooms | No usable speaker |
| Raspberry Pi + mic/speaker | Flexible satellite | Runs the OHF `linux-voice-assistant` project; most audio flexibility |

**Protocol:** ESPHome devices talk to HA over the ESPHome native API, not Wyoming. Wyoming is how HA talks to the speech services (Whisper, Piper). Wake word detection runs on the device, so no audio leaves it until triggered.

**Wake word:** There is no pretrained "Hey Terra" model. On-device detection uses microWakeWord, which ships with a few stock models (`okay_nabu`, `hey_jarvis`, and others). Terrarium ships with `okay_nabu` as the default and treats "Hey Terra" as a custom-trained microWakeWord model, added in Phase C once enough samples are recorded. The wake word is configurable per device.

### 3.3 Smart Devices — Phase A

| Device | Protocol | Purpose |
|---|---|---|
| Shelly Plug S 4th Gen (×2) | WiFi (local RPC API) | Per-device power and energy monitoring |

The Shelly plugs report wattage, voltage, and cumulative energy through HA's local Shelly integration — no cloud. Update both to current firmware before Phase A; one unit has already needed a replacement for spontaneous reboots.

### 3.4 Smart Devices — Phase B

| Device | Protocol | Purpose |
|---|---|---|
| IKEA ALPSTUGA | Matter over Thread | CO₂, PM 2.5, temperature, humidity |
| SONOFF Zigbee 3.0 Dongle Lite (EFR32MG21) — owned, unflashed | Thread RCP | Radio for the OpenThread Border Router — flashed to OpenThread RCP firmware |
| Shelly BLU Door/Window (candidate) | Bluetooth LE via a Shelly Gen4 plug as gateway | Apartment door open/close events — pending confirmation that the Plug S Gen4 works as a BLU gateway in HA |

**Thread in HA Container.** Thread and Matter need three pieces running together: the SONOFF dongle flashed to OpenThread RCP firmware, an OpenThread Border Router (OTBR) container on the host network, and the Matter server container. The Matter server is `matterjs-server` — the matter.js rewrite that replaced the archived `python-matter-server` in 2026. Running these without the add-on store is officially unsupported but well trodden by the community.

**Host networking prerequisites.** What actually matters is the Linux host, not whether your router hands out IPv6 addresses. The host must accept IPv6 router advertisements, including route information options, so it learns routes into the Thread mesh (`net.ipv6.conf.<iface>.accept_ra=2` and `accept_ra_rt_info_max_plen=64`). Docker's IPv6 forwarding settings can interfere and must be checked. Details belong in `docs/setup-thread-matter.md`.

**Commissioning.** Adding a Matter device to HA goes through the HA Companion app on a phone, over Bluetooth, and the phone needs the OTBR's Thread network credentials. This is the most likely place for Phase B to get stuck and should be tested first.

**Why not multiprotocol (Zigbee + Thread on one dongle)?** The Silicon Labs multiprotocol firmware is deprecated. One dongle, one protocol.

### 3.5 Zigbee (when needed)

No current device uses Zigbee. Buy a second SONOFF Dongle Lite and flash it to Zigbee coordinator firmware when the first Zigbee devices are actually on the way. Same hardware and flasher as the Thread dongle.

### 3.6 Energy Monitoring

Per-device monitoring comes from the Shelly plugs in Phase A. Whole-home monitoring (Shelly EM clamp meter) is a future consideration; it needs breaker panel access, which a rental may not allow.

---

## 4. Infrastructure

### 4.1 Operating System

Ubuntu LTS, Linux-first. Paths follow the XDG Base Directory Specification. The repo holds only code and templates; everything written at runtime lives outside it.

```
~/.config/terrarium/.env       # secrets + settings; the repo's .env is a gitignored symlink to it
~/.local/share/terrarium/      # TERRARIUM_DATA — the data root, always an absolute path in .env
├── postgres-data/
├── ha-config/                 # HA's /config, including .storage/ and secrets.yaml
├── grafana-data/
└── ollama-models/             # Phase D
```

Compose files never use `~`. Compose expands it for whichever user runs the command, so a single `sudo` run would point every volume at `/root` and start an empty database. All volumes use `${TERRARIUM_DATA}`.

### 4.2 Docker

Every service runs in a container, configured through environment variables and files in the data root. Image tags are pinned to exact versions. `docker compose up -d` from a clean clone, with an existing data root and `.env`, reproduces the running system.

**What must be preserved across a reinstall:**

| Path | Contents |
|---|---|
| `postgres-data/` | All history and statistics |
| `ha-config/` | HA's entity and device registries, users, integration entries (`.storage/`), and `secrets.yaml` |
| `ollama-models/` | Model weights — re-downloadable, but slow |
| `~/.config/terrarium/.env` | Secrets and settings |

Everything else — the repo, containers, images, `grafana-data/` — is disposable. Losing `ha-config/.storage/` is not fatal to history, but HA re-adds devices with new entity IDs, which splits history and breaks dashboards.

**Reinstall procedure:**
1. `docker compose down`
2. Delete or move the repo clone
3. Re-clone, run `scripts/setup.sh` (it detects the existing `.env` and data root and changes nothing)
4. `docker compose up -d`

Containers restart automatically after a reboot through `restart: unless-stopped`; no systemd unit is used.

### 4.3 Setup Script

`scripts/setup.sh` asks for the LAN IP, the database and Grafana passwords, and a backup target. It writes `~/.config/terrarium/.env` (with `TZ=UTC` and an absolute `TERRARIUM_DATA`), creates the data root, links `.env` into the repo, and seeds HA's `configuration.yaml` and `secrets.yaml` in the data root. It is idempotent: re-running it fills in what's missing and never overwrites existing data or config. Later phases add checks for USB devices and host networking.

### 4.4 Schema Migrations

HA owns the `public` schema; its recorder creates and migrates its own tables there. Terrarium's tables live in a separate `terrarium` schema in the same database, created empty in Phase A. This keeps joins against HA's tables possible while isolating the two migration systems.

Alembic manages the `terrarium` schema from Phase D onward. Its environment is configured with `version_table_schema="terrarium"` and an `include_object` filter that ignores every other schema. Without that filter, autogenerate sees HA's recorder tables as "not in the models" and generates `drop_table` operations for them. Migrations are reversible.

### 4.5 Services (docker-compose)

Services are added to the single `docker-compose.yml` as each phase lands. All image tags are pinned to exact versions at implementation time.

| Service | Phase | Image | Notes |
|---|---|---|---|
| `postgres` | **A** | postgres:17 | `${TERRARIUM_DATA}/postgres-data`; published on `127.0.0.1:5432` only |
| `homeassistant` | **A** | ghcr.io/home-assistant/home-assistant | Host network; config at `${TERRARIUM_DATA}/ha-config` |
| `grafana` | **A** | grafana/grafana | Bridge network; reads via `hass_readonly`; provisioned from repo |
| `otbr` | B | OpenThread Border Router image, chosen in Phase B | Host network; requires the Thread-flashed SONOFF dongle |
| `matter-server` | B | ghcr.io/matter-js/matterjs-server | Matter controller; host network; port 5580 |
| `wyoming-whisper` | C | rhasspy/wyoming-faster-whisper | Local STT |
| `wyoming-piper` | C | rhasspy/wyoming-piper | Local TTS |
| `intent-router` | C | custom Python | Custom HA conversation agent (see §7) |
| `notification-worker` | C | custom Python | Proactive spoken alerts |
| `ha-observer` | D | custom Python | Subscribes to HA's websocket; persists automation traces and context chains (see §13.4) |
| `ollama` | D | ollama/ollama | `${TERRARIUM_DATA}/ollama-models` |
| `curation-worker` | D | custom Python | Background LLM job |
| `read-api` | E | custom Python/FastAPI | Read-only DB access for the control UI and external tools |
| `mosquitto` | F | eclipse-mosquitto | MQTT broker for ESPresense |
| `presence-worker` | F | custom Python | Presence fusion |

Backups are a host script (`scripts/backup.sh`), not a service.

### 4.6 Backup

`scripts/backup.sh` writes two compressed files to `BACKUP_TARGET`: a `pg_dump` taken through `docker compose exec` (no Postgres client needed on the host) and a tarball of `${TERRARIUM_DATA}/ha-config`, including `.storage/` and `secrets.yaml`. Rotation keeps 30 of each by default. `scripts/restore-test.sh` restores the newest dump into a throwaway container and compares row counts — a backup is only trusted once a restore has been tested.

Phase A runs the script manually. Scheduling (nightly, via host cron) and an off-box target — a NAS mount works with no script changes, since the target is just a path — come later. Cloud backup is opt-in: if configured, archives are encrypted with `age` before upload. No plaintext data leaves the LAN. The HA config archive contains secrets; the backup target must be trusted like the server itself.

### 4.7 Observability

Every service emits structured JSON logs. Every service exposes a `/health` endpoint. Prometheus scrape targets are pre-configured but collection is optional. This is a first-class concern for the open-source release.

---

## 5. Home Assistant

### 5.1 Install Method

**HA Container on Ubuntu.** Runs Home Assistant as a Docker container alongside all other Terrarium services. This is one of the two officially supported HA installation methods as of 2025.12 (the other being HA OS). HA binds to the LAN IP on port 8123. Not exposed externally.

**Why not HA OS?** HA OS takes over the entire machine and manages its own Linux distribution. Terrarium requires direct Ubuntu access to run PostgreSQL, Ollama, and the custom Python services alongside HA. HA Container preserves that access.

**Trade-off vs HA Supervised:** HA Supervised was deprecated in 2025.6 and unsupported from 2025.12. HA Container has no add-on store — Whisper, Piper, OTBR, and the Matter server run as standalone containers in docker-compose instead. This fits Terrarium's Docker-first approach anyway.

**Networking:** HA runs with `network_mode: host` for device discovery. It is therefore not on the Compose network and cannot resolve service names. Anything it connects to is published on the host and addressed as `127.0.0.1` — the recorder's database URL is `postgresql://hass:…@127.0.0.1:5432/homeassistant`.

### 5.2 Integrations by Phase

| Integration | Phase | Purpose |
|---|---|---|
| Recorder | A | Writes entity state history to PostgreSQL |
| Shelly | A | Local polling for Plug S power data |
| Thread / OpenThread Border Router | B | Connects to the `otbr` container |
| Matter | B | ALPSTUGA, via the `matter-server` container |
| Wyoming Protocol | C | Connects to `wyoming-whisper` and `wyoming-piper` |
| ESPHome | C | Voice satellites |
| Ollama | D | Local LLM for the Assist pipeline |

### 5.3 Recorder Configuration

PostgreSQL from first boot; HA never writes SQLite. The connection string lives in HA's `secrets.yaml` in the data root. The recorder excludes noisy entities (link quality, RSSI, update entities). State retention is 730 days; long-term statistics are kept indefinitely regardless. `commit_interval` is 5 seconds.

### 5.4 Voice Containers (Phase C)

Whisper and Piper run as standalone Wyoming Protocol containers. HA's Wyoming integration connects to them through ports published on the host.

| Container | Image | Role |
|---|---|---|
| `wyoming-whisper` | rhasspy/wyoming-faster-whisper | Local STT — model size configurable via `.env` |
| `wyoming-piper` | rhasspy/wyoming-piper | Local TTS — voice configurable |

Both are added in HA through Settings → Devices & Services → Wyoming Protocol, pointing at `127.0.0.1` and each container's published port.

---

## 6. Data Layer

### 6.1 PostgreSQL

Single instance, published on `127.0.0.1:5432` only. One database, `homeassistant`, with two schemas: `public` (owned by HA's recorder) and `terrarium` (Terrarium's own tables).

| Role | Access | Used by |
|---|---|---|
| `postgres` | Superuser | Administration only; no service connects as it |
| `hass` | Owns the database and both schemas | HA recorder and Terrarium services |
| `hass_readonly` | `SELECT` on both schemas | Grafana, the read API, external tools |

`hass_readonly`'s access to future tables comes from `ALTER DEFAULT PRIVILEGES FOR ROLE hass ...`. The `FOR ROLE hass` clause is required because default privileges only apply to objects created by the named role, and the recorder creates its tables as `hass`. Passwords come from the environment through `db/init.sh`; none are stored in the repo.

Per-service roles with narrower write access (for example, the curation worker writing only its own tables) are a later hardening step.

### 6.2 Core Tables

| Table | Schema | Written by | Purpose |
|---|---|---|---|
| `states`, `states_meta` | public | HA recorder | Every entity state change; entity IDs live in `states_meta` |
| `state_attributes` | public | HA recorder | Attribute JSON per state |
| `statistics`, `statistics_meta` | public | HA recorder | Long-term aggregated stats, kept indefinitely |
| `events` | public | HA recorder | Event bus, including `automation_triggered` and context IDs |
| `query_log` | terrarium | Intent router | Every transcript + response + latency + session |
| `response_lookup` | terrarium | Router + curation worker | Intent patterns and response templates |
| `lookup_curation_log` | terrarium | Curation worker | Every curation decision with reasoning |
| `automation_proposals` | terrarium | Curation worker | Proposed automations pending approval |
| `automation_runs` | terrarium | HA observer | Every automation execution with trace and context |
| `llm_actions` | terrarium | All LLM jobs | Audit trail of every LLM write action |
| `presence_scores` | terrarium | Presence worker | Per-person confidence scores per signal |

**Recorder schema notes for anyone querying HA's tables:** timestamps are epoch doubles (`last_updated_ts`, `start_ts`), not the old `last_updated` column; `states` joins to `states_meta` on `metadata_id` for the entity ID; `state` is text and needs a cast plus filtering of `unavailable`/`unknown`; energy `statistics.sum` is cumulative, so a period's energy is the change in `sum` across it.

### 6.3 Read API

A small FastAPI service using the `hass_readonly` user. Exposes query endpoints for the dashboard and future tooling. Never touches the HA API for data reads. Runs on a local port, behind the VPN for remote access.

---

## 7. Intent Router

The intent router is the core of Terrarium's intelligence and the only latency-critical custom service.

### 7.0 How It Fits Into HA Assist

HA's Assist pipeline already handles much of a voice turn: wake word, STT through Wyoming, fast local intent matching for device control ("turn off the desk plug"), an optional LLM fallback through the Ollama integration with "prefer handling commands locally", and TTS. The router must not rebuild that.

Phase C therefore runs in two steps:

1. **Stock Assist first.** Whisper, Piper, a satellite, HA's built-in local intents. Measure real latency and see which questions fall through.
2. **Then the router, as a custom conversation agent.** HA's pipeline hands the router the transcript; the router answers from the lookup table, falls back to HA's local intents for device control, and only then calls Ollama — logging every turn to `query_log`. Packaging it as a conversation agent (a small HA custom integration that calls the router service) keeps HA in charge of audio and satellites.

The exact integration mechanism is decided in Phase C. It's listed in Open Questions.

### 7.1 Response Paths

Three paths, evaluated in priority order on every query. Latency targets are measured from transcript received to response text returned; speech-to-text and text-to-speech come on top (on CPU, transcription alone is roughly a second).

**Path 1 — Lookup hit** (target: < 200ms)
1. Receive transcript
2. Check `response_lookup` table for a pattern match
3. Resolve any `{entity_id}` variable slots via HA REST API
4. Fill response template, return to TTS

**Path 2 — HA local intent** (target: < 500ms)
1. No lookup hit
2. Hand the transcript to HA's built-in local intent matching (device control, simple state questions)
3. Return HA's response to TTS

**Path 3 — LLM fallback** (target: 2–8s)
1. No lookup hit, no local intent match
2. Forward transcript + conversation context to Ollama
3. Return response to TTS
4. Log to `query_log` — this entry feeds the curation loop

### 7.2 Conversation Context

The router maintains a short conversation window per device. Sessions have a 30-second TTL. Each `query_log` row has a nullable `session_id`. This enables follow-up queries: "Turn on the lights... actually make them blue" is handled correctly.

### 7.3 Logging

Every query is logged to `query_log` regardless of path. Fields include: transcript, session ID, matched lookup ID (null if LLM), response text, response source, latency in ms, presence context snapshot, device ID, and a nullable `speaker_id` reserved for future diarization.

### 7.4 Performance Architecture

The hot path (lookup check, template resolution, HA state fetch) runs within the intent router process. The lookup table is cached in memory and refreshed when the curation worker changes it, so Path 1 never waits on a database round trip. LLM requests on Path 3 are dispatched asynchronously so they don't block other incoming requests. The curation worker is a completely separate container and never on the hot path.

### 7.5 Graceful Degradation

| Failure | Behavior |
|---|---|
| Ollama unavailable | Paths 1 and 2 continue normally; Path 3 returns a spoken error |
| Postgres unavailable | Path 1 continues from the in-memory cache; logging is buffered and flushed when Postgres returns |
| HA unavailable | No audio reaches the router at all — satellites depend on HA. Nothing to degrade gracefully into |

Postgres and HA are the critical path. Their unavailability is announced to the user rather than failing silently.

---

## 8. Voice Pipeline

### 8.1 Flow

```
Wake word detection (on the satellite, microWakeWord)
    → Audio to HA over the ESPHome native API
        → Assist pipeline → Whisper via Wyoming → transcript
            → Conversation agent (intent router) → response text
                → Piper via Wyoming → audio
                    → Back to the satellite's speaker
```

### 8.2 Speaker Routing

Responses always route back to the originating device. Proactive notifications go to a configured default speaker until Phase F, then to the speaker nearest the detected person.

### 8.3 Wake Word

Default: `okay_nabu`, a stock microWakeWord model. "Hey Terra" requires training a custom microWakeWord model, planned for Phase C. Configurable per device in ESPHome config.

### 8.4 ESPHome OTA

After initial USB flashing, all firmware updates are delivered over the air via ESPHome's built-in OTA mechanism. No physical access required for updates.

---

## 9. Presence Detection

Phase F. A probabilistic fusion model produces a confidence score between 0.0 and 1.0 per person, updated continuously from multiple independent signals.

**Dependencies:** ESPresense publishes over MQTT, so Phase F adds a `mosquitto` broker. The Companion app's geofence reports to HA directly, which only works away from home if the phone can reach HA — that requires Phase E's VPN running on the phone.

### 9.1 Signal Sources

| Signal | Reliability | Latency | Notes |
|---|---|---|---|
| HA companion app geofence | Very high | 30–120s | GPS boundary crossing; reliable but slow |
| BLE scanning (ESPresense) | High | ~10s | Broadcasts even during phone deep sleep — solves the WiFi deep sleep problem |
| WiFi association (router API) | High | ~5s | Fast but unreliable during deep sleep |
| mmWave radar (LD2410/LD2450) | Very high | < 1s | Detects presence regardless of devices; per-room |
| PIR IR sensor | Medium | < 1s | Motion only, not static presence |
| Door contact sensor | Medium | Immediate | Entry/exit events |
| Temperature delta | Low | Minutes | Soft negative signal only — cooling room suggests absence |

### 9.2 Fusion Model

```
presence_score(person) = weighted_average(active_signals) × recency_decay(last_seen)
home_probability = sigmoid(presence_score)
```

Each signal has a weight, TTL, and floor/ceiling constraints. Temperature delta can only push confidence down, never to 100%. Fused scores live in `presence_scores`, updated by the presence worker. HA's native `person` entity becomes a consumer of this score rather than the source of truth.

### 9.3 Room-Level Location

ESPresense ESP32 nodes report BLE signal strength from known devices. Multiple nodes triangulate per-room location. This feeds the notification worker for speaker routing and provides context for automation conditions.

### 9.4 Multi-Person Support

Each tracked person has a `user_id`. The `query_log` and `automation_runs` tables include presence context at time of event. Speaker diarization is designed for but not scheduled — `speaker_id` is a nullable column reserved in `query_log`.

---

## 10. Notification Worker

Handles proactive, system-initiated output. Distinct from the intent router, which handles reactive user-initiated queries.

### 10.1 Architecture

The notification worker and intent router share an output queue to avoid competing for speaker access. Urgent alerts have priority over lower-priority notifications.

### 10.2 Trigger Types

| Trigger | Example |
|---|---|
| Entity state threshold | CO₂ exceeds 1000ppm, PM 2.5 exceeds 15 μg/m³ |
| Duration condition | Front door open for > 5 minutes |
| Scheduled | User-defined reminders |
| Device event | Shelly Plug reports unusual power draw |
| Automation completion | "Washing machine finished" (inferred from power drop) |

### 10.3 Speaker Targeting

Phase C: notifications go to a configured default speaker, with HA Companion app push as a second channel. Phase F: notifications route to the speaker nearest the detected person via ESPresense.

---

## 11. AI Layer

### 11.1 Ollama

Runs as a Docker container on the home server. No external API calls. No telemetry. Two separately configured models specified independently in `.env`:

- **Fast model** (voice fallback): small model sized for a few seconds of latency on the current hardware (see §3.1)
- **Capable model** (curation worker, automation drafts): larger model; no latency constraint

Specific models are chosen and benchmarked in Phase D.

The curation worker has no latency constraint and can use a larger, slower model than the intent router.

### 11.2 System Prompts

Each usage context has its own versioned prompt file in the repository:

- `prompts/intent_classification.txt`
- `prompts/response_generation.txt`
- `prompts/lookup_curation.txt`
- `prompts/automation_proposal.txt`

Prompt changes are tracked in git like code.

---

## 12. Lookup Table and Curation

### 12.1 How Lookup Entries Work

Each entry in `response_lookup` contains an intent pattern, query type (`state` / `history` / `static`), an optional `entity_id` resolved at runtime via the HA REST API, and a response template with variable slots:

```
"The living room is currently {value}{unit}."
```

Variables are always resolved live. No hardcoded values are stored in the table. Every entry also stores priority, confidence score, hit count, whether it was LLM-generated, and the LLM's reasoning for generating it.

### 12.2 Curation Loop

The curation worker runs on a configurable schedule (default: every 4 hours):

1. Pull N recent `query_log` rows where `response_source = 'llm'`
2. Send to Ollama with the curation prompt: *"Which of these are high-frequency, deterministic queries that could be templated? For each, propose a pattern, template, and explain your reasoning."*
3. Insert proposed rows with `llm_generated = true` and a confidence score
4. Log every decision to `lookup_curation_log` with full reasoning
5. Low-confidence entries not hit within a TTL window are automatically pruned

### 12.3 Self-Answering Audit Queries

Audit questions use the same voice pipeline as everything else. "Why did that happen last Tuesday?" is routed to the audit handler (§14.2), which queries `automation_runs` and `llm_actions` and hands the context to the LLM to narrate. The system answers its own audit questions.

---

## 13. Automation System

### 13.1 Core Constraint

**The LLM never silently creates or modifies automations.** All proposals go through explicit user approval. No exceptions.

### 13.2 Proposal Sources

Automations may be proposed when:
- A user explicitly asks: *"Hey Terra, remind me to take out the trash every Thursday at 8pm"*
- The curation worker detects a recurring manual action pattern in `query_log`

The curation worker never infers automations from passive `states` table observation — only from user-initiated voice interactions.

### 13.3 Approval Flow

1. LLM generates automation YAML with a plain-language explanation and reasoning
2. Row inserted into `automation_proposals` with `status = 'pending'`
3. User is notified (voice or dashboard)
4. User reviews — can approve, reject, or edit
5. On approval: automation written to HA, row updated to `status = 'committed'`

How the automation gets written is a Phase D decision. HA's UI saves automations through an internal config endpoint that is not a documented public API and could change between releases; the alternative is writing `automations.yaml` in the data root and calling `automation.reload`. Either way, the committed automation carries the proposal ID in its description so runs can be linked back.

### 13.4 Automation Runs Logging

HA does not write to Terrarium's tables, and its own automation traces are short-lived: HA keeps only the last few traces per automation in `.storage`. Explainability therefore needs its own collector.

The **HA observer** service (Phase D) subscribes to HA's websocket API. On every `automation_triggered` event it fetches the full trace (trigger, each condition with pass/fail, actions taken) and writes it to `automation_runs`, along with HA's `context_id`. It also records the presence snapshot (once Phase F exists) and the `reasoning` field — the LLM's original justification for proposed automations, or the user's description for manual ones.

HA's context IDs let any state change be traced to the automation, script, or user action that caused it. That chain — not a new mechanism — is what answers "why did the lights turn off at 2am". The earlier the observer runs, the more history audit queries can draw on, so it can move ahead of Phase D if needed.

---

## 14. Explainability and Audit

The user should never need to ask why the system did something without getting a clear, traceable answer.

### 14.1 Principles

- Every state change from an automation traces to a specific `automation_runs` row through HA's `context_id`
- Every `automation_runs` row links to the automation definition and LLM reasoning if applicable
- Every LLM write action is logged in `llm_actions` with input summary, output summary, and reasoning
- The voice interface can query any of these tables: *"Hey Terra, why did the lights turn off at 2am last Tuesday?"*

### 14.2 Audit Query Path

The intent router detects audit-intent queries (pattern: "why did", "what happened", "when did") and routes them to a dedicated handler. This handler queries `automation_runs`, `query_log`, and `llm_actions` with time filters, assembles the relevant context, and passes it to the LLM for narration. The response is spoken back.

---

## 15. Remote Access

### 15.1 Personal Use — VPN

WireGuard or Tailscale (decision deferred to setup). The server initiates an outbound connection; no inbound ports are opened on the home network. The application stack is tunnel-agnostic.

- **Tailscale:** 30-minute setup, free personal tier, works through NAT, third-party coordination server sees device list but not traffic
- **WireGuard:** fully self-hosted, no third party, requires more manual setup

### 15.2 Open-Source Users — Cloudflare Tunnel

For users who want public remote access without a VPN, Cloudflare Tunnel (`cloudflared`) runs as a Docker container and provides TLS termination and auth via Cloudflare Access. Free for personal use, no port forwarding or static IP required.

Both deployment modes are documented. The active mode is set in `.env`.

### 15.3 Authentication

- **Local / VPN:** no application auth required for read-only dashboard
- **Control endpoints:** JWT tokens on all write endpoints, generated at setup time
- **Multi-user:** Authentik self-hosted SSO is the documented upgrade path

---

## 16. Dashboard

### 16.1 Data Visualization — Grafana

Reads directly from PostgreSQL via `hass_readonly`. No HA API dependency for data queries. Datasources and dashboards are provisioned from the repo, so a reinstall recreates them. Dashboards select entities through query-driven variables rather than hardcoded entity IDs, so they work on any install. Phase A ships energy and recorder-health panels; later phases add air quality and query-log analytics.

### 16.2 Controls — Custom UI (Phase E)

A web app for device control, automation approval, and lookup-table management. It reads data through the Read API. Device control goes to HA's API — HA is the control plane. Terrarium-specific actions (approving proposals, editing lookup entries) go to a small, separate control endpoint with write access to the `terrarium` schema only, behind the JWT auth in §15.3. The Read API stays read-only. The voice router is not involved; coupling the dashboard to the voice service would make a voice outage take down the controls too.

Grafana and the control UI are independently deployable.

---

## 17. Database Schema

All Terrarium tables live in the `terrarium` schema and are managed by Alembic from Phase D. HA's recorder tables (`states`, `states_meta`, `state_attributes`, `statistics`, `statistics_meta`, `events`) live in `public` and are managed by HA.

```sql
-- Session tracking for multi-turn conversation
CREATE TABLE terrarium.sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_id TEXT NOT NULL,
    started_at TIMESTAMPTZ DEFAULT NOW(),
    last_active TIMESTAMPTZ DEFAULT NOW()
);

-- Intent patterns and response templates
CREATE TABLE terrarium.response_lookup (
    id SERIAL PRIMARY KEY,
    intent_pattern TEXT NOT NULL,
    query_type TEXT NOT NULL
        CHECK (query_type IN ('state', 'history', 'static')),
    entity_id TEXT,
    response_template TEXT NOT NULL,    -- e.g. "The living room is {value}{unit}"
    priority INT DEFAULT 0,
    confidence FLOAT DEFAULT 1.0,
    hit_count INT DEFAULT 0,
    last_hit TIMESTAMPTZ,
    llm_generated BOOLEAN DEFAULT FALSE,
    llm_reasoning TEXT,
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Every voice query and response
CREATE TABLE terrarium.query_log (
    id BIGSERIAL PRIMARY KEY,
    session_id UUID REFERENCES terrarium.sessions(id),
    transcript TEXT NOT NULL,
    matched_lookup_id INT REFERENCES terrarium.response_lookup(id),
    llm_response TEXT,
    response_source TEXT NOT NULL
        CHECK (response_source IN ('lookup', 'ha_intent', 'llm')),
    speaker_id INT,                     -- nullable, reserved for future diarization
    presence_context JSONB,
    device_id TEXT,
    latency_ms INT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- LLM curation decisions
CREATE TABLE terrarium.lookup_curation_log (
    id SERIAL PRIMARY KEY,
    action_type TEXT NOT NULL
        CHECK (action_type IN ('propose', 'promote', 'prune')),
    query_log_ids BIGINT[],
    affected_lookup_id INT,
    llm_reasoning TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- LLM-proposed automations pending approval
CREATE TABLE terrarium.automation_proposals (
    id SERIAL PRIMARY KEY,
    trigger_transcript TEXT,
    proposed_yaml TEXT NOT NULL,
    plain_english TEXT NOT NULL,
    llm_reasoning TEXT,
    status TEXT DEFAULT 'pending'
        CHECK (status IN ('pending', 'approved', 'rejected', 'committed')),
    committed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Every automation execution
CREATE TABLE terrarium.automation_runs (
    id BIGSERIAL PRIMARY KEY,
    automation_id TEXT NOT NULL,
    trigger_type TEXT NOT NULL,
    trigger_detail JSONB,
    conditions_evaluated JSONB,
    actions_taken JSONB,
    reasoning TEXT,
    presence_context JSONB,
    proposal_id INT REFERENCES terrarium.automation_proposals(id),
    ha_context_id TEXT,                 -- HA context ID; links to states/events that this run caused
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Audit trail of every LLM write action
CREATE TABLE terrarium.llm_actions (
    id BIGSERIAL PRIMARY KEY,
    action_type TEXT NOT NULL
        CHECK (action_type IN ('curate_lookup', 'propose_automation', 'prune_lookup')),
    input_summary TEXT,
    output_summary TEXT,
    reasoning TEXT,
    affected_ids INT[],
    model_used TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Presence confidence scores per person per signal
CREATE TABLE terrarium.presence_scores (
    id BIGSERIAL PRIMARY KEY,
    user_id INT NOT NULL,
    signal_type TEXT NOT NULL,
    score FLOAT NOT NULL,
    room TEXT,
    raw_data JSONB,
    recorded_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_query_log_session ON terrarium.query_log (session_id);
CREATE INDEX idx_query_log_created ON terrarium.query_log (created_at DESC);
CREATE INDEX idx_query_log_source ON terrarium.query_log (response_source, created_at DESC);
CREATE INDEX idx_lookup_priority ON terrarium.response_lookup (priority DESC, hit_count DESC);
CREATE INDEX idx_automation_proposals_status ON terrarium.automation_proposals (status);
CREATE INDEX idx_automation_runs_created ON terrarium.automation_runs (created_at DESC);
CREATE INDEX idx_presence_scores_user ON terrarium.presence_scores (user_id, recorded_at DESC);

-- Roles and grants are created by db/init.sh in Phase A, not by migrations:
--   hass owns the database and the terrarium schema;
--   hass_readonly gets SELECT on public and terrarium via
--   ALTER DEFAULT PRIVILEGES FOR ROLE hass IN SCHEMA <schema> GRANT SELECT ON TABLES TO hass_readonly;
```

---

## 18. Repository Structure

The repo holds code, templates, and docs only. Runtime state lives under `TERRARIUM_DATA` (§4.1).

```
terrarium/
├── README.md
├── DESIGN.md                          # this document
├── PHASE-A.md                         # one design doc per phase…
├── PHASE-A-GUIDE.md                   # …and one step-by-step guide
├── LICENSE                            # MIT
├── .env.example
├── .gitignore                         # .env symlink, local artifacts
├── docker-compose.yml
│
├── scripts/
│   ├── setup.sh                       # idempotent first-run setup
│   ├── backup.sh                      # pg_dump + HA config archive
│   ├── restore-test.sh                # restores the newest dump into a throwaway container
│   └── migrate.sh                     # Phase D
│
├── db/
│   ├── init.sh                        # Phase A: roles, ownership, terrarium schema, grants
│   ├── alembic.ini                    # Phase D
│   ├── env.py
│   └── versions/
│
├── ha-config/                         # TEMPLATES, copied into the data root by setup.sh
│   └── configuration.yaml
│
├── grafana/
│   └── provisioning/
│       ├── datasources/
│       └── dashboards/
│
├── services/
│   ├── intent-router/                 # Phase C
│   ├── notification-worker/           # Phase C
│   ├── ha-observer/                   # Phase D
│   ├── curation-worker/               # Phase D
│   ├── read-api/                      # Phase E
│   └── presence-worker/               # Phase F
│
├── custom_components/                 # Phase C: HA conversation-agent integration for the router
│
├── prompts/
│   ├── intent_classification.txt
│   ├── response_generation.txt
│   ├── lookup_curation.txt
│   └── automation_proposal.txt
│
├── esphome/
│   ├── voice-pe.yaml                  # Phase C
│   └── espresense-node.yaml           # Phase F
│
└── docs/
    ├── hardware.md
    ├── setup-ha-container.md
    ├── setup-sonoff-dongle.md         # flash OpenThread RCP firmware
    ├── setup-thread-matter.md         # OTBR, matterjs-server, host IPv6 settings, commissioning
    ├── setup-esphome.md
    ├── setup-vpn-tailscale.md
    ├── setup-vpn-wireguard.md
    ├── setup-cloudflare-tunnel.md
    └── ollama-model-guide.md
```

---

## 19. Capability Tiers

Tiers describe what Terrarium can do; each is independently useful. Phases (§20) describe the build order.

### Tier 1 — Core
Any modern PC running Ubuntu.
- HA Container + PostgreSQL
- Shelly energy monitoring
- Grafana dashboards
- Full state history and querying
- Optional: Thread/Matter sensors such as the ALPSTUGA (needs a Thread radio, the OTBR and Matter server containers, and host IPv6 settings)

### Tier 2 — Voice
Requires a voice satellite (Voice PE or similar).
- Whisper STT + Piper TTS through HA Assist
- Intent router with lookup table
- Multi-turn conversation context
- Proactive spoken notifications

### Tier 3 — AI
Requires a capable CPU or any GPU.
- Ollama fast model (voice fallback) and capable model (curation)
- Lookup table self-curation
- Automation proposal and approval
- Automation trace capture and audit query answering

### Tier 4 — Remote Access
Optional, never required.
- WireGuard or Tailscale (personal VPN), or Cloudflare Tunnel (public access)
- JWT-secured control endpoints
- Dashboard and control UI from anywhere

### Tier 5 — Presence
Requires additional sensors, an MQTT broker, and Tier 4 for away-from-home geofencing.
- ESPresense BLE scanning nodes
- mmWave or PIR room sensors
- Probabilistic presence fusion
- Room-level location and speaker routing
- Presence-aware automations

---

## 20. Implementation Phases

Each phase produces a working, stable system before the next begins.

| Phase | Name | Key deliverables | Document |
|---|---|---|---|
| A | **MVP** | Postgres, HA Container, Shelly ×2, Grafana, recorder, backup + tested restore | [PHASE-A.md](./PHASE-A.md) |
| B | Sensors | ALPSTUGA via SONOFF Thread dongle, OTBR, matterjs-server; Shelly BLU door sensor if the Gen4 gateway works | PHASE-B.md |
| C | Voice | Whisper, Piper, Voice PE satellite, stock Assist first, then the intent router as a conversation agent; notification worker; "Hey Terra" wake word training | PHASE-C.md |
| D | AI | Alembic + `terrarium` tables, Ollama, HA observer, curation worker, automation proposals, audit queries | PHASE-D.md |
| E | Remote | VPN or Cloudflare Tunnel, read API, control endpoint, control UI | PHASE-E.md |
| F | Presence | Mosquitto, ESPresense nodes, presence worker, room-level speaker routing | PHASE-F.md |

Zigbee is not tied to a phase. The second SONOFF dongle is bought and flashed when the first Zigbee device is.

Phase documents are the authoritative implementation references; DESIGN.md is the architectural reference. When they conflict, DESIGN.md wins and the phase document is updated.

---

## 21. Open Questions

Deliberately deferred. None of these block Phase A.

| Item | Phase | Notes |
|---|---|---|
| ~~License~~ | — | **Resolved: MIT** |
| ~~HA install method~~ | — | **Resolved: HA Container** |
| ~~Dongle strategy~~ | — | **Resolved: one SONOFF Dongle Lite (owned, unflashed) for Thread; a second only when Zigbee is needed** |
| ~~Data and state layout~~ | — | **Resolved: data root outside the repo; HA config and `.storage` preserved with Postgres; `terrarium` schema** |
| ~~Default wake word~~ | — | **Resolved: `okay_nabu` until "Hey Terra" is trained** |
| OTBR container image | B | Pick a maintained image; confirm Thread 1.4 support |
| Matter DCL online vs offline | B | matterjs-server checks device certificates online by default; `--dcl-offline` keeps it local at some cost to certificate validation. Tension with §1.3 |
| Phone Thread credentials | B | Confirm the Companion app on your phone can receive the OTBR's credentials for commissioning |
| Shelly BLU via Gen4 plug gateway | B | Confirm HA exposes BLU sensors through the Plug S Gen4 |
| Router integration with Assist | C | Custom conversation-agent integration vs another mechanism |
| "Hey Terra" training | C | Sample collection and microWakeWord training workflow |
| Pattern matching: regex vs embeddings | C | Start with regex against normalized transcripts; schema can add an embedding column |
| Writing approved automations into HA | D | Internal config endpoint vs `automations.yaml` + reload |
| Model selection | D | Benchmark current small and mid-size models on the 7600X, then the 3070 Ti |
| Curation schedule and confidence thresholds | D | Configurable; start conservative and tune with real data |
| Automation approval UX | D/E | Voice prompt as the interim; control UI later |
| VPN choice: Tailscale vs WireGuard | E | Both documented |
| Backup scheduling and off-box target | Post-A | Host cron; a NAS mount works with no script changes |
| Whole-home energy monitoring | — | Shelly EM; needs panel access |
| Speaker diarization | — | `speaker_id` column is reserved; not scheduled |
