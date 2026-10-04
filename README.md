# Neutrino

**Compute-Energy Optimization Application for Enterprise Linux Server Fleets**

Version **0.6.8.40** · Linux amd64 · signed trial packages

This repository ships **signed packages and documentation only**. It is not engine source and is not the licensing desk.

Neutrino is an enterprise-grade compute-energy optimization application for Linux server environments. Operating strictly in user space, it optimizes active compute paths so workloads finish sooner. Because the server completes its work in less time while remaining inside its normal power operating envelope, total kilowatt-hours consumed per job drop. Instantaneous package watts may rise during the shorter burst; energy per job still falls.

---

## The infrastructure dilemma

Increasing energy-intense AI usage in data centers is straining CapEx, OpEx, and facility limits for both corporate and government operators, especially against rising energy costs. Workload volume — high-throughput analytics, continuous integration, and AI serving — continues to expand. The rooms that house those workloads do not.

- **Rack power density ceilings.** Electrical supply per rack stops additional servers even when floor space remains.
- **Escalating hardware CapEx.** Scale-out means continuous procurement, switching, and depreciation.
- **Compounding power and cooling OpEx.** High-utilization fleets drive cost at the socket and again at the chiller and CRAH.

The usual answers are both compromises. Adding more physical servers absorbs demand by buying more boxes. Frequency throttling and power caps cut peak watts but stretch runtimes, inflate queues, and threaten SLAs.

### The Neutrino value engine

| Engine step | Business result |
| --- | --- |
| Optimized execution paths | Shorter active compute window (up to 3.5×+ faster on intercepted work) |
| Shorter execution window | Proportional drop in energy per job (−38% to −90%+ on intercepted work) |
| Higher server throughput | Server purchase deferral (CapEx, growth case) |
| Reduced thermal output | Cascading air-conditioning and chiller savings (OpEx) |

---

## Why accelerated computing saves power

A common assumption is that saving electricity means slowing processors down. In production servers the opposite is true: excess execution duration is the largest driver of electrical waste.

Modern sockets draw power in two regimes — dynamic switching power while gates fire, and static base power just to keep the box, memory buses, and board alive. When a job stalls, contends, or serializes, the server stays in its active high-power state and burns baseline watts while it waits.

Neutrino shortens the active window. Cores finish the payload and return to deep idle (C-states). The cut in duration outweighs any brief rise in intensity, so joules per job fall.

### Verified production telemetry (instrumented Linux node)

Energy from native package counters (Intel RAPL), checksum-matched across runs.

| Execution metric | Standard Linux baseline | Neutrino optimized | Measured impact |
| --- | --- | --- | --- |
| Average task duration | 5.64 s | 1.58 s | **3.57× faster** |
| Average instantaneous power | 22.6 W | 49.5 W | High-efficiency burst |
| Total energy consumed | 127.6 J | 78.3 J | **38.7% electricity cut** |
| Data output integrity | Baseline checksum | 100% bit-identical | Zero output divergence |

The 22.6 W / 49.5 W figures are socket package draw during the active window. They are not the 350 W whole-server planning figure used in fleet models.

Do not treat a single-job 3.57× / −38.7% burst as an estate average. Slice gains stay at the job. Node and fleet figures use multi-component Amdahl.

---

## Strategic performance (anonymized)

Intercepted enterprise pipelines, not recipe names. Job-level figures are not fleet averages.

| Workload category | Common business applications | Throughput gain | Direct energy cut |
| --- | --- | --- | --- |
| A · Enterprise batch | Risk analysis, ETL pipelines, billing runs | 2.5× to 3.6× | −38.7% to −57.9% |
| B · Message persistence | Event streaming, persistent queues | 5.0× | −80.4% |
| C · Transactional commit | High-frequency database commit paths | 17.5× | −94.1% |
| D · Continuous integration | Software build systems, test suites | 140×+ | −99%+ |
| E · NVIDIA serving | Multi-turn inference and agent file work on NVML-metered Tensor Core hosts | Estate ~1.27× on the modeled AI mix | Node energy ~23% on that mix |

CI/CD −99% is workload-specific upside, not an estate average. GPU prefix reuse and agent-dump reduction are separate levers; do not stack their percentages.

### Three-tier financial dividend

1. **Direct server OpEx** — less electrical energy per job at the processor socket.
2. **Facility cooling OpEx** — lower thermal output reduces chiller and CRAH load.
3. **Infrastructure CapEx** — Amdahl estate speedup expands current capacity and can defer purchases in the growth case (1.46× CPU banking mix; 1.27× NVIDIA AI mix).

---

## Fleet sizing (planning cases)

The two estates below are derived separately. Figures from one case are not inputs to the other. CapEx (absorb growth on the same boxes) and OpEx (hold today’s job count fixed) are different counterfactuals and are never added into one “first-year value.”

Field results vary with socket generation, duty cycle, mix, tariff, and PUE. Validate on your own fleet with a non-actuating Passive Audit.

