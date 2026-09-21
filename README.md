# Neutrino Energy Governor 0.6.5.0

**Use Autopilot.** Install, license, approve. Then stop.

This repository ships **packages and docs only**, not engine source.

Autopilot injects an allowlist when the class is approved and the license has `claims.apply`:

| Class | Lever | What it wraps | Quote |
| compile | occupancy | `make` / `gmake` / `ninja` (`-j` except `-j1`) | silicon occupancy |
| batch | occupancy | `gzip` → `pigz` if present; else `xz -T0` | −72% package E vs gzip on 400 MB (n=5) when pigz is installed |
| mq | batch_fsync | `neutrino-mq-append` | our append tool, not Kafka |
| oltp | group-commit | `neutrino-oltp-load` | our loader, not mysqld; direction-only on NVMe journal |
| etl | residency | `neutrino-etl` stream vs slurp | joules, not warehouse |

Also stamp (no silent rewrite of engines): `infer-cpu`, `hpc`, `render` if approved.

Not Autopilot savings: `idle`, `heartbeat`. GPU is not in this CPU release.

`POST /v1/actuate` is always 403. Neutrino does not write governors or sysfs.

There is no 19+1 unwrap cycle.

---

## 1. Install (once)

Download `neutrino_0.6.5.0_amd64.deb`, `SHA256SUMS`, and `SHA256SUMS.sig`.

```bash
neutrino pkg verify neutrino_0.6.5.0_amd64.deb SHA256SUMS SHA256SUMS.sig
sudo install -d -m 0700 /var/lib/neutrino/staging
sudo cp neutrino_0.6.5.0_amd64.deb /var/lib/neutrino/staging/neutrino-install.deb
sudo dpkg -i /var/lib/neutrino/staging/neutrino-install.deb
sudo neutrino-setup --validate
curl -sS http://127.0.0.1:8741/health
ss -ltn '( sport = :8741 )'    # 127.0.0.1:8741 only
```

Batch occupancy needs distro `pigz` (`apt install pigz`). Without it, gzip is unchanged.

## 2. License paper (once)

Send **only** `node_id` and `pk_sha256` to Xylonix. Never send private keys. Do not call a public license server.

```bash
sudo neutrino license enroll-print
sudo install -m 0640 -o root -g neutrino lease.json /etc/neutrino/lease.json
sudo install -m 0640 -o root -g neutrino lease.sig  /etc/neutrino/lease.sig
sudo neutrino license show
```

Required: `sku=NEUTRINO`, `source=desk`, `apply=true`.

## 3. Approve Autopilot (once)

```bash
for c in compile batch oltp mq etl infer-cpu hpc render; do
  sudo neutrino apply preview --class "$c"
  sudo neutrino apply approve --class "$c"
done
```

**Setup is finished.**

---

## Security

- `127.0.0.1:8741` only. Never `0.0.0.0`.
- Package contains verify keys only. Host private keys stay on the host.
- Neutrino will not wrap: `mysqld`, `postgres`, `mariadbd`, `oracle`, `sshd`, `systemd`, Kafka/RabbitMQ servers.

| Paper | Mode |
| --- | --- |
| None / expired lease | Observe (meter only) |
| `sku=NEUTRINO`, `apply=false` | Licensed observe |
| `sku=NEUTRINO`, `apply=true` + approve | Autopilot |

Copyright © 2026 Xylonix. All rights reserved.
