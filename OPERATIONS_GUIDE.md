---
layout: default
title: Operations Guide
description: Backup, recovery, HA, secrets, monitoring, and support operations
---

# Operations Guide

This guide covers operating Invenzo after deployment: backup and restore,
disaster recovery, high availability, secrets, monitoring, AV/EDR exclusions,
and support checks.

For install and upgrade commands, see [Deployment Guide](DEPLOYMENT_GUIDE.md).

---

## Backup And Restore

Configure scheduled backups from **Settings > Backup & Restore**.

For embedded PostgreSQL, the platform can run `pg_dump` from the Invenzo stack.
For external PostgreSQL, the customer DBA/platform team usually owns backup,
restore, point-in-time recovery, HA, monitoring, and patching.

Manual embedded backup:

```bash
cd /opt/invenzo
docker compose exec postgres pg_dump -U invenzo invenzo | gzip > backup_$(date +%Y%m%d).sql.gz
```

Manual embedded restore:

```bash
cd /opt/invenzo
docker compose stop api worker worker-beat pinger
gunzip -c backup_20260519.sql.gz | docker compose exec -T postgres psql -U invenzo invenzo
docker compose up -d
```

External PostgreSQL restore should follow the customer's approved PostgreSQL
restore process. Restart Invenzo API and workers after restore so all containers
reconnect cleanly.

Run restore drills. A backup that has never been restored is not proof.

---

## Disaster Recovery

Keep offline copies of:

- latest database backup or PITR recovery point
- `/etc/invenzo/age.key`
- `/opt/invenzo/.env`
- `/opt/invenzo/secrets/*.enc`
- customer TLS certificates and Caddy customizations
- current install package and image version

Basic full restore for embedded PostgreSQL:

1. provision a clean host
2. install Docker
3. restore `/etc/invenzo/age.key`
4. restore `/opt/invenzo` configuration and encrypted secrets
5. start the stack
6. restore the database backup if the DB volume was lost
7. verify login, schema sync, discovery schedules, agents, and reports

For external PostgreSQL, DB recovery happens in the customer DB platform. Invenzo
recovery focuses on restoring app configuration, encrypted secrets, proxy/TLS,
and containers.

---

## High Availability

The standard compose deployment is appliance-style and suitable for many
on-premises customers. HA designs should be planned deliberately.

Recommended HA responsibilities:

| Component | HA approach |
|---|---|
| PostgreSQL | External managed PostgreSQL, streaming replica, or customer HA platform. |
| Invenzo API/UI | Multiple app nodes behind a load balancer only when shared DB, Redis, secrets, and storage are planned. |
| Redis | Customer-managed Redis HA if queue/cache availability is critical. |
| Secrets | Back up age key and encrypted secret blobs offline. |
| Reports/backups | Store exports and backups on resilient storage. |
| Collectors | Use multiple collectors across network segments. |

Do not claim HA from running extra containers without testing failover, session
behavior, worker queues, migrations, backups, and report delivery.

---

## Secrets

Invenzo stores service secrets as age-encrypted files under `secrets/*.enc`.
The private age key is normally stored at:

```text
/etc/invenzo/age.key
```

Rules:

- keep the age key root-readable only
- back it up offline immediately after install
- never paste secret values into tickets or screenshots
- rotate credentials through approved support or admin workflows
- keep `.env` for non-secret configuration where possible

If the age key is lost, encrypted secrets cannot be decrypted. Recovery requires
regenerating secrets and, for embedded PostgreSQL, resetting the DB password
through the supported recovery process.

---

## Monitoring

Use built-in health surfaces and infrastructure monitoring together.

Check:

- API health endpoint
- worker and beat status
- pinger status
- PostgreSQL disk, CPU, memory, connections, and replication if external/HA
- Redis health
- backup success and restore drill age
- discovery queue depth and scan failures
- agent offline/outdated counts
- report delivery failures

Useful commands:

```bash
cd /opt/invenzo
docker compose ps
docker compose logs --tail=200 api
docker compose logs --tail=200 worker
curl -k https://<host>/api/v1/health
```

---

## AV / EDR Exclusions

Endpoint protection tools can block agent install, service creation, script
execution, network callbacks, or self-update. Coordinate with the customer's
security team before broad rollout.

Typical Windows paths:

```text
C:\ProgramData\InvenzoAgent\
C:\ProgramData\InvenzoAgent\invenzo-agent.exe
C:\ProgramData\InvenzoAgent\config.yaml
C:\ProgramData\InvenzoAgent\logs\
```

Typical exclusions:

- agent install directory
- agent service executable
- temporary installer bundle path used during deployment
- outbound HTTPS from endpoints to the Invenzo callback URL
- service creation and upgrade actions when using managed rollout

Security assurance:

- the agent should run with the least local privilege required for collection
- outbound communication should point only to the configured Invenzo server
- installer bundles should be downloaded from the authenticated Agents page
- agent update evidence should be visible in Invenzo

---

## SSO And Directory Operations

Invenzo supports enterprise identity patterns such as OIDC/SAML SSO and
LDAP/Active Directory integration models.

Operational checklist:

1. configure provider metadata and callback URLs
2. map groups or claims to Invenzo roles
3. test with a non-admin pilot user
4. keep at least one local break-glass admin
5. audit login and role changes

Common SSO providers include Google, Microsoft Entra ID, Okta, ADFS,
PingFederate, Shibboleth, and compatible OIDC/SAML providers.

---

## Common Incident Checks

| Incident | First checks |
|---|---|
| API down | `docker compose ps`, API logs, DB reachability, secrets decrypt, migrations. |
| Redis down | Redis logs, memory/disk pressure, worker queue behavior. |
| All agents offline | Callback URL, DNS, TLS, firewall, proxy, server availability. |
| Discovery failing | Credentials, collector health, subnet assignment, firewall, SNMP/WMI/SSH reachability. |
| Backups failing | Disk space, DB credentials, external DB network path, `pg_dump` availability. |
| Reports not delivered | SMTP/webhook config, schedule status, report history, failed checks. |
| Performance degraded | DB load, missing indexes/schema drift, queue backlog, large exports, host resource pressure. |

Escalate with:

- application version
- DB schema version
- failing page or API endpoint
- recent update/install action
- relevant container logs
- screenshots of visible errors
- backup status
- whether DB is embedded or external
