# Terrarium — System Design Document

**Status:** Pre-implementation draft  
**Last updated:** 2026-04-18  
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
│  ESP32-S3-BOX-3          Smart Devices         ALPSTUGA         │
│  (wake word + mic        (Shelly Plug S,        (Thread/Matter   │
│   + speaker)              Zigbee Phase 2)        air quality)    │
└────────────┬────────────────────┬──────────────────┬────────────┘
             │ Wyoming Protocol   │ WiFi / Zigbee    │ Thread
             ▼                    ▼                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    HOME ASSISTANT CONTAINER                      │
│                                                                 │
│  Whisper (Wyoming)  Piper (Wyoming)  Assist Pipeline  Automations│
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    INTENT ROUTER                         │  │
│  │  (1) Lookup hit → template fill → HA state → respond    │  │
│  │  (2) HA state query → REST API → respond                │  │
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
    │  Curation Worker │  (background job — LLM curates lookup
    │  Presence Worker │   table, proposes automations)
    │  Notif Worker    │
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

**Ollama model selection:**

| Phase | Role | Model | Notes |
|---|---|---|---|
| CPU-only (now) | Intent router fallback | Phi-3 Mini 3.8B Q4 | ~3GB RAM, fast enough for voice |
| CPU-only (now) | Curation worker | Mistral 7B Q4 or Llama 3.1 8B Q4 | Background job, no latency constraint |
| With 3070 Ti | Both roles | Llama 3.1 8B Q4 | Fits fully in VRAM (~5GB), 5-10x faster |

### 3.2 Voice Devices

| Device | Role | Cost | Notes |
|---|---|---|---|
| ESP32-S3-BOX-3 | Primary room voice satellite | ~$50 | Built-in mic + speaker, ESPHome native, microWakeWord on-device |
| ATOM Echo (M5Stack) | Secondary rooms (listen only) | ~$13 | No speaker — good for command-only rooms |
| Raspberry Pi Zero 2W | Flexible satellite | ~$40 built | Runs `wyoming-satellite`, max audio flexibility |

All devices use **Wyoming Protocol** to stream audio to the server after wake word detection. Wake word runs on-device so audio is never transmitted until triggered.

**Wake word:** `hey_terra` — configurable per device in ESPHome config. Custom wake word training via OpenWakeWord is documented but optional. The default pretrained model will be selected at implementation time.

### 3.3 Smart Devices — Phase 1

| Device | Protocol | Purpose |
|---|---|---|
| Shelly Plug S 4th Gen (×2) | WiFi (local API + MQTT) | Per-device power monitoring |
| IKEA ALPSTUGA | Thread / Matter | CO₂, PM 2.5, temperature, humidity |
| SONOFF Zigbee 3.0 Dongle Lite #1 (EFR32MG21) | Thread RCP only | Thread border router for ALPSTUGA — flashed to OpenThread RCP firmware |

**Note on ALPSTUGA and Thread:** The ALPSTUGA uses Thread protocol, not WiFi. In HA Container mode, Thread support requires three things running together: the SONOFF dongle flashed to OpenThread RCP firmware, a community OTBR container (`ownbee/hass-otbr-docker`) acting as the border router, and a Matter server container (`python-matter-server`). This is officially unsupported by HA but well-documented by the community and confirmed working.

**Critical network prerequisite:** IPv6 must be enabled on your LAN for the OTBR container to route Thread traffic correctly. This is not optional. Verify before attempting Phase 1 setup.

**Why not multiprotocol (Zigbee + Thread on one dongle)?** The Silicon Labs add-on that enabled this was deprecated and abandoned. SONOFF confirmed no replacement is ready for production use. One dongle, one protocol is the only stable path.

### 3.4 Smart Devices — Phase 2

- SONOFF Zigbee 3.0 Dongle Lite #2 (EFR32MG21) — flashed to Zigbee coordinator firmware, dedicated Zigbee dongle
- Zigbee sensors, lights, and switches TBD based on needs
- ESPresense ESP32 nodes for BLE room-level presence detection

**Note:** Two SONOFF Dongle Lites total (~$40) — one dedicated to Thread (Phase 1), one dedicated to Zigbee (Phase 2). Same hardware, same flasher tool, separate protocols. Cheaper than one ZBT-2 and better range due to external antenna.

