# Fermentator
An IoT platform for monitoring and controlling fermentation box (ESP32 → MQTT → Go Ingest → TimescaleDB → FastAPI → Next.js).
Goal: a home project that mirrors a production-like stack (k3s, Helm, CI/CD, Cloudflare Tunnel).


## Architecture Overview

**ESP32**: telemetry every 5 s over MQTT/TLS; downlink commands to change temperature setpoint; OTA via HTTPS (esp_https_ota).

**MQTT broker:** Eclipse Mosquitto (auth, ACL; only state is retained).

**Ingest (Go):** subscribes to MQTT, validates, writes to TimescaleDB, maintains Redis cache of last readings, and dispatches commands (Redis Streams → MQTT devices/<id>/cmd).

**API (FastAPI):** REST for devices, firmware, commands; also serves OTA binaries over HTTPS.

**Web (Next.js):** dashboard, graphs, setpoint control.

**DB:** TimescaleDB (hypertables, continuous aggregates, compression, retention).

**Redis:** cache of last values and command queue (Streams).

**Grafana:** dashboards & alerting.

**k3s + Traefik:** Kubernetes single‑node with Ingress & TLS termination.

**Cloudflare Tunnel:** public access without router port‑forwarding.


