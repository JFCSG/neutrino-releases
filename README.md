# Neutrino trial packages

Signed install pack only. Observe/metering works without a license.
Apply requires papers issued by Xylonix — this repository cannot mint them.

## Files

- `neutrino_*_amd64.deb` — package
- `SHA256SUMS` — checksums
- `SHA256SUMS.sig` — operator signature (DSA-16 Level 3)

## Install

```bash
neutrino pkg verify neutrino_0.5.4_amd64.deb SHA256SUMS SHA256SUMS.sig
sudo install -d -m 0700 /var/lib/neutrino/staging
sudo cp neutrino_0.5.4_amd64.deb /var/lib/neutrino/staging/neutrino-install.deb
sudo dpkg -i /var/lib/neutrino/staging/neutrino-install.deb
```

Do not install from `/var/tmp`. Do not skip verify.

## License

Package download ≠ license. Run `neutrino license enroll-print` and
send that output through the Xylonix site. You will receive `lease`
and `recipe` files separately.
