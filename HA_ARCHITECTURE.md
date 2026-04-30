---
layout: default
title: Invenzo ITAM — HA Reference Architecture
description: Production-grade high-availability deployment topology for Invenzo. Managed Postgres + managed Redis + multi-replica API + worker leader election.
---

# HA Reference Architecture

This page documents the **production-grade deployment topology** for Invenzo. It's the architecture we recommend for any install above ~500 endpoints, anything where downtime has a dollar cost, and every install on the [Enterprise tier](PRICING.md).

If you're running a small lab / pilot install, you do NOT need any of this — the [single-node Docker Compose deployment](DEPLOYMENT_GUIDE.md) is fine. This document exists so your security review passes when procurement asks "what's the HA story?"

<small>© <a href="https://mitvaris.com">Mitvaris</a>. All rights reserved.</small>

---

## At-a-glance

| Component | Single-node default | HA reference |
|---|---|---|
| **Postgres** | Bundled StatefulSet, single replica, no PITR | Managed (RDS / Cloud SQL / Patroni cluster), multi-AZ, PITR enabled |
| **Redis** | Bundled StatefulSet, single replica | Managed (ElastiCache / Cloud Memorystore), primary + replica with automatic failover |
| **API** | 1 container, 1 worker process | 2–3 replicas, HPA-scaled, behind ingress LB |
| **Discovery worker** | 1 container | 2 replicas (Celery distributes work) |
| **Worker-beat** | 1 container | **Always 1 replica** (RedBeat handles leader election internally; do NOT scale this) |
| **UI** | 1 nginx container | 2 replicas behind ingress LB |
| **Ingress** | Caddy single-host | Cloud LB (ALB / NLB / Cloud Load Balancing) → Caddy or ingress-nginx → services |
| **Backups** | Local pg_dump volume | WAL archiving to S3/GCS + nightly pg_dump for full-state snapshots |
| **Secrets** | Age key on disk | TPM-bound age key OR external secret manager (Vault / AWS Secrets Manager / Azure Key Vault) |

---

## Reference topology

```
                                  ┌─────────────────────────────────┐
                                  │  Cloud LB (ALB / NLB / GCLB)    │
                                  │   - TLS termination             │
                                  │   - Health checks               │
                                  └────────────┬────────────────────┘
                                               │
                                               ▼
                          ┌──────────────────────────────────────────┐
                          │  Ingress controller (ingress-nginx /     │
                          │  Caddy / Cloudflare Tunnel)              │
                          └────────┬─────────────────┬───────────────┘
                                   │                 │
                          ┌────────▼─────┐   ┌──────▼────────┐
                          │  api (×2-3)  │   │   ui (×2)     │
                          │  (HPA)       │   │   (nginx,     │
                          │              │   │    static)    │
                          └────┬────┬────┘   └───────────────┘
                               │    │
              ┌────────────────┘    └──────────────┐
              │                                    │
   ┌──────────▼─────────┐              ┌──────────▼────────────┐
   │  worker (×2)       │              │  worker-beat (×1)     │
   │  - discovery       │              │  - RedBeat scheduler  │
   │  - rollouts        │              │  - leader-elected     │
   │  - sweeps          │              │  - DO NOT scale       │
   └──────┬─────────────┘              └─────────┬─────────────┘
          │                                      │
          └──────────────┬───────────────────────┘
                         │
            ┌────────────▼─────────────┐
            │  Managed Redis           │
            │  (ElastiCache /          │
            │   Cloud Memorystore /    │
            │   Redis Sentinel)        │
            │   - primary + replica    │
            │   - automatic failover   │
            └──────────────────────────┘
                         ▲
                         │
            ┌────────────┴─────────────┐
            │  Managed Postgres        │
            │  (RDS / Cloud SQL /      │
            │   Patroni cluster)       │
            │   - multi-AZ             │
            │   - PITR (5-min WAL)     │
            │   - read replica (opt.)  │
            │   - automatic failover   │
            └──────────────────────────┘
                         │
                         ▼
            ┌──────────────────────────┐
            │  Object storage          │
            │  (S3 / GCS / Azure Blob) │
            │   - WAL archive          │
            │   - nightly pg_dump      │
            │   - 30-day retention     │
            └──────────────────────────┘
```

---

## Sizing guidance

