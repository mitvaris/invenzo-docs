---
layout: default
title: Deployment Guide
description: Install, upgrade, and uninstall Invenzo ITAM
---

# Deployment Guide

This guide covers customer installation and lifecycle management for Invenzo.

For day-to-day administration after install, see [Admin Guide](ADMIN_GUIDE.md).
For backups, HA, secrets, and disaster recovery, see
[Operations Guide](OPERATIONS_GUIDE.md).

---

## Requirements

| Resource | Minimum | Recommended |
|---|---|---|
| OS | Ubuntu 22.04, Debian 12, RHEL 9, Rocky 9 | Ubuntu 22.04 LTS or Rocky 9 |
| CPU | 4 vCPU | 8+ vCPU |
| RAM | 8 GB | 16+ GB |
| Disk | 50 GB | 200+ GB, SSD-backed |
| Network | Outbound HTTPS for image pulls | Inbound 80/443 and routed access to discovery targets |
| Runtime | Docker Engine 24+ with Compose v2 | Current stable Docker Engine |

For large environments, size CPU, RAM, disk, and PostgreSQL capacity from pilot
scan volume, asset count, software count, retention, and reporting load.

---

## Repository Model

Customers install from the public package repository:

```bash
git clone https://github.com/mitvaris/invenzo-package.git /tmp/invenzo-package
cd /tmp/invenzo-package
```

The package contains install scripts, update scripts, compose files, and
operator runbooks. It does not contain private application source code.

---

## Choose Database Mode

Database mode is selected by the server installer, not by the browser
`/install` wizard.

| Mode | Command | Use when |
|---|---|---|
| Embedded PostgreSQL | `sudo bash install.sh` | Standard appliance install. Invenzo manages its own PostgreSQL container. |
| External PostgreSQL | `sudo bash install.sh --external-db` | Customer already has a production PostgreSQL service and wants Invenzo to use it. |

The API must already have a working database connection before `/install` can
load. The browser wizard validates the selected DB mode, creates the first
admin, runs schema bootstrap/migrations, and seeds default data.

---

## Default Install: Embedded PostgreSQL

```bash
sudo bash install.sh
```

The installer:

1. checks OS, root access, ports, CPU, RAM, and disk
2. installs Docker Engine and Compose if needed
3. copies files to the install directory, usually `/opt/invenzo`
4. creates an age keypair for encrypted secrets
5. generates secrets and encrypts them into `secrets/*.enc`
6. prompts for hostname, admin account, and registry token when needed
7. pulls images
8. starts services
9. waits for health checks

Open:

```text
https://<your-hostname>/install
```

Complete the browser wizard and then sign in with the admin account.

---

## External PostgreSQL Install

Before running the installer, ask the DBA to create a dedicated database and
application role:

```sql
CREATE ROLE invenzo LOGIN PASSWORD '<strong-password>';
CREATE DATABASE invenzo OWNER invenzo;

\c invenzo

CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS citext;
CREATE EXTENSION IF NOT EXISTS btree_gist;

GRANT CREATE, USAGE ON SCHEMA public TO invenzo;
```

The application role must be able to create, alter, and index tables inside the
Invenzo database. Do not use DBA credentials as the application account.

Interactive:

```bash
sudo bash install.sh --external-db
```

Non-interactive:

```bash
sudo INVENZO_DB_MODE=external \
  INVENZO_POSTGRES_HOST=10.10.20.15 \
  INVENZO_POSTGRES_PORT=5432 \
  INVENZO_POSTGRES_DB=invenzo \
  INVENZO_POSTGRES_USER=invenzo \
  INVENZO_POSTGRES_PASSWORD='strong-password' \
  INVENZO_POSTGRES_SSLMODE=require \
  bash install.sh --external-db
```

For certificate validation, place the CA file under the install directory
`certs/` folder and pass a container path such as:

```bash
INVENZO_POSTGRES_SSLROOTCERT=/certs/postgres-ca.crt
```

Use `POSTGRES_SSLMODE=require` at minimum for production. Use `verify-full`
when the customer certificate and DNS naming support it.

The setup wizard preflight checks:

- database reachability and login
- PostgreSQL server version
- `pgcrypto`, `citext`, and `btree_gist`
- schema state and Alembic revision
- migration privileges
- SSL mode and SSL activity

After the first admin exists, setup endpoints are locked and the database
preflight endpoint returns `404` by design.

---

## Offline Install

On an internet-connected machine:

```bash
git clone https://github.com/mitvaris/invenzo-package.git
tar czf invenzo-package.tar.gz invenzo-package/

for img in invenzo-api invenzo-discovery invenzo-ui invenzo-updater; do
  docker pull ghcr.io/mitvaris/$img:<version>
  docker save ghcr.io/mitvaris/$img:<version> | gzip > $img-<version>.tar.gz
done
```

Transfer the package and image archives to the offline server:

```bash
tar xzf invenzo-package.tar.gz
cd invenzo-package

docker load < /path/to/invenzo-api-<version>.tar.gz
docker load < /path/to/invenzo-discovery-<version>.tar.gz
docker load < /path/to/invenzo-ui-<version>.tar.gz
docker load < /path/to/invenzo-updater-<version>.tar.gz

sudo bash install.sh --offline
```

External DB mode can also be used offline if the external PostgreSQL endpoint is
reachable from the Invenzo server.

---

## Upgrade

```bash
cd /opt/invenzo
sudo bash update.sh <version>
```

The update script:

1. creates a pre-upgrade backup when using embedded PostgreSQL
2. pulls or uses the requested images
3. drains workers
4. updates the image version
5. restarts services
6. waits for health checks

External PostgreSQL customers usually use their DBA backup/PITR process before
upgrades. The API still runs Alembic migrations, so the external database must
be reachable during the upgrade.

After upgrade:

1. hard-refresh the browser
2. open **Settings > Version & Schema**
3. confirm application version and DB schema are in sync
4. spot-check Discovery, Assets, Agents, Reports, and Backup status

---

## Uninstall

```bash
cd /opt/invenzo
sudo bash uninstall.sh
```

Use `--force` for unattended uninstall flows:

```bash
sudo bash uninstall.sh --force
```

For external PostgreSQL, uninstalling Invenzo does not remove the customer DB.
DB cleanup must be performed by the customer DBA.

---

## Useful Commands

```bash
cd /opt/invenzo
docker compose ps
docker compose logs -f api
docker compose logs -f worker
docker compose restart api worker worker-beat pinger
docker compose pull
```

Health check:

```bash
curl -k https://<your-hostname>/api/v1/health
```

Install status:

```bash
curl -k https://<your-hostname>/api/v1/setup/status
```

---

## Troubleshooting Install

| Symptom | Likely cause | Action |
|---|---|---|
| Port check fails | 80/443 already in use | Stop the existing service or change proxy binding. |
| API unhealthy | Database, secrets, Redis, or migration issue | Check `docker compose logs api`. |
| `/install` redirects to login | First admin already exists | Installation is completed; sign in. |
| External DB preflight fails | Firewall, SSL, credentials, extension, or privileges | Fix the DB side and rerun preflight. |
| Images cannot pull | Registry token/network issue | Confirm registry credentials and outbound HTTPS. |
| Browser shows stale UI after update | Cached assets/service worker | Hard-refresh with Ctrl+Shift+R. |
