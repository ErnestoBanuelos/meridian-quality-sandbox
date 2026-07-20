---
case: Meridian Retail Group — Click & Collect
feature: Cross-channel Click & Collect (Phase 1)
date: 2026-07-20
author: Ernesto Bañuelos
execution_context: Simulated QA Stub — Meridian Reference Case. All metrics derived from artefacts 00 through 04. No production execution.
---

# Test Report — Cross-channel Click & Collect (Phase 1)

## 1. Coverage

Testing covered all five in-scope surfaces defined in `00-test-plan.md`: the web cart reservation flow, customer identity stitch, SAP inventory validation during pickup confirmation, cross-region loyalty points credit, and POS QR-code pickup confirmation. The ten highest-priority test cases from `01-test-cases.md` were executed manually against a Simulated QA Stub using the synthetic records from `02-test-data.json`. Two areas were intentionally excluded per the test plan: SAP ECC inventory ground-truth correctness (owned by Finance and covered by their own controls) and legacy Shopify storefronts being retired during Phase 1.

---

## 2. Pass Rate and Defect Density

- **Executed cases:** 10
- **Passed:** 4
- **Failed:** 5
- **Story (not reproducible):** 1
- **Pass rate:** 40% (4 / 10)

The 40% pass rate does not meet the exit criterion of ≥ 95% critical-path pass rate defined in `00-test-plan.md`.

#### Results by surface

**Web Reservation Flow**
2 cases executed — 1 passed (TC-01A), 1 failed (TC-01C).
1 defect: DEF-001. Density: 1 defect / 2 cases.

**Identity Stitch**
2 cases executed — 1 passed (TC-02A), 1 failed (TC-02C).
1 defect: DEF-002. Density: 1 defect / 2 cases.

**SAP Inventory Validation**
3 cases executed — 1 passed (TC-03A), 2 failed (TC-03B, TC-03C).
2 defects: DEF-003, DEF-004. Density: 2 defects / 3 cases.

**Loyalty Credit**
Covered as a secondary surface in TC-03C. 1 defect recorded against this surface (DEF-004, shared with SAP Inventory Validation).

**POS Pickup Confirmation**
2 cases executed — 1 passed (TC-05A), 1 failed (TC-05C). 1 story raised (TC-05B).
1 defect: DEF-005. Density: 1 defect / 2 cases.

---

## 3. Top Two Problem Areas

### SAP Inventory Validation

- **Defects:** DEF-003, DEF-004
- **Business impact:** DEF-004 (Severity 1, Priority 1) is the most serious defect in this test cycle. When SAP returns zero stock at pickup, the POS completes the handover and credits loyalty points, meaning the customer leaves empty-handed while Meridian's records assert delivery occurred. This directly worsens the existing ~7% phantom-stock cancellation rate documented in `00-test-plan.md` by adding false "Collected" records and incorrect loyalty-programme liability. DEF-003 (Severity 2) independently sustains phantom-stock creation by propagating stale availability data to the web channel after the last unit is collected.
- **Rollout risk:** Both defects must be resolved before the Italy pilot. DEF-004 alone disqualifies the current build under the exit criterion of zero Priority-1 phantom-stock failures.

### Customer Identity Stitch

- **Defects:** DEF-002
- **Business impact:** DEF-002 (Severity 1, Priority 1) caused the identity-stitch service to silently merge two distinct loyalty profiles sharing an email address, exposing the second customer's purchase history and combined points balance to the first. This constitutes a GDPR Article 5 personal-data breach, requiring mandatory Data Protection Authority notification and carrying a potential fine of up to 4% of global annual turnover, as identified in Risk 2 of `00-test-plan.md`.
- **Rollout risk:** Launching the Italy pilot without resolving this defect would expose Meridian to regulatory and reputational consequences disproportionate to the commercial value of Phase 1.

---

## 4. Improvement Backlog

### BL-001 — Inventory-confirmation hard gate in POS pickup workflow

- **Engineering improvement:** Introduce a mandatory precondition check so that the "Collected" state transition and loyalty-credit event are only permitted when the SAP inventory-validation response explicitly confirms a quantity greater than zero. Any other result must route to a "Pickup Blocked" terminal state.
- **Business rationale:** Removes the fail-open condition identified as the root cause of DEF-004 (RCA Hypothesis A). Directly addresses the exit criterion of zero Priority-1 phantom-stock failures.
- **Suggested owner:** POS Platform Engineering Lead
- **Priority:** P1

### BL-002 — Loyalty points engine gated on confirmed inventory result

- **Engineering improvement:** The loyalty-credit trigger must consume an explicit inventory-confirmation token from the pickup-completion event. Points must not be posted when that token is absent or indicates zero stock.
- **Business rationale:** Removes the secondary condition identified in RCA Hypothesis B. Prevents loyalty liability accrual for unfulfilled orders, reducing financial reconciliation exposure identified in DEF-004.
- **Suggested owner:** Loyalty Platform Engineering Lead
- **Priority:** P1

### BL-003 — Conflict-detection gate in identity-stitch service

- **Engineering improvement:** Before committing any loyalty account merge, the identity-stitch service must perform a deterministic check for multiple distinct profiles sharing the same email address. A confirmed conflict must halt the merge and present a structured resolution notice to the customer; no profile data from a second account may be exposed.
- **Business rationale:** Removes the missing conflict-check condition that caused DEF-002. Prevents GDPR Article 5 violations and the associated DPA notification obligation.
- **Suggested owner:** Identity and Access Management Engineering Lead
- **Priority:** P1

### BL-004 — Server-side slot-expiry validation at order submission

- **Engineering improvement:** The checkout service must validate pickup-slot validity at the moment of order submission, not only at slot selection time. A slot whose expiry timestamp has passed must cause the submission to be rejected and no SAP reservation to be created.
- **Business rationale:** Removes the condition that allowed DEF-001 — expired-slot reservations being accepted — which creates unfulfillable SAP entries and drives contact-centre volume through subsequent cancellations.
- **Suggested owner:** Web Commerce Engineering Lead
- **Priority:** P1

### BL-005 — Event-driven inventory propagation from SAP to web channel

- **Engineering improvement:** Replace the polling-based SAP-to-web-channel inventory synchronisation with an event-driven mechanism so that a stock decrement in SAP propagates to the web availability cache within an operationally defined maximum latency, rather than on a fixed polling interval.
- **Business rationale:** Removes the propagation-delay condition identified in DEF-003. Prevents new phantom-stock reservations from being accepted during the window between a physical pickup and the next polling cycle, directly reducing the ~7% cancellation rate.
- **Suggested owner:** SAP Integration Engineering Lead
- **Priority:** P2

---

## 5. Release Recommendation

**NO GO**

The current build does not satisfy the Phase 1 exit criteria defined in `00-test-plan.md`. The observed pass rate of 40% falls materially below the required threshold of ≥ 95%. Two Severity-1 defects are open: DEF-002 exposes personal data from a second loyalty account during identity stitch, constituting a GDPR Article 5 breach; DEF-004 marks orders as "Collected" and credits loyalty points when SAP confirms zero stock, producing false fulfilment records against captured payments. Both defects are Priority 1 and block the Italy pilot. The root cause analysis confirms that DEF-004 arises from a structural condition in the POS pickup workflow — the absence of an inventory-confirmation hard gate — which has not been remediated. Four Priority-1 backlog items (BL-001 through BL-004) must be implemented and re-tested before the release recommendation can be revisited. Formal sign-off from David Park and Sarah Chen, as required by the exit criteria, is not appropriate at this stage.
