---
layout: default
title: Admin Guide
description: Daily administration for Invenzo ITAM
---

# Admin Guide

This guide is for Invenzo administrators and operators after the platform is
installed.

Use it for discovery setup, asset operations, CMDB relationships, SAM workflows,
agent lifecycle, reports, user access, and integrations.

---

## First Admin Checklist

1. Complete `/install` and create the first admin.
2. Confirm **Settings > Version & Schema** shows the application and DB schema in sync.
3. Configure allowed host, callback URL, SMTP/notifications, and backup settings.
4. Create roles or teams before inviting operators.
5. Add discovery credentials and test each source in a small scope.
6. Configure at least one recurring backup and one restore drill.
7. Deploy agents to a pilot group before broad rollout.

---

## Navigation Model

The sidebar is organized around ITAM operating areas:

| Area | Typical pages |
|---|---|
| Command / Executive | Dashboards, ITAM command center, executive reports, market proof. |
| Discovery | Discovery center, network scanner, AD, VMware/Hyper-V, cloud, MDM, discovery rules. |
| Inventory / CMDB | Assets, relationships, topology, locations, assignments, components. |
| Software / SAM | Software inventory, licenses, policies, compliance, usage and audit evidence. |
| Agents | Agent fleet, auto-deploy queue, rollouts, blocked assets, diagnostics. |
| Governance | Tasks, change workflows, audit logs, compliance, reporting, settings. |

Pages should stay operational and scanner-first. Heavy readiness panels should
not bury the workflow a user came to operate.

---

## Discovery Operations

Start with a controlled pilot:

1. Define in-scope subnets and exclusions.
2. Add credentials by platform: Windows/WMI/WinRM, Linux/macOS SSH, SNMP, AD, virtualization, and cloud as applicable.
3. Assign collectors to routed network segments.
4. Run a small scan and review discovery explanations before scanning broadly.
5. Fix credential, firewall, DNS, and subnet coverage issues.
6. Schedule recurring scans after the first successful baseline.

The Discovery Center should show:

- sources configured and active
- scans running, completed, and failed
- assets discovered
- credential successes and failures
- unreachable IPs
- stale assets
- duplicate candidates
- scan duration and last successful source per asset
- troubleshooting reasons by source, collector, subnet, and asset

---

## Asset Identity And Reconciliation

Invenzo should merge evidence using strong identifiers before weak network
signals:

| Evidence | Strength |
|---|---|
| Serial number, BIOS UUID, VM UUID, cloud instance ID, agent ID, AD computer SID | Strong |
| MAC addresses, hostname/FQDN | Medium to strong depending on source quality |
| Primary IP address | Weak; useful only with supporting evidence |

Every asset should maintain discovery sources, last-seen-by-source, and a
confidence score so operators can explain why an asset exists or why records
merged.

---

## Inventory Workflow

Use Inventory for the current operational state:

- identity, hostname, FQDN, IP, MAC, serial, manufacturer, model
- OS, CPU, memory, disks, peripherals, components
- installed software and licenses
- contracts, warranty, owner, department, environment, lifecycle state
- location and assignment
- tasks, history, relationships, and service context

When data is wrong, prefer correcting the source or reconciliation evidence over
manually editing a value that will be overwritten by the next scan.

---

## CMDB And Relationship Mapping

Use CMDB relationships to convert inventory into service context.

Supported relationship patterns include:

- `runs_on`
- `hosted_on`
- `connected_to`
- `depends_on`
- `installed_on`
- `member_of`
- `assigned_to`
- `uses_database`
- `exposes_service`
- `backed_by_storage`
- `network_attached_to`

Start practical: map hosts, VMs, installed software, listening ports, active
connections, databases, web/app servers, and service groups. Add passive flow
evidence later when available.

---

## SAM Workflow

SAM should be evidence-driven:

1. Normalize discovered software titles and publishers.
2. Define license entitlements, editions, downgrade rights, bundles, and audit notes.
3. Compare installs and usage against entitlement.
4. Flag risky software, unlicensed installs, over-allocation, and stale usage.
5. Export audit evidence from reports instead of relying on screenshots.

Strong SAM requires real entitlement data from procurement, contracts, vendor
portals, or customer-provided license statements.

---

## Agent Lifecycle

Use agents for endpoint facts that network scans cannot collect reliably:

- hardware and OS telemetry
- installed software
- health and performance facts
- security posture and patch signals
- local user/device context

Recommended rollout:

1. Generate installer bundle from the Agents page.
2. Pilot on a small Windows/Linux/macOS group.
3. Confirm callback URL, registration, and check-in.
4. Enable auto-deploy only after credentials and firewall access are proven.
5. Use rollouts for version upgrades and watch blocked assets.
6. Treat repeated install failures as credential, network, AV/EDR, or OS-policy issues until proven otherwise.

Agents should self-update when the server publishes a newer compatible version,
but administrators should still monitor outdated and blocked endpoints.

---

## Users, RBAC, And Audit

Use least privilege:

- admins manage platform configuration and security
- discovery operators manage sources and scans
- SAM operators manage software and license evidence
- asset operators maintain lifecycle and ownership
- executives view dashboards and reports
- auditors view evidence and audit logs

All sensitive actions should be visible in audit logs, including login,
configuration changes, credential updates, exports, report delivery, backup
actions, and lifecycle changes.

---

## Integrations

Common enterprise integrations:

- SSO/OIDC/SAML with Google, Microsoft Entra ID, Okta, ADFS, PingFederate, or similar providers
- LDAP/Active Directory for identity and directory context
- SCCM/ConfigMgr, Intune, Jamf for endpoint/device-management evidence
- Tenable/Qualys for vulnerability context
- Jira/ServiceNow for ITSM workflows
- SMTP/webhooks for notifications and report delivery

Connector health should be based on live configuration and last successful sync,
not only a catalog entry.

---

## Reporting

Use reports for repeatable evidence:

- executive ITAM scorecards
- asset and software inventory
- license compliance
- stale assets and discovery coverage
- warranty/contracts
- agent fleet health
- audit and compliance evidence
- market-proof benchmark and recognition evidence

Reports intended for boards or auditors should have delivery history, export
evidence, source timestamps, and no hidden failed checks.
