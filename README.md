# Neutrino Energy Governor 0.6.8.35

Enterprise userspace runtime governor for Linux environments. Delivers automated workload acceleration and energy optimization across Intel/AMD CPUs and NVIDIA GPUs without kernel modifications or semantic changes.

This repository ships **signed packages and documentation only**, not engine source, and is not the licensing desk.

---

## Measured Performance & Silicon Verification

Workload acceleration directly cuts wall-clock cycles, allowing processors and accelerators to return rapidly to low-power idle states.

### 24-Core Workstation CPU Levers

Measured via direct on-die RAPL package accounting (`uj_delta`) across 24 logical cores, compared against published 12-core reference baselines:

| Class | Lever | Workload Description | 24-Core \(\Delta E\) | 12-Core Reference \(\Delta E\) | Scaling Status |
|---|---|---|---|---|---|
| **batch** | pack-then-idle (occupancy) | Parallel reduction across logical cores | **−79.0%** (\(14.5\,\mathrm{s} \to 1.7\,\mathrm{s}\)) | −57.9% | MOVE (accelerated race-to-idle) |
| **mq** | batched fsync (\(K=50\)) | Append persistence with batched fsync | **−96.5%** (\(37.7\,\mathrm{s} \to 1.3\,\mathrm{s}\)) | −80.4% | MOVE (NVMe transaction grouping) |
| **oltp** | connection pooling | SQLite insert autocommit session reuse | **−21.7%** (\(6.3\,\mathrm{s} \to 4.9\,\mathrm{s}\)) | −29.7% (−5.3% quiet) | HOLD (session-bound overhead) |
| **etl** | memory residency | Wide-record streaming vs memory slurp | **−29.6%** (\(5.7\,\mathrm{s} \to 4.1\,\mathrm{s}\)) | −2.1% (narrow records) | MOVE (wide-record residency win) |

### GPU (0.6.8.35)

- **Classes:** `infer-gpu` (prefix reuse), `dump_guard` (file-agent rewrite only).
- **Meter:** NVML when the card is readable; RAPL package always.
- **Integration:** Wrap the completion worker with `neutrino-run --class infer-gpu`.
  Do not wrap `llama-server`. Neutrino is not on `:8080`.
- **Scope:** `dump_guard` is for file dumps. Do not use it on binder JSON.
- **Hardware baseline:** Quoted GPU bands are on one 24 GB card + local 27B; not an H100 farm certificate.

---

## 1. Installation

Download `neutrino_0.6.8.35_amd64.deb`, `SHA256SUMS`, and `SHA256SUMS.sig` from this release.

Package SHA-256:
```
5a4e0c61f81cffd51a0b0714b3c525ce49c583f2db3ba7a01891a0a0e2566f54  neutrino_0.6.8.35_amd64.deb
```

Always verify package integrity and operator signature before staging:

```bash
# Verify DSA-16 Level 3 operator detached signature
neutrino pkg verify neutrino_0.6.8.35_amd64.deb SHA256SUMS SHA256SUMS.sig

# Stage into protected root directory
sudo install -d -m 0700 /var/lib/neutrino/staging
sudo cp neutrino_0.6.8.35_amd64.deb /var/lib/neutrino/staging/neutrino-install.deb

# Install package
sudo dpkg -i /var/lib/neutrino/staging/neutrino-install.deb

# Validate service installation
sudo neutrino-setup --validate
curl -sS http://127.0.0.1:8741/health
ss -ltn '( sport = :8741 )'    # 127.0.0.1:8741 loopback only
```

---

## 2. Licensing Paper & Modes

Neutrino operates in **observe mode** for free (real-time RAPL/NVML energy accounting and workload telemetry). Applying runtime optimization pathways requires an active license paper with `sku=NEUTRINO` and `claims.apply=true`.

Send **only** `node_id` and `pk_sha256` to Xylonix to request license paper. Never send private keys. No public `:8750` port is exposed or contacted over the WAN.

```bash
# Print enrollment identity for licensing desk
sudo neutrino license enroll-print

# Install signed paper issued by Xylonix
sudo install -m 0640 -o root -g neutrino lease.json /etc/neutrino/lease.json
sudo install -m 0640 -o root -g neutrino lease.sig  /etc/neutrino/lease.sig
sudo neutrino license show
```

### Autopilot Operation

On licensed hosts with `auto_apply` enabled, Neutrino Autopilot automatically plants pre-approved recipes for all standard classes (`batch`, `compile`, `compile_handoff`, `dump_guard`, `hpc`, `infer-cpu`, `mq`, `oltp`, `render`) on service start with zero manual approval commands required.

---

## 3. Security & Operational Guardrails

- **Loopback Enforcement:** API listens strictly on `127.0.0.1:8741`. It never binds `0.0.0.0` or public interfaces.
- **Actuation Refusal:** `POST /v1/actuate` always returns `403 Forbidden`. Neutrino never modifies kernel governors, clock frequencies, execution precision, or sysfs knobs.
- **Protected Processes:** Neutrino will never wrap or intercept: `mysqld`, `postgres`, `mariadbd`, `oracle`, `sshd`, `systemd`, or message broker daemons.
- **Key Isolation:** Distribution packages contain verification public keys only. Host private keys never leave the host.

| License Status | Operational Mode | Policy Execution |
|---|---|---|
| None / Unlicensed | Observe | Real-time hardware telemetry and job logging |
| `sku=NEUTRINO`, `apply=false` | Licensed Observe | Accounted telemetry with licensed audit trails |
| `sku=NEUTRINO`, `apply=true` | Autopilot | Automated runtime acceleration across approved classes |

Copyright © 2026 Xylonix. All rights reserved.
