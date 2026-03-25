# Tentacle Platform Documentation

Tentacle is a distributed IIoT platform built on Deno with NATS as the message bus. It bridges industrial PLCs to MQTT/GraphQL for real-time monitoring and control.

## Quick Links

- [Architecture Overview](./architecture.md) - System design and data flow
- [Getting Started](./getting-started.md) - Development setup and running locally
- [Troubleshooting](./troubleshooting.md) - Common issues and debugging

## Services

| Service | Runtime | Description | Docs |
|---------|---------|-------------|------|
| tentacle-graphql | Deno | GraphQL API with real-time subscriptions (SSE) | [Docs](./services/graphql.md) |
| tentacle-web | Deno | SvelteKit dashboard with topology view and log streaming | [Docs](./services/web.md) |
| tentacle-gateway | Deno | Config-driven PLC runtime — reads device/variable config from NATS KV and builds a tentacle-plc instance | — |
| tentacle-ethernetip-go | Go | Allen-Bradley PLC scanner (EtherNet/IP via libplctag) | [Docs](./services/ethernetip.md) |
| tentacle-ethernetip-server-go | Go | EtherNet/IP server — exposes PLC data to EtherNet/IP clients | — |
| tentacle-opcua-go | Go | OPC UA client (gopcua) | — |
| tentacle-modbus | Deno | Modbus TCP scanner with block reads | [Docs](./services/modbus.md) |
| tentacle-modbus-server | Deno | Modbus TCP server — exposes PLC data to Modbus clients | [Docs](./services/modbus-server.md) |
| tentacle-snmp | Go | SNMP scanner and trap listener (gosnmp) | — |
| tentacle-mqtt | Deno | NATS to MQTT bridge (Sparkplug B) with store-and-forward | [Docs](./services/mqtt.md) |
| tentacle-history | Deno | Edge-local TimescaleDB historian with RBE filtering and time-bucketed aggregation | — |
| tentacle-mcp | Deno | MCP server — exposes GraphQL API as tools for AI agents | — |
| tentacle-network | Deno | Network interface monitoring and netplan configuration (Linux) | — |
| tentacle-nftables | Deno | Firewall and NAT management via nftables (Linux) | — |
| tentacle-demo | Deno | Example PLC project using tentacle-plc (EtherNet/IP and SNMP demos) | — |

## Libraries

| Package | Description | Docs |
|---------|-------------|------|
| tentacle-plc | PLC runtime library for variables and tasks | [Docs](./services/plc.md) |
| tentacle-nats-schema | Shared NATS topic and message type definitions (published to JSR) | [Docs](./services/nats-schema.md) |
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
- [tentacle-gateway](https://github.com/joyautomation/tentacle-gateway)
- [tentacle-ethernetip-go](https://github.com/joyautomation/tentacle-ethernetip-go)
- [tentacle-ethernetip-server-go](https://github.com/joyautomation/tentacle-ethernetip-server-go)
- [tentacle-opcua-go](https://github.com/joyautomation/tentacle-opcua-go)
- [tentacle-modbus](https://github.com/joyautomation/tentacle-modbus)
- [tentacle-modbus-server](https://github.com/joyautomation/tentacle-modbus-server)
- [tentacle-snmp](https://github.com/joyautomation/tentacle-snmp)
- [tentacle-demo](https://github.com/joyautomation/tentacle-demo)
- [tentacle-mqtt](https://github.com/joyautomation/tentacle-mqtt)
- [tentacle-history](https://github.com/joyautomation/tentacle-history)
- [tentacle-mcp](https://github.com/joyautomation/tentacle-mcp)
- [tentacle-network](https://github.com/joyautomation/tentacle-network)
- [tentacle-nftables](https://github.com/joyautomation/tentacle-nftables)

**Libraries:**
- [tentacle-plc](https://github.com/joyautomation/tentacle-plc)
- [tentacle-nats-schema](https://github.com/joyautomation/tentacle-nats-schema)
- [create-tentacle-plc](https://github.com/joyautomation/create-tentacle-plc)