### 3.5 Energy Monitoring (Phase 1)

The two Shelly Plug S units provide real-time per-device wattage, voltage, and energy totals via the local Shelly integration (no cloud required). All data flows directly into PostgreSQL via the HA recorder.

Whole-home monitoring (Shelly EM clamp meter) is a Phase 2 consideration.

---

## 4. Infrastructure

### 4.1 Operating System

Ubuntu LTS, Linux-first. All configuration follows the XDG Base Directory Specification:

```
~/.config/terrarium/          # user config (.env, service configs)
~/.local/share/terrarium/     # persistent Docker volumes
~/.cache/terrarium/           # logs, ephemeral data
/etc/terrarium/               # system-wide config (if installed as service)
```

### 4.2 Docker

Every service runs in a container. Configuration is entirely via environment variables. `docker compose up` from a clean clone plus a generated `.env` should produce a running system.

**Reinstall procedure:**
1. `docker compose down`
2. Delete everything except `~/.local/share/terrarium/postgres-data/` and `~/.local/share/terrarium/ollama-models/`
3. Re-clone repo, run setup script
4. `docker compose up`

The database is the only stateful artifact that must be preserved. Everything else regenerates from config.

### 4.3 Setup Script

An interactive setup script (`scripts/setup.sh`) asks three questions — LAN IP, timezone, Postgres password — and writes `~/.config/terrarium/.env`. It also validates dependencies (Docker version, available disk space, USB devices present) and prints a hardware checklist.

### 4.4 Schema Migrations

Alembic manages all database schema changes. No raw `CREATE TABLE` scripts in production. Migrations run automatically on service startup. Every migration is reversible.

### 4.5 Services (docker-compose)

Services are introduced progressively across implementation phases. The single `docker-compose.yml` contains all services; earlier phases simply don't activate later ones.

| Service | Phase | Image | Notes |
|---|---|---|---|
| `postgres` | **A** | postgres:16 | Volume at `~/.local/share/terrarium/postgres-data/` |
| `homeassistant` | **A** | ghcr.io/home-assistant/home-assistant | HA Container |
| `grafana` | **A** | grafana/grafana | Reads via `hass_readonly` DB user; provisioned from repo |
| `backup` | **A** | custom shell | Script ready; target configured via `.env` |
| `otbr` | B | ghcr.io/ownbee/hass-otbr-docker | OpenThread Border Router; `network_mode: host`; requires Thread-flashed SONOFF |
| `matter-server` | B | ghcr.io/home-assistant-libs/python-matter-server | Matter controller for ALPSTUGA |
| `wyoming-whisper` | C | rhasspy/wyoming-faster-whisper | Local STT; `small-int8` default |
| `wyoming-piper` | C | rhasspy/wyoming-piper | Local TTS; voice configurable |
| `intent-router` | C | custom Python/FastAPI | Hot path — localhost network only |
| `ollama` | D | ollama/ollama | Model storage at `~/.local/share/terrarium/ollama-models/` |
| `curation-worker` | D | custom Python | Background LLM job |
| `read-api` | E | custom Python/FastAPI | Read-only DB access for dashboard and external tooling |

### 4.6 Backup

`pg_dump` runs nightly at 3am (configurable via `.env`) to a local target directory (USB drive or NAS path). Backups are compressed and rotated, keeping 30 days by default. HA config files are backed up separately via the HA built-in backup system. Cloud backup is opt-in: if an S3 bucket is configured in `.env`, backups are encrypted with `age` before upload. No plaintext data ever leaves the LAN.

### 4.7 Observability

Every service emits structured JSON logs. Every service exposes a `/health` endpoint. Prometheus scrape targets are pre-configured but collection is optional. This is a first-class concern for the open-source release.

---

## 5. Home Assistant

### 5.1 Install Method

**HA Container on Ubuntu.** Runs Home Assistant as a Docker container alongside all other Terrarium services. This is one of the two officially supported HA installation methods as of 2025.12 (the other being HA OS). HA binds to the LAN IP on port 8123. Not exposed externally.

**Why not HA OS?** HA OS takes over the entire machine and manages its own Linux distribution. Terrarium requires direct Ubuntu access to run PostgreSQL, Ollama, and the custom Python services alongside HA. HA Container preserves that access.

