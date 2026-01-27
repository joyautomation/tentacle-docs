# tentacle-nats-schema

Shared TypeScript types and message validators for the Tentacle platform.

## Package

Published to JSR as `@joyautomation/tentacle-nats-schema`

## What's Included

- NATS topic patterns and substitution helpers
- Message type definitions (PlcDataMessage, ServiceHeartbeat, etc.)
- Runtime validators (`is*Message()` functions)
- KV bucket name constants

## Usage

```typescript
import {
  NATS_TOPICS,
  substituteTopic,
  isPlcDataMessage,
  type PlcDataMessage
} from "@joyautomation/tentacle-nats-schema";

// Build topic
const topic = substituteTopic(NATS_TOPICS.plc.data, {
  projectId: "my-project",
  variableId: "Temperature"
});

// Validate message
if (isPlcDataMessage(data)) {
  // TypeScript knows data is PlcDataMessage
}
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