These are guidelines, not contracts. Real numbers depend on your discovery cadence, agent fleet size, and how many compliance scans you run. See [BENCHMARKS.md](BENCHMARKS.md) for measured numbers at 10K assets.

| Endpoint count | Postgres | Redis | API | Worker | Worker-beat |
|---|---|---|---|---|---|
| **≤ 1,000** | db.t3.medium / 2 vCPU / 4 GB | cache.t3.micro / 0.5 GB | 2 × (200m CPU / 512 MB) | 1 × (500m CPU / 1 GB) | 1 × (100m CPU / 256 MB) |
| **1,000–5,000** | db.m6i.large / 2 vCPU / 8 GB | cache.t3.small / 1.5 GB | 2 × (500m CPU / 1 GB) | 2 × (1 CPU / 2 GB) | 1 × (200m CPU / 512 MB) |
| **5,000–10,000** | db.m6i.xlarge / 4 vCPU / 16 GB | cache.m6g.large / 6 GB | 3 × (1 CPU / 2 GB) | 2 × (2 CPU / 4 GB) | 1 × (500m CPU / 1 GB) |
| **10,000+** | db.r6i.xlarge / 4 vCPU / 32 GB + read replica | cache.m6g.large / 6 GB + replica | 3-5 × (1 CPU / 2 GB), HPA on CPU | 4 × (2 CPU / 4 GB), HPA on queue depth | 1 × (500m CPU / 1 GB) |

Storage: budget **2 GB Postgres / 1,000 endpoints** as a rough starting point. Asset history rows compound over time — prune retention via Settings or accept growth.

---

## RPO / RTO targets

What "high availability" actually means in numbers. These are the targets the reference architecture is built to hit.

| Metric | Single-node | HA reference |
|---|---|---|
| **RPO** (max data loss in worst case) | up to last nightly backup (≤ 24h) | **≤ 5 minutes** (Postgres WAL archiving) |
| **RTO** (time to restore service) | manual restore: ~1 hour | **≤ 5 minutes** for AZ failure (automatic failover) · ~30 min for region failure (manual failover to DR) |
| **Backup retention** | local volume only | 30 days minimum (WAL + nightly snapshots in S3/GCS) |
| **Backup geo-redundancy** | none | Cross-region replication on the backup bucket |

**Key call:** the bundled StatefulSet Helm path **cannot meet these targets** — there's no replication, no PITR, no automatic failover. If your contract / regulation requires <5 min RPO, the reference architecture is mandatory.

---

## Component-by-component

### Postgres (the load-bearing one)

This is the single most important component to get right. Everything else is replaceable; data loss isn't.

