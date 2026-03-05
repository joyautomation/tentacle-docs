# Tentacle Platform Documentation

Tentacle is a distributed IIoT platform built on Deno with NATS as the message bus. It bridges industrial PLCs to MQTT/GraphQL for real-time monitoring and control.

## Quick Links

- [Architecture Overview](./architecture.md) - System design and data flow
- [Getting Started](./getting-started.md) - Development setup and running locally
- [Troubleshooting](./troubleshooting.md) - Common issues and debugging

## Services

| Service | Description | Docs |
|---------|-------------|------|
| tentacle-graphql | GraphQL API with real-time subscriptions (SSE) | [Docs](./services/graphql.md) |
| tentacle-web | SvelteKit dashboard with topology view and log streaming | [Docs](./services/web.md) |
| tentacle-ethernetip | Allen-Bradley PLC scanner (EtherNet/IP) | [Docs](./services/ethernetip.md) |
| tentacle-opcua-go | OPC UA client (Go) | — |
| tentacle-modbus | Modbus TCP scanner with block reads | [Docs](./services/modbus.md) |
| tentacle-mqtt | NATS to MQTT bridge (Sparkplug B) | [Docs](./services/mqtt.md) |
| tentacle-network | Network interface monitoring (Linux) | — |
| tentacle-nftables | Firewall and NAT management (Linux) | — |

## Libraries

| Package | Description | Docs |
|---------|-------------|------|
| tentacle-plc | PLC runtime library for variables and tasks | [Docs](./services/plc.md) |
| tentacle-nats-schema | Shared NATS topic and message type definitions | [Docs](./services/nats-schema.md) |
| create-tentacle-plc | Project scaffolding tool | [Docs](./services/plc.md) |

## Protocols

- [NATS Topics & KV](./protocols/nats.md) - Message bus topics and KV buckets
- [Sparkplug B](./protocols/sparkplug.md) - MQTT Sparkplug B integration
- [CIP / EtherNet/IP](./protocols/cip.md) - Allen-Bradley PLC communication

## Repositories

All repos live under `github.com/joyautomation/`:

**Distribution:**
- [tentacle](https://github.com/joyautomation/tentacle) - Platform installer and release packaging

**Services:**
- [tentacle-graphql](https://github.com/joyautomation/tentacle-graphql)
- [tentacle-web](https://github.com/joyautomation/tentacle-web)
- [tentacle-ethernetip](https://github.com/joyautomation/tentacle-ethernetip)
- [tentacle-opcua-go](https://github.com/joyautomation/tentacle-opcua-go)
- [tentacle-modbus](https://github.com/joyautomation/tentacle-modbus)
- [tentacle-mqtt](https://github.com/joyautomation/tentacle-mqtt)
- [tentacle-network](https://github.com/joyautomation/tentacle-network)
- [tentacle-nftables](https://github.com/joyautomation/tentacle-nftables)

**Libraries:**
- [tentacle-plc](https://github.com/joyautomation/tentacle-plc)
- [tentacle-nats-schema](https://github.com/joyautomation/tentacle-nats-schema)
- [create-tentacle-plc](https://github.com/joyautomation/create-tentacle-plc)
