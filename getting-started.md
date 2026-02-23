# Getting Started

## Prerequisites

- [Deno](https://deno.land/) installed (backend services)
- [Node.js](https://nodejs.org/) 20+ installed (tentacle-web)
- [NATS Server](https://nats.io/) with JetStream enabled
- (Optional) MQTT broker for Sparkplug B integration
- (Optional) Allen-Bradley PLC for EtherNet/IP testing
- (Optional) OPC UA server for OPC UA testing
- (Optional) Modbus TCP device for Modbus testing

## Running NATS

```bash
# Start NATS with JetStream
docker run -d --name nats -p 4222:4222 nats -js

# Or if already created
docker start nats
```

## Running All Services (Recommended)

Use the `dev.sh` script in the repo root to start all services in parallel using [tdev](https://github.com/joyautomation/tdev):

```bash
./dev.sh
```

This starts all services with live reload:
- tentacle-graphql (port 4000)
- tentacle-ethernetip
- tentacle-opcua-go
- tentacle-mqtt
- tentacle-modbus
- tentacle-network
- tentacle-nftables
- tentacle-demo (reference PLC application)
- tentacle-web (port 3012)

## Running Services Individually

Each backend service (Deno) has a `dev` task:

```bash
# EtherNet/IP scanner
cd tentacle-ethernetip && deno task dev

# Modbus TCP scanner
cd tentacle-modbus && deno task dev

# MQTT Sparkplug B bridge
cd tentacle-mqtt && deno task dev

# GraphQL API
cd tentacle-graphql && deno task dev

# Web UI (Node.js/npm — NOT Deno)
cd tentacle-web && npm run dev
```

## Service Ports

| Service | Port |
|---------|------|
| NATS | 4222 |
| tentacle-graphql | 4000 |
| tentacle-web | 3012 |

## Environment Variables

Create a `.env` file in each service directory:

```bash
# Shared (all Deno services)
NATS_SERVERS=localhost:4222

# tentacle-plc / tentacle-demo
PROJECT_ID=my-project

# tentacle-mqtt
MQTT_BROKER_URL=mqtt://localhost:1883
MQTT_GROUP_ID=TentacleGroup
MQTT_EDGE_NODE=EdgeNode

# tentacle-graphql
GRAPHQL_PORT=4000

# tentacle-web
GRAPHQL_URL=http://localhost:4000/graphql
```

## Verifying Everything Works

1. **Check NATS**: `nats server info`
2. **Check GraphQL**: Open `http://localhost:4000/graphql` in browser
3. **Check Web UI**: Open `http://localhost:3012`
4. **Check service topology**: In the web UI, the topology view shows all services with active heartbeats

## Service Discovery

Service discovery is heartbeat-driven — no manual registration needed. Any service that publishes a valid `ServiceHeartbeat` to the `service_heartbeats` KV bucket automatically appears on the topology. Heartbeats are published every 10 seconds with a 60-second TTL.

## Common First-Time Issues

- **"Connection refused" on NATS**: Make sure NATS is running with the `-js` flag (JetStream required)
- **Web UI shows no services**: Check that tentacle-graphql is running and connected to NATS
- **Services don't see each other**: Verify `NATS_SERVERS` is set correctly in each `.env` file
- **tentacle-web blank page**: Run `npm install` first — Node.js deps aren't auto-installed
