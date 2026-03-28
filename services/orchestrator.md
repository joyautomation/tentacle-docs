# tentacle-orchestrator

Bare-metal service orchestrator for the tentacle platform. Watches the `desired_services` NATS KV bucket and reconciles with systemd — downloading, installing, version-switching, and starting/stopping modules automatically.

**Analogy:** NATS KV is etcd, tentacle-orchestrator is the kubelet, systemd units are pods.

## Overview

```
desired_services KV (source of truth)
        │
        ▼
  tentacle-orchestrator (reconciler)
        │
        ├── checks installed versions on disk
        ├── downloads from GitHub releases if needed
        ├── manages symlinks to active version
        ├── starts/stops systemd units
        │
        ▼
  service_status KV (reports back)
```

**What orchestrator manages:** All modules except NATS, Deno runtime, and itself.
**What install.sh manages:** NATS, Deno runtime, orchestrator itself, initial bootstrap.

## Key Architecture Decisions

1. **KV-driven reconciliation**: The `desired_services` KV bucket is the single source of truth. Write a key to install/start a module; delete it to stop managing it. No imperative API.

2. **Two reconciliation mechanisms**:
   - **KV Watch (reactive):** Watches `desired_services` for changes, triggers immediate reconcile
   - **Periodic sweep (defensive, every 30s):** Catches drift from crashes, manual `systemctl`, etc.

3. **Symlink-based versioning**: Each version is stored in its own directory under `versions/{moduleId}/{version}/`. The active version is a symlink in `bin/` (Go) or `services/` (Deno). Rollback = update KV version.

4. **Offline resilience**: If `version: "latest"` but no internet, falls back to the highest locally-installed version. If no local version exists, sets `reconcileState: "version_unavailable"`.

5. **Bootstrap from existing installs**: On first boot, scans existing `bin/` and `services/` directories, moves files to `versions/{moduleId}/unknown/`, and populates `desired_services` with current systemd state.

## NATS KV Buckets

### `desired_services` (no TTL)

One key per module. Missing key = orchestrator ignores that module.

```typescript
type DesiredServiceKV = {
  moduleId: string;    // e.g. "tentacle-mqtt"
  version: string;     // e.g. "0.0.5" or "latest"
  running: boolean;    // should the systemd unit be active?
  updatedAt: number;   // epoch ms
};
```

### `service_status` (TTL 120s)

Written by the orchestrator every reconcile loop. Auto-expires if orchestrator dies.

```typescript
type ReconcileState = "ok" | "pending" | "downloading" | "installing"
  | "starting" | "stopping" | "error" | "version_unavailable";

type ServiceStatusKV = {
  moduleId: string;
  installedVersions: string[];
  activeVersion: string | null;
  systemdState: "active" | "inactive" | "failed" | "not-found";
  reconcileState: ReconcileState;
  lastError: string | null;
  runtime: ModuleRuntime;        // "go" | "deno" | "deno-web"
  category: ModuleCategory;      // "core" | "optional"
  repo: string;
  updatedAt: number;
};
```

### Relationship to `service_enabled`

- `desired_services.running` = should the systemd **process** be running?
- `service_enabled` = should a running service **do work** vs idle heartbeat-only?
- Orthogonal: `running: true` + `enabled: false` = process alive but paused

## Reconciliation Algorithm

Per-module, each reconcile cycle:

1. **Resolve version** — if `"latest"`, query GitHub API (cached 5 minutes). If offline, use highest local version.
2. **Download if missing** — if resolved version not on disk, download from GitHub releases and install to `versions/{moduleId}/{version}/`.
3. **Switch version** — if active version differs from desired, stop service, update symlink, regenerate systemd unit, `daemon-reload`.
4. **Start/stop** — if `running` state doesn't match systemd state, start or stop as needed.
5. **Report status** — write `ServiceStatusKV` to the `service_status` KV bucket.

## Version Storage Layout

