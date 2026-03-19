# Tentacle Platform

Distributed IIoT platform built on Deno with NATS as the message bus. Bridges industrial devices to MQTT/GraphQL for real-time monitoring and control.

## Architecture Overview

```
┌──────────────────┐         ┌──────────────────┐
│ tentacle-plc     │         │ tentacle-web     │
│ (PLC Runtime)    │         │ (SvelteKit UI)   │
└────────┬─────────┘         └────────┬─────────┘
         │                            │
         │ NATS Topics                │ GraphQL (SSE)
         ▼                            ▼
    ┌────────────────────────────────────────────────┐
    │        NATS Message Bus + JetStream + KV       │
    └────────────────────────────────────────────────┘
     │            │            │            │
┌────▼──────┐ ┌───▼──────┐ ┌───▼──────┐ ┌───▼──────────┐
│ tentacle- │ │ tentacle-│ │ tentacle-│ │ tentacle-    │
│ ethernetip│ │ opcua-go │ │ modbus   │ │ graphql      │
└────┬──────┘ └───┬──────┘ └───┬──────┘ └──────────────┘
     │            │            │
     ▼            ▼            ▼
Allen-Bradley  OPC UA       Modbus TCP
PLCs           Servers      Devices
     │
     └──── tentacle-mqtt ──── MQTT Broker (Sparkplug B)
```

## Services

| Service | Purpose | Runtime | Port |
|---------|---------|---------|------|
| **tentacle-plc** | Lightweight PLC runtime library with task-based programming | Deno (library) | N/A |
| **tentacle-ethernetip-go** | Polls Allen-Bradley PLCs via EtherNet/IP (Go + libplctag), publishes to NATS | Go | N/A |
| **tentacle-opcua-go** | OPC UA client, subscribes to nodes, publishes to NATS | Go | N/A |
| **tentacle-modbus** | Modbus TCP scanner, polls registers/coils, publishes to NATS | Deno | N/A |
| **tentacle-modbus-server** | Modbus TCP server, exposes PLC data to Modbus clients | Deno | N/A |
| **tentacle-mqtt** | Bridges NATS to MQTT using Sparkplug B protocol | Deno | N/A |
| **tentacle-graphql** | GraphQL API with real-time subscriptions | Deno | 4000 |
| **tentacle-web** | SvelteKit frontend with D3 topology visualization | Node.js | 3012 |
| **tentacle-network** | Network interface monitoring and netplan configuration | Deno | N/A |
| **tentacle-nftables** | nftables NAT rule management | Deno | N/A |
| **tentacle-nats-schema** | Shared TypeScript types and message validators | N/A (library) | N/A |

## Running Services

```bash
# All services (recommended)
./dev.sh

# Individual Deno services
cd tentacle-{service} && deno task dev

# Web UI (Node.js — NOT Deno)
cd tentacle-web && npm run dev
```

## Environment Variables

```bash
# Shared (all Deno services)
NATS_SERVERS=localhost:4222

# tentacle-plc / tentacle-demo
PROJECT_ID=my-project

# tentacle-mqtt
MQTT_BROKER_URL=mqtt://localhost:1883
MQTT_GROUP_ID=TentacleGroup
MQTT_EDGE_NODE=EdgeNode

# tentacle-graphql
GRAPHQL_PORT=4000

# tentacle-web
GRAPHQL_URL=http://localhost:4000/graphql
```

## NATS Topics

```
# Variable data (any module publishing variables)
{moduleId}.data.{variableId}     # Variable value update
{moduleId}.command.{variableId}  # Write command to variable
{moduleId}.variables             # Request all variables (request/reply)
{moduleId}.shutdown              # Graceful shutdown

# EtherNet/IP specific
ethernetip.subscribe             # Add tag subscription
ethernetip.unsubscribe           # Remove subscription
ethernetip.browse                # Browse PLC tags
ethernetip.browse.progress.{browseId}  # Browse progress

# OPC UA specific
opcua.subscribe / opcua.unsubscribe / opcua.browse

# Modbus specific
modbus.subscribe / modbus.unsubscribe
modbus.data.{deviceId}           # Batch values (all tags per device, one subject)
modbus.command.{tagId}           # Write tag value

# Log streaming
service.logs.{serviceType}.{moduleId}

# Network
network.interfaces / network.state / network.command

# nftables
nftables.rules / nftables.state / nftables.command
```

## NATS KV Buckets

