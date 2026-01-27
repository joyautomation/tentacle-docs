# Tentacle Platform

Distributed IIoT platform built on Deno with NATS as the message bus. Bridges industrial PLCs to MQTT/GraphQL for real-time monitoring.

## Architecture Overview

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

## Services

| Service | Purpose | Port |
|---------|---------|------|
| **tentacle-plc** | Lightweight PLC runtime library with task-based programming | N/A (library) |
| **tentacle-ethernetip** | Polls Allen-Bradley PLCs via EtherNet/IP, publishes to NATS | N/A |
| **tentacle-mqtt** | Bridges NATS to MQTT using Sparkplug B protocol | N/A |
| **tentacle-graphql** | GraphQL API with real-time subscriptions | 4000 |
| **tentacle-web** | SvelteKit frontend with D3 visualizations | 5173 (dev) |
| **tentacle-nats-schema** | Shared TypeScript types and message validators | N/A (library) |

## Tech Stack

- **Runtime**: Deno (all backend services)
- **Frontend**: SvelteKit 2.9, Svelte 5, TypeScript 5.6, Vite 6
- **Messaging**: NATS with JetStream and KV stores
- **MQTT**: Sparkplug B via @joyautomation/synapse
- **GraphQL**: graphql-yoga + Pothos schema builder

## NATS Topics

```
plc.data.{projectId}.{variableId}   # Variable value updates
plc.status.{projectId}              # Project status (running/stopped/error)
plc.browse.{projectId}              # Browse available tags
plc.subscribe.{projectId}           # Subscribe to tag polling
plc.unsubscribe.{projectId}         # Unsubscribe from polling
field.sensor.{deviceId}             # Field sensor readings
field.command.{deviceId}            # Control commands
comm.event.{projectId}              # Events (error/warning/info)
```

## NATS KV Buckets

| Bucket | Purpose |
|--------|---------|
| `plc_variables` | Current state of all PLC variables |
| `service_heartbeats` | Service discovery (10s interval, 1min TTL) |
| `mqtt-config-{projectId}` | Per-variable MQTT settings |
| `field-config-{projectId}` | EtherNet/IP device/tag configuration |

## Key Message Types (from nats-schema)

```typescript
// Variable data - the core message type
interface PlcDataMessage {
  projectId: string
  deviceId: string
  variableId: string
  value: number | boolean | string | Record<string, unknown>
  timestamp: number
  datatype: "number" | "boolean" | "string" | "udt"
  deadband?: { value: number; maxTime?: number }
}

// Service heartbeat for discovery
interface ServiceHeartbeat {
  serviceType: "ethernetip" | "plc" | "mqtt" | "graphql"
  instanceId: string
  projectId: string
  lastSeen: number
  startedAt: number
}
```

All message types have `is*Message()` validators in nats-schema for runtime type checking.

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

## Running Services

```bash
# Each service
cd tentacle-{service}
deno task dev

# Web UI
cd tentacle-web
deno run -A npm:vite dev
```

## Environment Variables

```bash
# Shared
NATS_SERVERS=localhost:4222
PROJECT_ID=my-project

# tentacle-mqtt
MQTT_BROKER_URL=mqtt://localhost:1883
MQTT_GROUP_ID=TentacleGroup
MQTT_EDGE_NODE=EdgeNode

# tentacle-graphql
GRAPHQL_PORT=4000
GRAPHQL_PLAYGROUND=true

# tentacle-ethernetip
CLEAR_CACHE=false
```

## Data Flow Example

1. tentacle-ethernetip polls PLC tag "Temperature" = 20.5
2. Publishes to `plc.data.my-project.Temperature` with deadband config
3. tentacle-mqtt subscribes to `plc.data.my-project.>`
4. Reads RBE settings from `mqtt-config-my-project` KV
5. If change exceeds deadband, publishes via Sparkplug B
6. tentacle-graphql watches `plc_variables` KV bucket
7. Emits GraphQL subscription update to connected clients

## Key Dependencies

