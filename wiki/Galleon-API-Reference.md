# Galleon API reference

Galleon exposes a set of API routes under `app/api/` that serve two audiences: the dashboard frontend and the Vergil edge stations. The full specification is available as an OpenAPI 3.0 document at `/api-docs` (Swagger UI) or `/openapi.yaml` (raw spec).

## Ingestion endpoints

These routes receive data uploaded from Vergil workers running on each station. They require station authentication (Supabase JWT).

### `POST /api/vod/segment/upload`

Receives VOD recording segments from the Vergil daemon's VOD module. Each request contains an HLS segment (`.ts` file) and associated metadata. The API stores the segment in Cloudflare R2 and indexes it in Supabase for timeline playback.

### `POST /api/clip`

Receives detection clip bundles from the detection worker. A bundle includes:
- Thumbnail image (JPEG)
- Video clip (MP4)
- JSON metadata (label, confidence, zone, timestamp, camera, station)

### `POST /api/motions`

Receives motion event bundles from the motion worker. A bundle includes:
- Thumbnail image (JPEG)
- JSON metadata (zone, timestamp, camera, station)

## Resource endpoints

Standard CRUD routes for the dashboard frontend. All require user authentication and respect organization-level RLS policies.

### Devices and stations

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/devices` | List devices for the current organization |
| GET | `/api/devices/:id` | Device detail |
| POST | `/api/devices` | Register a new device |
| PATCH | `/api/devices/:id` | Update device configuration |
| GET | `/api/stations` | List stations with health status |

### Organizations and invitations

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/organizations` | List user's organizations |
| POST | `/api/organizations` | Create a new organization |
| PATCH | `/api/organizations/:id` | Update organization |
| POST | `/api/invitations` | Send an invitation email |
| GET | `/api/invitations` | List pending invitations |
| PATCH | `/api/invitations/:id` | Accept or reject invitation |

### Payments

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/api/webhooks/stripe` | Stripe webhook handler for payment events |

### Authentication

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/api/auth/*` | Auth helper routes (token refresh, session management) |

## API documentation

The OpenAPI spec lives at `public/openapi.yaml` and renders through Swagger UI at the `/api-docs` page. When API routes change, the spec should be updated to stay in sync.

> [!IMPORTANT]
> Ingestion endpoints (`/api/vod/*`, `/api/clip`, `/api/motions`) validate that the request comes from an authenticated station user. The station's `machine-id` is matched against the registered device to ensure stations can only upload data for their own cameras.