| Bucket | Key | Purpose |
|--------|-----|---------|
| `service_heartbeats` | `{moduleId}` | Service discovery (10s interval, 60s TTL) |
| `plc_variables` | `{variableId}` | Shared variable state |
| `plc-variables-{projectId}` | `{variableId}` | Per-project PLC variable persistence |
| `mqtt-config-{projectId}` | `{variableId}` | Per-variable MQTT settings |

## Key Message Types (from tentacle-nats-schema)

```typescript
// Variable data — the core message type
type PlcDataMessage = {
  moduleId: string;    // Source module (e.g., "ethernetip", "mixing-process")
  deviceId: string;
  variableId: string;
  value: number | boolean | string | Record<string, unknown>;
  timestamp: number;
  datatype: "number" | "boolean" | "string" | "udt";
  deadband?: { value: number; maxTime?: number };
  udtTemplate?: UdtTemplateDefinition;
};

// Service heartbeat for discovery
type ServiceHeartbeat = {
  serviceType: "ethernetip" | "plc" | "mqtt" | "graphql" | "modbus" | "modbus-server" | "opcua" | "network" | "nftables";
  moduleId: string;    // Unique module identifier
  lastSeen: number;
  startedAt: number;
  version?: string;
  metadata?: Record<string, unknown>;
};
```

## Service Discovery

Heartbeat-driven — no manual registration needed. Any service publishing valid heartbeats to the `service_heartbeats` KV bucket auto-appears in the topology. The web UI shows live topology from heartbeats.

## Data Flow Example

Each service publishes to its own NATS namespace. The web UI subscribes per-module:

1. `tentacle-ethernetip-go` polls PLC tags via libplctag (parallel goroutine reads)
2. Publishes to `ethernetip.data.{deviceId}.{tagName}` (its own namespace)
3. `tentacle-plc` subscribes to `ethernetip.data.>` for source variables
4. PLC processes/composes values, publishes to `{projectId}.data.{variableId}`
5. `tentacle-mqtt` subscribes to `*.data.>`, reads RBE settings from `mqtt-config-...` KV
6. If change exceeds deadband, publishes via Sparkplug B
7. `tentacle-graphql` subscribes to `{moduleId}.data.>` per client request, batches updates every 2.5s
8. `tentacle-web` receives batched SSE updates scoped to the page's module

### Per-Module Subscription Architecture

- **Devices page** (EtherNet/IP): `variableBatchUpdates(moduleId: "ethernetip")`
- **PLC variables page**: `variableBatchUpdates(moduleId: "{projectId}")`
- **MQTT metrics page**: Polls `mqttMetrics` via `invalidateAll()` (MQTT uses Sparkplug naming, not variable IDs)

## Key Dependencies

| Package | Purpose |
|---------|---------|
| `@nats-io/transport-deno` | NATS transport |
| `@nats-io/jetstream` | JetStream features |
| `@nats-io/kv` | Key-Value store |
| `@joyautomation/coral` | Logging |
| `@joyautomation/synapse` | Sparkplug B MQTT |
| `@joyautomation/tentacle-nats-schema` | Shared schemas |
| `@joyautomation/tentacle-plc` | PLC runtime (JSR) |

## Common Debugging

- **Service not in topology**: Check NATS heartbeat — `nats kv get service_heartbeats {moduleId}`
- **No data from PLC**: Check subscription exists in scanner; only subscribed tags are polled
- **MQTT not publishing**: Check `mqtt-config-{projectId}` KV has variable enabled
- **GraphQL subscription stale**: Check browser console for SSE errors; verify tentacle-graphql is running
- **EtherNet/IP not polling**: Verify subscription — EIP only polls subscribed tags

---

## tentacle-ethernetip-go Deep Dive

### Runtime

**Go + CGo + libplctag** — use `go run .` or `go build`. Requires `libplctag.so` installed.

### Key Architecture Decisions

1. **Subscription-based polling**: Only polls tags with active subscriptions (from tentacle-plc or MQTT config). Browse discovers available tags but doesn't continuously poll them.

2. **Parallel goroutine reads**: Tags are read in parallel batches using goroutines. libplctag internally uses CIP Multi-Service Packets for wire-level batching, reducing round-trips over high-latency links.

3. **Own NATS namespace**: Publishes to `ethernetip.data.{deviceId}.{tagName}` — multi-level subjects. Use `>` wildcard for subscriptions.

4. **UDT resolution**: Uses `@tags` for tag listing, `@udt/<id>` for UDT templates. Resolves TIMER, COUNTER, and custom UDTs with zero unnamed members.

5. **One-time value read on browse**: Reads all tag values once during browse so UI shows actual values instead of 0.

### Key Files