| Package | Purpose |
|---------|---------|
| @nats-io/transport-deno | NATS transport |
| @nats-io/jetstream | JetStream features |
| @nats-io/kv | Key-Value store |
| @joyautomation/coral | Logging |
| @joyautomation/dark-matter | Configuration |
| @joyautomation/synapse | Sparkplug B MQTT (see Synapse section below) |
| @joyautomation/tentacle-nats-schema | Shared schemas (renamed from nats-schema) |

## Common Debugging

- **Service not receiving messages**: Check NATS topic subscriptions match publish patterns
- **MQTT not publishing**: Verify mqtt-config KV has variable enabled
- **GraphQL subscription stale**: Check service_heartbeats KV for service health
- **EtherNet/IP not polling**: Verify subscription exists (only polls subscribed tags)

---

## tentacle-ethernetip Deep Dive

### Key Architecture Decisions

1. **Subscription-based polling**: Only polls tags with active subscriptions (from MQTT config or tentacle-plcs). Browse discovers tags but doesn't continuously poll them.

2. **One-time value read on browse**: Reads all tag values once during browse so UI shows actual values instead of 0/"unknown". After that, only subscribed tags are polled.

3. **NATS KV for caching**: Variable cache stored in NATS JetStream KV. Use `kv.purge()` not `kv.delete()` for complete removal (delete just adds tombstone).

4. **MQTT config change handling**: Scanner listens for MQTT config changes via `mqttConfigManager.onChange()` to dynamically subscribe/unsubscribe tags when user enables/disables them in UI.

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
- ForwardOpen may fail (status 0x1) - falls back to UCMM (unconnected messaging), which works fine
- Template parsing for UDTs can have alignment issues - **trust actual PLC response typeCode over template**

### Observability Features

- **Connection state tracking**: disconnected → connecting → connected
- **Exponential backoff**: 2s, 4s, 8s... up to 60s for reconnection
- **Periodic status logging**: Every 30s with sample values
- **"Data flowing" log**: On first successful read
- **Connection error detection**: Triggers automatic reconnect

### Bugs Previously Fixed (Don't Reintroduce!)

1. **Wrong UDT member datatypes** (VALUE showing as "boolean" instead of "number"):
   - Root cause: `decodeTagValue()` checked cached datatype BEFORE typeCode from PLC response
   - Fix: Prioritize typeCode from actual PLC response over cached datatype
   - Location: `scanner.ts` `decodeTagValue()` function

2. **Stale data appearing after refresh**:
   - Root cause: Multiple instances of tentacle-ethernetip running simultaneously
   - NATS requests answered by old instances with stale cached data
   - Fix: Kill all instances before starting fresh
   - Lesson: Always check `ps aux | grep tentacle` when seeing weird stale data

3. **Newly enabled MQTT tags not being polled**:
   - Root cause: `autoSubscribeMqttTags()` only called at startup, not on config changes
   - Fix: Added `handleMqttConfigChange()` that syncs subscriptions when MQTT config changes
   - Location: `scanner.ts`

### Gotchas & Lessons Learned

- **Multiple instances**: NATS request/reply can be answered by ANY running instance - stale data mystery usually means old instances running
- **Cache persistence**: NATS KV `delete()` just adds a tombstone marker; use `purge()` for complete removal
- **Datatype trust hierarchy**: PLC response typeCode > cached datatype > template parsing
- **Debug logging**: Most poll-level logging is at debug level. Use `DEBUG=* deno run -A main.ts` for verbose output
- **Deadband filtering**: MQTT publishes are filtered by deadband (default 1). Values changing by < deadband won't publish, but data IS being read
- **Values look "flat"**: If MQTT values look constant, check: (a) PLC values actually changing in RSLogix, (b) deadband setting, (c) the 30s status log shows sample values to confirm reads are working

### Common Commands

```bash
# Check for running instances (DO THIS FIRST when debugging stale data)
ps aux | grep tentacle

# Kill all tentacle processes
pkill -f tentacle

# Start with cache clear
deno run -A main.ts --clear-cache

# Run NATS locally
docker start nats
# or fresh: docker run -d --name nats -p 4222:4222 nats -js

# Check NATS KV cache
nats kv get field-config-tentacle-dev cache.variables.rtu45

# Verbose debug output
DEBUG=* deno run -A main.ts
```

---

