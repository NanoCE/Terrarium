# Terrarium

> A terrarium is a self-contained living ecosystem — controlled, observed, and cared for. So is your home.

Terrarium is a local-first, AI-assisted home automation system built on Home Assistant and PostgreSQL. It listens, learns, and explains itself — replacing cloud voice assistants with something that runs entirely on your hardware, stores everything you care about, and never needs to phone home.

---

## Philosophy

Most smart home systems are built around convenience for the vendor, not the user. Data lives in clouds you don't control, automations are black boxes, and when the service shuts down, your home gets dumber.

Terrarium is built around three principles that don't bend:

**Automation should be effortless.**  
The biggest barrier to home automation isn't hardware — it's the tedious work of setting things up and maintaining them. Terrarium reduces that friction as close to zero as possible. You speak, it responds. Over time it learns which questions you ask repeatedly and answers them faster. It notices when you keep doing the same thing manually and asks if you'd like it automated. You stay in control; it does the legwork.

**Data is fast, transparent, and integrable.**  
Every state change, every voice query, every AI decision is logged with enough context to explain itself — days or weeks later. All of it lives in PostgreSQL, queryable by anything. There are no black boxes. If you ask "why did the lights turn off at 2am last Tuesday?", Terrarium can answer that question through the same voice interface you use for everything else.

**Local-first, cloud-never.**  
Your home data stays on your hardware. LLM inference runs on your server via Ollama. Speech recognition and synthesis run locally via Whisper and Piper. The only time data leaves your network is for backups you explicitly configure, encrypted before they go. Remote access is handled by a VPN — the network perimeter secures you, not a third-party auth service. Cloud integrations are a secondary concern and always opt-in.

---

## What Terrarium Is

- A **voice assistant** that replaces Google Home or Alexa, runs entirely on your LAN, and improves over time
- A **data platform** for your home — every sensor reading, power measurement, and air quality sample stored in Postgres and queryable from anything
- A **learning system** that curates its own response library and proposes automations based on your actual usage patterns
- An **auditable AI layer** where every LLM decision is logged with reasoning and every automation has a traceable history
- An **open platform** designed for replication — `docker compose up` and a setup script get you running on your own hardware

## What Terrarium Is Not

- A cloud service
- A subscription
- A system that makes decisions without you

---

## Key Features

- 🎙️ **Local voice pipeline** — wake word on-device, STT and TTS on your server, no audio leaving your LAN
- ⚡ **Energy monitoring** — per-device power tracking via Shelly plugs, queryable history from day one
- 🌱 **Air quality tracking** — CO₂, PM 2.5, temperature, and humidity via Matter-compatible sensors
- 🧠 **Self-curating lookup table** — the LLM learns which queries can be answered faster without it
- 📋 **Automation proposals** — the system notices patterns and asks if you want them automated; you always approve
- 🔍 **Full audit trail** — ask why anything happened; the system answers from its own logs
- 🔒 **VPN-first remote access** — your data, your network, your rules
- 📊 **Grafana dashboard** — out-of-the-box panels for all sensor and query data
- 🔄 **Clean reinstall** — all runtime data lives in one directory outside the repo; keep it, re-clone everything else, back up and running in minutes

---

## Architecture Overview

```
Voice satellite (ESPHome)
    → Home Assistant Assist pipeline
        → Whisper (STT) → Intent Router → Piper (TTS)
                               │
                    ┌──────────┼──────────┐
                    ▼          ▼          ▼
              Lookup Table  HA intents  Ollama
                    │                    │
                    └──────┬─────────────┘
                           ▼
                       PostgreSQL
                    (states, queries,
                     lookups, audit log)
```

Full architecture is documented in [DESIGN.md](./DESIGN.md).

---

## Hardware Requirements

Terrarium is designed in capability tiers. Each tier is independently useful — you don't need to implement all of them.

| Tier | Requires | What you get |
|---|---|---|
| 1 — Core | Any modern PC, Ubuntu | HA + Postgres + energy/air monitoring + Grafana |
| 2 — Voice | Voice PE or similar satellite | Local voice assistant, replaces Google Home |
| 3 — AI | Decent CPU or any GPU | LLM fallback, self-curation, automation proposals, audit answers |
| 4 — Remote | VPN or Cloudflare Tunnel | Secure access from anywhere |
| 5 — Presence | ESP32 BLE nodes, room sensors, Tier 4 | Room-level location, presence-aware automations |

**Reference server (planned):**
- AMD Ryzen 5 7600X, 32GB DDR5, Ubuntu 24.04
- GPU optional — an RTX 3070 Ti (8GB VRAM) makes local LLM responses several times faster