```
/opt/tentacle/
  versions/                           # Versioned storage
    tentacle-mqtt/
      0.0.5/                         # Deno source
        deno.json, main.ts, ...
      0.0.6/
        deno.json, main.ts, ...
    tentacle-snmp/
      0.0.3/                         # Go binary
        tentacle-snmp
  bin/
    tentacle-snmp -> ../versions/tentacle-snmp/0.0.4/tentacle-snmp
    deno                              # NOT managed by orchestrator
    nats-server                       # NOT managed by orchestrator
  services/
    tentacle-mqtt -> ../versions/tentacle-mqtt/0.0.6/
    tentacle-web -> ../versions/tentacle-web/0.0.7/
```

## Download URLs (per runtime)

| Runtime | Release Asset | Install Location | Symlink |
|---------|--------------|------------------|---------|
| go | `{moduleId}-linux-{arch}` | `versions/{moduleId}/{ver}/{moduleId}` | `bin/{moduleId}` |
| deno | `{repo}-src.tar.gz` | `versions/{moduleId}/{ver}/` | `services/{repo}` |
| deno-web | `{repo}-build.tar.gz` | `versions/{moduleId}/{ver}/` | `services/{repo}` |

## Module Registry

Static registry in `types/registry.ts` mirrors install.sh's MODULES array. Each entry defines `repo`, `moduleId`, `category`, `runtime`, and optional `extraEnv`.

The orchestrator only manages modules listed in this registry. Unknown moduleIds in `desired_services` are logged and skipped.

## Configuration

Environment variables (all optional — sensible defaults for production):

| Variable | Default | Description |
|----------|---------|-------------|
| `NATS_SERVERS` | `localhost:4222` | NATS server address |
| `TENTACLE_INSTALL_DIR` | `/opt/tentacle` | Base install directory |
| `TENTACLE_SYSTEMD_DIR` | `/etc/systemd/system` | Systemd unit directory |
| `TENTACLE_NATS_UNIT` | `tentacle-nats` | NATS systemd unit name (without `.service`) |
| `TENTACLE_GH_ORG` | `joyautomation` | GitHub org for release downloads |
| `TENTACLE_RECONCILE_INTERVAL` | `30000` | Periodic sweep interval (ms) |
| `TENTACLE_LATEST_CACHE_TTL` | `300000` | Cache duration for "latest" version resolution (ms) |

## GraphQL API

The orchestrator's KV data is exposed via tentacle-graphql:

**Queries:**
- `desiredServices: [DesiredService]` — list all entries in `desired_services` KV
- `serviceStatuses: [ServiceStatus]` — list all entries in `service_status` KV

**Mutations:**
- `setDesiredService(moduleId, version, running): DesiredService` — create/update a desired service entry
- `deleteDesiredService(moduleId): Boolean` — remove a desired service entry

## Key Files

| File | Purpose |
|------|---------|
| `main.ts` | Entry point: NATS connect, heartbeat, start reconciler |
| `types/config.ts` | Env-based config with defaults |
| `types/registry.ts` | MODULE_REGISTRY constant |
| `reconciler/reconciler.ts` | Main loop (KV watch + periodic sweep) |
| `reconciler/download.ts` | GitHub release download, version resolution |
| `reconciler/install.ts` | Extract, install, symlink management |
| `reconciler/systemd.ts` | systemctl start/stop/status, unit file generation |
| `reconciler/status.ts` | Write ServiceStatusKV to KV |
| `reconciler/migration.ts` | Bootstrap from pre-orchestrator installs |
| `nats/client.ts` | NATS connection, KV handles |

## Gotchas

- **Self-update**: The orchestrator cannot restart itself in-place. It writes a shell script that updates the symlink and runs `systemctl restart tentacle-orchestrator`, then executes it fire-and-forget.
- **NATS unit name**: In dev environments where NATS runs as `nats.service` instead of `tentacle-nats.service`, set `TENTACLE_NATS_UNIT=nats`.
- **Bootstrap marker**: After first migration, writes `/opt/tentacle/config/.orchestrator-migrated` to avoid re-migrating on restart.
- **Rate limiting**: GitHub API calls for "latest" resolution are cached for 5 minutes to avoid rate limits.
