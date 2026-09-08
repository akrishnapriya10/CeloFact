# Procure-to-Pay Process Mining — CeloFact (Celonis Value Engineer Challenge)

Process mining analysis and Celonis application built, simulating the workflow of identifying, quantifying, and pitching process improvement opportunities in a manufacturer's Procure-to-Pay process.

## Context

**CeloFact** is a mid-sized German manufacturer of paper and packaging solutions made from recycled materials. Facing rising costs and supply chain disruption, CeloFact engaged Celonis to find and fix the biggest inefficiencies in its value chain — starting with **Procurement**.

This project simulates the full engagement: a Discovery Workshop to identify pain points, a Proof of Value (PoV) built on real transactional data, and an executive readout presenting the opportunities found.

| | |
|---|---|
| **Target process** | Procure-to-Pay (Requisition → PO → Goods Receipt → Invoice → Payment) |
| **Source system** | SAP ECC |
| **Data scope** | ~1.2M purchase order line items, 841K purchase orders, 50 vendors |
| **Platform** | Celonis Process Intelligence Platform (Process Explorer, PQL, Action Flows) |

## Pain Points Identified (Discovery Workshop)

1. **High manual effort** — a large share of activities are manual, driving errors and repeated vendor back-and-forth.
2. **Suboptimal payment terms** — many POs don't leverage optimal payment terms, hurting cash flow at a time of high interest rates.
3. **Stock-outs from late/partial deliveries** — no clear visibility into which vendors are driving delivery risk.
4. **Poor cross-team visibility** — lack of shared data makes it hard to diagnose and unblock stuck purchase orders.

## What I Built

A Celonis application with four views, moving from raw process discovery to quantified, prioritized opportunities:

### 1. Process Explorer
End-to-end variant map of the P2P process from PO creation through invoice clearing, surfacing rework loops (e.g., repeated *Change Price* / *Postponed Delivery Date* activities) and deviations from the happy path.

### 2. Executive Dashboard
Top-line KPIs and the three headline pain points sized in euros, giving leadership a one-glance view of where value is trapped.

| Metric | Value |
|---|---|
| Net Order Value | €2.27B |
| Total Purchase Orders | 841K |
| Order Items | 4.34M |
| Total Vendors | 50 |
| **Maverick spending** | **€1.67B** (635K POs) |
| **Manual activities** | **€2.87M** (277K POs) |
| **Order value exposed to delivery risk** | **€364M** (113K POs) |

### 3. Opportunity Deep Dive
Activity-level breakdown of occurrence counts vs. automation rate, plus vendor-level cuts (partial delivery volume, total PO volume by vendor) to pinpoint which vendors and activities drive the most cost.

### 4. MVP1: AI Findings
A first pass at AI-assisted analysis on invoice payment behavior — flagging early/late/on-time clearing patterns and payment date deviation, split by contract vs. no-contract usage.

