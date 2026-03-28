# NATS Topics & KV Buckets

## Topic Patterns

Topics use a consistent naming convention with substitutable parameters in `{curly_braces}`:

```
# Module data (any service publishing variables)
# Each service owns its own namespace — web UI subscribes per-module
{moduleId}.data.{variableId}     # Variable value update (pub/sub)
{moduleId}.command.{variableId}  # Write command to a module's variable
{moduleId}.variables             # Request all variables from module (request/reply)
{moduleId}.status                # Module status
{moduleId}.shutdown              # Graceful shutdown command

# EtherNet/IP scanner (Go + libplctag)
# NOTE: EIP data subjects are multi-level: ethernetip.data.{deviceId}.{tagName}
# Use `>` wildcard (not `*`) for subscriptions: ethernetip.data.>
ethernetip.data.{deviceId}.{tagName}   # Tag value update (multi-level subject!)
ethernetip.subscribe             # Add tag subscription (request/reply)
ethernetip.unsubscribe           # Remove tag subscription (request/reply)
ethernetip.browse                # Trigger tag browse (request/reply)
ethernetip.browse.progress.{browseId}  # Browse progress updates
ethernetip.variables             # Request all variables (request/reply)

# OPC UA scanner
opcua.subscribe                  # Add node subscription (request/reply)
opcua.unsubscribe                # Remove node subscription (request/reply)
opcua.browse                     # Browse address space (request/reply)
opcua.browse.progress.{browseId} # Browse progress updates

# Modbus TCP scanner
modbus.subscribe                 # Add device/tag subscription (request/reply)
modbus.unsubscribe               # Remove subscription (request/reply)
modbus.data.{deviceId}           # Batch tag values (one subject per device)
modbus.command.{tagId}           # Write a tag value

# Service log streaming
service.logs.{serviceType}.{moduleId}  # Real-time log entries

# Network management
network.interfaces               # Periodic interface state snapshot
network.state                    # On-demand state request (request/reply)
network.command                  # Apply netplan config (request/reply)

# nftables NAT management
nftables.rules                   # Periodic config broadcast
nftables.state                   # On-demand ruleset request (request/reply)
nftables.command                 # Apply/get NAT config (request/reply)

# Field devices
field.sensor.{deviceId}          # Sensor readings
field.command.{deviceId}         # Control commands
field.response.{deviceId}        # Command responses
```

### Wildcard Subscriptions

```
*.data.>                  # All variable data from all modules (multi-level)
{moduleId}.data.>         # All data from a specific module (recommended for web pages)
service.logs.>            # All service logs
service.logs.{svcType}.>  # Logs from one service type
```

> **Important**: Use `>` (multi-level) not `*` (single-level). EtherNet/IP uses multi-level subjects like `ethernetip.data.rtu45.TagName`.

## KV Buckets

| Bucket | Key Pattern | Purpose |
|--------|------------|---------|
| `service_heartbeats` | `{moduleId}` | Service discovery — 60s TTL, published every 10s |
| `plc_variables` | `{variableId}` | Shared variable state (from nats-schema) |
| `plc-variables-{projectId}` | `{variableId}` | Per-project PLC variable persistence |
| `mqtt-config-{projectId}` | `{variableId}` | Per-variable MQTT settings |
| `service_enabled` | `{moduleId}` | Service enable/disable state (boolean) |
| `desired_services` | `{moduleId}` | Orchestrator desired state — version + running (no TTL) |
| `service_status` | `{moduleId}` | Orchestrator reported status — 120s TTL, auto-expires if orchestrator dies |

> **Note**: `plc_variables` is defined in the nats-schema spec. `plc-variables-{projectId}` is used by tentacle-plc instances for their own state persistence.

### Orchestrator KV Detail

The `desired_services` and `service_status` buckets are used by [tentacle-orchestrator](../services/orchestrator.md) for bare-metal service management:

```typescript
// desired_services — what should be running
type DesiredServiceKV = {
  moduleId: string;    // e.g. "tentacle-mqtt"
  version: string;     // e.g. "0.0.5" or "latest"
  running: boolean;    // should the systemd unit be active?
  updatedAt: number;
};

// service_status — what is actually running (written by orchestrator)
type ServiceStatusKV = {
  moduleId: string;
  installedVersions: string[];
  activeVersion: string | null;
  systemdState: "active" | "inactive" | "failed" | "not-found";
  reconcileState: "ok" | "pending" | "downloading" | "installing"
    | "starting" | "stopping" | "error" | "version_unavailable";
  lastError: string | null;
  runtime: "go" | "deno" | "deno-web";
  category: "core" | "optional";
  repo: string;
  updatedAt: number;
};
```

## Key Message Types

Defined in `@joyautomation/tentacle-nats-schema`:

```typescript
// Variable data — published by any module when a variable changes
type PlcDataMessage = {
  moduleId: string;          // Source module (e.g., "ethernetip", "mixing-process")
  deviceId: string;          // Device/PLC this variable belongs to
  variableId: string;
  value: number | boolean | string | Record<string, unknown>;
  timestamp: number;
  datatype: "number" | "boolean" | "string" | "udt";
  deadband?: { value: number; maxTime?: number };
  disableRBE?: boolean;
  description?: string;
  udtTemplate?: UdtTemplateDefinition;  // Present when datatype === "udt"
};

// Service heartbeat — published to service_heartbeats KV bucket every 10s
type ServiceHeartbeat = {
  serviceType: "ethernetip" | "plc" | "mqtt" | "graphql" | "modbus" | "opcua" | "network" | "nftables" | "orchestrator" | "history";
  moduleId: string;          // Unique module identifier
  lastSeen: number;
  startedAt: number;
  version?: string;
  metadata?: Record<string, unknown>;
};

// Service log entry — published for real-time log streaming
type ServiceLogEntry = {
  timestamp: number;
  level: "info" | "warn" | "error" | "debug";
  message: string;
  serviceType: string;
  moduleId: string;
  logger?: string;
};

// Browse progress — async progress updates during tag/node browse
type BrowseProgressMessage = {
  browseId: string;
  moduleId: string;
  deviceId: string;
  phase: "discovering" | "expanding" | "reading" | "caching" | "completed" | "failed";
  totalTags: number;
  completedTags: number;
  errorCount: number;
  message?: string;
  timestamp: number;
};
```

## Using the nats-schema Package

```typescript
import {
  NATS_TOPICS,
  NATS_SUBSCRIPTIONS,
  substituteTopic,
  type PlcDataMessage,
  type ServiceHeartbeat,
} from "@joyautomation/tentacle-nats-schema";

// Build a topic with substitution
const topic = substituteTopic(NATS_TOPICS.module.data, {
  moduleId: "mixing-process",
  variableId: "temperature",
});
// → "mixing-process.data.temperature"

// Subscribe to all data from all modules
const allData = NATS_SUBSCRIPTIONS.allData(); // → "*.data.>"

// Subscribe to logs from a specific service type
const eipLogs = NATS_SUBSCRIPTIONS.serviceTypeLogs("ethernetip");
// → "service.logs.ethernetip.>"
```

## Data Flow

```
tentacle-ethernetip → {moduleId}.data.{variableId}  →  tentacle-graphql
tentacle-opcua-go   → {moduleId}.data.{variableId}  →    (subscribes to *.data.>)
tentacle-modbus     → modbus.data.{deviceId}         →
tentacle-plc        → {projectId}.data.{variableId}  →

tentacle-graphql → {moduleId}.command.{variableId} → scanner service → writes to device
```

## KV vs Pub/Sub

- **Use KV** for: Current state, configuration, service discovery
- **Use Pub/Sub** for: Real-time events, streaming data, commands

## Gotchas

- **KV delete vs purge**: `delete()` adds a tombstone, `purge()` completely removes
- **Topic wildcards**: Use `>` for multi-level, `*` for single level
- **Request/reply timeout**: Default 5s, increase for slow operations like browse
- **`{moduleId}` vs `{projectId}`**: The PLC runtime uses `projectId` as its `moduleId` — they are the same value
- **Modbus data subject**: `modbus.data.{deviceId}` is one subject per device (all tags batched), unlike EtherNet/IP which uses `{moduleId}.data.{variableId}` (one subject per tag)
- **NATS encoding**: Always publish as `Uint8Array` — use `new TextEncoder().encode(JSON.stringify(payload))`
