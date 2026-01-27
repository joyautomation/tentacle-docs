# Troubleshooting

## Quick Checks

```bash
# Check for running instances (DO THIS FIRST when debugging stale data)
ps aux | grep tentacle

# Kill all tentacle processes
pkill -f tentacle

# Check NATS is running
nats server info

# Check NATS KV cache
nats kv get field-config-{projectId} cache.variables.{variableId}
```

## Common Issues

### Stale data appearing after refresh

**Symptom**: Old values appear even after restarting services

**Cause**: Multiple instances of tentacle-ethernetip running simultaneously. NATS requests can be answered by ANY running instance.

**Fix**:
```bash
ps aux | grep tentacle
pkill -f tentacle
# Then restart only the services you need
```

### MQTT not publishing / values look "flat"

**Symptoms**:
- MQTT values don't change even though PLC values are changing
- Sparkplug client shows constant values

**Causes & Fixes**:
1. **Deadband filtering**: Values changing by less than deadband won't publish. Check `mqtt-config-{projectId}` KV bucket
2. **Variable not enabled**: Verify variable is enabled in mqtt-config
3. **Data IS being read**: Check the 30s status log in tentacle-ethernetip - it shows sample values

### DCMD commands not working

**Symptom**: Write commands from Ignition/SCADA don't reach the PLC

**Debugging steps**:
1. Check tentacle-mqtt logs for "Received DCMD" message
2. If no DCMD log: synapse version issue (need 0.0.87+)
3. If DCMD received but no "Publishing command": wrong event signature
4. Check tentacle-ethernetip logs for write attempts

**Common causes**:
- Wrong synapse version (need 0.0.87+ for reconnection fixes)
- MQTT reconnection dropped DCMD subscription
- NATS publish encoding issue (must use `TextEncoder().encode()`)

### EtherNet/IP not polling tags

**Symptom**: No data from PLC, tags show as "unknown" or 0

**Causes & Fixes**:
1. **No subscription**: Only polls tags with active subscriptions (from MQTT config or tentacle-plcs)
2. **Connection failed**: Check logs for connection errors, verify PLC IP
3. **Browse first**: Tags must be discovered via browse before polling

### GraphQL subscriptions stale

**Symptom**: Web UI not updating in real-time

**Fixes**:
1. Check `service_heartbeats` KV for service health
2. Verify tentacle-graphql is connected to NATS
3. Check browser console for WebSocket errors

### Service not receiving NATS messages

**Debugging**:
1. Verify topic patterns match between publisher and subscriber
2. Check `PROJECT_ID` matches across services
3. Use `nats sub "plc.data.>"` to see what's being published

## Debug Mode

```bash
# Verbose output for any service
DEBUG=* deno task dev
```

## Cache Issues

NATS KV `delete()` just adds a tombstone marker. For complete removal:

```bash
# In code
await kv.purge(key);

# Or start with clean cache
deno run -A main.ts --clear-cache
```

## Bugs Previously Fixed (Don't Reintroduce!)

### 1. Wrong UDT member datatypes

**Symptom**: VALUE showing as "boolean" instead of "number"

**Root cause**: `decodeTagValue()` checked cached datatype BEFORE typeCode from PLC response

**Fix**: Prioritize typeCode from actual PLC response over cached datatype

**Location**: `scanner.ts` `decodeTagValue()` function

### 2. Newly enabled MQTT tags not being polled

**Root cause**: `autoSubscribeMqttTags()` only called at startup, not on config changes

**Fix**: Added `handleMqttConfigChange()` that syncs subscriptions when MQTT config changes

**Location**: `scanner.ts`
