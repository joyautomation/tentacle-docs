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
         │          │              │           │
    ┌────▼────┐ ┌───▼──────┐ ┌────▼────┐ ┌───▼─────────────┐
    │tentacle-│ │tentacle- │ │tentacle-│ │tentacle-        │
    │ethernet │ │modbus    │ │mqtt     │ │graphql          │
    │ip       │ │          │ │         │ │                 │
    └────┬────┘ └───┬──────┘ └────┬────┘ └─────────────────┘
         │          │             │
    Allen-Bradley  Modbus    MQTT Broker
    PLCs           TCP       (Sparkplug B)
                   Devices
```

## Tech Stack

- **Runtime**: Deno (most backend services), Go (tentacle-ethernetip-go, tentacle-opcua-go)
- **Frontend**: SvelteKit 2.9, Svelte 5, TypeScript 5.6, Vite 6
- **Messaging**: NATS with JetStream and KV stores
- **MQTT**: Sparkplug B via @joyautomation/synapse
- **GraphQL**: graphql-yoga + Pothos schema builder
- **EtherNet/IP**: Go + CGo + libplctag (CIP Multi-Service Packet batching)

## Data Flow

### Reading from PLCs (PLC → Cloud)

Each service publishes to its own NATS namespace. The web UI subscribes per-module:

1. `tentacle-ethernetip-go` polls PLC tags via libplctag (CIP batch reads)
2. Publishes to `ethernetip.data.{deviceId}.{tagName}` (its own namespace)
3. `tentacle-plc` subscribes to `ethernetip.data.>` for its source variables
4. PLC processes/composes values, publishes to `{projectId}.data.{variableId}` (its own namespace)
5. `tentacle-mqtt` subscribes to `*.data.>`, reads RBE settings from `mqtt-config-{projectId}` KV
6. If change exceeds deadband, publishes via Sparkplug B DDATA
7. `tentacle-graphql` subscribes to `{moduleId}.data.>` per client request, batches updates every 2.5s
8. `tentacle-web` receives batched SSE updates scoped to the page's module

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
