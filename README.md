# Neutrino 0.5.8

Signed **CPU installer drop**. This repository is not the product source tree.

Neutrino is an energy governor for Linux servers. It meters package joules
(Intel RAPL, `method: uj_delta`), wraps named jobs, and — when licensed —
applies a **read-only policy wrap** around work the operator already runs.
It does not rewrite customer binaries, does not change CPU governors or
sysfs, and does not expose an actuation API.

| File | Role |
| --- | --- |
| `neutrino_0.5.8_amd64.deb` | Linux amd64 package |
| `SHA256SUMS` | SHA-256 of the `.deb` |
| `SHA256SUMS.sig` | DSA-16 Level 3 signature of `SHA256SUMS` |
| `sample-report.html` | Buyer energy-report layout (existing preview) |

Open `sample-report.html` for the report page layout. Numbers there are a
preview of campaign-style class stats, not a live host dump you should quote
as a contract.

---

## What it does

Two modes on one daemon:

**Observe (always).** After install the service listens on `127.0.0.1:8741`,
records RAPL package energy, and can wrap jobs with `neutrino-run` /
`neutrino-srun` / `neutrino-bsub`. No desk paper is required. This is the
try-before-buy meter.

**Apply (licensed).** A desk-signed paper (`sku=NEUTRINO`, unexpired,
fingerprint-bound, unseen nonce) with `claims.apply=true` lets the operator
turn **autopilot** on. Every class on the host recipe goes `watching`, then
independently `saving` or `off (retry next pass)`. Apply is a wrap and a
ledger stamp. Neutrino does not patch the workload.

`POST /v1/actuate` stays HTTP 403. Governors, frequencies, cgroups, and
sysfs are out of scope.

### Workload classes

| Class | Lever (when apply is on) | Notes |
| --- | --- | --- |
| `compile` | materialize / warm artifacts | Large matched-work ΔE in campaign |
| `infer-cpu` | materialize | Same |
| `batch` | occupancy (pack-then-idle) | HPC/render cousin |
| `oltp` | group-commit | Commit coalescing analog |
| `mq` | batched fsync | Persist analog, not a broker |
| `hpc` / `render` | occupancy | Scheduler wrap (`SLURM_*` / LSF env) |
| `etl` | residency (stream vs retain) | Weak ΔE — meter, do not sell as a win |
| `idle` / `heartbeat` | none | Floor / SLA only — never a savings class |

Savings are **same class, matched work, package joules**. No idle
subtraction. Sub-second jobs stay in the ledger but are excluded from mean
power. Autopilot needs a holdout window before it may label a class
`saving`.

### Report labels

```
watching              apply on; holdout not yet full
saving                live window ≤ −5% package E vs holdout
off (retry next pass) this window failed; re-armed next pass
off                   not on the recipe / not approved
```

`neutrino report` is the operator page. `sample-report.html` is the buyer
layout shipped with this drop.

---

## Verify, then install

Do not `dpkg -i` from a world-writable directory.

```bash
neutrino pkg verify neutrino_0.5.8_amd64.deb SHA256SUMS SHA256SUMS.sig
sudo install -d -m 0700 /var/lib/neutrino/staging
sudo cp neutrino_0.5.8_amd64.deb /var/lib/neutrino/staging/neutrino-install.deb
sudo dpkg -i /var/lib/neutrino/staging/neutrino-install.deb
sudo neutrino-setup --validate
```

`pkg verify` checks the `.deb` hash against `SHA256SUMS` and the DSA-16
Level 3 operator signature on that manifest. The operator public key ships
in the package. There is no signing key in this repository.

Expect:

- `neutrino.service` active
- `ss -ltn` shows `127.0.0.1:8741` only
- `GET /health` → `status=ok`
- `POST /v1/actuate` → `403`

Without a paper, health reports observe / no apply. That is correct.

### Uninstall

```bash
sudo neutrino-setup --uninstall
```

Touched install files restore from the pre-install backup. Ledger and node
identity under `/var/lib/neutrino` are kept unless you delete them. Secrets
(`token`, `lease.sig`, `node.sk`) are not extracted back from the backup
tar.

---

## Identity and license

Neutrino does not mint operator signatures. The license desk signs papers.
This host only **verifies**.

1. After first boot:

   ```bash
   sudo neutrino license enroll-print
   ```

   Send **only** `sku=NEUTRINO`, `node_id`, and `pk_sha256`. Never send
   `node.sk`.

2. Staff place `lease.json` + `lease.sig` (or voucher pair) under
   `/etc/neutrino/`. Hosts do not call the desk over the internet.

3. Effective license:

   | Paper | `sku` | `apply` |
   | --- | --- | --- |
   | none / expired / bad sig | none | false (observe only) |
   | valid Neutrino paper, `claims.apply` false | NEUTRINO | false |
   | valid Neutrino paper, `claims.apply` true | NEUTRINO | true |

Papers are canonical JSON + raw DSA-16 Level 3 signature (1043 bytes).
Ed25519 is retired. A leftover `license.sig` on disk is ignored.

---

## Daily use

### 1. Wrap work so it can be metered

```bash
sudo neutrino-run --class compile --id build-1 -- make -j
sudo neutrino-run --class batch --id nightly -- /usr/local/bin/nightly-job
SLURM_JOB_ID=123 SLURM_JOB_CPUS=8 sudo neutrino-srun --class hpc -- ./solver
```

Unwrapped processes are not in the job ledger. Observe works without a paper.

### 2. Turn autopilot on (all recipe classes)

Do **not** approve classes one by one unless you are debugging a single lever.

Licensed node + recipe (classes the desk or operator listed for this host):

```bash
sudo neutrino apply preview          # whole recipe, not one class
sudo neutrino apply autopilot on     # arms every class on the recipe
sudo neutrino apply status
sudo neutrino report
```

`autopilot on` is the product switch. Every class on the recipe goes
`watching`. After the holdout window, each class independently becomes
`saving` or `off (retry next pass)`. Neutrino re-arms `off` on the next
pass. It does not flip CPU governors.

`etl` may sit on the recipe as meter-only. It will not be sold as
`saving` unless live ΔE clears the threshold.

To stop:

```bash
sudo neutrino apply autopilot off
```

### 3. Optional: one class only

Use this only to inspect or pin a single lever:

```bash
sudo neutrino apply preview --class batch
sudo neutrino apply approve --class batch
sudo neutrino apply history --class batch
```

That does **not** start the other recipe classes.

### 4. Export

```bash
neutrino export --format jsonl
neutrino export --format csv
```

---

## Security envelope

- Bind: `127.0.0.1:8741` only. No Funnel, no `0.0.0.0`.
- Auth: Bearer token for operator API routes. Missing token fails closed.
- Actuate: always 403.
- Package install path in sudoers: `/var/lib/neutrino/staging/neutrino-install.deb` only.
- Restore: backup ids must match the installer pattern; extract stays on
  the touch list.
- Crypto on host: `libkltu_dsa16_l3.so` — verify + node keygen only.
  No `dsa16_sign`, no KEM, no AEAD in this package.

---

## What this drop is not

- Not source. Product development stays in the private lab tree.
- Not a GPU / NVML build. CPU RAPL package rail only.
- Not a cluster admin plugin. Scheduler wrap reads user-space env only.
- Not a guarantee of the ΔE figures in `sample-report.html`. Those are
  campaign-class illustrations. Quote live `neutrino report` on the buyer
  node after holdout.

Version **0.5.8**. Copyright (c) 2026 Xylonix. All rights reserved.
