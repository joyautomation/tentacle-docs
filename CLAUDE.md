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

| Service | Repo | Purpose | Runtime | Port |
|---------|------|---------|---------|------|
| **tentacle** | [GitHub](https://github.com/joyautomation/tentacle) | Core orchestration and deployment | Deno | N/A |
| **tentacle-plc** | [GitHub](https://github.com/joyautomation/tentacle-plc) | Lightweight PLC runtime library with task-based programming | Deno (library) | N/A |
| **tentacle-ethernetip-go** | [GitHub](https://github.com/joyautomation/tentacle-ethernetip-go) | Polls Allen-Bradley PLCs via EtherNet/IP (Go + libplctag), publishes to NATS | Go | N/A |
| **tentacle-opcua-go** | [GitHub](https://github.com/joyautomation/tentacle-opcua-go) | OPC UA client, subscribes to nodes, publishes to NATS | Go | N/A |
| **tentacle-modbus** | [GitHub](https://github.com/joyautomation/tentacle-modbus) | Modbus TCP scanner, polls registers/coils, publishes to NATS | Deno | N/A |
| **tentacle-modbus-server** | [GitHub](https://github.com/joyautomation/tentacle-modbus-server) | Modbus TCP server, exposes PLC data to Modbus clients | Deno | N/A |
| **tentacle-mqtt** | [GitHub](https://github.com/joyautomation/tentacle-mqtt) | Bridges NATS to MQTT using Sparkplug B protocol | Deno | N/A |
| **tentacle-graphql** | [GitHub](https://github.com/joyautomation/tentacle-graphql) | GraphQL API with real-time subscriptions | Deno | 4000 |
| **tentacle-web** | [GitHub](https://github.com/joyautomation/tentacle-web) | SvelteKit frontend with D3 topology visualization | Deno | 3012 |
| **tentacle-network** | [GitHub](https://github.com/joyautomation/tentacle-network) | Network interface monitoring and netplan configuration | Deno | N/A |
| **tentacle-nftables** | [GitHub](https://github.com/joyautomation/tentacle-nftables) | nftables NAT rule management | Deno | N/A |
| **tentacle-nats-schema** | [GitHub](https://github.com/joyautomation/tentacle-nats-schema) | Shared TypeScript types and message validators | N/A (library) | N/A |
| **tentacle-docs** | [GitHub](https://github.com/joyautomation/tentacle-docs) | Platform documentation | N/A | N/A |

## Running Services

```bash
# All services (recommended)
./dev.sh

# Individual Deno services
cd tentacle-{service} && deno task dev

# Web UI
cd tentacle-web && deno task dev
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

**Runs on Deno** — use `deno task dev` and `deno task build`.

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

---

## tentacle-opcua-go Deep Dive

### Runtime

**Go + gopcua** — use `go run .` or `go build`. No CGo required (pure Go OPC UA client).

### Key Architecture Decisions

1. **Subscriber-driven**: Zero connections at startup. Devices connect on-demand when `opcua.subscribe` or `opcua.browse` requests arrive.

2. **Event-driven subscriptions**: Unlike EIP (continuous polling), uses gopcua's built-in `NodeMonitor` with `ChanSubscribe()` for push-based data change notifications.

3. **Temporary browse connections**: Browse creates a temporary connection, disconnects after completion, but retains cached variable metadata in memory.

4. **Per-device subscriptions**: Multiple subscribers can monitor the same device independently. A single `NodeMonitor` per connection serves ALL subscribers.

5. **Certificate-based auth**: Self-signed certificates auto-generated for secure OPC UA connections.

### Key Files

| File | Purpose |
|------|---------|
| `main.go` | Entry point, NATS connection, heartbeat, shutdown |
| `scanner.go` | Connection management, subscriber tracking, data publishing |
| `browse.go` | Recursive address space traversal, datatype resolution |
| `types.go` | Type definitions, datatype mapping functions |
| `cert.go` | Certificate generation/loading for secure channels |

### NATS Topics

```
opcua.subscribe              # Start monitoring node values
opcua.unsubscribe            # Stop monitoring nodes
opcua.browse                 # Discover variables (sync or async)
opcua.browse.progress.{browseId}  # Async browse progress
opcua.variables              # List all discovered/subscribed variables
opcua.data.{deviceId}.{sanitizedNodeId}  # Variable value changes
opcua.command.{variableId}   # Write value to OPC UA node
```

### NodeID Sanitization

OPC UA NodeIDs like `ns=2;s=MyTag.SubTag` are sanitized to `ns_2_s_MyTag_SubTag` for NATS subject compatibility. Subjects are multi-level — use `>` wildcard.

### Gotchas

- **Multiple instances**: Same as EIP — NATS request/reply answered by ANY instance
- **Temporary connections**: Browse disconnects after completion; variable cache persists for subsequent subscriptions
- **Channel buffer**: 256-element data change buffer. High-frequency updates (1000+/sec) may overflow
- **Cleanup on zero subscribers**: Last unsubscribe immediately closes the OPC UA connection
- **`>` vs `*` wildcards**: OPC UA data subjects are multi-level — always use `>`

### Environment Variables

```bash
NATS_SERVERS=localhost:4222
OPCUA_PKI_DIR=./pki
OPCUA_AUTO_ACCEPT_CERTS=true  # Auto-accept server certs (testing only)
```

---

## tentacle-network Deep Dive

### Runtime

**Deno** — use `deno task dev`. Requires Linux with sysfs (`/sys/class/net`) and `ip` command.

### Key Architecture Decisions

1. **Kernel-sourced ground truth**: Never caches interface state. Always reads fresh from `/sys/class/net` and `ip` commands.

2. **Two-layer state model**:
   - **Live state**: Actual interface state from kernel (operstate, carrier, speed, stats)
   - **Config state**: Persistent configuration via netplan (DHCP, static addresses, gateway, DNS, MTU)

3. **Netplan as source of truth**: Writes to `/etc/netplan/60-tentacle.yaml`, backs up other netplan files to `.yaml.bak`.

4. **Sparkplug B UDT metrics**: Network interfaces published as UDTs with change detection per interface.

5. **Virtual interface filtering**: Excludes container/VM interfaces (veth, docker, tap, vnet, dummy, etc.) from Sparkplug B metrics — live state still includes all.

### Key Files

| File | Purpose |
|------|---------|
| `main.ts` | Entry point, NATS handlers, heartbeat, periodic publishing |
| `src/monitor/interfaces.ts` | Reads live interface state from sysfs + `ip -j addr show` |
| `src/commands/netplan.ts` | YAML generation/parsing, netplan file management |
| `src/metrics/publisher.ts` | Sparkplug B UDT messages, change detection |
| `src/utils/logger.ts` | NATS log streaming |

### NATS Topics

```
network.interfaces                    # Periodic snapshots (every 10s)
network.state                         # Request/reply for fresh state
network.command                       # Config commands (get-config, apply-config, add-address, remove-address)
network.command.>                     # Field-level commands from Sparkplug B DCMD
network.data.{interfaceName}          # Sparkplug B UDT metrics per interface
network.shutdown                      # Graceful shutdown
service.logs.network.network          # Log streaming
```

### Configuration Commands

| Action | Purpose |
|--------|---------|
| `get-config` | Merges all `/etc/netplan/*.yaml` files in sorted order |
| `apply-config` | Validates, writes netplan YAML, backs up others, runs `netplan apply` |
| `add-address` | Runs `ip addr add` (treats "File exists" as success) |
| `remove-address` | Runs `ip addr del` (treats "Cannot assign" as success) |

### Gotchas

- **Netplan apply delay**: 2-second delay after config apply before re-publishing state
- **Field commands only on config**: Sparkplug B DCMD on live metrics (e.g., speed) silently ignored
- **Address format**: Requires CIDR notation with prefix length (e.g., `192.168.1.100/24`)
- **MTU bounds**: Validated 68–65535
- **No rollback on failure**: If `netplan apply` fails, YAML is already written but not applied

---

## tentacle-nftables Deep Dive

### Runtime

**Deno** — use `deno task dev`. Requires Linux with `nft` and `sysctl` commands.

### Key Architecture Decisions

1. **Unified NatRule model**: Single rule type combines DNAT + optional Double NAT (SNAT).

2. **Atomic rule application**: Uses `nft -f` to load rules atomically. Deletes and recreates only `table ip tentacle_nat` — preserves all other system firewall rules.

3. **Dual storage**: Config as structured JSON (`/etc/tentacle/nftables.json`) AND generated nft syntax (`/etc/nftables.d/60-tentacle-nat.conf`).

4. **Automatic IP alias management**: Computes required aliases from enabled NAT rules, syncs via `tentacle-network` request/reply.

5. **Live state always fresh**: Never caches ruleset; reads from kernel via `nft -j list ruleset` on every request.

6. **Backward compatibility**: Auto-converts old `portForwards` + `snatRules` format to unified `natRules`.

### Key Files

| File | Purpose |
|------|---------|
| `main.ts` | Entry point, NATS handlers, IP alias management, heartbeat |
| `src/commands/nftables.ts` | Config I/O, nft syntax generation, atomic rule application |
| `src/monitor/ruleset.ts` | Reads live nftables state via `nft -j list ruleset` |
| `src/metrics/publisher.ts` | Sparkplug B UDT metrics per NAT rule, change detection |
| `src/utils/logger.ts` | NATS log streaming |

### NATS Topics

```
nftables.rules                        # Periodic broadcast of config + metrics (10s)
nftables.state                        # Request/reply for live ruleset
nftables.command                      # Config commands (get-config, apply-config)
nftables.command.>                    # Field-level commands from Sparkplug B DCMD
nftables.data.{ruleKey}              # Sparkplug B UDT metrics per NAT rule
nftables.shutdown                     # Graceful shutdown
network.state                         # Request to tentacle-network for interface info
network.command                       # Request to tentacle-network for add/remove address
```

### NAT Rule Model

```typescript
type NatRule = {
  id: string;
  enabled: boolean;
  protocol: "tcp" | "udp" | "icmp" | "all";
  connectingDevices: string;     // "any", IP, range, or CIDR
  incomingInterface: string;
  outgoingInterface: string;
  natAddr: string;               // Destination NAT IP (on incoming interface subnet)
  originalPort: string;          // Port or range "80" or "80-90"
  translatedPort: string;
  deviceAddr: string;            // Target device IP
  deviceName: string;
  doubleNat: boolean;            // Enable SNAT for return traffic
  doubleNatAddr: string;         // SNAT alias IP on outgoing interface
  comment: string;
};
```

### Config File Paths

| Path | Purpose |
|------|---------|
| `/etc/tentacle/nftables.json` | Structured NAT rules config |
| `/etc/nftables.d/60-tentacle-nat.conf` | Generated nft syntax (applied atomically) |
| `/etc/tentacle/managed-aliases.json` | Tracked IP aliases for cleanup |
| `/etc/sysctl.d/60-tentacle.conf` | Persisted IPv4 forwarding |

### Gotchas

- **IPv4 forwarding**: Auto-enabled on startup via `sysctl` and persisted — required for NAT
- **Table isolation**: Only manages `table ip tentacle_nat` — never flushes system rules
- **IP alias prefix**: Queries `tentacle-network` for interface subnet prefix length, falls back to /24
- **Disabled rules**: Excluded from nft syntax generation; aliases only created for enabled rules
- **No partial applies**: Field-level DCMD updates re-apply the full config
- **Legacy conversion**: Old `portForwards`/`snatRules` auto-converted; standalone SNATs without matching DNAT are discarded
