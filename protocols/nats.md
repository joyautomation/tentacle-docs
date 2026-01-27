# NATS Topics & KV Buckets

## Topics

```
plc.data.{projectId}.{variableId}   # Variable value updates
plc.status.{projectId}              # Project status (running/stopped/error)
plc.browse.{projectId}              # Browse available tags (request/reply)
plc.subscribe.{projectId}           # Subscribe to tag polling
plc.unsubscribe.{projectId}         # Unsubscribe from polling
plc.variables.{projectId}           # Get all current variables (request/reply)
{projectId}/{variableId}            # Write commands (from tentacle-mqtt DCMD)
field.sensor.{deviceId}             # Field sensor readings
field.command.{deviceId}            # Control commands
comm.event.{projectId}              # Events (error/warning/info)
```

## KV Buckets

| Bucket | Purpose |
|--------|---------|
| `plc_variables` | Current state of all PLC variables |
| `service_heartbeats` | Service discovery (10s interval, 1min TTL) |
| `mqtt-config-{projectId}` | Per-variable MQTT settings |
| `field-config-{projectId}` | EtherNet/IP device/tag configuration |

## Key Message Types

Defined in `@joyautomation/tentacle-nats-schema`:

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

// Browse progress for async browsing
interface BrowseProgressMessage {
  browseId: string
  projectId: string
  deviceId: string
  phase: "discovering" | "expanding" | "reading" | "caching" | "completed" | "failed"
  totalTags: number
  completedTags: number
  errorCount: number
  message?: string
  timestamp: number
}
```

All message types have `is*Message()` validators for runtime type checking.

## KV vs Pub/Sub

- **Use KV** for: Current state, configuration, caching
- **Use Pub/Sub** for: Real-time events, data streaming

## Gotchas

- **KV delete vs purge**: `delete()` adds a tombstone, `purge()` completely removes
- **Topic wildcards**: Use `>` for multi-level, `*` for single level
- **Request/reply timeout**: Default 5s, increase for slow operations like browse
