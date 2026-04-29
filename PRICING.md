---
layout: default
title: Invenzo ITAM — Pricing
description: Annual subscription pricing for Invenzo ITAM. Three tiers, transparent, on the page.
---

# Pricing

Invenzo is sold as an **annual subscription**. Prices are listed below — no "contact sales" runaround. If your environment is unusual ([government contracting](#government--enterprise-procurement), [air-gapped install](#air-gapped--offline-installations), or [unusually large](#extra-large-fleets)) we'll talk; otherwise pick a tier and install.

<small>© <a href="https://mitvaris.com">Mitvaris</a>. All rights reserved.</small>

> **Important:** Invenzo is not for everyone. Read [Who is this for?](WHO_IS_THIS_FOR.md) before evaluating. If your situation matches anything in [Who this isn't for](WHO_THIS_ISNT_FOR.md), please don't buy.

---

## The three tiers

| | **Starter** | **Professional** | **Enterprise** |
|---|---|---|---|
| | $4,800 / year | $14,400 / year | $36,000 / year |
| | $400/mo equivalent | $1,200/mo equivalent | $3,000/mo equivalent |
| **Endpoints** | up to 500 | up to 2,000 | unlimited within a single install |
| **Sites** | 1 | up to 5 | unlimited |
| **Discovery modules** | Linux · Windows · macOS · Network devices (SNMP) | Above + AWS · Azure · GCP · Active Directory · VMware | Above + Intune · Jamf · SCCM · Hyper-V |
| **Compliance frameworks** | All 6 (HIPAA · SOC 2 · PCI-DSS · ISO 27001 · NIST CSF · CIS Controls) | All 6 + custom controls | All 6 + custom controls + framework consulting (4 hrs) |
| **SAM (Software Asset Management)** | Inventory only | Inventory + reconciliation + entitlements | Inventory + reconciliation + entitlements + SaaS subscription tracking |
| **Change Management** | Basic | Enterprise (multi-stage approvals · CAB · templates · blackouts · PIR) | Enterprise + ServiceNow bidirectional sync |
| **RBAC** | Built-in roles | Built-in + custom roles | Built-in + custom roles + per-team workspaces |
| **CVE / vulnerability matching** | Built-in DB only | Built-in + custom CVE definitions | Above + monthly threat intel feed |
| **Backup & restore** | Manual | Scheduled + manual | Scheduled + manual + cross-site replication guidance |
| **Updates** | Quarterly bundle | Monthly | Monthly + early-access channel |
| **Support** | Email, 2 business day response | Email, 1 business day response | Email + scheduled video call, same business day |
| **Onboarding** | Self-serve via [docs](DEPLOYMENT_GUIDE.md) | 1 hr kickoff call + setup review | 4 hr guided install + first-month-check-in |
| **Reference HA architecture review** | — | — | Included |
| **Custom roadmap input** | — | Quarterly survey | Direct line to Mitvaris (the developer) |

All prices are USD, annual, paid in advance via wire / ACH / SEPA. Add 18% GST for Indian invoices.

---

## What's the same across every tier

The product is the same product. We don't gate features behind tiers because you're "small."

* All 95 RBAC pages
* All 6 compliance frameworks
* Asset history + audit log
* Self-hosted, your-infrastructure, your-data
* Encrypted secrets at rest (age + tmpfs)
* Ed25519-signed module licenses
* Optional TPM key binding
* Webhook delivery, SSO (SAML / OIDC), SCIM-style user provisioning via your IdP
* Asset auto-deploy via SSH / SMB
* Agent self-update + fleet rollouts
* Backup / restore / disaster recovery scripts

What changes between tiers: scale ceilings, optional integrations, support tier, onboarding hand-holding.

---

## What's NOT in any tier

We're transparent about what we don't ship — so you don't pay for something we can't deliver.

* **No managed / hosted Invenzo — ever.** Self-hosted on your infrastructure only. There will not be a `app.invenzo.com`. This is a deliberate, durable strategic choice — not a roadmap item we haven't gotten to. If you want SaaS ITAM, look at ServiceNow / Lansweeper / Asset Panda. If a future ask materially changes this, you'll see it on the [docs site](index.md) — not in a sales call.
* **No multi-tenancy — ever.** One install = one customer = one on-prem machine (or one customer-owned cloud cluster). The data model has no tenant isolation, the secret-management layer is single-tenant, and the LICENSE explicitly forbids resale / MSP hosting. This too is durable — multi-tenancy is not on the roadmap.
* **No 24/7 support phone line.** We're a small shop; we ship great software, not 3-shift call centers.
* **No "all-you-can-eat" RMM.** We discover, inventory, govern. We don't push patches, run scripts, or remote into endpoints. Pair us with your existing patching/RMM tool.
* **No sub-200-endpoint pricing.** If you operate fewer than 200 endpoints, you don't need this much product. Use Snipe-IT (free) or a spreadsheet.

---

## Special situations

### Government & enterprise procurement

If your procurement office requires:
* MSAs with custom indemnification
* SOC 2 Type II report (we can share what we have)
* Net-90 payment terms
* GSA / framework-contract pricing
* Cosigned NDA before pilot

…[contact Mitvaris](https://mitvaris.com). We do this. Expect 2–6 weeks added to the timeline.

### Air-gapped / offline installations

Standard install needs internet only for:
1. Pulling Docker images from Docker Hub
2. Optional license sync (can be deferred to monthly USB transfer)

For fully air-gapped: same Enterprise price + a one-time $2,000 setup fee covers offline image bundle, signed license file, and on-call support during install.

### Extra-large fleets

Above 10,000 endpoints in a single install is past our published benchmark ceiling. We've run pilots up to ~25K but don't list it in the table because we haven't published numbers we'd stand behind for that scale yet. If that's you: [contact Mitvaris](https://mitvaris.com), we'll do a sizing review. Pricing in this range is custom and likely starts around $60K/yr.

### Multi-year commitments

* 2-year prepay: 8% off
* 3-year prepay: 12% off

We'd rather give you a discount than a cancellation clause.

---

## What you actually pay for

Honest version: you're paying for **someone else** to maintain this code. That someone is one person (Mitvaris) plus contractors, not a 200-person engineering org. We make the trade explicit:

* You get a product that's noticeably above the OSS alternatives in feature depth and polish.
* You get a focused roadmap (we don't chase 10 directions; see [Who is this for?](WHO_IS_THIS_FOR.md)).
* You DON'T get a 3-shift support center, a Gartner Magic Quadrant placement, or "global PSO" hours.

If "huge vendor with deep bench" is on your evaluation criteria as a hard requirement, choose ServiceNow. We're the deliberate alternative for buyers who'd rather pay 1/10th and own the relationship with the people writing the code.

---

## How to buy

1. [Read the install guide](DEPLOYMENT_GUIDE.md) — install Invenzo in your environment first. Pilot freely; license enforcement on free tier doesn't expire on day 1.
2. [Contact Mitvaris](https://mitvaris.com) with: your endpoint count, your industry, your timeline, your tier choice.
3. We send a 2-page MSA + invoice. Annual term. Wire / ACH / SEPA.
4. We issue an Ed25519-signed license file scoped to your endpoint count and modules.
5. You upload the license file in Settings → Licensing. Modules activate on save.

No demo gating. No sales-engineering hand-off. No multi-week "discovery" call sequence. You either fit the [ICP](WHO_IS_THIS_FOR.md) or you don't, and a 30-minute install will tell you faster than any pitch deck.

---

## Frequently asked

**Is the price per-endpoint?**
No, it's per-tier with an endpoint ceiling. If you outgrow your tier mid-year, we charge prorated to upgrade you.

**What happens if my license expires?**
The Invenzo install keeps running. Modules using paid features will display a banner asking you to renew. No data is deleted, no rows go missing — we're not in the business of holding your inventory hostage.

**Is the source code open?**
No. Invenzo is commercial software under a [restrictive license](https://github.com/raguyazhin/invenzo-package/blob/main/LICENSE). The customer install bundle is open, the source is not.

**Can I see the source as a customer?**
On Enterprise tier with NDA, yes. On Starter / Professional, no.

**Can MSPs resell Invenzo to their customers?**
Not under standard tiers. Contact Mitvaris for an MSP partner agreement.

**Refunds?**
30-day pro-rated refund if Invenzo doesn't work in your environment. After 30 days, no refunds — you've had the product, the license file is yours.

---

[← Back to docs index](index.md) · [Who is this for?](WHO_IS_THIS_FOR.md) · [Who this isn't for](WHO_THIS_ISNT_FOR.md)
