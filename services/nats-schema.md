# tentacle-nats-schema

Shared TypeScript types and message validators for the Tentacle platform.

## Package

Published to JSR as `@joyautomation/tentacle-nats-schema`

## What's Included

- NATS topic patterns and substitution helpers (`NATS_TOPICS`, `NATS_SUBSCRIPTIONS`, `substituteTopic`)
- Message type definitions (`PlcDataMessage`, `ServiceHeartbeat`, `ServiceLogEntry`, `BrowseProgressMessage`, etc.)
- Runtime validators (`createPlcDataValidator`, etc.)
- KV bucket name constants (`PLCVariablesBucket`, `ServiceHeartbeatBucket`, etc.)

## Import Paths

```typescript
// Main export — types, topics, validators
import { NATS_TOPICS, substituteTopic, type PlcDataMessage } from "@joyautomation/tentacle-nats-schema";

// Sub-path exports
import { NATS_TOPICS } from "@joyautomation/tentacle-nats-schema/topics";
import { type PlcDataMessage } from "@joyautomation/tentacle-nats-schema/types";
import { ServiceHeartbeatBucket } from "@joyautomation/tentacle-nats-schema/kv";
import { createPlcDataValidator } from "@joyautomation/tentacle-nats-schema/validate";
```

## Usage

```typescript
import {
  NATS_TOPICS,
  NATS_SUBSCRIPTIONS,
  substituteTopic,
  type PlcDataMessage,
  type ServiceHeartbeat,
} from "@joyautomation/tentacle-nats-schema";

// Build a topic
const topic = substituteTopic(NATS_TOPICS.module.data, {
  moduleId: "mixing-process",
  variableId: "temperature",
});
// → "mixing-process.data.temperature"

// Subscribe to all data from all modules
const subject = NATS_SUBSCRIPTIONS.allData(); // → "*.data.>"

// Subscribe to service logs
const logSubject = NATS_SUBSCRIPTIONS.serviceTypeLogs("ethernetip");
// → "service.logs.ethernetip.>"
```

## Key Types

See [NATS Topics & KV](../protocols/nats.md) for full type definitions.

## Publishing

```bash
cd tentacle-nats-schema
# Update version in deno.json
git add . && git commit && git push
deno publish  # Interactive auth
```
