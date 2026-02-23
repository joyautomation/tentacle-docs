# Troubleshooting

## Quick Checks

```bash
# Check for running instances (DO THIS FIRST when debugging stale data)
ps aux | grep tentacle

# Kill all tentacle processes
pkill -f tentacle

# Check NATS is running
nats server info

# Check service heartbeats
nats kv get service_heartbeats ethernetip

# Watch all variable data in real-time
nats sub "*.data.>"

# Watch modbus data for a specific device
nats sub "modbus.data.my-device"

# Check NATS KV cache
nats kv get field-config-{projectId} cache.variables.{variableId}
```

## Common Issues

### Stale data appearing after refresh

**Symptom**: Old values appear even after restarting services

**Cause**: Multiple instances of a scanner running simultaneously. NATS requests can be answered by ANY running instance.

**Fix**:
```bash
ps aux | grep tentacle
pkill -f tentacle
# Then restart only the services you need
```

### MQTT not publishing / values look "flat"

**Symptoms**:
- MQTT values don't change even though device values are changing
- Sparkplug client shows constant values

**Causes & Fixes**:
1. **Deadband filtering**: Values changing by less than deadband won't publish. Check `mqtt-config-{projectId}` KV bucket
2. **Variable not enabled**: Verify variable is enabled in mqtt-config
3. **Data IS being read**: Check the 30s status log in the scanner — it shows sample values

### DCMD commands not working

**Symptom**: Write commands from Ignition/SCADA don't reach the device

**Debugging steps**:
1. Check tentacle-mqtt logs for "Received DCMD" message
2. If no DCMD log: synapse version issue (need 0.0.87+)
3. If DCMD received but not forwarded: wrong event signature
4. Check scanner logs for write attempts

**Common causes**:
- Wrong synapse version (need 0.0.87+ for reconnection fixes)
- MQTT reconnection dropped DCMD subscription
- NATS publish encoding issue (must use `TextEncoder().encode()`)

### EtherNet/IP not polling tags

**Symptom**: No data from PLC, tags show as "unknown" or 0

**Causes & Fixes**:
1. **No subscription**: Only polls tags with active subscriptions (from MQTT config or tentacle-plc)
2. **Connection failed**: Check logs for connection errors, verify PLC IP
3. **Browse first**: Tags must be discovered via browse before they can be subscribed

### Modbus not reading values

**Symptom**: Modbus device shows no data; scanner connects but no `modbus.data.*` messages

**Causes & Fixes**:
1. **No subscription sent**: The scanner only polls after a `modbus.subscribe` request arrives
2. **Wrong function code**: Verify FC matches register type (coils vs. holding registers)
3. **Address off-by-one**: Modbus addresses are 0-based in tentacle-modbus. Some tools show 1-based addresses — subtract 1
4. **Byte order**: Default is ABCD (big-endian). CoDesys and some devices use CDAB or BADC — check `byteOrder` in tag config
5. **PDU byte offset**: Response is `[FC, ByteCount, Data...]` — data starts at byte index 2

**Debugging**:
```bash
# Watch modbus subscribe/unsubscribe requests
nats sub "modbus.>"

# Check modbus scanner logs
nats sub "service.logs.modbus.>"
```

### OPC UA (tentacle-opcua-go) certificate issues

**Symptom**: `Bad_CertificateUseNotAllowed` from Ignition/Eclipse Milo

**Cause**: Self-signed cert missing required key usage bits for Milo's strict validation.

**Required cert properties**:
- `KeyUsageCertSign` (bit 5) must be set — even for self-signed
- AKI must equal SKI
- ExtKeyUsage: `clientAuth` + `serverAuth`
- SAN must include ApplicationURI

See [troubleshooting OPC UA](services/opcua.md) for cert generation commands.

### Ignition OPC UA connection refused

**Symptom**: OPC UA client connects but gets "endpoint not found" or connection refused

**Fix**: In Ignition, add the hostname your client uses to connect:
`Config → OPC UA → Server → Endpoint Addresses`

### GraphQL subscriptions stale

**Symptom**: Web UI not updating in real-time

**Fixes**:
1. Check `service_heartbeats` KV for service health: `nats kv get service_heartbeats graphql`
2. Verify tentacle-graphql is connected to NATS
3. Check browser console for SSE connection errors
4. Check that `*.data.>` subscription is active in tentacle-graphql

### Service not appearing in topology

**Symptom**: Service is running but not visible in the web topology view

**Cause**: Service heartbeat not being published to `service_heartbeats` KV

**Debugging**:
```bash
# List all heartbeats
nats kv watch service_heartbeats

# Check if a specific service is publishing
nats kv get service_heartbeats {moduleId}
```

### Service not receiving NATS messages

**Debugging**:
1. Verify topic patterns match between publisher and subscriber
2. Use `nats sub "*.data.>"` to see what's being published
3. Check service `.env` — `NATS_SERVERS` must match

## Debug Mode

```bash
# Verbose output for Deno services
LOG_LEVEL=debug deno task dev
```

## Cache Issues

NATS KV `delete()` just adds a tombstone marker. For complete removal:

```bash
# In code
await kv.purge(key);

# Or start with clean cache (EtherNet/IP)
deno run -A main.ts --clear-cache
```

## Bugs Previously Fixed (Don't Reintroduce!)

### 1. Wrong UDT member datatypes (EtherNet/IP)

**Symptom**: VALUE showing as "boolean" instead of "number"

**Root cause**: `decodeTagValue()` checked cached datatype BEFORE typeCode from PLC response

**Fix**: Prioritize typeCode from actual PLC response over cached datatype

**Location**: `tentacle-ethernetip/scanner.ts` `decodeTagValue()` function

### 2. Newly enabled MQTT tags not being polled (EtherNet/IP)

**Root cause**: `autoSubscribeMqttTags()` only called at startup, not on config changes

**Fix**: Added `handleMqttConfigChange()` that syncs subscriptions when MQTT config changes

### 3. Modbus wrong PDU data offset

**Symptom**: All Modbus values read as 0 or garbage

**Root cause**: Modbus response PDU is `[FC, ByteCount, Data...]` — data starts at offset 2, not 1

**Fix**: Use offset 2 in `extractBits()` / `extractWords()` calls

### 4. Modbus stale readLoop closes new connection

**Symptom**: Modbus reconnection fails; connection closes immediately after reconnect

**Root cause**: Old `readLoop` callback runs `.catch()` after reconnect and calls `handleDisconnect()` on the new connection

**Fix**: Generation counter — `connect()` increments generation; readLoop `.catch()` checks `if (this.generation !== gen) return`
