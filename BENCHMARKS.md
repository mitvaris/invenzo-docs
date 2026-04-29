---
layout: default
title: Invenzo ITAM — Performance & Scale Benchmarks
description: Published latency numbers for the Invenzo asset list, search, and filter queries at 10K endpoints.
---

# Performance & Scale Benchmarks

How fast is Invenzo at real scale? This page publishes numbers we'd stand behind. We re-run these benchmarks before each minor release.

<small>© <a href="https://mitvaris.com">Mitvaris</a>. All rights reserved.</small>

> **Reading the numbers:** "p50" = median (50% of requests faster than this). "p95" = 95th percentile (only 1 in 20 requests slower). "p99" = the worst tail you'd typically see. **For UX, p95 < 200ms is the bar.** Anything below that feels instant in a normal browser.

---

## Test corpus

We seed **10,000 synthetic assets** into a fresh PostgreSQL database, then run the most common asset-list / search / filter queries 50 times each. The seed script ([`api/tests/scale/seed.py`](https://github.com/raguyazhin/Invenzo/blob/master/api/tests/scale/seed.py)) generates a realistic mix:

| Asset type | Share |
|---|---|
| Workstation | 40% |
| Laptop | 30% |
| Server | 10% |
| Virtual machine | 10% |
| Printer | 4% |
| Switch | 3% |
| Monitor | 2% |
| Access point | 1% |

OS distribution mirrors a real Microsoft-heavy enterprise: ~60% Windows (mix of 10/11/Server 2019/Server 2022), ~25% Linux (Ubuntu / RHEL / Debian), ~15% macOS. `last_seen_at` is randomized: 70% within 24h, 20% within a week, 10% within 30 days. Each row carries a realistic `hardware` JSONB blob (manufacturer, model, CPU, RAM, serial).

---

## Bench environment

Numbers below are from a **single-node Docker Desktop install on a developer laptop** — explicitly NOT a production-grade machine. Customer numbers on dedicated hardware will be **noticeably better**.

| | |
|---|---|
| Host | Windows 11, Docker Desktop 25.x |
| Postgres | 16 (single node, no replication, no tuning beyond defaults) |
| API | FastAPI / asyncpg, 1 worker process |
| Storage | NVMe SSD (~3.5 GB/s sequential, ~600K IOPS random) |
| Memory | 32 GB host, ~1 GB allocated to Postgres container |
| CPU | 12 vCPU available; Postgres typically uses 1 |

A production deployment with managed Postgres + tuned `shared_buffers` + dedicated CPU should beat these numbers by 2–4x.

---

## Headline number — DB query latency at 10K assets

Pure SQL latency, no API serialization. This is the floor — what the database layer delivers before any HTTP roundtrip.

```
AGGREGATE (n=450 queries across 9 query types):
    p50:     6.5ms
    p95:    18.8ms
    p99:    33.7ms
    max:    56.0ms
    mean:    8.0ms
```

**Conclusion for 10K endpoints:** every common asset-list query returns in **under 60 ms** even at the p99 tail. The DB is not the bottleneck.

---

## Per-query breakdown

| Query | What it tests | Rows returned | p50 | p95 | p99 |
|---|---|---|---|---|---|
| `count-all` | Total asset count for paginator | 1 | 4.7 ms | 7.1 ms | 11.2 ms |
| `page-1-recent-25` | Default asset list view, sorted by last seen | 25 | 9.5 ms | 14.5 ms | 17.5 ms |
| `page-100-deep-offset` | Same query at offset=2475 (paginating deep into the corpus) | 25 | 9.6 ms | 14.4 ms | 20.3 ms |
| `filter-asset-type` | `WHERE asset_type = 'laptop'` filter | 25 | 3.7 ms | 5.3 ms | 11.4 ms |
| `filter-os-ilike` | `WHERE os_name ILIKE '%Windows%'` | 25 | 13.7 ms | 21.1 ms | 22.0 ms |
| `search-q-ilike` | Text search across hostname + asset_tag | 25 | 14.7 ms | 35.2 ms | 56.0 ms |
| `hardware-jsonb-mfr` | `WHERE hardware->>'manufacturer' = 'Dell Inc.'` | 25 | 1.8 ms | 2.6 ms | 12.9 ms |
| `status-online` | `WHERE last_seen_at > NOW() - INTERVAL '15 minutes'` | 25 | 0.9 ms | 1.3 ms | 23.4 ms |
| `count-by-type` | `GROUP BY asset_type` aggregate | 8 | 6.3 ms | 22.7 ms | 30.1 ms |

### Observations

* **Pagination doesn't degrade with depth.** Page 100 is no slower than page 1 — Postgres handles `OFFSET 2475 LIMIT 25` just as efficiently as `OFFSET 0 LIMIT 25` at this corpus size.
* **`ILIKE` text search is the slowest query** (p99: 56 ms for hostname + asset_tag wildcard). Acceptable at 10K but the first to revisit when scaling past 50K. A trigram GIN index would cut this to <5 ms.
* **JSONB lookups are surprisingly fast** (`hardware->>'manufacturer'` exact-match: 1.8 ms p50). PostgreSQL's JSONB performance is excellent for our access patterns.
* **Time-range filters are fastest** (`last_seen_at > NOW() - INTERVAL`: 0.9 ms p50). The btree index on `last_seen_at` does the work.

---

## Seed throughput

The seed script inserts **10,000 rows in ~8.5 seconds** on the same hardware (≈1,170 rows/sec). Useful as a sanity check on Postgres write throughput, since it stresses the same code path the agent checkin handler hits at every endpoint registration.

```
$ docker compose exec api python -m tests.scale.seed --count 10000 --run-id bench-1
Seeding 10000 assets with run_id=bench-1 ...
  ... seeded 5000/10000 rows in 5.2s (959/s)
  ... seeded 10000/10000 rows in 8.5s (1173/s)
✓ Inserted 10000 rows in 8.5s (1173/s)
```

---

## What's NOT yet benchmarked

Honest list of what these numbers DON'T tell you:

* **End-to-end HTTP latency.** The benchmark above measures DB query cost only. The full `GET /api/v1/assets` HTTP roundtrip adds JSON serialization, permission resolution, async-DB-pool overhead, and (in production) reverse-proxy cost. Expect 2–3x the DB number as a reasonable estimate. We'll add direct HTTP measurements in a future run.
* **Concurrent users.** Single-threaded benchmark. We haven't measured what happens at 10/50/100 concurrent admins.
* **Agent checkin throughput.** A separate stress dimension. Roughly: how many endpoints can check in per second before the API starts queueing.
* **Discovery scan throughput.** /16 sweep + nmap fan-out vs Celery worker concurrency.
* **Larger scales (50K, 100K endpoints).** Planned. Single-node Postgres can almost certainly handle 50K with tuning; 100K likely needs read replicas.
* **Compliance recalc fleet-wide wall time.** Important for the daily compliance score refresh task.
* **Backup pg_dump time at scale.** Customer-relevant for RTO/RPO planning.

These will be added as we benchmark them.

---

## Run it yourself

The benchmark scripts ship with the source. Run them in your own environment:

### 1. Seed test data

```bash
# Inside the api container
docker compose exec api python -m tests.scale.seed --count 10000 --run-id my-bench
```

The `--run-id` tag makes the data trivially removable later — the seed script writes it into `hardware->>'_seed_run'` on every row.

### 2. Run the DB benchmark (no auth required)

```bash
docker compose exec api python -m tests.scale.benchmark_db --requests 50
```

### 3. Run the HTTP benchmark (requires a JWT — login user must exist)

```bash
BENCH_PASSWORD=your-admin-password docker compose exec -T api \
    python -m tests.scale.benchmark_assets_list --requests 50
```

### 4. Clean up afterward

```bash
docker compose exec api python -m tests.scale.seed --cleanup my-bench
```

This hard-deletes every row tagged with the run_id. Real assets aren't touched.

---

## Methodology notes

* **Warmup.** Each query type runs 3 warmup iterations before the timed run, so we measure steady-state with caches populated.
* **Sequential, not concurrent.** The default benchmark hits each query 50 times in series. This measures **per-request latency** (what one user sees), not **server throughput** (req/s under load).
* **Uses the same connection pool** the API uses, against the same Postgres instance. The numbers reflect production code paths — there's no special "benchmark mode."
* **Hardware JSONB queries are exact-match.** A `hardware @>` containment query (which the API doesn't use today) would be slower; we're measuring what's actually shipped.

---

## When these numbers will change

We'll re-run + republish on:
* Major Postgres version upgrade (16 → 17)
* Schema changes that touch the `assets` table (new indexes, new columns in the hot path)
* Any release that touches the asset-list query path
* Customer reports of unacceptable latency at scale we hadn't tested yet

---

[← Back to docs index](index.md) · [Pricing](PRICING.md) · [Deployment guide](DEPLOYMENT_GUIDE.md)
