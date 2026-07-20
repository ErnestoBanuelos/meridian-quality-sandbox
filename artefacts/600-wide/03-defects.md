---
case: Meridian Retail Group — Click & Collect
feature: Cross-channel Click & Collect (Phase 1)
date: 2026-07-20
author: Ernesto Bañuelos
source_artefacts:
  - artefacts/600-wide/00-test-plan.md
  - artefacts/600-wide/01-test-cases.md
  - artefacts/600-wide/02-test-data.json
---

# Defect Log — Cross-channel Click & Collect (Phase 1)

## Execution Summary

- **Execution Mode:** Simulated QA Stub
- **Environment:** Meridian Reference Case (no live QA environment; no production system; no Playwright trace)
- **Executed Cases:** 10
- **Passed:** 4
- **Failed:** 5
- **Stories:** 1

### Executed Cases

The ten highest-priority test cases were selected from `01-test-cases.md` — the eight P1 cases (TC-01A, TC-01C, TC-02A, TC-02C, TC-03A, TC-03C, TC-05A, TC-05C) plus the two P2 cases most relevant to core business risk (TC-03B, TC-05B).

#### Passed

- **TC-01A** — Successful web cart reservation for Click & Collect — Data: RR-001
- **TC-02A** — Successful stitch of loyalty account to web account — Data: RR-001
- **TC-03A** — Inventory confirmed as available at pickup — Data: RR-001
- **TC-05A** — Successful QR-code scan and pickup confirmation at POS — Data: RR-001

#### Failed (defects raised)

- **TC-01C** — Reservation attempt after pickup slot expiration — Data: EC-005 → DEF-001
- **TC-02C** — Identity stitch blocked by merge conflict — Data: EC-002 → DEF-002
- **TC-03B** — Inventory drops to exactly one unit at moment of pickup confirmation — Data: RR-002 → DEF-003
- **TC-03C** — SAP inventory becomes zero between reservation and pickup — Data: EC-006 → DEF-004
- **TC-05C** — PSD2 SCA failure at POS terminal — Data: EC-007 → DEF-005

#### Story raised

- **TC-05B** — Pickup attempted with incorrect government ID — Data: RR-001 (ID mismatch condition injected) → STORY-001

---

## Defects

---

### DEF-001

- **Defect ID:** DEF-001
- **Title:** Customer with an expired pickup slot can submit a Click & Collect order and receive a confirmation, leading to an unfulfillable reservation in SAP.
- **Related Test Case:** TC-01C
- **Related Test Data Record:** EC-005 — CUST-US-01192 / ORD-US-2026-011925 / reservation_age_hours: 72
- **Business Area:** Web cart reservation flow
- **Steps to Reproduce:**
  1. Customer has an item in the web cart with a Click & Collect pickup slot that expired 72 hours ago (simulated using EC-005).
  2. Customer returns to the checkout page without selecting a new slot and submits the order.
  3. Observe the checkout outcome and the SAP reservation state.
- **Expected Result:** The checkout flow rejects the order, informs the customer that the pickup slot is no longer available, prompts selection of a new slot, and creates no reservation in SAP.
- **Actual Result:** The checkout flow accepted the submission and displayed an order confirmation screen. A reservation entry was created in SAP against the expired slot. No slot-expiry error was presented to the customer.
- **Severity:** 2 — Major business functionality unavailable; payment may be captured for an order that cannot be fulfilled.
- **Priority:** 1 — Blocks Italy pilot.
- **Status:** Open
- **Notes:** This failure is consistent with Risk 1 (phantom-stock cancellation) identified in `00-test-plan.md`. The reservation guard must validate slot expiry at the point of order submission, not only at slot selection time. No charge reversal flow was exercised in this simulated execution.

---

### DEF-002