**Trade-off vs HA Supervised:** HA Supervised was deprecated in 2025.6 and unsupported from 2025.12. HA Container has no add-on store — Whisper, Piper, and OTBR run as standalone Wyoming and community containers in docker-compose instead. This is architecturally cleaner for Terrarium's Docker-first approach anyway.

### 5.2 Integrations — Phase 1

| Integration | Purpose |
|---|---|
| Shelly | Local polling for Plug S power data |
| Matter | ALPSTUGA air quality sensor — via `python-matter-server` container + OTBR container |
| OpenThread Border Router | Connects to `otbr` container; routes Thread traffic for ALPSTUGA |
| Wyoming Protocol | Voice pipeline — connects to `wyoming-whisper`, `wyoming-piper`, and satellites |
| Recorder | Writes entity state history to PostgreSQL |

### 5.3 Recorder Configuration

HA's default SQLite is replaced with PostgreSQL from day one. The recorder excludes noisy entities (Zigbee link quality metrics, HA update entities). Retention is 730 days. `commit_interval` is 5 seconds.

### 5.4 Voice Containers (Tier 2)

Whisper and Piper run as standalone Wyoming Protocol containers in docker-compose rather than HA add-ons. HA's Wyoming Protocol integration connects to them over the local Docker network exactly as it would to add-ons.

| Container | Image | Role |
|---|---|---|
| `wyoming-whisper` | rhasspy/wyoming-faster-whisper | Local STT — `small-int8` model default; configurable via `.env` |
| `wyoming-piper` | rhasspy/wyoming-piper | Local TTS — voice model configurable per device |

Both containers expose Wyoming Protocol endpoints that HA discovers via the Wyoming integration in Settings → Integrations.

---

## 6. Data Layer

### 6.1 PostgreSQL

Single instance, binds to `localhost` only. Two database users:

- `hass` — read/write, used by HA recorder and all Terrarium services
- `hass_readonly` — read-only, used by Grafana, the read API, and external tooling

### 6.2 Core Tables

| Table | Written by | Purpose |
|---|---|---|
| `states` | HA recorder | Every entity state change |
| `state_attributes` | HA recorder | Attribute JSON per state |
| `statistics` | HA recorder | Long-term aggregated stats |
| `query_log` | Intent router | Every transcript + response + latency + session |
| `response_lookup` | Router + curation worker | Intent patterns and response templates |
| `lookup_curation_log` | Curation worker | Every LLM curation decision with reasoning |
| `automation_proposals` | Curation worker | LLM-proposed automations pending user approval |
| `automation_runs` | HA + intent router | Every automation execution with full context |
| `llm_actions` | All LLM jobs | Audit trail of every LLM write action |
| `presence_scores` | Presence worker | Per-person confidence scores from all signals |

### 6.3 Read API

A small FastAPI service using the `hass_readonly` user. Exposes query endpoints for the dashboard and future tooling. Never touches the HA API for data reads. Runs on a local port, behind the VPN for remote access.

---

## 7. Intent Router

The intent router is the core of Terrarium's intelligence and the only latency-critical custom service.

### 7.1 Response Paths

Three paths, evaluated in priority order on every query:

**Path 1 — Lookup hit** (target: < 200ms)
1. Receive transcript
2. Check `response_lookup` table for a pattern match
3. Resolve any `{entity_id}` variable slots via HA REST API
4. Fill response template, return to TTS

**Path 2 — HA state query** (target: 1–2s)
1. No lookup hit
2. LLM classifies intent as a state or history query
3. Query HA REST API or Postgres `states` table
4. Format response, return to TTS

**Path 3 — LLM fallback** (target: 2–8s)
1. No lookup hit, not a state query
2. Forward transcript + conversation context to Ollama
3. Return response to TTS
4. Log to `query_log` — this entry feeds the curation loop

### 7.2 Conversation Context

The router maintains a short conversation window per device. Sessions have a 30-second TTL. Each `query_log` row has a nullable `session_id`. This enables follow-up queries: "Turn on the lights... actually make them blue" is handled correctly.

### 7.3 Logging

Every query is logged to `query_log` regardless of path. Fields include: transcript, session ID, matched lookup ID (null if LLM), response text, response source, latency in ms, presence context snapshot, device ID, and a nullable `speaker_id` reserved for future diarization.

