---
case: Meridian Retail Group — Click & Collect
feature: Cross-channel Click & Collect (Phase 1)
date: 2026-07-20
author: Ernesto Bañuelos
source_artefact: artefacts/600-wide/03-defects.md
execution_context: Simulated QA Stub — Meridian Reference Case. No live implementation, no production telemetry.
---

# Root Cause Analysis

## 1. Defect Summary

**Defect:** DEF-004 — When SAP confirms zero stock at pickup, the order status is marked "Collected" in error and loyalty points are credited despite the customer leaving the store without the item.

**Observed behaviour (from 03-defects.md):** During simulated execution of TC-03C against test data record EC-006 (CUST-IT-00782 / ORD-IT-2026-007824 / STORE-IT-NAP-001), the POS received a zero-stock response from SAP for the reserved SKU. Despite this, the POS completed the handover workflow and set the order status to "Collected". Loyalty points for the full order value were simultaneously posted to the customer's account. The customer did not receive the item.

**Why this represents the highest business risk:** This defect is the direct realisation of Risk 1 identified in `00-test-plan.md` — phantom-stock cancellation at pickup — which is already occurring at an estimated rate of ~7% of Click & Collect orders. DEF-004 worsens that scenario materially: not only does the customer leave the store empty-handed, but Meridian's own records assert the item was delivered and loyalty points are disbursed against a fulfilment that never occurred. This creates three simultaneous exposures — a financial discrepancy between payment capture and goods delivered, incorrect loyalty-programme liability, and a misleading order history that complicates any subsequent customer-service or refund interaction. It is rated Severity 1 and Priority 1 in `03-defects.md`, and the exit criterion in `00-test-plan.md` mandates zero Priority-1 phantom-stock failures before sign-off.

---

## 2. Root Cause Hypotheses

### Hypothesis A — The POS pickup workflow treats the SAP inventory-validation response as advisory rather than as a hard gate

**Condition:** The POS workflow is designed to proceed to order completion regardless of the value returned by the SAP inventory check. The zero-stock result may be logged but does not interrupt the state transition to "Collected".

**Evidence that would confirm it:** Reviewing the POS pickup workflow specification or state-machine definition would show that the inventory-validation step is marked as non-blocking, or that the state transition to "Collected" is triggered by the QR-code scan event rather than by a validated inventory-positive response.

**Evidence that would refute it:** Workflow documentation or a configuration record showing that an explicit inventory-positive assertion is required as a precondition for the "Collected" state transition, and that the zero-stock result in EC-006 was never actually delivered to the POS decision point.

---

### Hypothesis B — The loyalty points engine subscribes to an order-status event without validating the inventory outcome that caused the status change

**Condition:** The loyalty points credit is triggered by an "order status → Collected" event emitted after the QR scan. The points engine does not inspect whether the underlying inventory validation confirmed available stock; it reacts solely to the status change.

**Evidence that would confirm it:** The loyalty points posting history for ORD-IT-2026-007824 shows a credit timestamped within seconds of the "Collected" status transition, and there is no record of an inventory-confirmation token in the event payload consumed by the points engine.

**Evidence that would refute it:** The event payload consumed by the loyalty points engine includes an explicit inventory-confirmation field, and the points credit was triggered only after that field indicated a positive stock result.

---

### Hypothesis C — A race condition between the SAP inventory response and the POS state transition allows the "Collected" status to be written before the zero-stock result is processed

**Condition:** The POS initiates the SAP inventory check and the "Collected" state transition in parallel or on a short timeout. When SAP responds with zero stock, the status transition has already been committed and cannot be rolled back by the inventory-check result.

**Evidence that would confirm it:** Execution timing records (if available in a live environment) would show the "Collected" status written within milliseconds of the QR scan, and the SAP zero-stock response arriving after that write. The simulated stub in EC-006 does not include timing data, so this is a hypothesis only.

**Evidence that would refute it:** Documentation showing that the POS explicitly awaits the SAP inventory response before committing any state change, with a defined timeout that reverts to an error state rather than a "Collected" state.

---

### Hypothesis D — The SAP sandbox is returning a zero-stock response that the POS interprets as a connection error rather than a definitive stock-absence signal, causing a fail-open behaviour