| File | Purpose |
|------|---------|
| `main.go` | Entry point, NATS connection, heartbeat, request handlers |
| `scanner.go` | Tag handle creation, poll loop, parallel read logic |
| `browse.go` | Tag listing via `@tags`, UDT template resolution |
| `types.go` | Shared types (PlcConnection, Variable, etc.) |

### Observability Features

- **Poll cycle timing**: "Poll cycle for rtu45: 1896 tags in 5ms"
- **Bad status warnings**: Tags that fail with PLCTAG_ERR_NOT_FOUND
- **Connection state**: Device connections/disconnections

### Gotchas

- **Multiple instances**: NATS request/reply answered by ANY running instance — stale data means old instances
- **libplctag NOT_FOUND**: Some UDT hidden members don't exist on PLC. After `maxCreateRetries` (3), skipped.
- **Cellular/high-latency**: Parallel reads help but initial tag creation for 1900+ tags can be slow
- **`>` vs `*` wildcards**: EIP data subjects are multi-level — always use `>`

### Common Commands

```bash
# Check for running instances
ps aux | grep ethernetip

# Build and run
cd tentacle-ethernetip-go && go run .

# Run in container
./tdev.sh restart tentacle-ethernetip-go
```

---

## tentacle-modbus Deep Dive

### Key Architecture Decisions

1. **Subscriber-driven**: Zero connections at startup. Devices connect on-demand when `modbus.subscribe` arrives.

2. **Block reads**: Contiguous tags grouped into single Modbus requests (max gap = 10 addresses). One TCP request per block per poll cycle.

3. **One subject per device**: `modbus.data.{deviceId}` carries all tag values in a batch, unlike EIP which uses per-tag subjects.

4. **Custom TCP client**: No Modbus library — pure Deno TCP. MBAP header + PDU, promise-based request/response by transaction ID.

### Supported Function Codes

| FC | Name | Access |
|----|------|--------|
| 01 | Read Coils | Read |
| 02 | Read Discrete Inputs | Read |
| 03 | Read Holding Registers | Read |
| 04 | Read Input Registers | Read |
| 05 | Write Single Coil | Write |
| 06 | Write Single Register | Write |
| 15 | Write Multiple Coils | Write |
| 16 | Write Multiple Registers | Write |

### Supported Datatypes

| Modbus Datatype | PLC Datatype | Register Count |
|----------------|-------------|----------------|
| `boolean` | `boolean` | 1 coil/bit |
| `int16` / `uint16` | `number` | 1 register |
| `int32` / `uint32` / `float32` | `number` | 2 registers |
| `float64` | `number` | 4 registers |

### Byte Order

- Per-device default: `ABCD` (big-endian, standard Modbus)
- Per-tag override: `ABCD`, `BADC`, `CDAB`, `DCBA`

---

## @joyautomation/synapse Deep Dive

### Event Signatures (IMPORTANT!)

```typescript
// DCMD event - receives (topic, payload), NOT just (payload)
node.events.on("dcmd", (topic: SparkplugTopic, payload: UPayload) => {
  // topic.deviceId - the device ID
  // payload.metrics - array of metrics with name/value
});
```

### Version History & Fixes

| Version | Fix |
|---------|-----|
| 0.0.86 | Re-subscribes to DCMD/NCMD/STATE topics on MQTT reconnect |
| 0.0.87 | `disconnect` transition only removes internal "ncmd" listener |

---

## tentacle-web Deep Dive

### Runtime

**Runs on Node.js / npm** — use `npm run dev` and `npm run build`. Do NOT use Deno commands.

### Server-Side Architecture

All GraphQL communication is server-side only:

```
Browser → SvelteKit Server → tentacle-graphql (port 4000)
         (proxy layer)
```

- **Initial data**: `+page.server.ts` files fetch data server-side
- **Mutations**: Client POSTs to `/api/graphql` → SvelteKit proxies to GraphQL
- **SSE Subscriptions**: Client connects to `/api/graphql/subscribe` → SvelteKit proxies SSE stream

### Svelte 5 Patterns

```typescript
// State
let count = $state(0);

// Derived
let doubled = $derived(count * 2);

// Effect (for reactive initialization with changing props)
$effect(() => {
  // runs when deps change
});
```

### Environment Variables

```bash
GRAPHQL_URL=http://localhost:4000/graphql  # Server-side only
```

---

## tentacle-ethernetip-go Write Support

### How Writes Work

1. Subscribes to `ethernetip.command.{variableId}`
2. Finds which PLC connection has that variable
3. Creates a libplctag write handle, encodes value, writes to PLC
