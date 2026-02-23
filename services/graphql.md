# tentacle-graphql

GraphQL API with real-time subscriptions for the Tentacle platform.

## Tech Stack

- graphql-yoga
- Pothos schema builder
- NATS for data source

## Port

Default: **4000**

## Key Features

- **Real-time variable subscriptions**: Subscribes to `*.data.>` on NATS for updates from all modules
- **Service topology**: Reads `service_heartbeats` KV bucket to return live service list
- **Log streaming**: Subscribes to `service.logs.{serviceType}.>` and forwards via GraphQL subscription
- **NATS traffic inspector**: Real-time view of NATS message traffic
- **Browse tags**: Async browse mutation with progress subscription (for EtherNet/IP and OPC UA)
- **Network management**: Forwards network state and configuration commands
- **nftables NAT management**: Forwards NAT config state and apply commands

## GraphQL Subscriptions

```graphql
# All variable updates from all modules
subscription {
  variableUpdates {
    moduleId
    variableId
    value
    datatype
    timestamp
  }
}

# Updates for a specific variable
subscription {
  variableChanged(variableId: "temperature") {
    value
    timestamp
  }
}

# Real-time service logs
subscription {
  serviceLogs(serviceType: "ethernetip") {
    timestamp
    level
    message
    moduleId
    logger
  }
}

# Real-time NATS message traffic (optional filter)
subscription {
  natsTraffic(filter: "ethernetip.>") {
    subject
    payload
    timestamp
  }
}

# Live network interface state
subscription {
  networkState {
    interfaces { name operstate addresses { address } }
  }
}

# Live nftables NAT config
subscription {
  nftablesConfig {
    natRules { id natAddr deviceAddr deviceName enabled }
  }
}

# Browse progress for async browse operations
subscription {
  browseProgress(browseId: "...") {
    phase
    totalTags
    completedTags
    message
  }
}
```

## Key Files

| File | Purpose |
|------|---------|
| `main.ts` | Entry point, NATS + HTTP server setup |
| `server.ts` | graphql-yoga server configuration |
| `schema/schema.ts` | Schema assembly |
| `schema/types.ts` | GraphQL type definitions (Pothos) |
| `schema/queries.ts` | Queries (variables, services, etc.) |
| `schema/mutations.ts` | Mutations (browse, command, apply, etc.) |
| `schema/subscriptions.ts` | Real-time subscriptions |
| `nats/client.ts` | NATS connection, pub/sub, request helpers |
| `nats/watcher.ts` | Variable update async iterable with filtering |
| `modules/logs.ts` | Log streaming, ring buffer (200 entries per service) |
| `modules/nats-traffic.ts` | NATS traffic inspection |
| `modules/network.ts` | Network state forwarding |
| `modules/nftables.ts` | nftables state/config forwarding |
