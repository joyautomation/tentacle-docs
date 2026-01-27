# tentacle-plc

Lightweight PLC runtime library with task-based programming.

## Overview

A library (not a service) for creating soft PLC logic in TypeScript/Deno.

## Key Features

- Task-based execution model
- Publishes data to NATS
- Integrates with tentacle ecosystem

## Usage

```typescript
import { createPlc } from "@joyautomation/tentacle-plc";

const plc = createPlc({
  projectId: "my-project",
  // ...
});
```

## Publishing

Published to JSR as `@joyautomation/tentacle-plc`
