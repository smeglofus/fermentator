# Fermentator
Minimal fermentation controller/monitor built for a single Raspberry Pi with Docker Compose. Telemetry from ESP32 every 5 s via MQTT; processing and UI in Node‑RED; data persisted in PostgreSQL; optional Grafana for charts; Caddy for reverse‑proxy and OTA file serving.


## Architecture Overview


**ESP32**: telemetry every 5 s over MQTT/TLS; downlink commands to change temperature setpoint; OTA via HTTPS (esp_https_ota).

**MQTT broker:** Eclipse Mosquitto (auth, ACL; only state is retained).

**Node‑RED** – flows (ingest→DB), dashboard, commands, alerts, OTA orchestration

**DB:** PostgreSQL – simple time‑series table + retention job

**Grafana:** dashboards & alerting.

**Caddy** reverse‑proxy + static server for OTA files

**Cloudflare Tunnel:** public access without router port‑forwarding. or **Tailscale** – private remote access (no public ports)


## Features

Live dashboard (temps, SG, relay status) + charts

One‑click actions: set temperature, pause heating

Downlink commands to ESP32 via MQTT (cmd/setpoint, cmd/config)

OTA: upload FW, auto SHA‑256, update manifest, notify device

Basic alerts (out‑of‑range, sensor loss) via Telegram
