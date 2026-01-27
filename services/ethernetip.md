# tentacle-ethernetip

Polls Allen-Bradley PLCs via EtherNet/IP protocol and publishes data to NATS. Also handles write commands from NATS for bidirectional control.

## Key Architecture Decisions

1. **Subscription-based polling**: Only polls tags with active subscriptions (from MQTT config or tentacle-plcs). Browse discovers tags but doesn't continuously poll them.

2. **One-time value read on browse**: Reads all tag values once during browse so UI shows actual values instead of 0/"unknown". After that, only subscribed tags are polled.

3. **NATS KV for caching**: Variable cache stored in NATS JetStream KV. Use `kv.purge()` not `kv.delete()` for complete removal.

4. **MQTT config change handling**: Scanner listens for MQTT config changes via `mqttConfigManager.onChange()` to dynamically subscribe/unsubscribe tags.

## Key Files

| File | Purpose |
|------|---------|
| `scanner.ts` | Main polling loop, connection management, browse handler, write support |
| `template.ts` | UDT template parsing |
| `mqttConfig.ts` | MQTT variable configuration via NATS KV |
| `main.ts` | Entry point, `--clear-cache` flag uses `kv.purge()` |

## Write Support

Handles write commands from NATS for bidirectional PLC control:

1. Subscribes to NATS subject `>` and filters for `{projectId}/*`
2. When message arrives on `{projectId}/{variableId}`:
   - Finds which PLC connection has that variable
   - Looks up the variable's datatype
   - Encodes value to `Uint8Array` using CIP type codes
   - Calls `writeTag()` to write to PLC

### Key Functions

| Function | Purpose |
|----------|---------|
| `datatypeToTypeCode()` | Maps datatype string to CIP type code |
| `encodeTagValue()` | Encodes JS value to Uint8Array for PLC |
| `findPlcWithVariable()` | Finds which PLC connection has a variable |
| `handleWriteCommand()` | Processes write requests from NATS |

## Observability

- **Connection state tracking**: disconnected → connecting → connected
- **Exponential backoff**: 2s, 4s, 8s... up to 60s for reconnection
- **Periodic status logging**: Every 30s with sample values
- **"Data flowing" log**: On first successful read
- **Connection error detection**: Triggers automatic reconnect

## Gotchas

- **Multiple instances**: NATS request/reply can be answered by ANY running instance - stale data mystery usually means old instances running
- **Cache persistence**: NATS KV `delete()` just adds a tombstone marker; use `purge()` for complete removal
- **Datatype trust hierarchy**: PLC response typeCode > cached datatype > template parsing
- **ForwardOpen failures**: May fail (status 0x1) - falls back to UCMM (unconnected messaging), which works fine