**Requirements:**
* PostgreSQL 16+ (matches the bundled image; older versions may work but aren't tested)
* Connection limit ≥ 100 (each api / worker / worker-beat / pinger replica opens a small pool)
* PITR enabled (RDS: enable automated backups + set retention ≥ 7 days; Cloud SQL: same; Patroni: configure WAL archiving + base backups via wal-g / pgBackRest)
* `pgcrypto` and `citext` extensions available (Invenzo uses both)
* TLS-only connections strongly recommended (set `sslmode=require` in `DATABASE_URL`)

**Multi-AZ deployment:**
* RDS: `Multi-AZ` checkbox at instance creation. Failover is automatic, ~60s.
* Cloud SQL: `Regional` instance. Same behavior.
* Patroni: 3-node cluster across 3 AZs. Failover via etcd / Consul / ZooKeeper.

**Read replicas:**
* Optional. Invenzo doesn't use read replicas today (no read/write split in the API). You can attach one for ad-hoc analytics, monitoring queries, or to feed a BI tool — none of those will affect the main app.

**Connection string format:**
```
postgresql+asyncpg://USER:PASS@HOST:5432/DBNAME?sslmode=require
```

In Helm: set `external.postgres.host` + use the `secretKey: databaseUrl` (or compose the URL from individual secret keys — see chart README).

### Redis

Used for:
* Celery broker + result backend (worker queues)
* RedBeat scheduler state (worker-beat leader election)
* Per-asset metric SSE pub/sub
* Credential success cache (24h TTL)

**Failure mode:** if Redis goes down, in-flight discovery jobs lose their place and worker-beat stops firing scheduled tasks. Existing data is unaffected. Once Redis is back, beat tasks resume on their next tick.

**Requirements:**
* Redis 7+ (the bundled chart uses 7-alpine)
* `requirepass` set (no anonymous access)
* `maxmemory-policy allkeys-lru` for the cache use-case
* TLS optional but recommended for cross-AZ traffic

**HA options (in order of cost):**
1. **AWS ElastiCache for Redis** — Multi-AZ with automatic failover. Cheapest managed option. ~$50/mo for cache.t3.small.
2. **Google Cloud Memorystore** — equivalent. Standard tier = HA.
3. **Redis Sentinel** (self-managed) — 3 sentinels + 1 primary + 2 replicas. More work, more flexibility.

### API + UI replicas

Stateless. Scale horizontally. Behind a Layer-7 LB or ingress controller doing TLS termination.

* **API:** 2 replicas minimum. HPA on CPU (target 60%). Set `api.replicaCount: 2` (or higher) in values.yaml.
* **UI:** 2 replicas minimum (it's just nginx serving static assets — could even be a single replica behind a CDN).
* **Pod anti-affinity:** spread replicas across nodes / AZs so a single AZ failure doesn't knock all replicas. The Helm chart sets this when `affinity.zoneAware=true`.

### Discovery worker

Multiple replicas are SAFE — Celery distributes tasks across all listening workers. No coordination needed.

* Recommended: 2 replicas for fleets >1K endpoints, scale to 4 for >10K
* HPA on Redis queue depth (custom metric — requires Prometheus + KEDA, optional)
* Each worker uses CAP_NET_RAW for fping/nmap; the Helm chart sets this via `worker.netRaw=true`

### Worker-beat

**Exactly 1 replica.** Do NOT scale this up.

RedBeat handles leader election internally — the scheduler state lives in Redis, and only one Celery beat process can hold the leader lock at a time. Running 2 worker-beat replicas works in theory (only one will be leader, the other sits idle waiting), but it's wasted compute. The chart enforces `replicas: 1` and adds a PodDisruptionBudget so cluster operations can't take it down.

If worker-beat is down for >5 minutes, scheduled tasks (heal-offline-agents, peripheral-sweep, etc.) start to backlog. Most are idempotent and self-recovering on the next tick — the system tolerates short worker-beat outages well.

### Ingress / load balancer

Two reasonable patterns:

**Pattern A — Cloud LB → Helm `Ingress` resource:**
```yaml
ingress:
  enabled: true
  className: alb           # or "nginx", "gce", whatever your cluster uses
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: invenzo.example.com
      paths: [...]
  tls:
    - hosts: [invenzo.example.com]
      secretName: invenzo-tls
```

**Pattern B — Cloudflare Tunnel / Tailscale Funnel:**
* No public LB needed; the tunnel daemon connects outbound from the cluster
* Best for installs that genuinely don't need internet exposure (zero-trust networks)
* Caddy still runs inside the cluster as the in-mesh router

Both work. Pick based on your existing perimeter strategy.

---

## Secrets management at scale

At single-node scale, the [age-encrypted-secrets](SECRETS.md) approach is sufficient. At HA scale, you have additional options:

### Option 1 — Age key + ExternalSecrets / SealedSecrets

Keep using age-encrypted secrets at rest, but generate the age key once at install time and inject the master via ExternalSecrets / SealedSecrets / Vault CSI:

```yaml
secrets:
  externalSecretRef: invenzo-secrets   # ExternalSecret resource owned by your secrets operator
```

The chart skips the inline secret generation when `externalSecretRef` is set; the operator delivers the secret values into the cluster, the api/worker pods read them as files via `SECRETS_DIR`. Already wired in the chart.

### Option 2 — Remote secret provider (Vault / AWS / Azure)

Set `SECRETS_PROVIDER=vault | aws_sm | azure_kv` as an env var on the api + worker pods. The api side's `read_secret()` will fall through file → env → remote provider, so values can live in the secret manager and be fetched at startup.

This is the cleanest option for orgs with an existing Vault / AWS Secrets Manager investment. Documented under [SECRETS.md § Remote secret providers](SECRETS.md#9-remote-secret-providers).

### Option 3 — TPM-bound age key

For bare-metal installs where you control the host BIOS / Secure Boot state, bind the age key to a TPM key slot via `bind-age-key-to-tpm.sh`. The key cannot be exfiltrated even by an attacker with root on the host.

---

## Backup strategy

**Two layers, run independently:**

1. **Continuous WAL archiving** for sub-5-minute RPO.
   * RDS / Cloud SQL: enabled by checking the "automated backups" box.
   * Patroni: enable wal-g or pgBackRest in the PostgreSQL config.
   * Storage: cross-region S3 / GCS bucket with object-lock for ransomware protection.

2. **Nightly pg_dump** for full-state snapshots that are easy to restore on a different host.
   * Run via the existing Invenzo backup endpoint OR an external Kubernetes CronJob.
   * Encrypted at rest (server-side encryption + customer-managed KMS key).
   * Retained 30 days minimum; 90 days if regulated.

The two layers cover different failure modes:
* WAL archive recovers from a database corruption / accidental DELETE / failed migration — restore to a point 1 minute before the incident.
* pg_dump recovers from "we accidentally deleted the entire RDS instance" — re-create infra from scratch, restore from S3.

---

## Helm chart configuration template

```yaml
# values.production.yaml — minimal HA-grade settings
postgres:
  enabled: false      # NEVER use the bundled StatefulSet in production

redis:
  enabled: false      # NEVER use the bundled StatefulSet in production

external:
  postgres:
    host: invenzo-prod.cluster-xyz.us-east-1.rds.amazonaws.com
    port: 5432
  redis:
    host: invenzo-prod.abc.use1.cache.amazonaws.com
    port: 6379

secrets:
  externalSecretRef: invenzo-secrets

api:
  replicaCount: 3
  resources:
    requests: { cpu: "500m", memory: "1Gi" }
    limits:   { cpu: "2",    memory: "4Gi" }

worker:
  replicaCount: 2
  netRaw: true
  resources:
    requests: { cpu: "500m", memory: "1Gi" }
    limits:   { cpu: "2",    memory: "4Gi" }

beat:
  replicaCount: 1     # Always 1 — RedBeat enforces leader election

ui:
  replicaCount: 2

ingress:
  enabled: true
  className: alb
  hosts: [{host: invenzo.example.com, paths: [...]}]
  tls: [{hosts: [invenzo.example.com], secretName: invenzo-tls}]

autoscaling:
  api:    {enabled: true, minReplicas: 2, maxReplicas: 6, targetCPUUtilizationPercentage: 60}
  worker: {enabled: true, minReplicas: 2, maxReplicas: 4}

podDisruptionBudget:
  api:    {enabled: true, minAvailable: 1}
  worker: {enabled: true, minAvailable: 1}
  beat:   {enabled: false}   # Single-replica; PDB would block all node drains

affinity:
  zoneAware: true     # Pod anti-affinity across AZs

networkPolicy:
  enabled: true       # Default-deny ingress, allow only what's needed
```

Apply: `helm install invenzo charts/invenzo/ -f values.production.yaml -n invenzo --create-namespace`.

---

## What we DON'T promise

Honest about the limits:

* **No multi-region active-active.** The data model is single-tenant, single-cluster. Cross-cluster federation requires external coordination (DNS-level traffic split, async replication on managed Postgres, manual failover). We can advise; we can't make it one-click.
* **No 99.999% SLA.** With managed Postgres + Redis + multi-replica API, expect 99.9% (≈8h downtime/year) realistically. 99.99% requires multi-region active-passive at minimum.
* **No "infinite" scale.** Above 25K endpoints in a single install, you're past published benchmarks. Multi-cluster federation (per-region installs with read-only roll-up) is the path; this is not yet documented.

For any of these, [contact Mitvaris](https://mitvaris.com) — we do consulting engagements for non-standard topologies on Enterprise tier.

---

## Next steps

* [Disaster runbook](DISASTER_RUNBOOK.md) — what to do when Postgres dies / VAULT_KEY is corrupted / all agents go offline at once.
* [Pricing](PRICING.md) — Enterprise tier includes a 4-hour HA architecture review with Mitvaris.
* [Helm chart README](https://github.com/mitvaris/invenzo/blob/master/charts/invenzo/README.md) — chart-specific knobs not documented here.

---

[← Back to docs index](index.md) · [Disaster runbook →](DISASTER_RUNBOOK.md)
