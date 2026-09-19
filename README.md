# Neutrino 0.5.10

Signed **CPU installer drop**. This repository is not the product source tree.

Neutrino is an enterprise-grade energy accounting engine and autonomous governor for Linux servers. It meters package joules (Intel RAPL, `method: uj_delta`), wraps named jobs, and — when licensed — applies a **policy wrap** around work the operator already runs. It does not rewrite customer binaries, does not change CPU scaling governors or sysfs dials, and does not expose an actuation API.

| File | Role |
| --- | --- |
| `neutrino_0.5.10_amd64.deb` | Linux amd64 package |
| `SHA256SUMS` | SHA-256 manifest of the `.deb` release package |
| `SHA256SUMS.sig` | Detached Post-Quantum DSA-16 Level 3 signature of `SHA256SUMS` |
| `sample-report.html` | Buyer energy-report layout (existing preview) |

Open `sample-report.html` for the buyer report page layout. Numbers there are a layout preview of campaign-style class stats, not a live host dump or a quoted SLA contract.

---

## What It Does

Two modes on one daemon:

**Observe (always free).** After installation, the daemon listens exclusively on loopback `127.0.0.1:8741`, records RAPL package energy, and measures jobs wrapped with `neutrino-run`, `neutrino-srun`, or `neutrino-bsub`. Zero licensing, tokens, or vendor contact required. This is the try-before-buy meter.

**Apply (licensed).** A Post-Quantum DSA-16 Level 3 signed desk paper (`lease.json` + `lease.sig`, with `sku=NEUTRINO`, unexpired, fingerprint-bound, unseen nonce, and `claims.apply=true`) enables autonomous governing. When the operator turns **autopilot** on, eligible classes on the host recipe go `watching`, then independently transition to `saving` or `off (retry next pass)`. Apply is a wrapper and ledger timestamp. Neutrino never patches customer workloads.

`POST /v1/actuate` unconditionally returns HTTP `403 Forbidden`. Actuate is not a user feature. CPU frequencies, scaling governors, cgroups, and sysfs are strictly untouched.

---

## Workload Classes & The -5% Holdout Gate

Neutrino defines an exact schema of 11 workload classes. All 11 classes must be specified in `/etc/neutrino/recipe.json`; missing class slugs are **rejected** (they do not default to `meter_only`):

| Class | Lever (when apply is on) | Notes |
| --- | --- | --- |
| `compile` | materialize / warm artifacts | Materialize lever |
| `infer-cpu` | materialize | Materialize lever |
| `batch` | occupancy (pack-then-idle) | HPC/render analog |
| `oltp` | group-commit | Commit coalescing analog |
| `mq` | batched fsync | Persist-batch analog, not a message broker |
| `hpc` | occupancy | Scheduler wrap (`SLURM_*` / LSF env) |
| `render` | occupancy | Scheduler wrap (`SLURM_*` / LSF env) |
| `etl` | residency (stream vs retain) | Meter-only by default; cannot save unless explicitly configured and clearing threshold |
| `idle` | none | Mandatory meter-only floor — never a savings class |
| `heartbeat` | none | Mandatory meter-only floor — never a savings class |
| `unspecified` | none | Mandatory meter-only fallback — never a savings class |

### Empirical Proof on Your Host (No Quoted Lab SLAs)
Neutrino does **not** quote synthetic lab percentages as an operational SLA. Savings are proved empirically on *your* hardware:
- Every 20 job wraps, Neutrino samples a holdout run (un-optimized control run).
- A class is labeled `saving` only when $n \ge 5$ live runs and $n \ge 5$ holdout runs have completed ($t \ge 1\text{s}$), and live mean energy is at least 5% lower than holdout mean energy ($\Delta E \le -5\%$).
- If $\Delta E > -5\%$, the class is labeled `off (retry next pass)` for that window to avoid overhead, and re-evaluates automatically on the subsequent pass.

### Report Labels
```text
watching              apply on; holdout window not yet complete
saving                live window ≤ −5% package energy vs holdout
off (retry next pass) this window failed threshold; re-armed next pass
off                   not on the recipe / not approved
meter_only            passive observation only (idle, heartbeat, unspecified, etl)
```

`neutrino report` is the operator page. `sample-report.html` is the buyer layout shipped with this drop.

---

## Protected Targets: Deny List & Neutrino Service

To guarantee host safety, the capture helper and recipe validator strictly refuse to wrap:
- `oracle`
- `postgres`
- `mariadbd`
- `mysqld`
- `java`
- `sshd`
- `systemd`

Wrapping `neutrino.service` is unconditionally rejected.

---

## Verify, Stage, Then Install

Do not run `dpkg -i` from world-writable directories. Sudoers policies and installer integrity require staging in a root-protected path:

