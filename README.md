# Neutrino Energy Governor 0.6.3

**Use Autopilot.** Install, license, approve. Then stop. Neutrino applies policy to all savings classes with no further operator action.

This repository ships **packages and docs only**, not engine source.

Autopilot classes: `compile`, `infer-cpu`, `batch`, `oltp`, `mq`, `hpc`, `render`.

Not Autopilot: `etl`, `idle`, `heartbeat`, `unspecified`. GPU is not in this CPU release.

---

## 1. Install (once)

Download `neutrino_0.6.3_amd64.deb`, `SHA256SUMS`, and `SHA256SUMS.sig` from this release.

```bash
neutrino pkg verify neutrino_0.6.3_amd64.deb SHA256SUMS SHA256SUMS.sig
sudo install -d -m 0700 /var/lib/neutrino/staging
sudo cp neutrino_0.6.3_amd64.deb /var/lib/neutrino/staging/neutrino-install.deb
sudo dpkg -i /var/lib/neutrino/staging/neutrino-install.deb
sudo neutrino-setup --validate
curl -sS http://127.0.0.1:8741/health
ss -ltn '( sport = :8741 )'    # 127.0.0.1:8741 only
```

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
for c in compile infer-cpu batch oltp mq hpc render; do
  sudo neutrino apply preview --class "$c"
  sudo neutrino apply approve --class "$c"
done
```

**Setup is finished. Do not run anything else.**

---

## Autopilot (no operator)

- 19 optimized runs + 1 baseline per class, chosen by Neutrino
- Coverage held iff \(\Delta E \le -5\%\) vs baseline
- `recipe.json` never expires
- Lease is the only clock. Expiry or `claims.apply=false` stops Autopilot; metering continues
- `POST /v1/actuate` is always 403. Neutrino does not write governors or sysfs
- Human again only for a new paper after expiry or revoke

---

## Security

- `127.0.0.1:8741` only. Never `0.0.0.0`.
- Package contains verify keys only. Host private keys stay on the host.
- Neutrino will not govern: `oracle`, `postgres`, `mariadbd`, `mysqld`, `sshd`, `systemd`.

| Paper | Mode |
| --- | --- |
| None / expired lease | Observe (meter only) |
| `sku=NEUTRINO`, `apply=false` | Licensed observe |
| `sku=NEUTRINO`, `apply=true` + approve | **Autopilot — zero further operator action** |

Copyright © 2026 Xylonix. All rights reserved.
