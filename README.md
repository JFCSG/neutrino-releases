# Neutrino 0.5.8 installer

Signed CPU drop. Not the product source tree.

| File | Role |
| --- | --- |
| `neutrino_0.5.8_amd64.deb` | Linux amd64 package |
| `SHA256SUMS` | SHA-256 of the `.deb` |
| `SHA256SUMS.sig` | DSA-16 Level 3 signature of `SHA256SUMS` |
| `sample-report.html` | Buyer energy-report preview (existing layout) |

## Verify, then install

```bash
neutrino pkg verify neutrino_0.5.8_amd64.deb SHA256SUMS SHA256SUMS.sig
sudo install -d -m 0700 /var/lib/neutrino/staging
sudo cp neutrino_0.5.8_amd64.deb /var/lib/neutrino/staging/neutrino-install.deb
sudo dpkg -i /var/lib/neutrino/staging/neutrino-install.deb
```

After install: `curl -sS http://127.0.0.1:8741/health` then `neutrino report`.
API is `127.0.0.1:8741` only. `POST /v1/actuate` is HTTP 403.

Open `sample-report.html` for the report layout.
