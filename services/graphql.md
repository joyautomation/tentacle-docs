# tentacle-graphql

GraphQL API with real-time subscriptions for the Tentacle platform.

## Tech Stack

- graphql-yoga
- Pothos schema builder
- NATS for data source

## Port

Default: 4000

## Key Features

- Real-time subscriptions via WebSocket
- Watches `plc_variables` NATS KV bucket for updates
- Browse tags mutation with async progress support
- Service health via `service_heartbeats` KV

## Key Files

| File | Purpose |
|------|---------|
| `schema/types.ts` | GraphQL type definitions |
| `schema/mutations.ts` | Mutations (browse, subscribe, etc.) |
| `schema/subscriptions.ts` | Real-time subscriptions |
| `nats/client.ts` | NATS connection and helpers |
| `nats/watcher.ts` | KV bucket watchers |

## Browse Progress Subscription

For large PLCs, browse can be async with progress:

```graphql
mutation {
  browseTags(projectId: "my-project", async: true) {
    browseId
  }
}

subscription {
  browseProgress(browseId: "...") {
    phase
    totalTags
    completedTags
    message
  }
}
```
