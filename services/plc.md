# tentacle-plc

Lightweight PLC runtime library with task-based programming. Published to JSR as `@joyautomation/tentacle-plc`.

## Overview

A library (not a standalone service) for creating soft PLC logic in TypeScript/Deno. Manages variables, scan tasks, NATS integration, and sourcing values from protocol scanner services.

Runs as `tentacle-demo` in the dev environment — the demo project is the reference implementation of a tentacle-plc application.

## Usage

```typescript
import { createPlc } from "@joyautomation/tentacle-plc";

const plc = createPlc({
  projectId: "my-project",
  nats: { servers: "nats://localhost:4222" },
  variables: {
    temperature: {
      id: "temperature",
      description: "Process temperature",
      datatype: "number",
      default: 0,
      source: eipTag(myPlc, "Temperature"),
    },
  },
  tasks: {
    main: {
      scanRate: 100,
      program: (plc) => {
        // Logic runs every 100ms
      },
    },
  },
});
```

## Variable Sources

Variables can be sourced from any protocol scanner service. The PLC handles subscription management, reconnection, and retry automatically.

### EtherNet/IP

```typescript
import { eipTag } from "@tentacle/plc/ethernetip";
import { rtu45 } from "./generated/ethernetip.ts";

source: eipTag(rtu45, "Program:MainProgram.Motor_Speed")
// subscribes to: ethernetip.data.{deviceId}.{tag}
```

Codegen (requires running tentacle-ethernetip + live device):
```typescript
import { generateEipTypes } from "@tentacle/plc/codegen";
await generateEipTypes({
  nats: { servers: "nats://localhost:4222" },
  devices: [{ id: "rtu45", host: "192.168.1.10" }],
  outputDir: "./generated",
});
```

### OPC UA

```typescript
import { opcuaTag } from "@tentacle/plc/opcua";
import { myServer } from "./generated/opcua.ts";

source: opcuaTag(myServer, "ns=2;s=Temperature")
// subscribes to: opcua.data.{deviceId}.{sanitizedNodeId}
```

Codegen (requires running tentacle-opcua-go + live server):
```typescript
import { generateOpcuaTypes } from "@tentacle/plc/codegen";
await generateOpcuaTypes({
  nats: { servers: "nats://localhost:4222" },
  devices: [{ id: "ignition", endpointUrl: "opc.tcp://ignition:62541" }],
  outputDir: "./generated",
});
```

### Modbus TCP

```typescript
import { modbusTag } from "@tentacle/plc/modbus";
import { pumpSkid } from "./generated/modbus.ts";

source: modbusTag(pumpSkid, "pump_speed")
// subscribes to: modbus.data.{deviceId}  (filters by variableId in message body)
```

Codegen (**no live connection needed** — define the register map directly):
```typescript
import { generateModbusTypes } from "@tentacle/plc/codegen";
await generateModbusTypes({
  devices: [{
    id: "pump-skid",
    host: "192.168.1.100",
    port: 502,
    unitId: 1,
    byteOrder: "ABCD",
    tags: [
      { id: "pump_speed",   address: 0, functionCode: "holding", datatype: "float32" },
      { id: "pump_running", address: 0, functionCode: "coil",    datatype: "boolean" },
      { id: "tank_level",   address: 2, functionCode: "holding", datatype: "uint16",
        byteOrder: "BADC" },  // tag-level byte order override
    ],
  }],
  outputDir: "./generated",
});
```

The generated file (`generated/modbus.ts`) has full addressing info per tag and is type-safe:
```typescript
export const pump_skid = {
  id: "pump-skid", host: "192.168.1.100", port: 502, unitId: 1, byteOrder: "ABCD",
  tags: {
    "pump_speed":   { datatype: "number",  address: 0, functionCode: "holding", modbusDatatype: "float32", byteOrder: "ABCD" },
    "pump_running": { datatype: "boolean", address: 0, functionCode: "coil",    modbusDatatype: "boolean", byteOrder: "ABCD" },
    "tank_level":   { datatype: "number",  address: 2, functionCode: "holding", modbusDatatype: "uint16",  byteOrder: "BADC" },
  },
} as const;
```

`modbusTag(pumpSkid, "pump_speed")` — tag name is autocompleted and compile-checked against the generated constant.

## Variable Datatypes

| Datatype | TypeScript | Sparkplug B |
|----------|-----------|-------------|
| `"number"` | `number` | Float/Double/Int |
| `"boolean"` | `boolean` | Boolean |
| `"string"` | `string` | String |
| `"udt"` | `Record<string, unknown>` | Template Instance |

## Sparkplug B UDT Templates

For variables that map to structured data types in Sparkplug B:

```typescript
{
  id: "motor",
  datatype: "udt",
  default: { speed: 0, running: false },
  udtTemplate: {
    name: "Motor",
    version: "1",
    members: [
      { name: "speed",   datatype: "number" },
      { name: "running", datatype: "boolean" },
    ],
  },
}
```

When `udtTemplate` is set, `tentacle-mqtt` publishes this variable as a Sparkplug B Template Instance rather than a JSON string.

## Source vs Sparkplug B Types

The comms layer (EIP/OPC-UA/Modbus sources) and the Sparkplug B layer are **separate concerns**:

- Protocol sources expose what the downstream device publishes — these are raw inputs
- tentacle-plc maps/transforms/composes them into variables
- Variables are then optionally published upstream as Sparkplug B metrics or UDTs via tentacle-mqtt

Two common patterns:
1. **Mirror**: Device structure maps closely to Sparkplug B → codegen output can serve as a starting point for UDT definitions
2. **Compose**: Multiple sources + internal logic produce new types → Sparkplug B UDTs are defined independently

## NATS Integration

- Variables publish to `{projectId}.data.{variableId}` on change
- State persisted to KV bucket `plc-variables-{projectId}`
- Heartbeat published to `service_heartbeats` KV (10s interval, 60s TTL)
- Listens for `{projectId}.shutdown` for graceful shutdown
- Retries protocol scanner subscriptions every 10s if the scanner isn't available yet

## Key Files

| File | Purpose |
|------|---------|
| `nats.ts` | NATS connection, source subscriptions (EIP/OPC-UA/Modbus), variable publishing |
| `codegen.ts` | `generateEipTypes`, `generateOpcuaTypes`, `generateModbusTypes` |
| `ethernetip.ts` | `eipTag()` helper + `EipDevice` type |
| `opcua.ts` | `opcuaTag()` helper + `OpcUaDevice` type |
| `modbus.ts` | `modbusTag()` helper + `ModbusDevice` type |
| `types/variables.ts` | All variable config and runtime types |
