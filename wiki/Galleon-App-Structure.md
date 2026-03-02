# Galleon app structure

Galleon uses the Next.js App Router with route groups to separate public, authentication, and protected areas. Each area has its own layout that controls the visual shell around pages.

## Route tree

```
app/
├── layout.tsx                    # Root: fonts, metadata, global providers
├── (auth)/                       # Auth pages (minimal centered layout)
│   ├── layout.tsx
│   ├── login/
│   ├── signup/
│   ├── forgot-password/
│   ├── reset-password/
│   ├── verify-email/
│   └── callback/                 # OAuth redirect handler
├── account/                      # Protected dashboard (sidebar layout)
│   ├── layout.tsx
│   ├── dashboard/                # Overview: station status, recent events
│   ├── devices/                  # Device list
│   │   └── [id]/                 # Device detail
│   │       ├── stream/           # Full-screen live view
│   │       ├── recordings/       # VOD timeline player
│   │       ├── clips/            # Detection clip gallery
│   │       ├── events/           # Event log
│   │       └── settings/         # Device configuration
│   ├── subscriptions/            # Plan management
│   ├── orders/                   # Order history
│   ├── billing/                  # Payment methods, invoices
│   └── settings/                 # Profile, security, org management
├── (legal)/                      # Public legal pages
│   ├── privacy/
│   └── terms/
├── api/                          # API routes (server-side)
│   ├── vod/segment/upload/       # VOD segment ingestion
│   ├── clip/                     # Detection clip upload
│   ├── motions/                  # Motion bundle upload
│   ├── devices/                  # Device CRUD
│   ├── stations/                 # Station management
│   ├── organizations/            # Org CRUD
│   ├── invitations/              # Invitation system
│   ├── webhooks/stripe/          # Stripe payment webhooks
│   └── auth/                     # Auth helpers
├── api-docs/                     # Swagger UI page
└── shop/                         # Product catalog
```

## Layout hierarchy

The app uses three nested layouts:

1. **Root layout** (`app/layout.tsx`) defines HTML structure, loads Geist Sans and Geist Mono fonts, and sets global metadata.

2. **Auth layout** (`app/(auth)/layout.tsx`) wraps login/signup pages with a minimal centered design -- logo, background effect, and a constrained content area.

3. **Account layout** (`app/account/layout.tsx`) provides the full dashboard shell:
   - Responsive sidebar (collapsible on tablet, drawer on mobile)
   - Top header with menu toggle
   - Footer
   - Background visual effect

## Component organization

Components live in three directories, each with a barrel export (`index.ts`):

### `components/ui/`

Base UI primitives used across the app. Each component is a single `index.tsx` file using inline Tailwind classes.

`Button`, `Input`, `Select`, `Checkbox`, `Modal`, `Table`, `Card`, `Pagination`, `StatusPill`, `Logo`

### `components/layout/`

Structural components that form the page shell.

`BackgroundEffect`, `Header`, `Sidebar`, `Footer`

### `components/cards/`

Domain-specific card components that combine UI primitives with business logic.

`BalanceCard`, `CreditsCard`, `DeviceInfoCard`, `MetricCard`, `OrderInfoCard`, `PaymentMethodCard`, `ProfileCard`, `UsersTableCard`

### `components/players/`

Video player components for streaming and playback.

`HLSPlayer`, `WebRTCPlayer`, `TimelinePlayer`

## Navigation

The sidebar defines the primary navigation structure:

| Route | Section |
|---|---|
| `/account/dashboard` | Home |
| `/account/devices` | Device management |
| `/account/subscriptions` | Subscriptions |
| `/account/orders` | Order history |
| `/account/billing` | Billing and payments |
| `/account/settings` | Account settings |
| `/shop` | Product catalog (mobile only) |
| `/help` | Help center (mobile only) |

Active route highlighting works automatically by matching the current pathname.

## Server vs client components

- **Layout files** use `'use client'` for interactivity (sidebar toggle, navigation state).
- **Page components** are typically server components for data fetching.
- **UI components** that need state or event handlers use `'use client'`.
- **Player components** are always client components (browser APIs required).

## Styling approach

Galleon uses Tailwind CSS v4 with CSS variables defined in `app/globals.css`. There are no CSS Modules -- the migration from 45 `.module.css` files is complete. Shared patterns live in `lib/styles.ts` as reusable Tailwind class constants.

> [!TIP]
> See the project's `STYLES.md` for the full style reference, including CSS variable inventory, Tailwind v4 patterns, and component examples.