**Case 1 — Core banking (CPU-only, 10,000 servers)**  
S_overall ≈ 1.46× · ΔE_node ≈ 29.7% · P_base 0.350 kW · PUE 1.50 · $0.15/kWh · 75% duty · $10,000/server.

| Metric | Reference value |
| --- | --- |
| Virtual capacity | 14,600 server equivalents |
| Avoided purchases (growth) | 4,600 servers · **$46M CapEx deferred** |
| Annual utility OpEx | **$1.535M / year** ($1.023M IT + $0.512M cooling) |

**Case 2 — Enterprise AI serving (CPU + NVIDIA GPU, 10,000 nodes)**  
S_overall ≈ 1.27× · blended ΔE ≈ 22.8% · 6.5 kW IT/node · PUE 1.40 · $0.15/kWh · 75% duty · $250,000/node.

| Metric | Reference value |
| --- | --- |
| Avoided purchases (growth) | 2,700 nodes · **~$675M CapEx deferred** |
| Annual utility OpEx | **~$20.4M / year** |

---

## Certified scope

- **CPU:** Intel Xeon, Intel Core, and AMD EPYC multi-core x86_64. Package energy via RAPL MSRs / Linux sysfs.
- **GPU:** NVIDIA Tensor Core hosts metered through NVML. AMD ROCm is not certified in this release.
- **Architecture:** Non-invasive user-space application. No kernel modules, no sysfs actuation, no binary rewrite.
- **Target estate:** Self-managed Linux servers (on-prem or colocation) on capacity-constrained queues.

---

## Security and governance

- **User space only.** No proprietary kernel modules, no kernel config edits, no syscall-table changes.
- **No business-data exposure.** Neutrino does not inspect, parse, store, or transmit payloads, source, database records, or credentials.
- **Fail-closed.** Missing or unverified license paper returns Neutrino to passive, read-only metering. Production jobs keep running.
- **Loopback API.** `127.0.0.1:8741` only. Never bind the control plane on a public address.
- **Actuation refused.** `POST /v1/actuate` is always **403**. Neutrino does not set governors, clocks, cgroups, or sysfs knobs.
- **Verify-only crypto on the node.** DSA-16 Level 3. The operator secret never ships in this tree. Host private keys never leave the host.

---

## Install, license, Autopilot

After these three steps there is **no class wrap, no job wrap, and no daily operator action**. Autopilot is the product path. Zero human intervention is required after the apply lease is in place.

### 1. Install the signed package

Download `neutrino_0.6.8.40_amd64.deb`, `SHA256SUMS`, and `SHA256SUMS.sig` from this repository.

Package SHA-256:

```
7949b86235d33059a5712fe8eccac0b97677d5ac4dc24bfffbd611a03cc6f5d4  neutrino_0.6.8.40_amd64.deb
```

```bash
neutrino pkg verify neutrino_0.6.8.40_amd64.deb SHA256SUMS SHA256SUMS.sig

sudo install -d -m 0700 /var/lib/neutrino/staging
sudo cp neutrino_0.6.8.40_amd64.deb /var/lib/neutrino/staging/neutrino-install.deb
sudo dpkg -i /var/lib/neutrino/staging/neutrino-install.deb

sudo neutrino-setup --validate
curl -sS http://127.0.0.1:8741/health
```

The API must remain on `127.0.0.1:8741`.

### 2. Enroll and install the apply lease

Observe metering runs without paper. Apply and Autopilot require a desk-signed lease: `sku=NEUTRINO` and `claims.apply=true`.

Send **only** `sku`, `node_id`, and `pk_sha256`. Never send `node.sk` or any private key. No public license port is contacted over the WAN.

```bash
sudo neutrino license enroll-print

sudo install -m 0640 -o root -g neutrino lease.json /etc/neutrino/lease.json
sudo install -m 0640 -o root -g neutrino lease.sig  /etc/neutrino/lease.sig
sudo neutrino license show
```

Success: `/health` shows `"sku":"NEUTRINO"` and `"apply":true`.

### 3. Leave Autopilot on

On a licensed apply host, Autopilot attaches approved optimization on service start and keeps it on. You do not wrap jobs. You do not name classes. You do not run on/off after work. A service start is enough.

| License paper | Mode | What happens |
| --- | --- | --- |
| None | Observe | Hardware telemetry only. Jobs are untouched. |
| `sku=NEUTRINO`, `apply=false` | Licensed observe | Accounted telemetry. Still no apply. |
| `sku=NEUTRINO`, `apply=true` | **Autopilot** | Licensed apply runs without further human steps. |

Optional check, not an operating step:

```bash
curl -sS http://127.0.0.1:8741/health
neutrino report
```

---

## Commercial path

1. **Passive Audit** (14 days, no actuation) on a pilot of 10–50 Linux servers.
2. **FinOps review** of hardware-metered energy and latency logs.
3. **Targeted activation** under Autopilot on approved queues.

Copyright © 2026 Xylonix Pte. Ltd. All rights reserved.  
Document field alignment: WP-NEUTRINO-PROD-2026-V7 (reduced public edition).
