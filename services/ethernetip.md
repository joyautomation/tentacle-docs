# tentacle-ethernetip-go

Go + CGo + libplctag service that polls Allen-Bradley PLCs via EtherNet/IP (CIP) and publishes data to NATS. Replaced the original Deno `tentacle-ethernetip` service.

## Tech Stack

- **Go** with CGo bindings to libplctag C library
- **libplctag**: Handles CIP protocol, connection management, and tag I/O
- Requires `libplctag.so` installed on target (typically at `~/.local/lib/` or `/usr/local/lib/`)

## Key Architecture Decisions

1. **Subscription-based polling**: Only polls tags with active subscriptions (from tentacle-plc or MQTT config). Browse discovers tags but doesn't continuously poll them.

2. **Parallel goroutine reads**: Tags are read in parallel batches using goroutines. libplctag internally uses CIP Multi-Service Packets to batch reads over the wire, reducing round-trips especially over high-latency links (e.g., cellular).

3. **Own NATS namespace**: Publishes to `ethernetip.data.{deviceId}.{tagName}` — each scanner service owns its own data namespace. The `>` wildcard is needed to subscribe (multi-level subjects).

4. **UDT resolution**: Uses `@tags` for tag listing, `@udt/<id>` for UDT template definitions. Resolves TIMER, COUNTER, and custom UDTs. Zero unnamed members.

5. **Browse with one-time value read**: Reads all tag values once during browse so UI shows actual values instead of 0. After that, only subscribed tags are polled.

## NATS Topics

```
ethernetip.data.{deviceId}.{tagName}     # Variable value update (pub)
ethernetip.subscribe                      # Add tag subscription (request/reply)
ethernetip.unsubscribe                    # Remove subscription (request/reply)
ethernetip.browse                         # Browse PLC tags (request/reply)
ethernetip.browse.progress.{browseId}     # Browse progress updates (pub)
ethernetip.variables                      # Request all variables (request/reply)
```

**Important**: Use `>` wildcard for subscriptions (e.g., `ethernetip.data.>`), NOT `*`. The subjects have multiple tokens after `data.`.

## Key Files

| File | Purpose |
|------|---------|
| `main.go` | Entry point, NATS connection, heartbeat, request handlers |
| `scanner.go` | Tag handle creation, poll loop, parallel read logic |
| `browse.go` | Tag listing via `@tags`, UDT template resolution via `@udt/<id>` |
| `types.go` | Shared types (PlcConnection, Variable, etc.) |

## Write Support

Handles write commands from NATS:

1. Subscribes to `ethernetip.command.{variableId}`
2. Finds which PLC connection has that variable
3. Creates a libplctag write handle, encodes value, writes to PLC

## Observability

- **Poll cycle timing**: Logs duration of each poll cycle (e.g., "Poll cycle for rtu45: 1896 tags in 5ms")
- **Bad status warnings**: Logs tags that fail with PLCTAG_ERR_NOT_FOUND or other errors
- **Connection state**: Logs device connections and disconnections
- **Heartbeat**: Publishes to `service_heartbeats` KV every 10s (60s TTL)

## Gotchas

- **Multiple instances**: NATS request/reply can be answered by ANY running instance — stale data usually means old instances running
- **libplctag NOT_FOUND**: Some UDT hidden members legitimately don't exist on the PLC. After `maxCreateRetries` (3), the tag is skipped until next browse.
- **Cellular/high-latency links**: Parallel goroutine reads help, but initial connection for many tags can be slow. libplctag handles connection pooling internally.
- **`>` vs `*` wildcards**: EtherNet/IP data subjects are multi-level (`ethernetip.data.rtu45.TagName`). Use `>` not `*` for subscriptions.
