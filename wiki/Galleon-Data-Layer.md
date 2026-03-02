# Galleon data layer

Galleon uses Supabase as its primary backend -- PostgreSQL for structured data, Supabase Auth for user management, Supabase Storage for small files, and Cloudflare R2 for larger media assets. The data access layer provides typed helpers that abstract Supabase's client SDK.

## Database access patterns

Two helper modules exist for database operations:

### Modern helpers (`lib/supabase.ts`)

The preferred approach for new code. Provides typed functions that wrap Supabase's query builder:

```typescript
import { query, queryOne, insert, update, remove } from '@/lib/supabase';

// Fetch with filters, ordering, and pagination
const devices = await query<Device>('devices', {
  filter: [{ column: 'user_id', value: userId }],
  order: { column: 'created_at', ascending: false },
  limit: 10
});

// Single record lookup
const device = await queryOne<Device>('devices', {
  filter: [{ column: 'id', value: deviceId }]
});

// Write operations
await insert<Device>('devices', { name: 'Station Alpha' });
await update<Device>('devices', deviceId, { status: 'active' });
await remove('devices', deviceId);
```

### Legacy helpers (`lib/db.ts`)

An older SQL-style abstraction that parses SQL strings into Supabase API calls. Still used in some pages but not recommended for new code.

## Supabase client configuration

Galleon creates two Supabase clients:

- **Client-side** (`supabase` from `lib/supabase.ts`): uses the public anon key. Safe for browser usage. Respects RLS policies scoped to the authenticated user.
- **Server-side** (`createServerClient()`): uses the service role key. Bypasses RLS for admin operations like user registration and data migrations. Only used in API routes.

## Type system

All TypeScript types live in the `types/` directory with a barrel export at `types/index.ts`:

| File | Domain |
|---|---|
| `user.ts` | User accounts and profiles |
| `subscription.ts` | Subscription plans and status |
| `order.ts` | Order tracking |
| `invoice.ts` | Billing invoices |
| `device.ts` | IoT devices (stations, cameras) |
| `device-metric.ts` | Device telemetry (CPU, GPU, RAM, temp, network, storage) |
| `billing.ts` | Payment methods and billing info |
| `api.ts` | API response wrappers |
| `schemas.ts` | Shared schemas (e.g., `MetricTimeframe`) |

## Authentication

Supabase Auth handles two authentication methods:

- **Google OAuth** -- primary flow for end users. The `/callback` route handles the OAuth redirect.
- **Email/password** -- alternative flow with email verification. Also used by station machine accounts.

Session tokens are managed by the Supabase client SDK and automatically refreshed.

## Multi-tenancy model

Organizations are the tenancy boundary:

- Each user belongs to one or more organizations.
- Each organization has a role hierarchy: **owner** > **admin** > **member**.
- Stations (devices) are owned by organizations, not individual users.
- Supabase RLS policies enforce data isolation at the database level.
- The invitation system (`/api/invitations/`) allows owners to invite new members by email.

## Storage

Two storage backends serve different purposes:

| Backend | Usage | Access |
|---|---|---|
| Supabase Storage | User avatars, small assets | Supabase SDK |
| Cloudflare R2 | VOD segments, clips, thumbnails | S3-compatible API |

The `lib/storage.ts` module abstracts both backends, routing uploads and fetches to the appropriate service.

## Real-time subscriptions

Galleon uses two real-time channels:

1. **Supabase Realtime** -- subscribes to database changes (new events, status updates) via Supabase's built-in Realtime service.
2. **Flare (Socket.IO)** -- connects to the Flare WebSocket server for station presence (online/offline status) and viewer counts. See [[Flare-Overview]] for details.
