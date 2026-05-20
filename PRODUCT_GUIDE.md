---
layout: default
title: Product Guide
description: Invenzo ITAM product scope, modules, customer fit, and packaging
---

# Product Guide

Invenzo is an on-premises ITAM platform for organizations that need discovery,
inventory, CMDB, SAM, compliance evidence, agent lifecycle, and executive
reporting without sending asset data to a SaaS tenant.

It is positioned for internal IT, infrastructure, security, compliance,
procurement, and asset teams that need one operational source for assets,
software, ownership, dependencies, and reporting.

---

## Who Invenzo Is For

Invenzo is a strong fit when:

- asset and discovery data must stay inside the customer's network
- the customer wants an appliance-style deployment they can operate
- network discovery, endpoint agents, and credentialed scans all matter
- ITAM needs to connect with CMDB, SAM, compliance, and executive reporting
- buyers need evidence, not only inventory tables

It is usually not the best fit when:

- the customer wants a pure SaaS ITAM product
- the customer does not have anyone to operate an on-premises system
- the customer only needs a small spreadsheet-like asset register
- the customer cannot provide scanning credentials, network access, or endpoint rollout support

---

## Product Modules

| Module | What it covers |
|---|---|
| Discovery Center | Source setup, scanner status, collector coverage, scan history, credential results, troubleshooting, topology, and dependency readiness. |
| Inventory | Assets, asset detail, hardware/software facts, lifecycle, assignments, locations, contracts, components, peripherals, and history. |
| CMDB | CI relationships, dependency records, service links, topology data, and visual map foundations. |
| Software Asset Management | Software inventory, license positions, compliance policies, entitlement evidence, usage signals, and audit reporting. |
| Agents | Installer bundles, endpoint telemetry, auto-deploy queue, rollouts, blocked assets, version alignment, and diagnostics. |
| Governance | RBAC, audit logs, compliance checks, evidence packs, tasks, change workflows, and executive reports. |
| Integrations | SSO, LDAP/AD, directory and device-management sources, vulnerability and ITSM connector surfaces, and connector-health evidence. |
| Market Proof | Benchmark evidence, connector parity, device recognition evidence, certified device packs, and board-ready reporting proof. |

---

## Discovery Capabilities

Invenzo discovery is designed around multiple evidence sources instead of a
single scanner result:

- distributed on-prem collectors
- subnet and source coverage tracking
- credentialed and uncredentialed scans
- SNMP, SSH, WMI/WinRM, AD, virtualization, cloud, and agent data models
- identity reconciliation using serial, BIOS UUID, MAC, FQDN, VM UUID, cloud ID, agent ID, AD SID, and weak IP evidence
- per-source and per-subnet troubleshooting explanations
- dependency and topology evidence for enterprise CMDB workflows

The discovery goal is to move from "asset list" to "known IT estate with
confidence, source history, dependencies, and explainable scan gaps."

---

## On-Premises Positioning

Invenzo should be compared against on-prem or self-hosted expectations from
tools such as Lansweeper, Device42, ManageEngine, Ivanti, Snow, Flexera, and
ServiceNow ITAM/SAM deployments.

Do not claim parity from empty readiness screens. Use persisted evidence:

- real benchmark datasets
- live connector health
- certified SNMP/OID packs from real device samples
- customer pilot discovery results
- same-network device recognition comparisons where legally and practically available
- delivered executive report packs

---

## Packaging And Licensing

The customer install package is public and contains deployment scripts, compose
files, and operational runbooks. Product source code remains private.

The application is licensed commercially. Feature access may be controlled by
licensed modules, but historical data should remain readable when a module is
paused or unlicensed.

For pricing and commercial packaging, use the current sales quote or contract as
the authority rather than copying old website values into customer proposals.

---

## Product Boundaries

Invenzo is not:

- a SaaS inventory service
- an EDR or vulnerability scanner replacement
- a SIEM
- a full ITSM replacement
- a DBA-managed PostgreSQL product
- a guarantee that every device can be scanned without credentials, firewall access, or endpoint cooperation

It can integrate with those systems and preserve their evidence inside the ITAM
and CMDB workflow.