### 7.4 Performance Architecture

The hot path (lookup check, template resolution, HA API call) runs entirely within the intent router process — no inter-service hops on Paths 1 or 2. LLM requests on Path 3 are dispatched asynchronously so they don't block other incoming requests. The curation worker is a completely separate container and never on the hot path.

### 7.5 Graceful Degradation

| Failure | Behavior |
|---|---|
| Ollama unavailable | Paths 1 and 2 continue normally; Path 3 returns a spoken error |
| Postgres unavailable | Paths 2 and 3 still functional; Path 1 lookup unavailable |
| HA unavailable | Spoken error for state queries; static lookup templates still work |

Postgres and HA are the critical path. Their unavailability is announced to the user rather than failing silently.

---

## 8. Voice Pipeline

### 8.1 Flow

```
Wake word detection (on ESP32 device)
    → Audio stream over Wyoming Protocol to server
        → Whisper (STT) → transcript
            → Intent Router → response text
                → Piper (TTS) → audio stream
                    → Device speaker
```

### 8.2 Speaker Routing

Responses always route back to the originating device. Proactive notifications route to the speaker nearest the detected presence (Phase 2, via ESPresense).

### 8.3 Wake Word

Default: `hey_terra` using a pretrained OpenWakeWord model. Configurable per device in ESPHome config. Custom wake word training is documented as an optional advanced step.

### 8.4 ESPHome OTA

After initial USB flashing, all firmware updates are delivered over the air via ESPHome's built-in OTA mechanism. No physical access required for updates.

---

## 9. Presence Detection

A probabilistic fusion model produces a confidence score between 0.0 and 1.0 per person, updated continuously from multiple independent signals.

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

ESPresense ESP32 nodes (Phase 2) report BLE signal strength from known devices. Multiple nodes triangulate per-room location. This feeds the notification worker for speaker routing and provides context for automation conditions.

### 9.4 Multi-Person Support

Each tracked person has a `user_id`. The `query_log` and `automation_runs` tables include presence context at time of event. Speaker diarization is designed for but not implemented in Phase 1 — `speaker_id` is a nullable column reserved in `query_log`.

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

Phase 1: notifications go to a configured default speaker. Phase 2: notifications route to the speaker nearest the detected presence via ESPresense.

---

## 11. AI Layer

### 11.1 Ollama

Runs as a Docker container on the home server. No external API calls. No telemetry. Two separately configured models specified independently in `.env`:

- **Fast model** (intent router fallback): Phi-3 Mini 3.8B Q4 → Llama 3.1 8B Q4 once GPU arrives
- **Capable model** (curation worker, automation drafts): Mistral 7B Q4 or Llama 3.1 8B Q4

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

The lookup table is a data source the intent router can query directly. "Why did that happen last Tuesday?" routes through the same voice pipeline as everything else, querying `automation_runs` and `llm_actions` and handing the context to the LLM to narrate. The system answers its own audit questions.

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
5. On approval: automation committed to HA config via REST API, row updated to `status = 'committed'`

### 13.4 Automation Runs Logging

Every automation execution is recorded in `automation_runs` with: trigger type and detail, conditions evaluated (each with pass/fail), actions taken, presence context, and a `reasoning` field containing the LLM's original justification or user-supplied description. This table is the primary data source for audit queries.

---

## 14. Explainability and Audit

The user should never need to ask why the system did something without getting a clear, traceable answer.

### 14.1 Principles

- Every state change from an automation traces to a specific `automation_runs` row
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

Reads directly from PostgreSQL via `hass_readonly`. No HA API dependency for data queries. Pre-built panels for sensor history, energy monitoring, air quality trends, and query log analytics. Runs as a Docker container.

### 16.2 Controls — Custom UI (Phase 2+)

A React/Next.js web app for device control, automation approval, and lookup table management. Calls the Read API for data and the intent router API for control actions. The intent router is the single control surface — all device control flows through it rather than directly to HA.

Both tools are independently deployable.

---

## 17. Database Schema

Full Alembic-managed schema. HA recorder tables (`states`, `state_attributes`, `statistics`, `statistics_meta`, `events`) are managed by HA.

