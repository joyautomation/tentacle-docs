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
| **tentacle-ethernetip** | Polls Allen-Bradley PLCs via EtherNet/IP, publishes to NATS | Deno | N/A |
| **tentacle-opcua-go** | OPC UA client, subscribes to nodes, publishes to NATS | Go | N/A |
| **tentacle-modbus** | Modbus TCP scanner, polls registers/coils, publishes to NATS | Deno | N/A |
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
  serviceType: "ethernetip" | "plc" | "mqtt" | "graphql" | "modbus" | "opcua" | "network" | "nftables";
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

1. `tentacle-ethernetip` polls PLC tag "Temperature" = 20.5
2. Publishes to `ethernetip.data.Temperature`
3. `tentacle-mqtt` subscribes to `*.data.>`, reads RBE settings from `mqtt-config-...` KV
4. If change exceeds deadband, publishes via Sparkplug B
5. `tentacle-graphql` subscribes to `*.data.>`, forwards via GraphQL subscription
6. `tentacle-web` receives SSE update, renders new value

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

## tentacle-ethernetip Deep Dive

### Key Architecture Decisions

1. **Subscription-based polling**: Only polls tags with active subscriptions (from tentacle-plc or MQTT config). Browse discovers available tags but doesn't continuously poll them.

2. **One-time value read on browse**: Reads all tag values once during browse so UI shows actual values instead of 0/"unknown".

3. **NATS KV for caching**: Variable cache stored in NATS JetStream KV. Use `kv.purge()` not `kv.delete()` for complete removal.

4. **MQTT config change handling**: Scanner listens for `mqtt-config-...` KV changes to dynamically subscribe/unsubscribe tags.

### Key Files

| File | Purpose |
|------|---------|
| `scanner.ts` | Main polling loop, connection management, browse handler |
| `template.ts` | UDT template parsing |
| `mqttConfig.ts` | MQTT variable configuration via NATS KV |
| `main.ts` | Entry point, `--clear-cache` flag uses `kv.purge()` |

### CIP Protocol Reference

| Type Code | Type |
|-----------|------|
| 0xC1 | BOOL |
| 0xC2 | SINT |
| 0xC3 | INT |
| 0xC4 | DINT |
| 0xCA | REAL |
| 0xCB | LREAL |

- `typeCodeToDatatype()` converts CIP type codes to string names
- ForwardOpen may fail (status 0x1) — falls back to UCMM (unconnected messaging)
- Template parsing for UDTs can have alignment issues — **trust actual PLC response typeCode over template**

### Observability Features

- **Connection state tracking**: disconnected → connecting → connected
- **Exponential backoff**: 2s, 4s, 8s... up to 60s for reconnection
- **Periodic status logging**: Every 30s with sample values

### Bugs Previously Fixed (Don't Reintroduce!)

1. **Wrong UDT member datatypes** (VALUE showing as "boolean" instead of "number"):
   - Root cause: `decodeTagValue()` checked cached datatype BEFORE typeCode from PLC response
   - Fix: Prioritize typeCode from actual PLC response over cached datatype

2. **Stale data appearing after refresh**:
   - Cause: Multiple instances running simultaneously; NATS requests answered by old instances
   - Fix: Kill all instances before starting fresh (`pkill -f tentacle`)

3. **Newly enabled MQTT tags not being polled**:
   - Cause: `autoSubscribeMqttTags()` only called at startup
   - Fix: Added `handleMqttConfigChange()` for dynamic subscription sync

### Common Commands

```bash
# Check for running instances
ps aux | grep tentacle

# Kill all tentacle processes
pkill -f tentacle

# Start with cache clear
deno run -A main.ts --clear-cache

# Verbose debug output
DEBUG=* deno task dev

# Check NATS KV cache
nats kv get field-config-tentacle-dev cache.variables.rtu45
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

## tentacle-ethernetip Write Support

### How Writes Work

1. Subscribes to NATS subject `>` and filters for `{projectId}/*`
2. When message arrives on `{projectId}/{variableId}`:
   - Finds which PLC connection has that variable
   - Looks up the variable's datatype
   - Encodes value to `Uint8Array` using CIP type codes
   - Calls `writeTag()` to write to PLC

### CIP Type Codes for Writing

```typescript
const TYPE_MAP: Record<string, number> = {
  "BOOL": 0xc1, "SINT": 0xc2, "INT": 0xc3, "DINT": 0xc4,
  "LINT": 0xc5, "USINT": 0xc6, "UINT": 0xc7, "UDINT": 0xc8,
  "ULINT": 0xc9, "REAL": 0xca, "LREAL": 0xcb,
};
```
