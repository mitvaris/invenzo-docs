---
layout: default
title: Invenzo ITAM — Disaster Runbook
description: Step-by-step recovery procedures for common failure modes — Postgres down, Redis down, lost age key, full data loss, all-agents-offline.
---

# Disaster Runbook

When something goes wrong at 3am, you don't want to be reading architecture docs. This page is the **recipe-book** — find the failure that matches what you're seeing, follow the steps.

Every procedure here has been tested on a fresh install. If you find one that doesn't work in your environment, [open an issue](https://github.com/raguyazhin/invenzo-package/issues).

<small>© <a href="https://mitvaris.com">Mitvaris</a>. All rights reserved.</small>

> **Read first:** if the impact is "I can't see the dashboard but agents are still checking in," it's almost certainly the API container — skip to [Scenario 5: API container won't start](#scenario-5-api-container-wont-start). If the impact is "we just lost everything," skip to [Scenario 7: Full data loss / restore from backup](#scenario-7-full-data-loss--restore-from-backup).

---

## Triage flowchart

```
                    Is Invenzo responding at all?
                              │
                  ┌───────────┴───────────┐
                 YES                      NO
                  │                       │
        Are agent checkins         Is Caddy / ingress up?
        succeeding?                       │
                  │                ┌──────┴──────┐
        ┌─────────┴────────┐      YES            NO
        YES               NO       │             │
        │                 │   Is API up?    Cluster /
   "Slow"           "Specific      │       infra issue —
   issue            features        ┌──┴──┐    not Invenzo
   (Scenario 8)    failing"       YES    NO
                                   │     │
                              Is DB up?  └─→ Scenario 5
                                   │
                              ┌────┴────┐
                             YES       NO
                              │         │
                         Is Redis    Scenario 1
                         up?
                              │
                         ┌────┴────┐
                        YES       NO
                         │         │
                  Hardware    Scenario 2
                  / I/O issue
                  on host
```

---

## Scenario 1: Postgres is down or unreachable

**Symptoms:**
* API container restart-loops with `connection refused` / `connection failed` in logs
* `/api/v1/health` returns 503 or doesn't respond
* Agent checkins fail with 500 errors

**Diagnosis (1 minute):**

```bash
# Self-hosted Docker Compose:
docker compose ps postgres
docker compose logs postgres --tail=50

# Kubernetes:
kubectl get pods -n invenzo -l app=postgres
kubectl logs -n invenzo -l app=postgres --tail=50

# Managed (RDS):
aws rds describe-db-instances --db-instance-identifier invenzo-prod \
    --query 'DBInstances[0].DBInstanceStatus'
```

**Recovery paths:**

### A. Postgres container/pod won't start

Most common cause: disk full. Check:
```bash
# Docker
docker exec invenzo-postgres-1 df -h /var/lib/postgresql/data

# Kubernetes
kubectl exec -n invenzo postgres-0 -- df -h /var/lib/postgresql/data
```

If full, **don't delete files inside `pg_wal/` blindly** — that's WAL data. Steps:
1. Stop the API + worker so they stop writing.
2. Increase the volume size (`docker-compose down`, edit volume claim, `up -d`; or for k8s, edit the PVC).
3. Start Postgres back up. It will replay any pending WAL automatically.

### B. RDS / Cloud SQL primary failed over

If you have Multi-AZ: failover is automatic, expect ~60s of read errors then everything resumes.
* Check your `external.postgres.host` is using the cluster endpoint (not the writer endpoint) — RDS `cluster-xyz...` not `instance-xyz...`.
* Check `DATABASE_URL` doesn't pin to an AZ-specific replica.

If failover hasn't happened automatically (single-AZ setup):
```bash
aws rds reboot-db-instance --db-instance-identifier invenzo-prod --force-failover
```

### C. Patroni cluster lost quorum

Check cluster state:
```bash
patronictl -c /etc/patroni/patroni.yml list
```

If you see no leader and 3 replicas in "running" state, etcd / Consul / Zookeeper has lost quorum. Restore quorum first (separate runbook), then Patroni will elect a new leader.

**RPO impact:** ≤ 5 minutes (the WAL stream is the bottleneck). For point-in-time recovery to a specific timestamp, see [Scenario 7](#scenario-7-full-data-loss--restore-from-backup).

---

## Scenario 2: Redis is down

**Symptoms:**
* Discovery jobs queued but never executing
* Worker-beat logs `Cannot connect to redis://...`
* Asset list page works (Postgres reads), but background tasks stop firing
* Agent checkin completes but `recent activity` feed is stale

**Diagnosis:**
```bash
docker compose exec redis redis-cli -a "$(cat /dev/shm/invenzo-secrets/REDIS_PASSWORD)" PING
# Expected: PONG
```

**Recovery:**

### A. Bundled Redis container

```bash
docker compose restart redis
# Wait 10s, then verify:
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" PING
```

If it won't start: check disk (Redis persistence files in `/data`). Same disk-full pattern as Postgres above.

### B. Managed Redis (ElastiCache / Cloud Memorystore)

Failover is automatic when in HA mode. Verify in the AWS / GCP console. If both primary and replica are down (cluster-wide outage), you're waiting for the cloud provider — there's no customer-side recovery.

**Data loss expected:** in-flight Celery task state + RedBeat scheduler last-fire timestamps. This is non-load-bearing — beat tasks resume on the next tick (peripheral_sweep runs at most once / day; auto_rescan_unknown_hosts at most once / N min). No customer asset data is in Redis.

**Side effect:** the credential success cache (`credcache:*` keys) gets invalidated, so the next discovery scan retries every credential. Slower scan, but no functional damage.

---

## Scenario 3: Lost age key (encrypted-secrets mode)

**Symptoms:**
* api / worker / postgres / redis containers fail to start with `decryption failed` or `no identity matched any of the recipients`
* `/etc/invenzo/age.key` is missing, corrupted, or readable as junk

**Severity:** **HIGH but recoverable.** Every DB row, every backup, every user account is preserved. Only stored credential secret values (SSH/WMI/SNMP scan passwords) become unreadable and need re-entry via the UI.

**Recovery:**

The recovery script is shipped in the install package:

```bash
# Self-hosted:
sudo bash /opt/invenzo/recover-lost-age-key.sh

# It will:
# 1. Generate a new age keypair
# 2. Reset every secret in ./secrets/ (rebuild from secure random)
# 3. Reset the postgres password via a one-shot trust-auth dance
# 4. Restart the stack
```

Followed by:
1. Log in with `FIRST_ADMIN_USERNAME` / new admin password (printed by the script).
2. Settings → Credentials → re-enter every scan credential. The credential rows survived; only the encrypted secret_value column is gone.

**See:** [SECRETS.md § Recovery](SECRETS.md#recovery-lost-agekey--what-survives-vs-whats-lost) for the full script + manual procedure.

**Prevention:** keep an offline copy of the age key in a separate location (HSM / sealed envelope / password manager). NEVER back up the age key in the same archive as `./secrets/` — the encryption is moot if both are in the same tarball.

---

## Scenario 4: VAULT_KEY rotation / corruption

**Symptoms:**
* "Failed to decrypt credential" errors when running scans
* Stored scan credentials show in the UI but discovery jobs fail with credential errors
* `cryptography.exceptions.InvalidTag` in worker logs

**Diagnosis:** check whether `VAULT_KEY` in `.env` (or in `/dev/shm/invenzo-secrets/VAULT_KEY` for encrypted-secrets mode) matches what was used to encrypt the credentials.

**Recovery:**

### A. VAULT_KEY accidentally regenerated (most common)

Stored credential rows are unrecoverable — the encryption key was the only thing that could decrypt them.

1. Restore the original `VAULT_KEY` from your offline backup (same envelope as the age key).
2. Restart api + worker.
3. Verify by running a discovery scan against a host with stored credentials.

If you don't have the original `VAULT_KEY` backed up:
1. Settings → Credentials → re-enter every credential. Same as Scenario 3.

### B. Planned VAULT_KEY rotation

```bash
# 1. Read the OLD key from .env / secrets:
OLD=$(cat /dev/shm/invenzo-secrets/VAULT_KEY)

# 2. Generate the new key:
NEW=$(openssl rand -hex 32)

# 3. Set both temporarily (the credential service supports dual-key reads):
docker compose exec api sh -c "echo $OLD > /dev/shm/invenzo-secrets/VAULT_KEY_OLD"
docker compose exec api sh -c "echo $NEW > /dev/shm/invenzo-secrets/VAULT_KEY"

# 4. Trigger a re-encrypt sweep (admin endpoint):
curl -X POST -H "Authorization: Bearer $ADMIN_TOKEN" \
    https://your-invenzo/api/v1/credentials/admin/rekey

# 5. Verify all credentials decrypt with the NEW key, then remove VAULT_KEY_OLD.
```

See `credential_service.py` for the dual-key mechanism. Plan rotation for compliance regimes that require it (HIPAA / SOC 2 best practice = annual; PCI-DSS = annual minimum).

---

## Scenario 5: API container won't start

**Symptoms:**
* `docker compose ps api` shows `Restarting` or `Exited`
* `/api/v1/health` doesn't respond
* UI loads (it's nginx-static) but every API call returns 502

**Diagnosis:**
```bash
docker compose logs api --tail=100
```

**Top causes:**

### A. Database not ready yet (during install / restart)

API depends on Postgres for the schema bootstrap. Wait 30s, the api container will retry.
* If it doesn't recover: check Postgres is up (Scenario 1).

### B. Migration failure

Look for `alembic.runtime.migration` errors in the log. Most common:
* Schema already at HEAD but `alembic_version` table missing — API auto-stamps in this case; check it actually did.
* Migration tries to add a column that already exists (drift). The `_run_alembic_migrations` self-healer should catch this, but if it doesn't:
  ```bash
  # Stamp past the failing migration manually
  docker compose exec api alembic stamp <next_revision>
  docker compose restart api
  ```

### C. SECRET_KEY / VAULT_KEY missing or wrong format

Logs will say `entering setup mode (JWT operations will fail until fixed)`. The api stays up but auth is broken.
* Check `.env` has `SECRET_KEY` (≥64 hex chars) and `VAULT_KEY` (exactly 64 hex chars).
* For encrypted-secrets mode: check `/dev/shm/invenzo-secrets/` is populated. If not, the decrypt-and-exec entrypoint failed — check the age key is present and readable.

### D. Port collision

Another process is using port 8000 on the host. Change `HTTPS_PORT` / `HTTP_PORT` in `.env` and recreate.

---

## Scenario 6: All agents went offline simultaneously

**Symptoms:**
* Dashboard "online endpoints" count drops to near-zero in <5 minutes
* No agent checkins arriving (check `agent_registrations.last_checkin_at`)
* `worker.log` shows fleet-wide push-deploy attempts failing

**This is BAD. Almost certainly one of:**
1. Recent server-side `AGENT_CALLBACK_URL` change broke the URL agents use
2. TLS cert on the API endpoint expired / rotated and agents reject it
3. Bad agent self-update — broken binary pushed to the fleet
4. Network outage between agents and the API (firewall change?)

**Recovery decision tree:**

### A. If TLS cert expired

Renew the cert. Agents with TLS verify enabled (`tls_mode: pinned` or `mtls`) will reject mismatched certs and refuse to check in.

### B. If broken agent self-update

Roll back the `AGENT_SCRIPT_VERSION`:
```bash
# 1. Find the previous-known-good version
ls -lt agent-binaries/    # last-modified shows current
git log --oneline -- agent/Makefile | head    # bump history

# 2. Restore the previous binary
cp agent-binaries/invenzo-agent-windows-amd64.exe.previous \
   agent-binaries/invenzo-agent-windows-amd64.exe

# 3. Roll back AGENT_SCRIPT_VERSION in api/app/routers/agents.py
# 4. docker compose up -d --build api

# 5. Online agents pick up the rollback on next checkin via update_available
# 6. For fully-offline agents: SSH/SMB push-deploy with action=update
```

(W9 in the refactor plan adds auto-rollback on canary failure — until then this is manual.)

### C. If `AGENT_CALLBACK_URL` was changed

Agents have the old URL baked into `config.yaml`. Two recovery paths:

1. **Live rotation (v1.9.14+ agents):** API checkin response carries `config_update.api_url` to update agents in place. Hit the rebroadcast endpoint:
   ```bash
   curl -X POST -H "Authorization: Bearer $ADMIN_TOKEN" \
       "https://your-invenzo/api/v1/agent-auto-deploy/admin/rebroadcast-url?include_online=true"
   ```

2. **Push-deploy via SSH/SMB:** if agents can't reach the API at all, the worker can push a new install with the corrected URL using stored credentials. Same endpoint as above.

### D. If genuinely network-isolated

Network/firewall investigation. Not an Invenzo problem. Restore connectivity, agents will resume on their next checkin (default cadence 5 min).

---

## Scenario 7: Full data loss / restore from backup

**Symptoms:**
* Postgres data volume corrupted / deleted
* RDS instance accidentally deleted (yes, it happens)
* Need to clone an environment from a backup

**Recovery:**

### A. Restore a pg_dump backup (same Postgres major version)

```bash
# 1. Stop the API + worker so nothing writes during restore
docker compose stop api worker worker-beat

# 2. Drop the existing schema (if any) and restore
docker compose exec postgres psql -U invenzo -d invenzo -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"

# 3. Restore from backup (replace path with your actual backup file):
docker compose exec -T postgres pg_restore -U invenzo -d invenzo --no-owner --no-acl \
    < /path/to/invenzo_backup.dump

# 4. Start the API back up
docker compose start api worker worker-beat
```

### B. Restore via the Invenzo UI (admin endpoint)

Settings → Backup & Restore → upload the `.dump` file → click Restore. The endpoint validates the manifest first (Postgres version + VAULT_KEY fingerprint) and warns if the target environment differs.

### C. Cross-server restore

When restoring to a different host than the one that took the backup, **the VAULT_KEY mismatch is the gotcha.** The manifest reports `vault_key_fp: <SHA-256>` so the UI surfaces a banner: "Encrypted credentials in this backup were encrypted with a different VAULT_KEY than this server has. They will not decrypt."

Two options:
1. Copy the original `VAULT_KEY` to the new server before restore. Stored credentials work as-is.
2. Restore as-is and re-enter every credential post-restore. Faster, slight one-time pain.

### D. PITR (point-in-time recovery)

Only available if you have managed Postgres with PITR enabled. RDS / Cloud SQL: choose a recovery point in the console (down to the second within retention window).

For self-managed Patroni + wal-g:
```bash
wal-g backup-fetch /var/lib/postgresql/data LATEST --target-time "2026-04-29T03:15:00Z"
# Then start postgres in recovery mode
```

**RTO target:** ≤ 30 minutes for any of the above on a freshly provisioned instance.

---

## Scenario 8: Performance degradation

**Symptoms:**
* Asset list page p95 > 2s
* Agent checkins time out
* UI feels sluggish across the board

**Diagnosis order:**

1. **Check Postgres connections:**
   ```sql
   SELECT count(*), state FROM pg_stat_activity GROUP BY state;
   ```
   If `idle in transaction` > 50, something is leaking connections. Restart the worker.

2. **Check disk I/O on Postgres:**
   ```bash
   docker compose exec postgres iostat -x 1 5
   ```
   `%util > 80%` sustained = disk-bound. Need bigger volume or move to managed instance.

3. **Check Redis memory:**
   ```bash
   docker compose exec redis redis-cli -a "$REDIS_PASSWORD" INFO memory | grep used_memory_human
   ```
   If close to `maxmemory`, eviction is happening — bump it up or accept the cache misses.

4. **Check agent checkin volume:**
   ```sql
   SELECT date_trunc('minute', last_checkin_at) AS minute, count(*)
   FROM agent_registrations
   WHERE last_checkin_at > now() - interval '15 minutes'
   GROUP BY 1 ORDER BY 1 DESC;
   ```
   If you're seeing 5000 checkins/minute and the API can't keep up, you've outgrown the deployment size — see [HA Architecture sizing](HA_ARCHITECTURE.md#sizing-guidance).

5. **Check the slow query log** (PG `log_min_duration_statement = 1000` will surface anything taking >1s).

6. **Run the published benchmark** to confirm it's a workload issue, not a regression:
   ```bash
   docker compose exec api python -m tests.scale.benchmark_db --requests 50
   ```
   Compare against [BENCHMARKS.md](BENCHMARKS.md). If your numbers are 5x worse, something's wrong with your install — not the product.

---

## When all else fails

* **For Enterprise tier customers:** [contact Mitvaris](https://mitvaris.com) — same-business-day video call response. We've seen most of these before.
* **For Professional tier:** email support, 1-business-day response.
* **For Starter tier / open-source path:** [open a GitHub issue](https://github.com/raguyazhin/invenzo-package/issues) — best-effort response, no SLA.

---

## What we DON'T cover here

* **Kubernetes cluster failures** — that's your platform team's problem. We assume the cluster is healthy.
* **Cloud provider outages** — RDS / EKS / GCP regional failures. We can't help; you wait.
* **Network firewall / DNS issues** — same. Confirm Invenzo is the failure domain before reaching out.
* **OS-level failures on the host** (Linux kernel panics, filesystem corruption) — restore from backup is the path. There's no Invenzo-specific recovery.

---

[← HA Architecture](HA_ARCHITECTURE.md) · [Back to docs index](index.md) · [Pricing](PRICING.md)
