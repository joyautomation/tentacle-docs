# Architecture Overview

## System Diagram

```
┌──────────────────┐         ┌──────────────────┐
│ tentacle-plc     │         │ tentacle-web     │
│ (PLC Runtime)    │         │ (SvelteKit UI)   │
└────────┬─────────┘         └────────┬─────────┘
         │                            │
         │ NATS Topics                │ GraphQL
         ▼                            ▼
    ┌────────────────────────────────────────────────┐
    │        NATS Message Bus + JetStream + KV       │
    └────────────────────────────────────────────────┘
         │                     │                │
    ┌────▼────────┐   ┌────────▼────┐   ┌──────▼──────────┐
    │ tentacle-    │   │ tentacle-   │   │ tentacle-       │
    │ ethernetip   │   │ mqtt        │   │ graphql         │
    └────┬────────┘   └────┬───────┘   └─────────────────┘
         │                 │
    Allen-Bradley     MQTT Broker
    PLCs              (Sparkplug B)
```

## Tech Stack

- **Runtime**: Deno (all backend services)
- **Frontend**: SvelteKit 2.9, Svelte 5, TypeScript 5.6, Vite 6
- **Messaging**: NATS with JetStream and KV stores
- **MQTT**: Sparkplug B via @joyautomation/synapse
- **GraphQL**: graphql-yoga + Pothos schema builder

## Data Flow

### Reading from PLCs (PLC → Cloud)

1. `tentacle-ethernetip` polls PLC tag "Temperature" = 20.5
2. Publishes to NATS topic `plc.data.my-project.Temperature` with deadband config
3. `tentacle-mqtt` subscribes to `plc.data.my-project.>`
4. Reads RBE settings from `mqtt-config-my-project` KV bucket
5. If change exceeds deadband, publishes via Sparkplug B DDATA
6. `tentacle-graphql` watches `plc_variables` KV bucket
7. Emits GraphQL subscription update to connected clients

### Writing to PLCs (Cloud → PLC)

```
Ignition/SCADA → DCMD → MQTT Broker → tentacle-mqtt → NATS → tentacle-ethernetip → PLC
```

1. Sparkplug client (e.g., Ignition) sends DCMD message
2. `tentacle-mqtt` receives DCMD via synapse library
3. Extracts metric name/value, publishes to NATS `{projectId}/{variableId}`
4. `tentacle-ethernetip` receives, encodes value, calls `writeTag()` to PLC

## Report By Exception (RBE)

Traffic reduction via deadband filtering (80-95% reduction):

```typescript
deadband: {
  value: 0.5,      // Only publish if change > threshold
  maxTime: 60000   // Force publish at least every N ms
}
```

Configured per-variable, flows through entire stack:
1. Defined in tentacle-plc/ethernetip
2. Included in NATS messages
3. Honored by tentacle-mqtt when publishing to Sparkplug B

## Key Dependencies

| Package | Purpose |
|---------|---------|
| @nats-io/transport-deno | NATS transport |
| @nats-io/jetstream | JetStream features |
| @nats-io/kv | Key-Value store |
| @joyautomation/coral | Logging |
| @joyautomation/dark-matter | Configuration |
| @joyautomation/synapse | Sparkplug B MQTT |
| @joyautomation/tentacle-nats-schema | Shared schemas |
