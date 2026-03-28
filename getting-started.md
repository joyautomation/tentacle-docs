# Getting Started

## Installation

Tentacle runs on Linux (amd64 and arm64). The installer downloads modules from GitHub releases, installs NATS and Deno runtimes, creates systemd units, and starts services.

### Quick Install

```bash
# Download and run (interactive — prompts for module selection)
curl -fsSL https://raw.githubusercontent.com/joyautomation/tentacle/main/install.sh -o install.sh
sudo bash install.sh install
```

### Non-Interactive Install

```bash
# Install with specific optional modules, no prompts
sudo bash install.sh install --yes --modules tentacle-mqtt,tentacle-snmp
```

### Commands

```bash
sudo bash install.sh install    # Fresh install (select modules interactively or via --modules)
sudo bash install.sh update     # Update all installed modules to latest releases
sudo bash install.sh status     # Show installed modules and systemd service status
sudo bash install.sh uninstall  # Stop services and remove /opt/tentacle
```

### Flags

- `--modules mod1,mod2,...` — Install specific optional modules (comma-separated, no spaces)
- `--yes` / `-y` — Skip all confirmation prompts (for scripted installs)

### Core vs Optional Modules

Core modules are always installed:

| Module | Runtime | Description |
|--------|---------|-------------|
| nats | binary | NATS message broker (JetStream) |
| tentacle-graphql | deno | GraphQL API gateway |
| tentacle-web | deno-web | Web dashboard |
| tentacle-orchestrator | deno | Service orchestrator |

Optional modules are selected during install:

| Module | Runtime | Description |
|--------|---------|-------------|
| tentacle-ethernetip | go | EtherNet/IP scanner (Allen-Bradley) |
| tentacle-opcua | go | OPC UA client |
| tentacle-snmp | go | SNMP scanner & trap listener |
| tentacle-mqtt | deno | MQTT Sparkplug B bridge |
| tentacle-history | deno | Edge historian (TimescaleDB) |
| tentacle-modbus | deno | Modbus TCP scanner |
| tentacle-modbus-server | deno | Modbus TCP server |
| tentacle-network | deno | Network interface manager |
| tentacle-nftables | deno | Firewall manager |

### Directory Layout

```
/opt/tentacle/
  bin/              # Binaries: nats-server, deno, Go modules
  services/         # Deno service source (tentacle-graphql/, tentacle-web/, etc.)
  config/           # tentacle.env (shared environment file)
  data/             # NATS JetStream data (data/nats/)
  cache/            # Deno module cache (cache/deno/)
```

### Systemd Services

Each module runs as a systemd service. NATS uses the unit name `tentacle-nats.service`; all other modules use `{moduleId}.service` (e.g., `tentacle-graphql.service`).

```bash
# Service management
systemctl status tentacle-graphql        # Check a service
systemctl restart tentacle-mqtt          # Restart a service
journalctl -u tentacle-orchestrator -f   # Follow logs
```

### Configuration

All services share `/opt/tentacle/config/tentacle.env`:

```bash
# NATS Server
NATS_SERVERS=nats://localhost:4222

# GraphQL API
GRAPHQL_PORT=4000
GRAPHQL_HOSTNAME=0.0.0.0
TENTACLE_MODE=systemd

# Web Dashboard
GRAPHQL_URL=http://localhost:4000/graphql
PORT=3012

# MQTT Sparkplug B Bridge (uncomment to configure)
# MQTT_BROKER_URL=mqtt://localhost:1883
# MQTT_CLIENT_ID=tentacle-mqtt
# MQTT_GROUP_ID=TentacleGroup
# MQTT_EDGE_NODE=EdgeNode1

# OPC UA Client (uncomment to configure)
# OPCUA_PKI_DIR=/opt/tentacle/data/opcua/pki
# OPCUA_AUTO_ACCEPT_CERTS=true
```

Edit this file and restart the relevant service to apply changes.

### Creating a PLC Project

After installation (or standalone with Deno installed):

```bash
deno run -A jsr:@joyautomation/create-tentacle-plc my-plc
cd my-plc
deno task dev
```

---

## Development Setup

### Prerequisites

- [Deno](https://deno.land/) installed (all backend services and tentacle-web)
- [Go](https://go.dev/) 1.22+ (only for building Go scanner modules from source)
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
- tentacle-ethernetip-go
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
# EtherNet/IP scanner (Go)
cd tentacle-ethernetip-go && go run .

# Modbus TCP scanner
cd tentacle-modbus && deno task dev

# MQTT Sparkplug B bridge
cd tentacle-mqtt && deno task dev

# GraphQL API
cd tentacle-graphql && deno task dev

# Web UI
cd tentacle-web && deno task dev
```

## Service Ports

| Service | Port |
|---------|------|
| NATS | 4222 |
| tentacle-graphql | 4000 |
| tentacle-web | 3012 |

## Environment Variables

For development, create a `.env` file in each service directory. For production installs, all services share `/opt/tentacle/config/tentacle.env` (see Installation section above).

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
- **tentacle-web blank page**: Run `deno install` first if deps aren't cached
