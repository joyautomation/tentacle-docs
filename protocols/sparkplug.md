# Sparkplug B & Synapse

## Overview

Sparkplug B is an MQTT-based protocol for industrial IoT. Tentacle uses `@joyautomation/synapse` library for Sparkplug B support.

Synapse repo: `/home/joyja/Development/joyautomation/kraken/synapse`

## Sparkplug Topics

```
spBv1.0/{groupId}/NBIRTH/{edgeNode}           # Node birth
spBv1.0/{groupId}/NDEATH/{edgeNode}           # Node death
spBv1.0/{groupId}/NDATA/{edgeNode}            # Node data
spBv1.0/{groupId}/NCMD/{edgeNode}             # Node command
spBv1.0/{groupId}/DBIRTH/{edgeNode}/{device}  # Device birth
spBv1.0/{groupId}/DDEATH/{edgeNode}/{device}  # Device death
spBv1.0/{groupId}/DDATA/{edgeNode}/{device}   # Device data
spBv1.0/{groupId}/DCMD/{edgeNode}/{device}    # Device command (write)
STATE/{primaryHostId}                          # Host state (ONLINE/OFFLINE)
```

## Synapse Event Signatures

**IMPORTANT**: Getting these wrong causes silent failures.

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

## Synapse Version History

| Version | Fix |
|---------|-----|
| 0.0.86 | Re-subscribes to DCMD/NCMD/STATE topics on MQTT reconnect |
| 0.0.87 | Preserves external event listeners (like "dcmd") during disconnect |

### Why These Fixes Matter

1. **mqtt.js auto-reconnection**: When connection drops, mqtt.js reconnects but with `clean: true` the broker forgets all subscriptions. Synapse 0.0.86+ re-subscribes in `onConnect()`.

2. **Event listener preservation**: The `disconnect` transition used to call `cleanUpEventListeners(node.events)` which removed ALL listeners including external ones. Synapse 0.0.87+ only removes internal "ncmd" listener.

## Publishing Synapse to JSR

```bash
cd /path/to/synapse
# Update version in deno.json
git add deno.json && git commit -m "chore: bump version"
git tag X.X.X && git push origin main && git push origin X.X.X
# GitHub workflow auto-publishes on tag push
```

## Sparkplug Metric Types

| Type | Description |
|------|-------------|
| boolean | True/false |
| int8, int16, int32, int64 | Signed integers |
| uint8, uint16, uint32, uint64 | Unsigned integers |
| float, double | Floating point |
| string | Text |
| bytes | Binary data |
| datetime | Timestamp |
