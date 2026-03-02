# Ideas and recommendations

After analyzing all five Argus repositories, here are practical suggestions that align with the current architecture and could improve reliability, developer experience, and product capabilities.

## Reliability and resilience

### Retry logic for worker uploads

The Vergil workers use a SQLite queue with status tracking, which is a solid foundation. Enhancing the retry logic with exponential backoff and a dead-letter state (after N failures) would make the system more resilient to prolonged API outages. Events in `error` state could be retried manually via the MQTT listener CLI.

### Health check endpoint on Flare

Flare currently has no `/health` endpoint. Adding one that returns the active room count and connection count would enable container orchestrators (Docker Compose health checks, Kubernetes liveness probes) to detect and restart unhealthy instances.

### Redis-backed session state for Flare

Flare stores all session state in memory, which means a single instance is the scaling limit. Adding Redis as a session backend (using `websockets` broadcast or a simple pub/sub layer) would enable running multiple Flare instances behind a load balancer.

### Graceful degradation for VOD upload

If the VOD upload fails mid-segment, the daemon should track partial uploads and resume rather than re-uploading entire segments. Implementing multipart upload with checkpointing would reduce bandwidth waste on stations with limited connectivity.

### Watchdog for Docker containers on stations

The Vergil daemon manages ffmpeg restream processes, but the Docker workers (detection, motion, MQTT) and Frigate rely on Docker's restart policy. Adding a lightweight watchdog loop in the daemon that checks container health via the Docker API and alerts on repeated restarts would catch issues faster.

## Developer experience

### Monorepo tooling or cross-repo scripts

The five repositories are tightly coupled but managed independently. Consider:
- A top-level `Makefile` or `justfile` in argus-one with commands like `make start-all`, `make test-all`, `make status` that operate across repos
- Git submodules or a simple `clone-all.sh` script for onboarding new developers
- Shared environment variable templates that validate all required vars are set before starting

### Automated integration tests

The connection points between repos (workers to API, daemon to Supabase, Flare to Redis) are tested manually. A small integration test suite that spins up the stack with Docker Compose and verifies end-to-end flows (event published on MQTT -> appears in Supabase) would catch breaking changes early.

### OpenAPI client generation

Galleon already has an OpenAPI spec at `public/openapi.yaml`. Generating a typed Python client from this spec for use in Vergil workers would eliminate manual HTTP request construction and keep the workers in sync with API changes automatically.

### Shared type definitions

Station metrics, device status, and event structures are defined in both TypeScript (Galleon) and Python (Vergil). A shared schema definition (JSON Schema or Protocol Buffers) with code generation for both languages would prevent the two sides from drifting apart.

## Product features

### Station alerting system

The metrics infrastructure is already in place (Vergil collects CPU, GPU, RAM, temp, network, storage every 60 seconds). Adding threshold-based alerts would be high-value with low effort:
- Storage > 90%: warn that Frigate recordings may fail
- Temperature > 80C: warn of thermal throttling
- Network I/O drop to zero: connectivity lost
- No heartbeat for 5+ minutes: station offline

Alerts could be delivered via Supabase edge functions sending emails or webhook notifications.

### Camera health monitoring

Frigate knows when a camera feed drops. Exposing camera-level status (connected, disconnected, degraded frame rate) through the existing MQTT -> worker -> API pipeline would let Galleon show per-camera health rather than just per-station health.

### Recording timeline gaps visualization

The VOD timeline player could highlight gaps in recordings (periods where no segments were uploaded). This helps operators identify connectivity issues or storage problems without manually scrubbing through hours of footage.

### Thunder Board telemetry in the dashboard

The Thunder Board collects per-channel power data and temperature readings via USB-UART. If the Vergil daemon reads this data (serial port to MCU) and includes it in the metrics report, Galleon could display power consumption per channel, board temperature distribution, and alert on anomalies like overcurrent or overheating -- turning the dashboard into a complete station health monitor.

### Clip sharing and export

Adding the ability to share detection clips via a temporary public URL (time-limited signed URL from R2) would enable users to quickly share incidents with security teams or law enforcement without requiring them to have Galleon accounts.

## Architecture improvements

### Redis caching for Galleon

Galleon fetches device lists, station status, and event histories from Supabase on every page load. Adding a Redis cache layer for frequently accessed data like station status and recent events would reduce Supabase load and improve dashboard response times. Redis is already used in the Vergil VOD pipeline, so the operational knowledge exists in the team.

### Event deduplication

If a worker crashes and restarts, it may re-process MQTT events that were already uploaded. Adding idempotency keys (based on Frigate event ID + timestamp) to the ingestion API would prevent duplicate events in the database.

### Structured logging across services

Adopting a consistent structured logging format (JSON logs with `service`, `station_id`, `event_type`, `timestamp` fields) across Vergil daemon, workers, and Flare would make centralized log aggregation and debugging much easier. Tools like Loki or even CloudWatch Logs could then correlate events across services.

### MQTT QoS upgrade

The current Mosquitto configuration likely uses QoS 0 (fire-and-forget). For detection events where data loss matters, upgrading to QoS 1 (at-least-once delivery) between Frigate and the workers would ensure events survive brief broker restarts. Combined with the idempotency recommendation above, this provides at-least-once without duplicates.

## Infrastructure

### Container image registry

Building Docker images on each Jetson is slow and error-prone. Publishing pre-built images to a container registry (GitHub Container Registry is free for public repos) would speed up deployments and ensure all stations run identical versions.

### Automated station updates

Currently, updating a station requires SSH access, `git pull`, and service restarts. A simple pull-based update mechanism -- the daemon checks a version endpoint periodically and triggers an update script when a new version is available -- would enable fleet-wide updates without manual intervention.

### Centralized configuration management

Each station's configuration is generated locally by `init_station.sh`. As the fleet grows, managing per-station configs becomes harder. A configuration service in Supabase (a `station_config` table) that the daemon fetches on startup would centralize configuration and enable remote reconfiguration without SSH.
