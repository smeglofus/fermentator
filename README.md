# Fermentator
Minimal fermentation controller/monitor on a single Raspberry Pi (Docker Compose). ESP32 sends telemetry every 5 s over MQTT/TLS; Node-RED handles ingest + UI; SQLite stores data; optional Grafana for charts; Caddy works as reverse-proxy and OTA file server.

## Stack
- ESP32: telemetry + OTA via `esp_https_ota`
- MQTT broker: Eclipse Mosquitto (auth, ACL, only retained state)
- Node-RED: ingest -> DB, dashboard, commands, alerts, OTA orchestration
- DB: SQLite (simple time-series table, retention job)
- Caddy: reverse-proxy + static server for OTA files
- Cloudflare Tunnel or Tailscale: remote access without exposed ports

## Quick start (RPi/Docker)
1. Install Docker + Docker Compose.
2. `cp .env.example .env` and fill secrets (MQTT, Node-RED admin, Telegram bot/ID, Cloudflare/Tailscale token if used).
3. `docker compose up -d`
4. Open Node-RED (`http://<host>:1880`) and import `flows.json` (see Export/Import).
5. Configure Mosquitto users/ACL based on `mosquitto.conf`.
6. ESP32: set Wi-Fi, MQTT endpoint, TLS certs.

## Data flow and schema
- MQTT topic: `devices/<id>/telemetry` (QoS1, JSON).
- Table `reading` (SQLite):
  - ts TEXT default `strftime('%Y-%m-%dT%H:%M:%fZ','now')`
  - device_id TEXT
  - temperature_actual/wanted REAL
  - humidity_actual/wanted REAL
  - relay_heater INTEGER
  - relay_fan INTEGER
  - fw TEXT
- Retention: cron/Node-RED `cron-plus` -> `DELETE older than 400d` + `VACUUM`.

## Features
- Live dashboard: temperatures, SG, relay states, 24h chart.
- One-click: set temperature, pause heating.
- Downlink commands to ESP32 via MQTT (`cmd/setpoint`, `cmd/config`).
- OTA: upload firmware, auto SHA-256, manifest, device notification.
- Alerts (out of range, sensor loss) via Telegram.

## Operations
- Set temperature: dashboard slider -> MQTT `devices/<id>/cmd/setpoint`; ESP32 confirms in `state` within ~2 s.
- OTA: upload firmware to `/ota/<device>/`; Node-RED calculates SHA-256, writes `manifest.json`, publishes `fw/notify`.
- Backups: SQLite dump to `/backups` (cron). Restore: `sqlite3 fermentator.db < dump.sql`.

## Export/Import Node-RED flows
- The single source of truth for flows is `flows.json` in the repo (versioned in git).
- It is **not** auto-exported on `git push`. Before you commit/push, export the current flows from the running Node-RED instance and save to `flows.json`, then commit/push.
- Manual export (UI): Node-RED menu -> Export -> Clipboard -> All flows -> save into `flows.json` in the repo.
- Manual export (API, for script or pre-push hook):
  - Set `NODE_RED_API_KEY` (see `settings.js`).
  - `curl -H "Node-RED-API-KEY: $NODE_RED_API_KEY" http://localhost:1880/flows -o flows.json`
  - Commit `flows.json` together with code changes.
- Import after rebuild/on a new machine:
  - Start the docker stack, open Node-RED, Import -> select `flows.json` -> Deploy.
  - API option: `curl -X POST -H "Node-RED-API-KEY: $NODE_RED_API_KEY" -H "Content-Type: application/json" --data @flows.json http://localhost:1880/flows` then Deploy.
- Automation idea (optional): add a pre-push hook or a cron job that calls the export API before `git push` so the repo always carries the latest `flows.json`. This is opt-in and does not happen by default.

## Roadmap (milestones)
- M2 Ingest & Dashboard: ingest flow, dashboard + 24h chart; DoD: new rows in `reading`, dashboard shows live values.
- M3 Commands: UI slider for setpoint, confirmation within 2 s.
- M4 OTA: A/B rollback test, version visible in `state`.
- M5 Alerts & Retention: Telegram/email, delete >400d, VACUUM.
- M6 Backups & Docs: `backup.sh`, restore guide, export/import Node-RED flows.

## To-Do
- Add retention job (cron/Node-RED).
- Finish bidirectional Node-RED <-> ESP32 control for temperature changes.
- Solve regular `flows.json` export (script/hook) and SQLite backups.
