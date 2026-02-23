# Tentacle Platform Documentation

Tentacle is a distributed IIoT platform built on Deno with NATS as the message bus. It bridges industrial PLCs to MQTT/GraphQL for real-time monitoring and control.

## Quick Links

- [Architecture Overview](./architecture.md) - System design and data flow
- [Getting Started](./getting-started.md) - Development setup and running locally
- [Troubleshooting](./troubleshooting.md) - Common issues and debugging

## Services

| Service | Description | Docs |
|---------|-------------|------|
| tentacle-ethernetip | Polls Allen-Bradley PLCs via EtherNet/IP | [Docs](./services/ethernetip.md) |
| tentacle-modbus | Polls Modbus TCP devices, block reads, all FC groups | [Docs](./services/modbus.md) |
| tentacle-mqtt | Bridges NATS to MQTT using Sparkplug B | [Docs](./services/mqtt.md) |
| tentacle-graphql | GraphQL API with real-time subscriptions | [Docs](./services/graphql.md) |
| tentacle-web | SvelteKit frontend | [Docs](./services/web.md) |
| tentacle-plc | Lightweight PLC runtime library | [Docs](./services/plc.md) |
| tentacle-nats-schema | Shared TypeScript types and validators | [Docs](./services/nats-schema.md) |

## Protocols

- [NATS Topics & KV](./protocols/nats.md) - Message bus topics and KV buckets
- [Sparkplug B](./protocols/sparkplug.md) - MQTT Sparkplug B integration
- [CIP / EtherNet/IP](./protocols/cip.md) - Allen-Bradley PLC communication

## Repositories

All repos live under `github.com/joyautomation/`:

- [tentacle-ethernetip](https://github.com/joyautomation/tentacle-ethernetip)
- [tentacle-mqtt](https://github.com/joyautomation/tentacle-mqtt)
- [tentacle-graphql](https://github.com/joyautomation/tentacle-graphql)
- [tentacle-web](https://github.com/joyautomation/tentacle-web)
- [tentacle-plc](https://github.com/joyautomation/tentacle-plc)
- [tentacle-nats-schema](https://github.com/joyautomation/nats-schema)
