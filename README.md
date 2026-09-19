# Neutrino 0.5.10

Signed CPU Installer Drop

Notice: This repository distributes pre-compiled deployment packages and cryptographic verification manifests. Source code and proprietary engine internals are maintained in a private repository.

Neutrino is an enterprise-grade energy accounting engine and autonomous governor for Linux server infrastructure. It meters package energy via Intel RAPL, wraps standard job execution, and applies autonomous optimization policies to target workloads.

Neutrino operates non-invasively: it does not rewrite or patch binaries, does not manipulate kernel CPU scaling governors or sysfs parameters, and exposes no external actuation control plane.

## Release Artifacts

| File | Description |
| --- | --- |
| `neutrino_0.5.10_amd64.deb` | Linux amd64 production package |
| `SHA256SUMS` | SHA-256 cryptographic manifest |
| `SHA256SUMS.sig` | Detached Post-Quantum Level 3 signature of `SHA256SUMS` |
| `sample-report.html` | Client reporting dashboard preview (illustrative layout; not a contract or live host telemetry) |

## Operational Architecture

Neutrino runs as an isolated system daemon supporting two modes:

**Observe Mode (Standard / Default)**

Installs in a passive state. The daemon listens exclusively on local loopback (`127.0.0.1:8741`) to meter energy consumed by wrapped processes (`neutrino-run`, `neutrino-srun`, `neutrino-bsub`). Requires no licensing papers, external network connectivity, or vendor access tokens.

**Apply Mode (Licensed)**

Activated using cryptographically signed offline authorization papers (`lease.json` + `lease.sig`). Enables autonomous energy optimization policies across workloads defined in the host policy configuration.

**Execution Boundary:** `POST /v1/actuate` strictly returns HTTP 403 Forbidden. Neutrino is governed entirely through declarative local policies and wrapper lifecycles; it does not allow remote or external parameter actuation.

## Workload Classification and Autonomous Validation

### Supported Workload Profiles

Neutrino categorizes workloads into defined operational profiles declared in the host policy configuration. Workload policies define the governance boundary:

| Profile | Category | Scope |
| --- | --- | --- |
| `compile` | Build Systems | Compilation, linking, and artifact generation |
| `infer-cpu` | AI / ML | CPU-bound model inference and scoring pipelines |
| `batch` | Batch Compute | Throughput-intensive batch jobs |
| `oltp` | Data Processing | Transaction and record-oriented workloads |
| `mq` | Event Pipelines | High-frequency persistence and message queues |
| `hpc` | High-Performance Compute | Workload manager jobs (`SLURM_*`, LSF) |
| `render` | Asset Generation | Visual processing and simulation rendering |
| `etl` | Data Engineering | Stream and transform processing (meter-only by default) |
| `idle` | System Baseline | Passive system baseline (meter-only floor) |
| `heartbeat` | Health Check | Daemon diagnostic floor (meter-only) |
| `unspecified` | Fallback | Default unassigned workload envelope (meter-only) |

### Continuous Statistical Holdout Validation

Neutrino does not rely on synthetic benchmarks or static efficiency claims. Energy savings are proven dynamically on production hardware through integrated A/B control testing:

1. **Automated Control Holdouts:** The engine autonomously interleaves unoptimized baseline executions to maintain an ongoing control comparison.
2. **Promotion Gate:** Autonomous optimization remains active only when candidate workloads demonstrate statistically significant, sustained package energy reduction against the control baseline.
3. **Safety Fallback:** If a measurement window fails to meet required efficiency thresholds, governance automatically disengages (`off (retry next pass)`) to eliminate intervention overhead, re-evaluating during subsequent cycles.

### Telemetry Status Definitions

- `watching`: Policy active; holdout sampling window currently accumulating.
- `saving`: Workload actively cleared the threshold, demonstrating statistically verified energy reduction.
- `off (retry next pass)`: Performance threshold unreached in the current window; deferred to next evaluation cycle.
- `off`: Profile inactive or omitted from the active policy.
- `meter_only`: Dedicated energy accounting without policy intervention (`idle`, `heartbeat`, `unspecified`, `etl`).

## Safety Controls and Protected Targets

To prevent service disruption to core infrastructure, Neutrino strictly refuses to profile or govern critical system processes:

- Enterprise services and databases: `oracle`, `postgres`, `mariadbd`, `mysqld`, `java`
- System infrastructure: `sshd`, `systemd`, `neutrino.service`

