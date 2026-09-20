# Neutrino Energy Governor 0.6.3

Enterprise observe-first energy and performance accounting engine.

## Overview

**Neutrino Energy Governor 0.6.3** provides continuous, hardware-level CPU energy accounting and autonomous optimization across enterprise workloads:

- **Observe Mode (Free)**: Passive, transparent metering of wrapped processes (`neutrino-run`, `neutrino-srun`, `neutrino-bsub`). Requires no licensing papers, external network access, or credentials.
- **Apply / Autopilot Mode (Licensed)**: Activated by an offline cryptographically signed desk paper (`sku=NEUTRINO` with `claims.apply = true`). Autonomous governance optimizes eligible workloads based on live efficiency measurements.

> **Repository Scope**: This repository distributes **packages and documentation only**, not engine source code.

---

## Autonomous Governance: 19+1 Autopilot

Neutrino 0.6.3 operates an autonomous, closed-loop 19+1 coverage model:

- **19+1 Cycle**: Per workload class, Neutrino executes 19 optimized (wrapped) runs followed by 1 unoptimized (unwrapped) baseline reference run.
- **Statistical Savings Gate**: A workload class remains covered and actively optimized if and only if measured savings meet the efficiency threshold ($\Delta E \le -5\%$) compared against the unwrap reference.
- **Recipe Immortality**: The local recipe (`recipe.json`) serves as the node-owned coverage map and **never expires**. Expiration timestamps (`not_after`) in the recipe are treated as inert metadata.
- **Single Clock Authority**: The offline authenticator-signed lease (`lease.json`) is the **sole clock authority** governing licensing validity.

---

## Security Architecture & API Boundary

- **Local Loopback Only**: The daemon binds strictly to `127.0.0.1:8741`. It never listens on external interfaces (`0.0.0.0`) or connects outbound.
- **Immutable Actuation Boundary**: `POST /v1/actuate` strictly returns **403 Forbidden**. Neutrino does not expose remote or RPC actuation endpoints to alter kernel parameters; optimization is enforced purely through declarative local policies and process wrappers.
- **Zero Secrets**: Host packages contain only signature verification keys; signing keys and private credentials are never distributed.

---

## Verification & Installation

Always verify package signatures and cryptographic checksums prior to installation.

### 1. Verify Signed Manifest

```bash
# Verify debian package against DSA-16 Level 3 operator signature
neutrino pkg verify neutrino_0.6.3_amd64.deb SHA256SUMS SHA256SUMS.sig
```

### 2. Stage and Install

```bash
# Stage in restricted directory (mode 0700)
sudo install -d -m 0700 /var/lib/neutrino/staging
sudo cp neutrino_0.6.3_amd64.deb /var/lib/neutrino/staging/neutrino-install.deb

# Install package
sudo dpkg -i /var/lib/neutrino/staging/neutrino-install.deb
sudo neutrino-setup --validate
```

### 3. Service Verification

```bash
# Ensure daemon is active and listening strictly on local loopback
sudo systemctl status neutrino.service
ss -ltn '( sport = :8741 )'   # 127.0.0.1:8741 only

# Query daemon health
curl -s http://127.0.0.1:8741/health
```

---

## Workload Profiles

Neutrino categorizes execution profiles for accounting and governance:

| Profile | Category | Scope |
| --- | --- | --- |
| `compile` | Build Systems | Compilation, linking, and artifact generation |
| `infer-cpu` | AI / ML | CPU-bound model inference and scoring pipelines |
| `batch` | Batch Compute | Throughput-intensive batch processing |
| `oltp` | Data Processing | Transaction and record-oriented workloads |
| `mq` | Event Pipelines | High-frequency persistence and message queues |
| `hpc` | High-Performance Compute | Scheduler jobs (`SLURM_*`, LSF) |
| `render` | Asset Generation | Visual processing and simulation rendering |
| `etl` | Data Engineering | Stream and transform processing (meter-only floor) |
| `idle` | System Baseline | Passive system baseline (meter-only floor) |
| `heartbeat` | Health Check | Daemon diagnostic floor (meter-only floor) |
| `unspecified` | Fallback | Unassigned workload envelope (meter-only floor) |

### Protected System Targets

To preserve infrastructure integrity, execution wrappers immediately reject wrapping critical system processes:
- Database engines: `oracle`, `postgres`, `mariadbd`, `mysqld`
- System supervisors and infrastructure: `sshd`, `systemd`, `neutrino.service`

---

## Operator Usage

```bash
# General CLI execution (metered)
sudo neutrino-run --class compile --id build-01 -- make -j

# Scheduler integration (Slurm)
SLURM_JOB_ID=101 SLURM_JOB_CPUS=8 sudo neutrino-srun --class hpc -- ./solver

# Check system license and coverage state
sudo neutrino license show

# Telemetry export
neutrino export --format jsonl
```

---

## Licensing State

| Authorization Paper | Operational State | Capability |
| --- | --- | --- |
| Unlicensed / Expired Lease | Observe Mode | Passive hardware energy metering |
| Valid Lease, `claims.apply = false` | Licensed Observe | Passive metering under licensed terms |
| Valid Lease, `claims.apply = true` | Apply / Autopilot Mode | Autonomous 19+1 governance across covered profiles |

Private keys generated during enrollment must remain strictly on the host and must never be transmitted.

---

Copyright © 2026 Xylonix. All rights reserved.