- **Defect ID:** DEF-002
- **Title:** Customer is silently merged with a conflicting loyalty account during identity stitch, exposing a second customer's points balance and purchase history.
- **Related Test Case:** TC-02C
- **Related Test Data Record:** EC-002 — CUST-IT-00614 / loyalty_number: MLY-IT-91045 / conflicting_loyalty_number: MLY-IT-91046 / identity_merge_state: conflict
- **Business Area:** Customer identity stitch (existing loyalty account + web account)
- **Steps to Reproduce:**
  1. Customer CUST-IT-00614 logs in to the Meridian web account using the shared email address; the identity-provider stub is set to the conflict state (EC-002).
  2. The identity-stitch service detects two loyalty profiles (MLY-IT-91045 and MLY-IT-91046) mapped to the same email and attempts a merge.
  3. Observe the checkout page and the loyalty points balance displayed.
- **Expected Result:** The stitch is blocked. A conflict notice is shown directing the customer to contact Meridian Customer Services. Neither loyalty account is modified, and the second customer's data is not visible.
- **Actual Result:** The stitch proceeded without a conflict check. The checkout page displayed the combined points balance of both loyalty profiles. The second customer's most recent purchase was visible in the order history panel on the same page.
- **Severity:** 1 — Critical; personal data from a separate customer account was exposed, constituting a GDPR Article 5 breach.
- **Priority:** 1 — Blocks Italy pilot.
- **Status:** Open
- **Notes:** This failure directly realises Risk 2 from `00-test-plan.md`. The identity-stitch service must implement a deterministic conflict-detection gate before any merge is committed. This defect warrants immediate escalation to the Data Protection Officer prior to any pilot launch.

---

### DEF-003

- **Defect ID:** DEF-003
- **Title:** After collecting the last available unit, the web channel continues to display the item as available for Click & Collect at the same store.
- **Related Test Case:** TC-03B
- **Related Test Data Record:** RR-002 — CUST-DE-00831 / ORD-DE-2026-008315 / STORE-DE-MUC-003 / sap_inventory_at_reservation: 1
- **Business Area:** SAP inventory validation during pickup confirmation
- **Steps to Reproduce:**
  1. Customer CUST-DE-00831 holds a confirmed reservation for the last available unit at STORE-DE-MUC-003 (SAP inventory = 1 per RR-002).
  2. Store associate scans QR token QRT-7B3D1A-DE; SAP validates the single unit and confirms the handover.
  3. Navigate to the product page on the web channel and check Click & Collect availability for STORE-DE-MUC-003.
- **Expected Result:** After the pickup is confirmed, SAP decrements inventory to zero and the web channel immediately reflects zero stock for Click & Collect at that store for this SKU.
- **Actual Result:** SAP decremented inventory to zero and the order status updated to "Collected" correctly. However, the web channel product page continued to show one unit available for Click & Collect at STORE-DE-MUC-003 for more than five minutes after the pickup was confirmed. A second customer attempting a reservation during this window would receive a false availability signal.
- **Severity:** 2 — Major; stale availability data will produce phantom-stock reservations, directly contributing to the ~7% pickup cancellation rate documented in `00-test-plan.md`.
- **Priority:** 2 — High-value fix.
- **Status:** Open
- **Notes:** The SAP-to-web-channel inventory synchronisation appears to operate on a polling interval rather than an event-driven push. The acceptable propagation delay was not specified in Phase 1 requirements; this defect should be reviewed with David Park and Sarah Chen at sign-off.

---

### DEF-004

- **Defect ID:** DEF-004
- **Title:** When SAP confirms zero stock at pickup, the order status is marked "Collected" in error and loyalty points are credited despite the customer leaving the store without the item.
- **Related Test Case:** TC-03C
- **Related Test Data Record:** EC-006 — CUST-IT-00782 / ORD-IT-2026-007824 / STORE-IT-NAP-001 / sap_inventory_at_pickup: 0 / sap_inventory_at_reservation: 2
- **Business Area:** SAP inventory validation during pickup confirmation / Cross-region loyalty points credit
- **Steps to Reproduce:**
  1. Customer CUST-IT-00782 presents QR token QRT-0D6A3C-ZERO at STORE-IT-NAP-001; SAP inventory for the reserved SKU is zero at this moment (EC-006).
  2. The POS sends an inventory validation request; SAP returns a zero-stock response.
  3. Observe the order status in the Meridian web account and the loyalty points history page.
