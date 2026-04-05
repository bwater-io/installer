# Breakwater Installer

This repository exists to publish customer installer artifacts as GitHub Release
assets. It intentionally does not contain product source code.

## Customer install command

```bash
curl -fsSL https://install.bwater.io | sudo bash
```

`install.bwater.io` should redirect or proxy to the latest published
`bootstrap.sh` release asset from this repository.

## Release assets

Each release publishes:

- `bootstrap.sh`
- `install.sh`
- `breakwater`
- `docker-compose.customer.yml`
- `SHA256SUMS`
- `manifest.json`
- `breakwater-customer-installer-<version>.tar.gz`

## Stable GitHub asset URLs

GitHub exposes latest-release asset URLs in this form:

```text
https://github.com/bwater-io/installer/releases/latest/download/bootstrap.sh
```

That URL is the correct target for `install.bwater.io`.
