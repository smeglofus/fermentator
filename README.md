# Fermentator
Minimal fermentation controller/monitor built for a single Raspberry Pi with Docker Compose. Telemetry from ESP32 every 5 s via MQTT; processing and UI in Node‑RED; data persisted in SQLite; optional Grafana for charts; Caddy for reverse‑proxy and OTA file serving.


## Architecture Overview


**ESP32**: telemetry every 5 s over MQTT/TLS; downlink commands to change temperature setpoint; OTA via HTTPS (esp_https_ota).

**MQTT broker:** Eclipse Mosquitto (auth, ACL; only state is retained).

**Node‑RED** – flows (ingest→DB), dashboard, commands, alerts, OTA orchestration.

**DB:** SQLite – simple time‑series table + retention job

**Caddy** reverse‑proxy + static server for OTA files

**Cloudflare Tunnel:** public access without router port‑forwarding. or **Tailscale** – private remote access (no public ports).


## Features

Live dashboard (temps, SG, relay status) + charts

One‑click actions: set temperature, pause heating

Downlink commands to ESP32 via MQTT (cmd/setpoint, cmd/config)

OTA: upload FW, auto SHA‑256, update manifest, notify device

Basic alerts (out‑of‑range, sensor loss) via Telegram


## Roadmap / Milestones



---

### M2 – Ingest & Dashboard (1–2 days)
**Tasks**
- Node-RED: install palettes (`node-red-node-postgres`, Dashboard if used)
- **Ingest flow:** `mqtt in (devices/+/telemetry)` → `json` → `function(validate)` → Postgres **INSERT**
- **Dashboard flow:** live gauges + simple 24h chart (MQTT stream or SQL query)
- ESP32: sends telemetry every 5 s (QoS1, JSON)

**DoD**
- New rows appear in `reading`; dashboard shows latest values and a 24h curve.

---

### M3 – Commands (0.5 day)
**Tasks**
- Node-RED Dashboard: UI slider/button “Setpoint”
- Flow: UI → `devices/<id>/cmd/setpoint` (QoS1)
- ESP32: confirmation in `devices/<id>/state` (e.g., `applied_setpoint`)
- UI shows ACK (text/notification)

**DoD**
- Changing setpoint from UI → device confirms in `state` within ≤ 2 s.

---

### M4 – OTA (0.5–1 day)
**Tasks**
- Caddy (or Node-RED file server): folder `/ota/<device>/`
- Node-RED upload flow: accept firmware, compute **SHA-256**, write `manifest.json`
- Publish `fw/notify` (MQTT) with `{ version, url, sha256 }`
- ESP32: `esp_https_ota` + A/B rollback test

**DoD**
- Two consecutive successful OTA updates (A→B→A); version visible in `state`.

---

### M5 – Alerts & Retention (0.5 day)
**Tasks**
- Alert flow: out-of-range temperature / ROC → Telegram or email
- Retention: cron (Node-RED `cron-plus` or crontab) — `DELETE older than 400d` + `VACUUM`
- UI parameters for thresholds (e.g., min/max temperature)

**DoD**
- Test alert delivered; old data are pruned on schedule.

---

### M6 – Backups & Docs (0.5 day)
**Tasks**
- `postgres/backup.sh` + cron → daily `pg_dump` to `/backups` (SSD/NAS)
- README: deployment steps, troubleshooting, restore procedure
- Export/import Node-RED flows (`flows.json`) into the repo

**DoD**
- A fresh `.dump` exists and a restore into an empty DB was verified.


### Data scheme:

reading(
  ts TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ','now')),
  device_id TEXT,          -- může být NULL
  temperature_c REAL,
  ambient_c REAL,
  humidity_pct REAL,
  sg REAL,
  relay_on INTEGER,
  setpoint_c REAL,
  fw TEXT
)




Tailscale pro vzdálený přístup

Malé API (FastAPI) pro CSV exporty a jednoduchou autorizaci
