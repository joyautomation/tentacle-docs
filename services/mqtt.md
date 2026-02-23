# tentacle-mqtt

Bridges NATS to MQTT using Sparkplug B protocol. Handles both reading (variables → MQTT) and writing (MQTT DCMD → variables).

## Bidirectional Data Flow

```
Sources → NATS → tentacle-mqtt → Sparkplug B → MQTT Broker
                                                    ↓
Variables ← NATS ← tentacle-mqtt ← DCMD ←──────────
```

Data sources feeding NATS can include:
- `tentacle-ethernetip` → EtherNet/IP PLC data
- `tentacle-opcua-go` → OPC UA server data
- `tentacle-modbus` → Modbus TCP device data
- `tentacle-plc` → composed/processed variables

## Dependencies

Uses `@joyautomation/synapse` for Sparkplug B protocol. See [Sparkplug docs](../protocols/sparkplug.md) for synapse details.

## Sparkplug B UDT Templates

When a variable has a `udtTemplate` defined (in tentacle-plc), tentacle-mqtt publishes it as a Sparkplug B Template Instance rather than a JSON string. Template definitions are published in NBIRTH.

## DCMD Handling (Write Commands)

When Ignition or another Sparkplug client sends a DCMD:

1. Synapse receives DCMD on `spBv1.0/{group}/DCMD/{edgeNode}/{deviceId}`
2. Emits `dcmd` event with `(topic, payload)`
3. tentacle-mqtt extracts metric name/value from payload
4. Publishes to NATS: `{projectId}/{variableId}` with value as string
5. Scanner service (EtherNet/IP, Modbus, etc.) receives and writes to device

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

## NATS Publishing

NATS requires `Uint8Array`, not strings:

```typescript
// WRONG - silently fails or errors
nc.publish(subject, String(value));

// CORRECT
nc.publish(subject, new TextEncoder().encode(String(value)));
```

## Configuration

Uses NATS KV bucket `mqtt-config-{projectId}` for:
- Which variables are enabled for MQTT
- Per-variable deadband settings
- Default deadband values

## Key Files

| File | Purpose |
|------|---------|
| `mqtt.ts` | Main bridge logic, DCMD handling |
| `main.ts` | Entry point, config loading |
| `types/config.ts` | Configuration types |
| `types/mappings.ts` | Sparkplug ↔ variable type conversions |
