---
layout: default
title: Invenzo ITAM — Who this isn't for
description: When Invenzo is the wrong tool. Read before evaluating.
---

# Who this isn't for

We'd rather lose a sale than win a support ticket from a wrong-fit customer. If your situation matches anything below, **please buy something else**. Most of these have a great alternative — we'll point you at it.

<small>© <a href="https://mitvaris.com">Mitvaris</a>. All rights reserved.</small>

---

## Hard disqualifiers

If any of these apply, Invenzo is the wrong tool. Don't waste your money or our time.

### 1. You operate fewer than 200 endpoints

**Why it doesn't fit:** Invenzo's depth (95-page RBAC, 6 compliance frameworks, ITIL change management, multi-stage approvals) is over-engineered for a small fleet. You'll spend more time configuring it than getting value out of it.

**Use instead:**
* **[Snipe-IT](https://snipeitapp.com/)** — free, open-source, self-hosted, perfectly fine for SMBs.
* A spreadsheet. Genuinely.

### 2. You operate more than 10,000 endpoints in one install

**Why it doesn't fit:** Invenzo is benchmarked at 1K / 10K / soon-to-be 25K endpoints in a single install. Beyond that, our published numbers run out. You'd be a lighthouse customer — exciting for us, risky for you.

**Use instead:**
* **[ServiceNow CMDB](https://www.servicenow.com/products/it-asset-management.html)** — multi-million-dollar deployment cost, but it scales.
* **[Tanium](https://www.tanium.com/)** — hyperscale endpoint platform with ITAM built-in.
* If you really want Invenzo at this scale: [contact us](https://mitvaris.com) for a sizing review. Don't just install and hope.

### 3. You want SaaS / hosted ITAM (vendor-managed)

**Why it doesn't fit:** Invenzo is **self-hosted on customer infrastructure only — and this is a durable strategic choice, not a roadmap gap.** There is no `app.invenzo.com`, there is no Mitvaris-managed cloud tier in development, and there isn't going to be one. Every customer runs their own install on their own machine (or their own cloud cluster). You install Docker, run docker-compose, and own the operations end-to-end.

If a future ask materially changes this, it will be announced on the [docs site](index.md) — not in a sales pitch. As of v1.15.6, the answer is a clear no.

**Use instead:**
* **[ServiceNow ITAM](https://www.servicenow.com/products/it-asset-management.html)**
* **[Lansweeper Cloud](https://www.lansweeper.com/cloud/)**
* **[Asset Panda](https://www.assetpanda.com/)**
* **[Freshservice](https://www.freshworks.com/freshservice/)**

### 4. You need to PUSH changes to endpoints (RMM / endpoint management)

**Why it doesn't fit:** Invenzo's agent is **read-only by design**. It collects inventory + posture; it does NOT push patches, run scripts, install software, change config, or remote-control endpoints. This is a deliberate principle, not a missing feature.

**Use instead (alone or alongside Invenzo):**
* **[NinjaOne](https://www.ninjaone.com/)** — RMM with patch management
* **[Atera](https://www.atera.com/)** — RMM + ITSM for MSPs
* **[Microsoft Intune](https://www.microsoft.com/en-us/security/business/microsoft-intune)** — Windows/macOS/iOS/Android MDM + patch
* **[ManageEngine Endpoint Central](https://www.manageengine.com/products/desktop-central/)** — patching + remote control + software deployment
* **Pair with Invenzo:** Many customers run Intune + Invenzo together. Invenzo is the source of truth + compliance evidence; Intune executes the changes.

### 5. You need 24/7 phone support / 1-hour SLA

**Why it doesn't fit:** We're a small team. Email + scheduled video call is what we offer. If your operations require around-the-clock vendor coverage, you need a Gartner-tier vendor with a global support footprint.

**Use instead:**
* **[ServiceNow](https://www.servicenow.com/)** with a partner SI for support
* **[Ivanti](https://www.ivanti.com/)** with their managed services tier
* **[BMC Helix](https://www.bmc.com/it-solutions/itsm.html)**

### 6. You're a managed service provider (MSP) wanting to host Invenzo for your customers

**Why it doesn't fit:** Invenzo is **single-tenant by design — one install per customer, on the customer's own on-prem machine or their own cloud cluster.** The data model has no tenant isolation, the secrets layer is single-tenant, and the [LICENSE](https://github.com/raguyazhin/invenzo-package/blob/main/LICENSE) explicitly forbids MSP / SaaS hosting. This is not a v1 limitation that goes away later — it's a deliberate strategic choice. We will not build multi-tenancy or a managed-hosting model.

If you're an MSP wanting to manage Invenzo installs **on each of your customers' own infrastructure** (one install per customer, you operate them remotely with a per-customer license), that's a different conversation — [contact Mitvaris](https://mitvaris.com) about a partner agreement. Same product, customer-owned infrastructure, you billed for the operations layer.

**Use instead (for shared multi-tenant hosting):**
* **[NinjaOne](https://www.ninjaone.com/)** — built for MSP multi-tenancy
* **[Atera](https://www.atera.com/)** — MSP-native pricing
* **[Auvik](https://www.auvik.com/)** — MSP-friendly network monitoring

### 7. You want zero-cost / free open-source

**Why it doesn't fit:** Invenzo is commercial software. The install package is public; the source code is closed.

**Use instead:**
* **[Snipe-IT](https://snipeitapp.com/)** — solid free option, smaller scope
* **[GLPI](https://glpi-project.org/)** — French open-source ITSM/CMDB, very capable
* **[NetBox](https://github.com/netbox-community/netbox)** — DCIM + IPAM, narrow focus
* **[OCS Inventory NG](https://ocsinventory-ng.org/)** — old but free hardware/software inventory

---

## Soft disqualifiers — proceed with eyes open

These cases CAN work but you'll feel friction. Read carefully.

### A. You don't have an in-house IT admin

Invenzo expects someone who can run `docker compose up -d`, edit a `.env` file, configure SSO, and pick up a PostgreSQL backup if it goes wrong. If your team outsources all IT, you'll need either a managed-service partner OR your VAR / consultancy to run it for you.

### B. You're cloud-only (no on-prem endpoints)

Invenzo's discovery engine is most powerful on-prem (network sweeps, WMI, SNMP). For pure-cloud (AWS / Azure / GCP) inventories, our cloud collectors work but you'll get more value from cloud-native tooling like AWS Config, Azure Resource Graph, or **[Steampipe](https://steampipe.io/)**. Invenzo + cloud-only = you're paying for half the product.

### C. You have a strong existing CMDB you can't migrate

If you already have a working ServiceNow CMDB and you're not allowed to replace it, Invenzo's role becomes "discovery + posture" — useful, but a lot of the platform's value is the CMDB layer. Consider whether the lighter discovery-only license would be a better choice.

### D. You're in active acquisition / merger turbulence

ITAM tooling decisions made during M&A get re-decided 6 months later. Wait for the dust to settle, then call us. We've watched too many pilots get killed by post-merger consolidation.

### E. Your compliance regime requires FedRAMP / SOC 2 Type II / ISO 27001 *certification of the vendor*

Invenzo helps your organization achieve those certifications by collecting and reporting on the relevant controls. **Mitvaris itself is not yet FedRAMP / SOC 2 Type II / ISO 27001 certified.** If your contracting authority requires the vendor be certified (not just the customer), you can't use us yet. Ask anyway — we'll tell you when we expect to have certifications, if ever.

### F. You want all-in-one IT management (ITSM ticketing + RMM + ITAM)

Invenzo doesn't do ticketing or RMM. Many customers pair us with **[Freshservice](https://www.freshworks.com/freshservice/)**, **[Jira Service Management](https://www.atlassian.com/software/jira/service-management)**, or **[Zendesk](https://www.zendesk.com/)** for tickets, and **[Intune](https://www.microsoft.com/en-us/security/business/microsoft-intune)** / **[NinjaOne](https://www.ninjaone.com/)** for RMM. If you wanted one product to do everything, choose **[Atera](https://www.atera.com/)** or **[Freshservice + Freshteam + RMM bundle](https://www.freshworks.com/)** instead.

---

## What if you're on the line?

We'd rather you skip the pilot than waste a quarter on a poor fit. If you're unsure:

1. Read [Who is this for?](WHO_IS_THIS_FOR.md) — the 5-question fit check is the most honest filter.
2. [Install Invenzo in a lab](DEPLOYMENT_GUIDE.md) — 30 minutes on Ubuntu 22.04. No license required for a small lab fleet. The product will show you whether it fits faster than any sales call.
3. If you still aren't sure, [email Mitvaris](https://mitvaris.com). We'll tell you honestly whether to evaluate further or pick a different tool. Selling badly fits is a worse outcome for both of us than pointing you elsewhere.

---

[← Back to docs index](index.md) · [Who is this for?](WHO_IS_THIS_FOR.md) · [Pricing](PRICING.md)