- **Expected Result:** The POS displays a cancellation message. The order remains in "Ready for Collection" status. No loyalty points are credited, and any pre-posted points are reversed.
- **Actual Result:** Despite receiving a zero-stock response from SAP, the POS completed the handover workflow and updated the order status to "Collected". Loyalty points for the full order value were posted to the customer's account. The customer did not receive the item.
- **Severity:** 1 — Critical; payment was captured and loyalty points were credited for an item that was not delivered. This constitutes a fulfilment failure with financial and regulatory exposure.
- **Priority:** 1 — Blocks Italy pilot.
- **Status:** Open
- **Notes:** This is the primary realisation of Risk 1 from `00-test-plan.md`. The POS pickup workflow must treat a zero-stock SAP response as a hard stop, not a warning. The loyalty points engine must be gated on a confirmed non-zero inventory validation result. This defect must reach zero occurrences before any pilot launch, per the exit criteria in `00-test-plan.md`.

---

### DEF-005

- **Defect ID:** DEF-005
- **Title:** After a PSD2 SCA failure at the POS, the customer's reservation is permanently cancelled, preventing re-attempt and leaving the customer without recourse.
- **Related Test Case:** TC-05C
- **Related Test Data Record:** EC-007 — CUST-DE-01088 / ORD-DE-2026-010882 / STORE-DE-HAM-006 / payment_status: sca_failed / sca_challenge_result: rejected
- **Business Area:** POS QR-code pickup confirmation
- **Steps to Reproduce:**
  1. Customer CUST-DE-01088 presents QR token QRT-2E9B7F-SCA at STORE-DE-HAM-006; the SCA provider stub is set to return a rejected response (EC-007).
  2. The POS initiates the PSD2 SCA challenge; the authentication is rejected.
  3. Observe the order status in the Meridian web account and the customer's available options on the POS screen.
- **Expected Result:** The POS displays a clear SCA failure message. The order remains in "Ready for Collection" status so the customer can retry or contact Meridian Customer Services. Inventory is not decremented.
- **Actual Result:** The POS displayed an SCA failure message correctly. However, the order status was simultaneously updated to "Cancelled" in the Meridian web account. The reservation was voided and inventory was released. The customer had no way to retry the collection. No reference number or escalation path was provided on the POS screen.
- **Severity:** 2 — Major; a valid, paid order is cancelled solely because of a transient authentication failure, leaving the customer with no self-service recovery path.
- **Priority:** 1 — Blocks Italy pilot.
- **Status:** Open
- **Notes:** This failure is the direct realisation of Risk 3 from `00-test-plan.md`. The SCA failure handling must be decoupled from reservation cancellation. A transient SCA failure must result in a retry-eligible "Held" state, not an irreversible cancellation. The absence of a reference number on the POS screen is a secondary issue that should be tracked under this defect.

---

## Stories

---

### STORY-001

- **Observation:** During simulated execution of TC-05B (pickup with incorrect government ID presented, using RR-001 with a manually injected ID mismatch condition), the POS correctly blocked the handover and displayed an ID mismatch warning. However, the escalation instruction shown on screen ("See store manager") did not include a reference number or any mechanism for the store manager to locate the held order, which differed from the expected result in TC-05B.
- **Why it is not reproducible:** The exact text of the escalation message and the presence or absence of a reference number depends on POS terminal firmware configuration, which is outside the scope of the simulated QA stub. The condition could not be reproduced consistently across different simulated terminal configurations. The behaviour may be environment-specific rather than a defect in the application logic.
- **Suggested follow-up:** During live QA execution in the Italy pilot environment, a dedicated sub-step should be added to TC-05B to capture the full text of the escalation message and verify that a unique order reference is displayed. If the reference number is absent in the live environment, a formal defect should be raised at that point.