```sql
-- Session tracking for multi-turn conversation
CREATE TABLE sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_id TEXT NOT NULL,
    started_at TIMESTAMPTZ DEFAULT NOW(),
    last_active TIMESTAMPTZ DEFAULT NOW()
);

-- Every voice query and response
CREATE TABLE query_log (
    id BIGSERIAL PRIMARY KEY,
    session_id UUID REFERENCES sessions(id),
    transcript TEXT NOT NULL,
    matched_lookup_id INT,
    llm_response TEXT,
    response_source TEXT NOT NULL
        CHECK (response_source IN ('lookup', 'llm', 'state_query')),
    speaker_id INT,                     -- nullable, reserved for future diarization
    presence_context JSONB,
    device_id TEXT,
    latency_ms INT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Intent patterns and response templates
CREATE TABLE response_lookup (
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

-- LLM curation decisions
CREATE TABLE lookup_curation_log (
    id SERIAL PRIMARY KEY,
    action_type TEXT NOT NULL
        CHECK (action_type IN ('propose', 'promote', 'prune')),
    query_log_ids BIGINT[],
    affected_lookup_id INT,
    llm_reasoning TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- LLM-proposed automations pending approval
CREATE TABLE automation_proposals (
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
CREATE TABLE automation_runs (
    id BIGSERIAL PRIMARY KEY,
    automation_id TEXT NOT NULL,
    trigger_type TEXT NOT NULL,
    trigger_detail JSONB,
    conditions_evaluated JSONB,
    actions_taken JSONB,
    reasoning TEXT,
    presence_context JSONB,
    proposal_id INT REFERENCES automation_proposals(id),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Audit trail of every LLM write action
CREATE TABLE llm_actions (
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
CREATE TABLE presence_scores (
    id BIGSERIAL PRIMARY KEY,
    user_id INT NOT NULL,
    signal_type TEXT NOT NULL,
    score FLOAT NOT NULL,
    room TEXT,
    raw_data JSONB,
    recorded_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_query_log_session ON query_log (session_id);
CREATE INDEX idx_query_log_created ON query_log (created_at DESC);
CREATE INDEX idx_query_log_source ON query_log (response_source, created_at DESC);
CREATE INDEX idx_lookup_priority ON response_lookup (priority DESC, hit_count DESC);
CREATE INDEX idx_automation_proposals_status ON automation_proposals (status);
CREATE INDEX idx_automation_runs_created ON automation_runs (created_at DESC);
CREATE INDEX idx_presence_scores_user ON presence_scores (user_id, recorded_at DESC);

-- Read-only role
CREATE ROLE hass_readonly WITH LOGIN PASSWORD 'changeme_readonly';
GRANT CONNECT ON DATABASE homeassistant TO hass_readonly;
GRANT USAGE ON SCHEMA public TO hass_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO hass_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO hass_readonly;
```

---

## 18. Repository Structure

```
terrarium/
├── README.md
├── DESIGN.md                          # this document
├── LICENSE                            # MIT
├── .env.example
├── docker-compose.yml
├── docker-compose.dev.yml
│
├── scripts/
│   ├── setup.sh                       # interactive first-run setup
│   ├── backup.sh                      # pg_dump + optional encrypted cloud upload
│   └── migrate.sh
│
├── services/
│   ├── intent-router/
│   │   ├── Dockerfile
│   │   ├── main.py
│   │   ├── router/
│   │   │   ├── lookup.py
│   │   │   ├── state_query.py
│   │   │   └── llm_fallback.py
│   │   └── tests/
│   │
│   ├── curation-worker/
│   │   ├── Dockerfile
│   │   ├── worker.py
│   │   ├── curator.py
│   │   └── proposer.py
│   │
│   ├── read-api/
│   │   ├── Dockerfile
│   │   └── main.py
│   │
│   ├── presence-worker/
│   │   ├── Dockerfile
│   │   └── worker.py
│   │
│   └── notification-worker/
│       ├── Dockerfile
│       └── worker.py
│
├── db/
│   ├── alembic.ini
│   ├── env.py
│   └── versions/
│
├── prompts/
│   ├── intent_classification.txt
│   ├── response_generation.txt
│   ├── lookup_curation.txt
│   └── automation_proposal.txt
│
├── ha-config/
│   ├── configuration.yaml
│   ├── automations.yaml
│   └── scripts.yaml
│
├── esphome/
│   ├── s3-box-3.yaml
│   ├── atom-echo.yaml
│   └── espresense-node.yaml           # Phase 2
│
└── docs/
    ├── hardware.md
    ├── setup-ha-container.md
    ├── setup-sonoff-dongle.md             # flash Thread RCP (#1) and Zigbee (#2) firmware
    ├── setup-thread-matter.md             # OTBR container + Matter server + IPv6 prereqs
    ├── setup-esphome.md
    ├── setup-vpn-tailscale.md
    ├── setup-vpn-wireguard.md
    ├── setup-cloudflare-tunnel.md
    └── ollama-model-guide.md
```