**Condition:** The POS error-handling logic does not distinguish between a definitive zero-stock response and a communication failure. Both conditions cause the POS to default to "allow pickup" in order to avoid blocking customers when SAP is temporarily unavailable.

**Evidence that would confirm it:** The POS error-handling specification shows a single catch-all path for any non-positive SAP response (including zero stock, timeout, and network error) that defaults to completing the pickup. The simulated execution in EC-006 produced the same outcome as the SAP-timeout scenario (DEF-004 and TC-03C share the same observed result).

**Evidence that would refute it:** The POS specification distinguishes between a "SAP returned zero stock" response and a "SAP did not respond" condition, routing each to a different outcome, and the simulated execution of TC-03C (timeout) produced a different POS behaviour from EC-006 (zero stock).

---

### Hypothesis E — The order-status write and the loyalty-points credit share a single transactional boundary that does not include the inventory-validation result, so both succeed or fail together independent of stock state

**Condition:** The final step of the POS pickup workflow bundles the order-status update and the loyalty-points credit trigger into a single operation. Because the inventory check is a separate upstream call and its result is not passed into this bundle, the bundle executes successfully regardless of the inventory outcome.

**Evidence that would confirm it:** The pickup completion operation in the POS workflow specification shows a single commit containing both the status update and the loyalty-credit trigger, with the inventory-check result stored only as a log entry rather than as an input parameter.

**Evidence that would refute it:** The pickup completion operation specification shows inventory-confirmation as a required parameter to the commit, meaning a zero or absent inventory result would prevent the bundle from executing.

---

## 3. Selected Root Cause

Hypothesis A is the strongest because it accounts for all observed symptoms — the "Collected" status, the loyalty points credit, and the absence of any error message to the associate — with a single structural condition, and it is directly supported by the observed behaviour in EC-006 where the POS gave no indication of having received the zero-stock result.

**The condition that made this defect possible was** that the POS pickup workflow transitions the order to "Collected" and emits the loyalty-credit event in response to a successful QR-code scan, without requiring a confirmed non-zero inventory result from SAP as a hard precondition for that transition.

---

## 4. Guard Test

### TC-RCA-001 — POS must not complete pickup when SAP returns zero stock for a multi-unit reservation at a UK store using Klarna

- **Category:** Regression
- **Priority:** P1 — Blocks Italy pilot
- **In-Scope Surface:** SAP inventory validation during pickup confirmation

**Preconditions:**
- Customer CUST-GB-00934 holds a confirmed Click & Collect reservation for two units at STORE-GB-BIR-003, paid via Klarna Split-Pay (order ORD-GB-2026-009341, QR token QRT-7A1C4D-MAX, from test data record EC-008).
- SAP sandbox is reconfigured so that the inventory count for the reserved SKU at STORE-GB-BIR-003 is zero at the moment of pickup confirmation (overriding the EC-008 value of 55 units).
- Store associate is at the POS terminal and the Klarna Split-Pay authorisation is in an "authorised" state.

**Steps:**
1. Store associate scans QR token QRT-7A1C4D-MAX at the STORE-GB-BIR-003 POS terminal.
2. POS sends an inventory validation request to SAP; SAP returns a zero-stock response for the reserved SKU.
3. Observe the order status in the Meridian web account for ORD-GB-2026-009341.
4. Observe the loyalty points history page for loyalty number MLY-GB-29056.

**Expected Result:**
The POS halts the pickup workflow and displays a clear cancellation notice to the associate. The order status remains "Ready for Collection". No loyalty points are posted to MLY-GB-29056. The Klarna Split-Pay authorisation is not marked as settled. The associate is presented with an escalation path.

---

## 5. Recommended Fix

The engineering team should introduce a mandatory inventory-confirmation gate as the final precondition for the POS state-transition commit: the "Collected" status write and the loyalty-credit event must be placed inside a single conditional block that executes only when the SAP inventory-validation response explicitly confirms a stock quantity greater than zero; any other result — including zero stock, a null response, or a timeout — must route the workflow to a clearly labelled "Pickup Blocked" terminal state that preserves the reservation, suppresses the loyalty-credit trigger, and surfaces a structured escalation notice to the store associate. This gate makes the inventory-confirmation result a first-class input to the commit operation rather than a side-channel log entry, eliminating the fail-open condition identified in Hypothesis A and, as a secondary benefit, closing the fail-open path described in Hypothesis D.
