---
layout: default
title: Validation Guide
description: Benchmarks, evidence, test coverage, and production readiness
---

# Validation Guide

This guide explains how to prove Invenzo is ready for a customer environment and
how to avoid unsupported market-leader claims.

Invenzo should never claim production scale, connector parity, recognition
accuracy, or executive-report readiness from placeholder data.

---

## Evidence Standards

Use real persisted evidence for:

- 50k and 100k asset benchmark results
- customer pilot discovery datasets
- same-network device recognition comparison
- live connector health
- certified SNMP/OID/vendor packs
- board-ready report delivery
- SAM/license compliance results
- backup and restore proof

Seeded proof-lab data is useful for demos, but label it clearly. Do not present
it as customer evidence.

---

## Production Readiness Gates

| Gate | Target evidence |
|---|---|
| Installation | Fresh install and upgrade complete without manual DB edits. |
| Database | Embedded and external PostgreSQL paths tested. |
| Discovery | Subnets, collectors, credentials, stale assets, duplicates, and troubleshooting visible. |
| Recognition | Device/vendor recognition measured against known ground truth. |
| Scale | 50k/100k asset benchmark run with scan, import, query, report, and UI timings. |
| SAM | Entitlement, install, usage, edition, downgrade, bundle, and audit evidence tested. |
| Integrations | Live connector configs tested for the advertised vendors. |
| Reporting | Executive reports delivered on schedule with history and export proof. |
| Backup | Restore drill completed and documented. |
| Security | RBAC, audit, SSO, secrets, and agent update flows tested. |

---

## Benchmark Method

A benchmark should record:

- environment details: CPU, RAM, disk, Docker version, DB mode
- app version and schema version
- asset count and software/install count
- discovery source mix
- data generation or import method
- ingestion duration
- query latency for key pages
- report generation duration
- background queue depth
- error count
- screenshots or exported evidence pack

For large-enterprise proof, run at least:

- 50k assets
- 100k assets
- realistic software install distribution
- realistic subnet/source distribution
- duplicate and stale asset scenarios
- agent and agentless evidence mixed together

---

## Device Recognition Validation

Recognition claims should be measured against a known baseline:

1. scan the same network scope with Invenzo
2. collect ground truth from customer inventory, device labels, SNMP, agent data, or existing tool exports
3. compare vendor, model, serial, device type, OS, and network identity
4. record false positives, false negatives, and unknown devices
5. store evidence with timestamp, scope, and version

Same-network comparison against tools such as Lansweeper should be performed
only when legally and practically allowed by the customer.

---

## Connector Validation

Connector readiness should show live health, not just a catalog card.

For each connector:

- configuration exists
- credentials are stored securely
- test connection has a timestamp
- last successful sync is visible
- last failure is visible
- imported object counts are visible
- retry and troubleshooting guidance is visible

Priority connector categories:

- Microsoft SCCM / ConfigMgr
- Microsoft Intune
- Jamf Pro
- Tenable / Nessus
- Qualys VMDR
- Jira / Jira Service Management
- ServiceNow
- Active Directory
- Microsoft Entra ID

---

## SNMP/OID Pack Certification

Certified device packs should be based on real samples:

- vendor and model
- firmware version
- sysObjectID
- interface inventory
- LLDP/CDP neighbors
- switch MAC table
- ARP table
- VLANs
- serial number
- port status
- sample collection date

Do not mark a vendor pack certified until at least one real sample has been
collected, parsed, and regression-tested.

---

## Executive Reporting Proof

Board-ready reporting should prove:

- scheduled delivery works
- PDF/Excel exports are generated
- failed checks are visible
- source timestamps are included
- report history is retained
- evidence can be reproduced

Useful executive packs:

- estate summary
- discovery coverage
- stale/unknown assets
- license compliance
- risky software
- support/warranty exposure
- agent health
- compliance trend
- market-proof benchmark summary

---

## Test Coverage Checklist

Before shipping a major release, verify:

- login, logout, MFA/SSO where enabled
- RBAC permission boundaries
- asset list and asset detail
- discovery source CRUD and scan flow
- discovery troubleshooting
- agent install, check-in, update, block, and uninstall paths
- SAM imports, policy checks, license positions, and reports
- CMDB relationship creation and map rendering
- backup, restore, and restore drill evidence
- update flow and rollback path
- audit log coverage
- scheduled reports and notifications
- external PostgreSQL preflight if the release touches DB/config

---

## Scoring Guidance

Use scores as an internal readiness signal, not as a substitute for proof.

| Score | Meaning |
|---|---|
| 6-7 | Strong product foundation, but proof or scale gaps remain. |
| 8-8.5 | Serious on-prem ITAM alternative with visible enterprise workflows. |
| 9-9.5 | Production evidence exists for scale, connectors, recognition, SAM, and reports. |
| 10 | Repeatable customer proof across target market segments, with live integrations and audited evidence. |

The path to 9.5+ is mostly evidence and reliability, not more placeholder
screens.