---

## 19. Capability Tiers

Each tier is independently useful. A user can run Tier 1 indefinitely without moving to Tier 2.

### Tier 1 — Core
Any modern hardware. Day one.
- HA Container + PostgreSQL
- Shelly Plug S energy monitoring
- ALPSTUGA air quality (requires SONOFF dongle #1 flashed to Thread RCP + OTBR container + Matter server container + IPv6 on LAN)
- Grafana data dashboard
- Full state history and querying

### Tier 2 — Voice
Requires mic/speaker hardware.
- ESP32-S3-BOX-3 or equivalent satellite
- Whisper STT + Piper TTS
- Intent router with lookup table
- Multi-turn conversation context
- Proactive notifications to speakers

### Tier 3 — AI
Requires capable CPU or any GPU.
- Ollama fast model (intent router fallback)
- Ollama capable model (curation worker)
- Lookup table self-curation
- Automation proposal and approval
- Explainability and audit query answering

### Tier 4 — Presence
Requires additional sensors.
- ESPresense BLE scanning nodes
- mmWave or PIR room sensors
- Probabilistic presence fusion
- Room-level location and speaker routing
- Presence-aware automations

### Tier 5 — Remote Access
Optional, never required.
- WireGuard or Tailscale (personal VPN)
- Or Cloudflare Tunnel (public access)
- JWT-secured control endpoints
- Full dashboard access from anywhere

---

## 20. Implementation Phases

The capability tiers define what Terrarium can do. The implementation phases define the order in which it gets built. Each phase produces a working, stable system before the next phase begins.

| Phase | Name | Key deliverables | Document |
|---|---|---|---|
| A | **MVP** | Postgres, HA Container, Shelly ×2, Grafana, recorder, backup infrastructure | [PHASE-A.md](./PHASE-A.md) |
| B | Sensors | ALPSTUGA via SONOFF Thread dongle, OTBR container, Matter server | PHASE-B.md |
| C | Voice | SONOFF Zigbee dongle, Whisper, Piper, Wyoming satellites, intent router | PHASE-C.md |
| D | AI | Ollama, curation worker, automation proposals, audit queries | PHASE-D.md |
| E | Remote | VPN/Cloudflare Tunnel, read API, control UI | PHASE-E.md |

Phase documents are the authoritative implementation references. DESIGN.md is the architectural reference. When they conflict, DESIGN.md wins and the phase doc should be updated.

---

## 21. Open Questions

Deliberately deferred to implementation. None of these block starting the repository.

| Item | Notes |
|---|---|
| ~~License: MIT vs Apache 2.0~~ | **Resolved: MIT** |
| ~~HA install method~~ | **Resolved: HA Container** |
| ~~Dongle strategy~~ | **Resolved: Two SONOFF Dongle Lites — #1 Thread RCP, #2 Zigbee** |
| Wake word model selection | Test pretrained OpenWakeWord models at implementation time |
| Pattern matching: regex vs embeddings | Start with regex; schema supports adding an embedding column later |
| Curation job schedule | Configurable via `.env`; default TBD, suggest every 4 hours |
| Automation approval UX | Depends on Phase 2 control UI timeline; voice notification is viable interim |
| Confidence thresholds | Start conservative; tune with real usage data |
| VPN choice: Tailscale vs WireGuard | Both documented; pick at setup time |
| Whole-home energy monitoring | Shelly EM for Phase 2; requires panel access |
| Custom wake word training | Easy to add later; not blocking |
| Speaker diarization | `pyannote.audio` integration; schema is ready, implementation is Phase 3+ |