> Screenshots of each view are in [`assets/`](./assets) — see [Screenshots](#screenshots) below.

## Opportunities Quantified

Two opportunities were sized and packaged as one-pagers (challenge → root cause → solution → tracking KPIs → value):

### Opportunity 1 — Resolve Contract Non-Compliance (Maverick Spending)
- **Challenge:** 75% of PO value is spent outside approved contracts (contract usage at 25% vs. a 90%+ best-in-class benchmark).
- **Root cause:** Free-text requisitions bypass contracts 94% of the time; even proper purchase requisitions bypass contracts 65% of the time.
- **Solution:** Real-time Celonis Action Flow flagging any no-contract PO above a value threshold for approval; enforce catalog-based requisitioning to cut free-text entry.
- **Value:** €1.67B in no-contract spend → **€83.3M potential savings**, ~5% conservatively recoverable per Hackett Group benchmarks for maverick buying.

### Opportunity 2 — Resolve Manual Rework
- **Challenge:** 156K "Change Price" events with 0% automation; no-contract cases show a 24% rework rate vs. 4% for contract cases.
- **Root cause:** Free-text entry skips upfront contract/price validation, pushing rework downstream.
- **Solution:** Auto-validate pricing against contract terms at PO creation; automate the highest-volume, lowest-automation manual steps.
- **Value:** 277K cases with manual rework × ~20 min at €31/hr (German procurement analyst average) → **€2.87M potential savings**.

## Recommended Roadmap

| Phase | Timeline | Focus |
|---|---|---|
| **1. Stop the Bleeding** | 0–3 months | Automate highest-volume manual activities (e.g., Change Price); deploy real-time no-contract PO flagging |
| **2. Close Structural Gaps** | 3–9 months | Renegotiate/extend contract coverage for high-volume materials; enforce catalog/structured requisitioning above a value threshold; activate invoice value verification |
| **3. Govern** | 9+ months | Vendor performance review and SLA enforcement; live Celonis monitoring dashboard as a permanent governance layer; quarterly compliance reviews |

**Projected trajectory:**

| | Today | Phase 1 | Phase 2 | Phase 3 |
|---|---|---|---|---|
| Contract Compliance | 25% | 35–40% | 55–60% | 70–75% |
| Automation Rate | 74% | 76% | 78–80% | 80–85% |

## Deliverables

- [`Solution Blueprint.pptx`](./docs/Solution%20Blueprint.pptx) — the problem statement: company background, strategic alignment, pain points, and PoV scope agreed at Discovery.
- [`Final Presentation.pptx`](./docs/Final%20Presentation.pptx) — the executive readout: platform intro, opportunity sizing, roadmap.
- Celonis app (Process Explorer, Executive Dashboard, Deep Dive, MVP1: AI Findings) — see screenshots below.

## Screenshots

| | |
|---|---|
| <img width="1440" height="671" alt="image" src="https://github.com/user-attachments/assets/c442c626-0c34-40c5-9073-24824cabc03b" />
| <img width="1440" height="670" alt="image" src="https://github.com/user-attachments/assets/259c9a39-4a54-4e73-abd1-4bab832ecbae" />
 |
| *Process Explorer — full P2P variant map* | *Executive Dashboard — top-line KPIs and pain points* |
| ![Opportunity Deep Dive](./assets/opportunity-deep-dive.png) | ![Maverick Spending Deep Dive](./assets/maverick-spending-deep-dive.png) |
| *Opportunity Deep Dive — activity & vendor breakdown* | *Deep Dive — maverick spending by vendor and requisition type* |
| ![MVP1 AI Findings](./assets/ai-findings.png) | |
| *MVP1: AI Findings — payment deviation analysis* | |

## KPI Definitions (PQL)

The data model (tables, relationships, activity table) was provided; the KPI logic below is what I authored on top of it.

**Order Value Exposed to Late Delivery** — sums PO value for any case with at least one `Partial Delivery` activity:
```pql
SUM(CASE
  WHEN PU_COUNT("PurchaseOrders",
    CASE WHEN "ACTIVITY_TABLE"."ACTIVITY_EN" = 'Partial Delivery' THEN 1 END
  ) > 0
  THEN "PurchaseOrders"."NetOrderValue"
  ELSE NULL
END
)
```

**Manual Rework Cost / Potential Savings** — counts cases with at least one manual, rework-type activity (price/date/quantity changes, free-text requisitions, order confirmation chasing, cancellations, expediting) and converts to cost at 20 minutes per case and a €31/hr procurement analyst rate:
```pql
COUNT(DISTINCT CASE WHEN
  PU_COUNT("PurchaseOrders",
    CASE WHEN "ACTIVITY_TABLE"."ACTIVITY_EN" IN (
      'Change Price', 'Change Requested Date' , 'Change Quantity', 'Create Free-Text Requisition Item'
      , 'Dun Order Confirmation' , 'Refuse Purchase Order' , 'Cancel Good Receipt'
      , 'Send Express Purchase Order' ,'Receive Order Confirmation' , 'Postponed Delivery Date'
    ) AND "ACTIVITY_TABLE"."USER_TYPE" = 'Manual' THEN 1 END
  ) > 0
THEN "PurchaseOrders"."_CASE_KEY" END) * (20.0/60.0) * 31
```
*(Used for both the "Manual Activities" savings figure on the Executive Dashboard and the "Rework Cost" figure in the Opportunity Deep Dive — same rework-activity list and cost basis, applied at each view's grain.)*

**Contract Non-Compliance Rate** — share of cases using no approved contract:
```pql
COUNT(DISTINCT CASE WHEN "PurchaseOrders"."ContractUsage" = 'NoContract'
THEN "PurchaseOrders"."_CASE_KEY" END)
/
COUNT(DISTINCT "PurchaseOrders"."_CASE_KEY")
```

**No-Contract (Maverick) Spend Value:**
```pql
SUM(CASE
  WHEN "PurchaseOrders"."ContractUsage" = 'NoContract'
  THEN "PurchaseOrders"."NetOrderValue"
  ELSE NULL
END
)
```

**Order Value Paid Late** — sums PO value where actual payment deviates positively from terms:
```pql
SUM(CASE WHEN "PurchaseOrders"."payment_deviation" > 0
THEN "PurchaseOrders"."NetOrderValue" END)
```

**% Cleared (MVP1: AI Findings)** — compares an AI-extracted invoice value against the PO's net order value, using a document data extraction table (`DOC_EXTRACTION`) built to pull invoice amounts out of unstructured documents:
```pql
TO_FLOAT("DOC_EXTRACTION_KRISHNAPRIYA_A"."invoice_value") / "PurchaseOrders"."NetOrderValue"
```

## Skills Applied

- Process mining & root cause analysis (Celonis Process Explorer)
- KPI authoring in PQL on top of a provided data model
- AI-assisted document data extraction (invoice value extraction feeding the MVP1 findings)
- Business case quantification (savings modeling, benchmark research)
- Executive dashboarding and stakeholder-ready storytelling (problem → root cause → solution → value)

---
*Built as a simulated Value Engineer case study; CeloFact is a fictional company used for the exercise.*