**Phase A devices:**
- Shelly Plug S 4th Gen ×2 (power monitoring, WiFi, fully local)

**Phase B devices:**
- IKEA ALPSTUGA air quality sensor (Matter over Thread)
- SONOFF Zigbee 3.0 Dongle Lite (EFR32MG21), flashed to OpenThread RCP firmware as the Thread radio

> **Thread note:** The server needs specific IPv6 settings to route into the Thread network, and adding Matter devices goes through the HA Companion app on your phone. See [DESIGN.md §3.4](./DESIGN.md#34-smart-devices--phase-b).

---

## Getting Started

> ⚠️ Terrarium is in active design phase. Implementation is in progress. This section will be updated as services are built.

### Prerequisites

- Ubuntu 24.04 LTS (other Linux distributions should work; untested)
- Docker Engine + the Compose plugin
- Git
- For Phase B: one SONOFF Zigbee 3.0 Dongle Lite (EFR32MG21) and the HA Companion app on your phone

### Quick Start

```bash
git clone https://github.com/yourname/terrarium.git
cd terrarium
./scripts/setup.sh
docker compose up -d
```

The setup script writes your `.env` outside the repo, creates the data directory, and seeds Home Assistant's config. The repo itself never holds runtime data — you can delete and re-clone it at any time. For the full walkthrough, see [PHASE-A-GUIDE.md](./PHASE-A-GUIDE.md).

### Home Assistant

Terrarium uses Home Assistant Container (the officially supported Docker-based install method as of 2025.12). See [docs/setup-ha-container.md](./docs/setup-ha-container.md) for installation instructions.

### Voice Satellites

ESPHome configs for supported devices live in `esphome/`. See [docs/setup-esphome.md](./docs/setup-esphome.md).

### Remote Access

- Tailscale: [docs/setup-vpn-tailscale.md](./docs/setup-vpn-tailscale.md)
- WireGuard: [docs/setup-vpn-wireguard.md](./docs/setup-vpn-wireguard.md)
- Cloudflare Tunnel: [docs/setup-cloudflare-tunnel.md](./docs/setup-cloudflare-tunnel.md)

---

## Repository Structure

```
terrarium/
├── README.md
├── DESIGN.md                 # full system design document
├── PHASE-A.md                # phase design docs + step-by-step guides
├── PHASE-A-GUIDE.md
├── docker-compose.yml
├── .env.example
├── scripts/                  # setup, backup, restore test
├── db/                       # init script, later Alembic migrations
├── ha-config/                # Home Assistant config templates
├── grafana/                  # provisioned datasources and dashboards
├── services/
│   ├── intent-router/        # voice query handler (hot path)
│   ├── notification-worker/  # proactive alerts
│   ├── ha-observer/          # automation trace capture for audit queries
│   ├── curation-worker/      # background LLM jobs
│   ├── read-api/             # read-only data API
│   └── presence-worker/      # presence fusion
├── prompts/                  # versioned LLM system prompts
├── esphome/                  # voice satellite and presence node configs
└── docs/                     # setup guides per component
```

---

## Wake Word

The planned wake word is **"Hey Terra"**. It needs a custom-trained microWakeWord model, so Terrarium ships with the stock `okay_nabu` until that model exists. The wake word is configurable per device in ESPHome — see [docs/setup-esphome.md](./docs/setup-esphome.md).

---

## Roadmap

- [ ] **Phase A — MVP:** HA Container, PostgreSQL, Shelly ×2, Grafana, recorder, backup with a tested restore
- [ ] **Phase B — Sensors:** ALPSTUGA air quality via SONOFF Thread dongle + OTBR + matterjs-server
- [ ] **Phase C — Voice:** Whisper, Piper, Voice PE satellite, intent router, spoken notifications, "Hey Terra" wake word
- [ ] **Phase D — AI:** Ollama, automation trace capture, curation worker, automation proposals, audit query answering
- [ ] **Phase E — Remote:** VPN/Cloudflare Tunnel, read API, control UI
- [ ] **Phase F — Presence:** MQTT, ESPresense, presence fusion, room-level speaker routing

See [DESIGN.md §20](./DESIGN.md#20-implementation-phases) for the full phase breakdown.

---

## Design Document

The full system design — architecture, schema, service contracts, capability tiers, and open questions — is in [DESIGN.md](./DESIGN.md). Read it before contributing.

---

## Contributing

Contributions welcome. A few ground rules:

- Read `DESIGN.md` first — understand the principles before proposing changes
- No cloud dependencies in the core path
- Every LLM action must be logged and reversible
- Test coverage is expected for the intent router and curation worker

---

## License

MIT. See [LICENSE](./LICENSE).
