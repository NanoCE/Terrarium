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
- 🔄 **Clean reinstall** — preserve your Postgres volume, wipe everything else, back up and running in minutes

---

## Architecture Overview

```
Voice Device (ESP32)
    → Wyoming Protocol
        → Whisper (STT) → Intent Router → Piper (TTS)
                               │
                    ┌──────────┼──────────┐
                    ▼          ▼          ▼
              Lookup Table   HA API    Ollama
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
| 2 — Voice | ESP32-S3-BOX-3 or similar | Local voice assistant, replaces Google Home |
| 3 — AI | Decent CPU or any GPU | LLM fallback, self-curation, automation proposals |
| 4 — Presence | ESP32 BLE nodes, room sensors | Room-level location, presence-aware automations |
| 5 — Remote | VPN or Cloudflare Tunnel | Secure access from anywhere |

**Tested server configuration:**
- AMD Ryzen 5 7600X, 32GB DDR5, Ubuntu 24.04
- GPU optional — RTX 3070 Ti (8GB VRAM) when available improves LLM response times 5-10x

**Phase 1 devices:**
- Shelly Plug S 4th Gen (power monitoring, WiFi, fully local)
- IKEA ALPSTUGA air quality sensor (Thread/Matter)
- SONOFF Zigbee 3.0 Dongle Lite #1 (EFR32MG21) — flashed to OpenThread RCP firmware, acts as Thread border router for ALPSTUGA
- SONOFF Zigbee 3.0 Dongle Lite #2 (EFR32MG21) — flashed to Zigbee coordinator firmware (Phase 2, buy alongside #1)

> **Note:** IPv6 must be enabled on your LAN for Thread to work. Verify this before Phase 1 setup.

---

## Getting Started

> ⚠️ Terrarium is in active design phase. Implementation is in progress. This section will be updated as services are built.

### Prerequisites

- Ubuntu 24.04 LTS (other Linux distributions should work; untested)
- Docker + Docker Compose
- Git
- 2× SONOFF Zigbee 3.0 Dongle Lite (EFR32MG21) — one for Thread, one for Zigbee
- IPv6 enabled on your LAN (required for Thread/Matter)

### Quick Start

```bash
git clone https://github.com/yourname/terrarium.git
cd terrarium
./scripts/setup.sh
docker compose up -d
```

The setup script will walk you through initial configuration and write your `.env` file.

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
├── docker-compose.yml
├── .env.example
├── scripts/                  # setup, backup, migration
├── services/
│   ├── intent-router/        # voice query handler (hot path)
│   ├── curation-worker/      # background LLM jobs
│   ├── read-api/             # read-only data API
│   ├── presence-worker/      # presence fusion
│   └── notification-worker/  # proactive alerts
├── db/                       # Alembic migrations
├── prompts/                  # versioned LLM system prompts
├── ha-config/                # Home Assistant configuration
├── esphome/                  # voice satellite firmware configs
└── docs/                     # setup guides per component
```

---

## Wake Word

The default wake word is **"Hey Terra"**.

Configurable per device in ESPHome config. Custom wake word training via OpenWakeWord is supported — see [docs/setup-esphome.md](./docs/setup-esphome.md).

---

## Roadmap

- [ ] **Phase A — MVP:** HA Container, PostgreSQL, Shelly ×2, Grafana, recorder, backup infrastructure
- [ ] **Phase B — Sensors:** ALPSTUGA air quality via SONOFF Thread dongle + OTBR + Matter server
- [ ] **Phase C — Voice:** Zigbee dongle, Whisper, Piper, ESP32-S3-BOX-3 satellites, intent router
- [ ] **Phase D — AI:** Ollama, curation worker, automation proposals, audit query answering
- [ ] **Phase E — Remote:** VPN/Cloudflare Tunnel, read API, control UI

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
