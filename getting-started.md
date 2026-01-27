# Getting Started

## Prerequisites

- [Deno](https://deno.land/) installed
- [NATS Server](https://nats.io/) with JetStream enabled
- (Optional) MQTT broker for Sparkplug B integration
- (Optional) Allen-Bradley PLC for EtherNet/IP testing

## Running NATS

```bash
# Start NATS with JetStream
docker run -d --name nats -p 4222:4222 nats -js

# Or if already created
docker start nats
```

## Environment Variables

Create `.env` file in each service directory:

```bash
# Shared
NATS_SERVERS=localhost:4222
PROJECT_ID=my-project

# tentacle-mqtt
MQTT_BROKER_URL=mqtt://localhost:1883
MQTT_GROUP_ID=TentacleGroup
MQTT_EDGE_NODE=EdgeNode

# tentacle-graphql
GRAPHQL_PORT=4000
GRAPHQL_PLAYGROUND=true

# tentacle-ethernetip
CLEAR_CACHE=false
```

## Running Services

Each service has a `dev` task:

```bash
# Terminal 1 - EtherNet/IP scanner
cd tentacle-ethernetip
deno task dev

# Terminal 2 - MQTT bridge
cd tentacle-mqtt
deno task dev

# Terminal 3 - GraphQL API
cd tentacle-graphql
deno task dev

# Terminal 4 - Web UI
cd tentacle-web
deno run -A npm:vite dev
```

## Service Ports

| Service | Port |
|---------|------|
| NATS | 4222 |
| tentacle-graphql | 4000 |
| tentacle-web | 5173 (dev) |

## Verifying Everything Works

1. **Check NATS**: `nats server info`
2. **Check GraphQL**: Open `http://localhost:4000/graphql` in browser
3. **Check Web UI**: Open `http://localhost:5173`
4. **Check services**: `ps aux | grep tentacle`

## Common First-Time Issues

- **"Connection refused" on NATS**: Make sure NATS is running with `-js` flag
- **Services don't see each other**: Verify `PROJECT_ID` matches across all `.env` files
- **Web UI shows no data**: Check that tentacle-graphql is running and connected to NATS
