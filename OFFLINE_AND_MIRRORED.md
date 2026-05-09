# Offline / Air-Gapped & Hosted-Mirror Installs

ROADMAP item C.3 — operational guidance, not a code feature.

Some customer environments cannot reach `ghcr.io/mitvaris` directly:

- Air-gapped networks (defense, regulated industries)
- Networks with strict allowlists where adding ghcr.io is procurement-heavy
- Customers without GitHub PAT management capability

There are two paths.

## Path A — fully offline (no Mitvaris involvement at runtime)

Documented in [DEPLOYMENT_GUIDE.md](DEPLOYMENT_GUIDE.md) "Offline / air-gapped install".

Workflow:
1. Mitvaris pulls the four images on a connected machine, runs `docker save` to produce `invenzo-images-<version>.tar.gz`
2. Customer copies the tarball to the air-gapped server, runs `docker load`
3. `sudo bash install.sh --offline` skips the registry-auth + pull steps; customer runs the locally-loaded images

`update.sh` follows the same `--offline` path on subsequent versions.

## Path B — Mitvaris-hosted mirror (managed offering)

For customers who can reach the Internet but want a single Mitvaris-managed pull endpoint instead of GHCR-with-PAT, Mitvaris operates a mirror at `mirror.invenzo.mitvaris.com`. Customer-side workflow:

1. Customer's procurement signs the mirror SLA + receives credentials (HTTP basic, rotated quarterly)
2. `INVENZO_REGISTRY=mirror.invenzo.mitvaris.com` set in `.env`
3. Re-run `install.sh` / `update.sh` — same flow, just a different registry hostname

The mirror itself runs as a Mitvaris-operated reverse-proxy (Caddy) that sits in front of GHCR with credential translation. Customer never sees a GitHub PAT.

**This is sold as a managed offering (Path B);** the existing GHCR-with-PAT path (Path A in the existing `install.sh`) stays the default.

## What's in core Invenzo for either path

- `install.sh` already supports `--offline` (Path A)
- `INVENZO_REGISTRY` env var override has been in `build-and-push.sh` since v1.15.31
- `setup_registry_auth()` in `install.sh` short-circuits when the auth probe succeeds against any registry, not just ghcr.io — so an HTTPS-fronted mirror works transparently

No code changes needed in core to support either path. Path B is purely Mitvaris-side infrastructure operated under separate SLA.