Execution wrappers will immediately reject attempts to run against these targets.

## Installation and Deployment

Package installation requires root staging to comply with standard enterprise privilege policies.

### 1. Verify and Stage

```bash
# Verify the installer against the post-quantum signed manifest
neutrino pkg verify neutrino_0.5.10_amd64.deb SHA256SUMS SHA256SUMS.sig

# Stage package inside a restricted directory (mode 0700)
sudo install -d -m 0700 /var/lib/neutrino/staging
sudo cp neutrino_0.5.10_amd64.deb /var/lib/neutrino/staging/neutrino-install.deb
```

### 2. Install Package

```bash
sudo dpkg -i /var/lib/neutrino/staging/neutrino-install.deb
sudo neutrino-setup --validate
```

### 3. Verify Daemon State

```bash
# Check service status and loopback binding
sudo systemctl status neutrino.service
ss -ltn '( sport = :8741 )'   # Must show 127.0.0.1:8741 only

# Query daemon health status
curl -s http://127.0.0.1:8741/health
```

## Uninstallation

```bash
sudo neutrino-setup --uninstall
```

System configurations are restored from pre-install backups. Local ledgers and cryptographic identity files under `/var/lib/neutrino` are preserved unless explicitly purged.

## Licensing and Offline Verification

Neutrino operates completely air-gapped. Authorization papers are signed offline and validated locally via quantum-resistant digital signatures.

### 1. Generate Node Identity

```bash
sudo neutrino license enroll-print
```

Transmit only `sku=NEUTRINO`, `node_id`, and `pk_sha256` to the license authority. Private keys (`/var/lib/neutrino/node.sk`) must never leave the host.

### 2. Install Authorization Papers

Place the signed cryptographic papers in the configuration directory:

```bash
sudo install -m 0640 -o root -g neutrino lease.json /etc/neutrino/lease.json
sudo install -m 0640 -o root -g neutrino lease.sig  /etc/neutrino/lease.sig
```

### Licensing State

| License File Status | Autonomous Optimization | Operational State |
| --- | --- | --- |
| Unlicensed / Expired / Invalid Signature | Disabled | Passive Metering (Observe Mode) |
| Valid Signature, `claims.apply = false` | Disabled | Licensed Passive Metering |
| Valid Signature, `claims.apply = true` | Enabled | Autonomous Governance (Apply Mode) |

## Operator Commands

### 1. Meter Workloads

```bash
# General CLI execution
sudo neutrino-run --class compile --id build-01 -- make -j

# Batch job execution
sudo neutrino-run --class batch --id batch-run-01 -- /usr/local/bin/worker-job

# Workload scheduler integration (e.g., Slurm)
SLURM_JOB_ID=123 SLURM_JOB_CPUS=8 sudo neutrino-srun --class hpc -- ./solver
```

### 2. Autopilot Management

Autopilot coordinates governance across all approved profiles in the policy configuration:

```bash
# Dry run inspection (inspect without applying state)
sudo neutrino apply preview

# Engage autonomous governance across policy profiles
sudo neutrino apply autopilot on

# Check operational metrics and live status
sudo neutrino apply status
sudo neutrino report

# Disengage governance (instantly reverts to standard unoptimized execution)
sudo neutrino apply autopilot off
```

### 3. Telemetry Export

```bash
neutrino export --format jsonl
neutrino export --format csv
```

## Security Specifications

- **Network Surface:** Daemon binds exclusively to `127.0.0.1:8741`. It never listens on `0.0.0.0` or exposes WAN interfaces.
- **Authentication:** Local administrative endpoints require Bearer token authorization; missing tokens fail closed.
- **Actuation Isolation:** Actuation APIs are disabled (`403 Forbidden`). Neutrino introduces zero runtime RPC surfaces to alter host kernel dials.
- **Cryptographic Boundary:** Host packages include verification runtimes strictly for signature and identity checks; offline signing engines are not distributed within host drops.

## Release Scope and Boundaries

- **Packaging:** Binary release drop. Engine source code is private.
- **Hardware Scope:** Validated for Intel x86_64 CPU package rails via RAPL. GPU metrics are not handled by this build.
- **Empirical Verification:** Energy reporting is derived strictly from real-time execution telemetry on your host via `neutrino report`.

Version 0.5.10. Copyright © 2026 Xylonix. All rights reserved.
