# tentacle-modbus

Polls Modbus TCP devices and publishes data to NATS. Also handles write commands from NATS for bidirectional control.

## Key Architecture Decisions

1. **Custom Modbus TCP client**: No third-party library. The MBAP header + PDU framing is simple enough to implement directly in Deno, avoiding Node.js dependencies.

2. **Stateless, subscriber-driven**: Zero connections at startup. Devices connect on-demand when `modbus.subscribe` requests arrive. Last subscriber leaving closes the connection.

3. **Block reads**: Contiguous tags (within `MAX_GAP = 10` registers) are merged into a single request to minimize round-trips. Float32/Int32 spanning 2 registers and Float64 spanning 4 are handled transparently.

4. **All four function code groups**: Coils (FC01/05/15), Discrete Inputs (FC02), Holding Registers (FC03/06/16), Input Registers (FC04).

5. **Byte order support**: ABCD (big-endian standard), DCBA (little-endian), BADC (byte-swapped), CDAB (word-swapped). Configurable per-device with per-tag overrides.

## Key Files

| File | Purpose |
|------|---------|
| `main.ts` | Entry point, NATS connect, heartbeat, scanner |
| `src/modbus/client.ts` | `ModbusClient` TCP class — connect, readLoop, promise-based request/response |
| `src/modbus/protocol.ts` | MBAP header encode/decode |
| `src/modbus/functions.ts` | FC01–04 reads, FC05/06/15/16 writes |
| `src/modbus/decode.ts` | Register words → typed values, encode for writes |
| `src/modbus/blocks.ts` | `buildBlocks()` — groups tags into contiguous read blocks |
| `src/service/scanner.ts` | Subscriber-driven polling engine |
| `src/nats/publisher.ts` | `publishBatch()` → `modbus.data.{deviceId}` |

## NATS Subjects

| Subject | Direction | Purpose |
|---------|-----------|---------|
| `modbus.subscribe` | request/reply | Add subscriber with tag configs |
| `modbus.unsubscribe` | request/reply | Remove subscriber |
| `modbus.command.{tagId}` | subscribe | Write tag value |
| `modbus.data.{deviceId}` | publish | Per-tag `PlcDataMessage` on each scan |
| `modbus.shutdown` | subscribe | Graceful shutdown |

## Subscribe Request

```typescript
type ModbusSubscribeRequest = {
  deviceId: string;
  host: string;
  port?: number;       // default: 502
  unitId?: number;     // default: 1
  byteOrder?: "ABCD" | "BADC" | "CDAB" | "DCBA";  // default: "ABCD"
  scanRate?: number;   // ms, default: 1000
  tags: Array<{
    id: string;
    address: number;   // 0-based
    functionCode: "coil" | "discrete" | "holding" | "input";
    datatype: "boolean" | "int16" | "uint16" | "int32" | "uint32" | "float32" | "float64";
    byteOrder?: string;
    writable?: boolean;
  }>;
  subscriberId: string;
};
```

## Data Published

```typescript
// Published to modbus.data.{deviceId} once per tag per scan cycle
type PlcDataMessage = {
  moduleId: "modbus";
  deviceId: string;
  variableId: string;   // = tag id
  value: number | boolean;
  datatype: "number" | "boolean";
  timestamp: number;
};
```

## Datatypes

| Modbus datatype | Registers | tentacle-plc type |
|-----------------|-----------|-------------------|
| `boolean`       | 1         | `boolean`         |
| `int16`         | 1         | `number`          |
| `uint16`        | 1         | `number`          |
| `int32`         | 2         | `number`          |
| `uint32`        | 2         | `number`          |
| `float32`       | 2         | `number`          |
| `float64`       | 4         | `number`          |

## Reconnection

Exponential backoff: `2^failures * 1000ms`, capped at 60s, reset on successful connect.

## Testing

Three test layers:

```bash
# Layer 1+2: spec-byte unit tests + fixture server integration
deno task test:unit
deno task test:integration

# Layer 3: third-party validation (requires pymodbus)
pip install pymodbus
python scripts/pymodbus_server.py &
deno task test:smoke
```

## Gotchas

- **Block reads and multi-register types**: Float32 spans 2 registers, Float64 spans 4. The `buildBlocks()` algorithm accounts for this when calculating block extents and word offsets.
- **Byte order is self-inverse**: The same `arrangeBytes()` function is used for both decode and encode — all four byte orders are their own inverse.
- **Address is 0-based**: Modbus spec uses 0-based addresses. If a device datasheet lists addresses starting at 40001 (holding), that maps to address 0 here.
