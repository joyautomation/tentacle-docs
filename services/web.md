# tentacle-web

SvelteKit frontend for the Tentacle platform.

## Tech Stack

- SvelteKit 2.9
- Svelte 5 (runes syntax — `$state`, `$derived`, `$effect`)
- TypeScript 5.6
- Vite 6
- D3 for topology visualization
- `@joyautomation/salt` design system

## Port

Development: **3012**

## Running

```bash
cd tentacle-web
npm install  # First time only
npm run dev
```

> **Important**: tentacle-web runs on Node.js/npm, not Deno. Do not use `deno run` or `deno task`.

## Key Features

- **Topology view**: D3-force graph showing live service topology. Nodes are clickable — navigate to service detail pages. Topology is heartbeat-driven; services appear automatically when their heartbeat is detected.
- **Service detail pages**: `/services/[serviceType]` — Overview tab + Logs tab with real-time log streaming
- **Real-time log streaming**: GraphQL subscription → SSE → LogViewer component with level filtering and auto-scroll
- **Variable monitoring**: Real-time variable values via GraphQL subscriptions
- **Network management**: View and configure network interfaces
- **NAT rules**: View and manage nftables NAT configuration

## Server-Side Architecture

All GraphQL communication is server-side only — the browser never talks to tentacle-graphql directly:

```
Browser → SvelteKit server → tentacle-graphql (port 4000)
```

- **Initial data**: `+page.server.ts` files load data server-side
- **Mutations**: Client POSTs to `/api/graphql` → SvelteKit proxies to GraphQL
- **Subscriptions**: Client connects to `/api/graphql/subscribe` → SvelteKit proxies SSE stream

## Key Files

| File | Purpose |
|------|---------|
| `src/lib/server/graphql.ts` | Server-side GraphQL client |
| `src/lib/graphql/client.ts` | Client-side helpers (route through proxy) |
| `src/routes/api/graphql/+server.ts` | POST proxy for mutations |
| `src/routes/api/graphql/subscribe/+server.ts` | SSE proxy for subscriptions |
| `src/routes/(app)/+page.svelte` | Topology view |
| `src/routes/(app)/services/[serviceType]/+page.svelte` | Service detail page |
| `src/lib/components/LogViewer.svelte` | Real-time log streaming component |

## Environment Variables

```bash
GRAPHQL_URL=http://localhost:4000/graphql  # GraphQL server endpoint (server-side only)
```

## Svelte 5 Gotchas

- Use `$state`, `$derived`, `$effect` — not legacy `writable`/`readable` stores
- `{@const}` must be a direct child of `{#each}`, `{#if}`, etc. — NOT inside plain HTML elements
- `$state` captures initial prop values; use `$effect` for reactive re-initialization on prop changes

## Design System

Uses `@joyautomation/salt` for UI components:

```typescript
import { state as saltState } from '@joyautomation/salt';  // alias to avoid Svelte treating as a store
saltState.addNotification({ type: 'success', message: 'Saved!' });
```