```bash
# 1. Verify package against signed manifest
neutrino pkg verify neutrino_0.5.10_amd64.deb SHA256SUMS SHA256SUMS.sig

# 2. Stage to root-only path (mode 0700)
sudo install -d -m 0700 /var/lib/neutrino/staging
sudo cp neutrino_0.5.10_amd64.deb /var/lib/neutrino/staging/neutrino-install.deb

# 3. Install strictly from the staged path
sudo dpkg -i /var/lib/neutrino/staging/neutrino-install.deb
sudo neutrino-setup --validate
```

`pkg verify` checks the `.deb` hash against `SHA256SUMS` and validates the Post-Quantum DSA-16 Level 3 operator signature on that manifest. The operator public key ships in the package. There is no private signing key in this repository.

Expected status:
- `neutrino.service` active
- `ss -ltn` shows loopback `127.0.0.1:8741` only
- `GET /health` → `status: ok`, `version: 0.5.10`
- `POST /v1/actuate` → `403 Forbidden`

Without a license paper, health reports Observe mode (`apply: false`).

---

## Uninstall

```bash
sudo neutrino-setup --uninstall
```

Touched files restore from the pre-install backup. Ledger and node identity under `/var/lib/neutrino` are kept unless explicitly purged. Secrets (`token`, `lease.sig`, `node.sk`) are never restored back from backup tar archives.

---

## Identity & Licensing

Neutrino does not mint operator signatures on customer hosts. The license desk signs papers offline. The host only **verifies**.

1. **Print Identity**:
   ```bash
   sudo neutrino license enroll-print
   ```
   Transmit **only** `sku=NEUTRINO`, `node_id`, and `pk_sha256`. Never send `node.sk`.

2. **Install Offline Papers**:
   Place detached DSA-16 Level 3 signed papers (`lease.json` + `lease.sig`, and `recipe.json` + `recipe.sig` for capture) under `/etc/neutrino/`:
   ```bash
   sudo install -m 0640 -o root -g neutrino lease.json /etc/neutrino/lease.json
   sudo install -m 0640 -o root -g neutrino lease.sig  /etc/neutrino/lease.sig
   ```
   Customer hosts operate air-gapped and never make external network connections.

3. **Effective License**:
   | Paper | `sku` | `apply` |
   | --- | --- | --- |
   | none / expired / bad sig | none | false (observe only) |
   | valid Neutrino paper, `claims.apply` false | NEUTRINO | false |
   | valid Neutrino paper, `claims.apply` true | NEUTRINO | true |

Papers use canonical JSON + raw DSA-16 Level 3 signatures (1043 bytes). Ed25519 is retired. Leftover `license.sig` files are ignored.

---

## Daily Use & Autopilot

### 1. Wrap Work for Metering
```bash
sudo neutrino-run --class compile --id build-1 -- make -j
sudo neutrino-run --class batch --id nightly -- /usr/local/bin/nightly-job
SLURM_JOB_ID=123 SLURM_JOB_CPUS=8 sudo neutrino-srun --class hpc -- ./solver
```

### 2. Autopilot On (Whole Recipe)
Autopilot is the product on-switch. Do not approve classes one by one unless debugging a single lever.

```bash
sudo neutrino apply preview          # Read-only; does NOT bind nonces
sudo neutrino apply autopilot on     # Binds nonce once; arms eligible recipe classes
sudo neutrino apply status
sudo neutrino report
```

- `apply preview` is read-only and leaves `seen_nonces.jsonl` untouched.
- `autopilot on` binds the nonce once via Post-Quantum DSA-16 Level 3.
- To stop governance and restore original configuration byte-identically:
```bash
sudo neutrino apply autopilot off
```

### 3. Debug Single Lever
```bash
sudo neutrino apply preview --class batch
sudo neutrino apply approve --class batch
sudo neutrino apply history --class batch
```

### 4. Telemetry Export
```bash
neutrino export --format jsonl
neutrino export --format csv
```

---

## Security Envelope

- **Bind**: Loopback `127.0.0.1:8741` only. No `0.0.0.0`, no external listeners.
- **Auth**: Bearer token for operator API routes. Missing token fails closed.
- **Actuate**: Always `403 Forbidden`.
- **Sudoers Path**: `/var/lib/neutrino/staging/neutrino-install.deb` only.
- **Crypto on Host**: `libkltu_dsa16_l3.so` — verify + node keygen only. No `dsa16_sign`, no KEM, no AEAD in package.

---

## What This Drop Is NOT

- Not source. Product development remains in the private repository.
- Not a GPU / NVML build. CPU RAPL package rail only.
- Not an SLA guarantee of figures in `sample-report.html`. Live savings are verified on the buyer node via `neutrino report`.

Version **0.5.10**. Copyright (c) 2026 Xylonix. All rights reserved.
