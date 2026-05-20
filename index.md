---
layout: default
title: Invenzo ITAM Documentation
description: On-premises ITAM, discovery, CMDB, SAM, and reporting documentation
---

# Invenzo ITAM Documentation

Invenzo is an on-premises IT asset management platform for discovery,
inventory, CMDB relationships, software asset management, agent lifecycle,
compliance evidence, and executive reporting.

It is designed for customers who want to keep discovery data, credentials,
inventory, and reporting inside their own network instead of running ITAM as a
SaaS product.

<small>© <a href="https://mitvaris.com">Mitvaris</a>. All rights reserved.</small>

---

## Core Docs

| Guide | Use it for |
|---|---|
| [Product Guide](PRODUCT_GUIDE.md) | What Invenzo does, who it is for, major modules, packaging, and fit. |
| [Admin Guide](ADMIN_GUIDE.md) | Day-to-day use: discovery, assets, CMDB, SAM, agents, reports, users, and integrations. |
| [Deployment Guide](DEPLOYMENT_GUIDE.md) | Fresh install, embedded vs external PostgreSQL, offline install, upgrade, and uninstall. |
| [Operations Guide](OPERATIONS_GUIDE.md) | Backup, restore, HA, disaster recovery, secrets, monitoring, AV/EDR exclusions, and support checks. |
| [Validation Guide](VALIDATION_GUIDE.md) | Benchmarks, market-proof evidence, pilot validation, test coverage, and production readiness. |

---

## Fast Start

Default appliance install with bundled PostgreSQL:

```bash
git clone https://github.com/mitvaris/invenzo-package.git /tmp/invenzo-package
cd /tmp/invenzo-package
sudo bash install.sh
```

Customer-managed PostgreSQL install:

```bash
sudo bash install.sh --external-db
```

Database mode is selected by `install.sh`, not inside the browser wizard. After
containers start, open:

```text
https://<your-hostname>/install
```

The wizard validates the selected database, creates the first admin, applies the
schema, and seeds default application data.

---

## Current Product Areas

| Area | Included capabilities |
|---|---|
| Discovery | Distributed collectors, network scans, credentialed discovery, AD, SNMP, SSH, WMI/WinRM readiness, virtualization/cloud source models, troubleshooting, and discovery coverage. |
| Inventory | Hardware, OS, software, ownership, lifecycle, location, contracts, peripherals, components, and history. |
| CMDB | CI relationships, dependency and topology foundations, service mapping, network/virtualization/application map views. |
| SAM | Software installs, licenses, policies, compliance evidence, usage and entitlement foundations, and audit-ready reporting flows. |
| Agents | Endpoint telemetry, installer bundles, auto-deploy queue, rollout controls, blocked assets, self-update, and diagnostics. |
| Governance | RBAC, audit logs, evidence packs, compliance checks, executive dashboards, scheduled reporting, and market-proof workflows. |
| Integrations | SSO/LDAP, AD/Entra models, SCCM/Intune/Jamf/Tenable/Qualys/Jira/ServiceNow readiness surfaces and connector-health tracking. |

---

## Honest Evidence Policy

Invenzo should not claim market-leader parity from placeholder data. Benchmark
scores, connector health, certified SNMP packs, pilot evidence, and executive
proof packs must come from real persisted evidence or explicit proof-lab runs.

Use the [Validation Guide](VALIDATION_GUIDE.md) before publishing customer claims
about 50k/100k scale, device recognition accuracy, or competitor comparisons.

---

## Repository Model

| Repository | Audience | Purpose |
|---|---|---|
| `invenzo-package` | Customers | Public install/upgrade package. Contains scripts and compose files, not product source. |
| `Invenzo` | Developers | Private source repository for API, UI, discovery engine, agent, and docs. |
| `invenzo-docs` | Customers and operators | Public documentation generated from this folder. |
