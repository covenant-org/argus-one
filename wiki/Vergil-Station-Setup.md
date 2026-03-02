# Vergil station setup

Each Argus station is a Jetson device that needs to be initialized before it can join the network. The initialization process generates configuration files, registers the station with Supabase, and prepares the environment for the daemon and Docker containers.

## Initialization script

The `scripts/init_station.sh` script automates the setup:

```bash
# First run (defaults to stage environment)
./scripts/init_station.sh

# Production environment
./scripts/init_station.sh --env prod

# Overwrite existing configuration
./scripts/init_station.sh --env stage --force
```

### What the script does

1. Reads the Jetson's `/etc/machine-id` to derive the station's `HW_CODE`
2. Generates a Supabase email (`{HW_CODE}@station.covenant.space`) and random password
3. Resolves environment-specific URLs (Supabase, API endpoints)
4. Writes the `.env` file (used by Docker containers)
5. Writes `/etc/vergil/variables.conf` (used by the systemd daemon service)
6. Registers the station user in Supabase (if a service role key is provided)

### Generated files

| File | Used by | Contents |
|---|---|---|
| `.env` | Docker containers (docker-compose) | All environment variables |
| `/etc/vergil/variables.conf` | systemd daemon service | Same variables in systemd format |

### Environment support

The script supports three environments with different Supabase instances and API URLs:

| Environment | Use case |
|---|---|
| `local` | Development with Supabase CLI (built-in test credentials) |
| `stage` | Staging/testing environment |
| `prod` | Production deployment |

> [!IMPORTANT]
> Supabase credentials (URL, anon key, service role key) must be set as shell environment variables before running the script. They are not hardcoded for security reasons.

## Post-initialization steps

After running the script, complete the setup:

1. **Start the daemon** as a systemd service:
   ```bash
   sudo cp extras/daemon.service /etc/systemd/system/
   sudo systemctl daemon-reload
   sudo systemctl enable daemon.service
   sudo systemctl start daemon.service
   ```

2. **Start Docker containers**:
   ```bash
   docker-compose up -d
   ```

3. **Register cameras** for the station (currently a manual step).

## Services running on a station

Once fully deployed, six services run on each station:

| Service | Type | Function |
|---|---|---|
| `daemon` | systemd | Heartbeat, metrics, VOD, presence |
| `frigate` | Docker | NVR with AI detection |
| `mosquitto` | Docker | Local MQTT broker |
| `worker-mqtt` | Docker | Raw event forwarding |
| `worker-detection` | Docker | Detection clip packaging |
| `worker-motion` | Docker | Motion event packaging |

## Frigate configuration

Frigate's behavior is controlled by a YAML config file in `config/`. It defines:
- Camera sources (RTSP URLs)
- Detection zones (areas of interest per camera)
- Object detection settings (which objects to detect, confidence thresholds)
- Recording retention (default: 30 days of continuous recording)
- MQTT topic configuration for event publishing

## Mosquitto configuration

The local Mosquitto broker runs on port 1883 with a simple configuration:
- No authentication required (local-only communication)
- Persistence enabled for message durability
- Frigate publishes to `frigate/events` and `frigate/+/+` topics
- Workers subscribe to relevant topics based on their specialization
