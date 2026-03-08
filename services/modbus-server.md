# tentacle-modbus-server

Standalone Modbus TCP server that exposes PLC variable data to Modbus TCP clients. The reverse of tentacle-modbus — instead of reading from Modbus devices, it serves data to them.

## Use Case

When external systems (HMIs, SCADA, DCS) need to read PLC data via Modbus TCP, this service creates virtual Modbus devices on-demand. Each virtual device maps PLC variables to Modbus registers/coils and serves them over TCP.

## Key Architecture Decisions

1. **Stateless, on-demand**: Zero devices at startup. Virtual devices are created when `modbus-server.subscribe` requests arrive via NATS.

2. **Bidirectional**: Modbus clients can write to registers/coils. Writes are published back to NATS so the source PLC can act on them.

3. **Register mapping**: Each tag is mapped to a specific Modbus address, function code, and datatype. Multi-register types (float32, int32, float64) are handled with configurable byte ordering.

4. **NATS log streaming**: All logs published to `service.logs.modbus-server.modbus-server` for real-time viewing in the web dashboard.

## Key Files

| File | Purpose |
|------|---------|
| `main.ts` | Entry point, NATS connect, heartbeat, shutdown |
| `src/nats/subscriber.ts` | `ServerManager` — handles subscribe requests, creates virtual devices |
| `src/server/tcp_server.ts` | Modbus TCP server — accepts connections, handles all 8 function codes |
| `src/server/register_store.ts` | In-memory register storage with typed encoding/decoding |
| `src/utils/logger.ts` | Centralized logger with NATS log streaming |
| `src/types.ts` | Type definitions for subscribe requests and tag configs |

## NATS Subjects

| Subject | Direction | Purpose |
|---------|-----------|---------|
| `modbus-server.subscribe` | request/reply | Create a virtual Modbus device |
| `modbus-server.shutdown` | subscribe | Graceful shutdown |
| `plc.data.{sourceModuleId}.*` | subscribe | Receive PLC variable updates |
| `{sourceModuleId}/{variableId}` | publish | Write-back from Modbus client |
| `service.logs.modbus-server.modbus-server` | publish | Log streaming |

## Subscribe Request

```typescript
type SubscribeRequest = {
  deviceId: string;           // Unique ID for this virtual device
  port?: number;              // TCP port to listen on (default: 5020)
  unitId?: number;            // Modbus unit ID (default: 1)
  sourceModuleId: string;     // PLC module to subscribe to for data
  tags: Array<{
    variableId: string;       // PLC variable to map
    address: number;          // Modbus register/coil address (0-based)
    functionCode: "coil" | "discrete" | "holding" | "input";
    datatype: "boolean" | "int16" | "uint16" | "int32" | "uint32" | "float32" | "float64";
    byteOrder?: "ABCD" | "BADC" | "CDAB" | "DCBA";
    writable?: boolean;       // Allow Modbus clients to write (default: false)
  }>;
};
```

## Data Flow

```
PLC Runtime → NATS (plc.data.{moduleId}.*)
                    ↓
          modbus-server subscribes
                    ↓
          RegisterStore.updateFromVariable()
                    ↓
          Modbus TCP Client reads registers
                    ↓ (on write)
          onWrite → NATS ({moduleId}/{variableId})
                    ↓
          PLC Runtime receives write command
```

## Supported Function Codes

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

## Environment Variables

```bash
NATS_SERVERS=localhost:4222          # NATS server URL(s)
SUBSCRIBE_SUBJECT=modbus-server.subscribe  # Subject to listen for subscribe requests
```

## Running

```bash
cd tentacle-modbus-server
deno task dev
```

## Testing

```bash
deno test --allow-net --allow-env tests/
```