## @joyautomation/synapse Deep Dive

Synapse is the Sparkplug B MQTT library used by tentacle-mqtt. It's maintained in `/home/joyja/Development/joyautomation/kraken/synapse`.

### Event Signatures (IMPORTANT!)

Synapse emits events with specific signatures. Getting these wrong causes silent failures:

```typescript
// DCMD event - receives (topic, payload), NOT just (payload)
node.events.on("dcmd", (topic: SparkplugTopic, payload: UPayload) => {
  // topic.deviceId - the device ID
  // payload.metrics - array of metrics with name/value
});

// SparkplugTopic structure
type SparkplugTopic = {
  version: string;    // "spBv1.0"
  groupId: string;
  commandType: string; // "DCMD", "NCMD", etc.
  edgeNode: string;
  deviceId?: string;
};
```

### Version History & Fixes

| Version | Fix |
|---------|-----|
| 0.0.86 | Re-subscribes to DCMD/NCMD/STATE topics on MQTT reconnect (mqtt.js `clean: true` means broker forgets subscriptions) |
| 0.0.87 | `disconnect` transition only removes internal "ncmd" listener, preserves external listeners like "dcmd" |

### Why These Fixes Matter

1. **mqtt.js auto-reconnection**: When connection drops, mqtt.js reconnects automatically but with `clean: true` the broker forgets all subscriptions. Synapse 0.0.86+ re-subscribes in `onConnect()`.

2. **Event listener preservation**: The `disconnect` transition used to call `cleanUpEventListeners(node.events)` which removed ALL listeners including external ones. Synapse 0.0.87+ only removes internal "ncmd" listener.

### Publishing to JSR

```bash
cd /path/to/synapse
# Update version in deno.json
git add deno.json && git commit -m "chore: bump version"
git tag X.X.X && git push origin main && git push origin X.X.X
# GitHub workflow auto-publishes on tag push
```

---

## tentacle-mqtt Deep Dive

### Bidirectional Data Flow

```
PLC → tentacle-ethernetip → NATS → tentacle-mqtt → Sparkplug B → MQTT Broker
                                                                      ↓
PLC ← tentacle-ethernetip ← NATS ← tentacle-mqtt ← DCMD ←───────────────
```

### DCMD Handling (Write Commands)

When Ignition or another Sparkplug client sends a DCMD:

1. Synapse receives DCMD on `spBv1.0/{group}/DCMD/{edgeNode}/{deviceId}`
2. Emits `dcmd` event with `(topic, payload)`
3. tentacle-mqtt extracts metric name/value from payload
4. Publishes to NATS: `{projectId}/{variableId}` with value as string
5. tentacle-ethernetip receives and writes to PLC

### Key Code Pattern

```typescript
// CORRECT - use (topic, payload) signature
node.events.on("dcmd", async (topic: any, payload: { metrics?: any[] }) => {
  log.info(`Received DCMD for device ${topic.deviceId}`);
  if (payload.metrics) {
    for (const metric of payload.metrics) {
      // Process metric.name and metric.value
    }
  }
});

// WRONG - this silently fails (payload ends up being the topic)
node.events.on("dcmd", async (payload) => { ... });
```

### NATS Publishing Gotcha

NATS requires `Uint8Array`, not strings:

```typescript
// WRONG - silently fails or errors
nc.publish(subject, String(value));

// CORRECT
nc.publish(subject, new TextEncoder().encode(String(value)));
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
  "BOOL": 0xc1,
  "SINT": 0xc2,
  "INT": 0xc3,
  "DINT": 0xc4,
  "LINT": 0xc5,
  "USINT": 0xc6,
  "UINT": 0xc7,
  "UDINT": 0xc8,
  "ULINT": 0xc9,
  "REAL": 0xca,
  "LREAL": 0xcb,
};
```

### Key Functions in scanner.ts

| Function | Purpose |
|----------|---------|
| `datatypeToTypeCode()` | Maps datatype string to CIP type code |
| `encodeTagValue()` | Encodes JS value to Uint8Array for PLC |
| `findPlcWithVariable()` | Finds which PLC connection has a variable |
| `handleWriteCommand()` | Processes write requests from NATS |
