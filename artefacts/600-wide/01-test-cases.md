---
case: Meridian Retail Group — Click & Collect
feature: Cross-channel Click & Collect (Phase 1)
date: 2026-07-20
author: Ernesto Bañuelos
source: artefacts/600-wide/00-test-plan.md
---

# Test Cases — Cross-channel Click & Collect (Phase 1)

## In Scope Reference

The following surfaces are under test, sourced directly from `00-test-plan.md`:

- Web cart reservation flow
- Customer identity stitch (existing loyalty account + web account)
- SAP inventory validation during pickup confirmation
- Cross-region loyalty points credit
- POS QR-code pickup confirmation

---

## Surface 1 — Web Cart Reservation Flow

---

### TC-01A — Successful web cart reservation for Click & Collect

- **Category:** Smoke
- **Priority:** P1 — Blocks Italy pilot
- **In-Scope Surface:** Web cart reservation flow

**Preconditions:**
- Customer is logged in with a valid Meridian web account.
- At least two units of the selected SKU are available in the chosen store's SAP inventory.
- A Click & Collect pickup slot is open at the chosen store.

**Steps:**
1. Customer adds a single item to the web cart and selects Click & Collect as the fulfilment method.
2. Customer selects an available store and a pickup slot.
3. Customer proceeds through checkout and confirms the order.
4. Customer receives the order confirmation page and a confirmation email.

**Expected Result:**
The order is confirmed, the reservation is recorded against the store's inventory in SAP, and the confirmation email displays the pickup slot, store address, and a QR-code collection reference.

---

### TC-01B — Reservation succeeds when only the last unit is available

- **Category:** Edge
- **Priority:** P2 — High business impact
- **In-Scope Surface:** Web cart reservation flow

**Preconditions:**
- Customer is logged in with a valid Meridian web account.
- Exactly one unit of the selected SKU remains in SAP inventory for the chosen store.
- No other session has that unit in an active reservation at the start of the test.

**Steps:**
1. Customer adds the item to the web cart and selects Click & Collect.
2. Customer selects the store showing one unit available and proceeds to checkout.
3. Customer confirms the order before the inventory count changes.
4. Customer receives the order confirmation page.

**Expected Result:**
The reservation is accepted for the last unit, SAP inventory is decremented to zero for that store, and the item is no longer shown as available for Click & Collect at that store on the web channel.

---

### TC-01C — Reservation attempt after pickup slot expiration

- **Category:** Regression
- **Priority:** P1 — Blocks Italy pilot
- **In-Scope Surface:** Web cart reservation flow

**Preconditions:**
- Customer is logged in with a valid Meridian web account.
- Customer has an item in the web cart with a Click & Collect pickup slot selected.
- The selected pickup slot has expired (test environment clock advanced past slot end time).

**Steps:**
1. Customer returns to the checkout page after the selected pickup slot has expired.
2. Customer attempts to confirm the order without selecting a new slot.
3. Customer submits the order.
4. Customer observes the result on screen.

**Expected Result:**
The checkout flow rejects the order and displays a clear message that the selected pickup slot is no longer available. The customer is prompted to choose a new slot. No reservation is created in SAP and no charge is processed.

---

## Surface 2 — Customer Identity Stitch

---

### TC-02A — Successful stitch of loyalty account to web account

- **Category:** Critical-Path
- **Priority:** P1 — Blocks Italy pilot
- **In-Scope Surface:** Customer identity stitch (existing loyalty account + web account)

**Preconditions:**
- Customer holds an active Meridian loyalty account (loyalty ID present).
- Customer also holds a separate Meridian web account using the same email address.
- Identity-provider stub is configured for merged customer accounts.

**Steps:**
1. Customer logs in to the web account and initiates a Click & Collect checkout.
2. The identity-stitch service detects the matching loyalty account by email.
3. Customer is prompted to confirm the account link and completes the verification step.
4. Customer lands on the checkout page with the unified account active.

**Expected Result:**
The two accounts are merged into a single identity. The customer's loyalty points balance is visible and accurate on the checkout page. No duplicate accounts remain in the identity-provider after the stitch.

---

### TC-02B — Identity stitch for a loyalty account with a zero-point balance

- **Category:** Edge
- **Priority:** P3 — Moderate impact
- **In-Scope Surface:** Customer identity stitch (existing loyalty account + web account)

**Preconditions:**
- Customer holds an active Meridian loyalty account with a confirmed balance of zero points.
- Customer holds a separate Meridian web account using the same email address.
- Identity-provider stub is configured for merged customer accounts.

**Steps:**
1. Customer logs in to the web account and initiates a Click & Collect checkout.
2. The identity-stitch service detects the matching loyalty account by email.
3. Customer confirms the account link.
4. Customer lands on the checkout page with the unified account active.

**Expected Result:**
The stitch completes successfully. The loyalty points balance displayed is zero. No erroneous points are added or removed as a side effect of the merge. The customer can continue to checkout.

---

### TC-02C — Identity stitch blocked by a merge conflict between two distinct loyalty accounts

- **Category:** Regression
- **Priority:** P1 — Blocks Italy pilot
- **In-Scope Surface:** Customer identity stitch (existing loyalty account + web account)

**Preconditions:**
- Two separate Meridian loyalty accounts belong to two different customers who share the same email address (e.g., household inbox scenario).
- Identity-provider stub is configured to simulate a merge-conflict state for this email.
- Customer A is logged in to the web account using the shared email.

**Steps:**
1. Customer A logs in to the web account and initiates a Click & Collect checkout.
2. The identity-stitch service detects two distinct loyalty profiles mapped to the same email.
3. The system attempts to proceed with account merging.
4. Customer A observes the result.

**Expected Result:**
The stitch is blocked. The customer is shown a conflict notice and directed to contact Meridian Customer Services for manual resolution. Neither loyalty account is modified. Customer B's purchase history and points balance remain inaccessible to Customer A.

---

## Surface 3 — SAP Inventory Validation During Pickup Confirmation

---

### TC-03A — Inventory confirmed as available at pickup

- **Category:** Critical-Path
- **Priority:** P1 — Blocks Italy pilot
- **In-Scope Surface:** SAP inventory validation during pickup confirmation

**Preconditions:**
- Customer holds a confirmed Click & Collect reservation with a valid QR code.
- SAP sandbox shows the reserved SKU in stock at the pickup store.
- Store associate is at the POS terminal with the item ready.

**Steps:**
1. Store associate opens the Click & Collect pickup workflow on the POS terminal.
2. Associate scans the customer's QR code.
3. SAP inventory validation runs against the reservation.
4. Associate confirms the handover on the POS.

**Expected Result:**
SAP confirms stock availability, the pickup is completed, inventory is decremented by one unit, and the order status updates to "Collected" in the Meridian web account.

---

### TC-03B — Inventory drops to exactly one unit at the moment of pickup confirmation

- **Category:** Edge
- **Priority:** P2 — High business impact
- **In-Scope Surface:** SAP inventory validation during pickup confirmation

**Preconditions:**
- Customer holds a confirmed Click & Collect reservation with a valid QR code.
- SAP sandbox inventory for the reserved SKU at this store is exactly one unit at the time the associate scans the QR code.
- No other concurrent reservation exists for the same SKU at this store.

**Steps:**
1. Store associate scans the customer's QR code on the POS terminal.
2. SAP inventory validation runs; the current count is one unit.
3. The validation succeeds and the associate confirms the handover.
4. Customer collects the item.

**Expected Result:**
SAP decrements inventory to zero for that SKU at that store. The order status updates to "Collected". The web channel immediately reflects zero stock for Click & Collect at this store for the same SKU.

---

### TC-03C — SAP timeout during pickup confirmation

- **Category:** Regression
- **Priority:** P1 — Blocks Italy pilot
- **In-Scope Surface:** SAP inventory validation during pickup confirmation

**Preconditions:**
- Customer holds a confirmed Click & Collect reservation with a valid QR code.
- SAP sandbox is configured to simulate an inventory-check timeout (no response within the defined threshold).
- Store associate is at the POS terminal.

**Steps:**
1. Store associate scans the customer's QR code on the POS terminal.
2. POS sends an inventory validation request to SAP.
3. SAP does not respond within the timeout threshold.
4. Associate and customer observe the result on screen.

**Expected Result:**
The POS displays a clear timeout error message. The order is not marked as "Collected". Inventory in SAP is not decremented. The associate is presented with an escalation path (e.g., contact store manager) and the customer's reservation remains valid for retry.

---

## Surface 4 — Cross-Region Loyalty Points Credit

---

### TC-04A — Loyalty points credited correctly after inter-region Click & Collect purchase

- **Category:** Regression
- **Priority:** P2 — High business impact
- **In-Scope Surface:** Cross-region loyalty points credit

**Preconditions:**
- Customer has a stitched Meridian account with a known starting loyalty points balance.
- Customer places a Click & Collect order at a store in a different Meridian region than their registered home region.
- The pickup is completed successfully (order status "Collected").

**Steps:**
1. Customer completes a Click & Collect pickup at an out-of-home-region Meridian store.
2. The order status transitions to "Collected" in the Meridian web account.
3. Customer navigates to the loyalty points history page.
4. Customer reviews the latest points transaction.

**Expected Result:**
The correct number of loyalty points for the purchase is credited to the customer's account within the agreed posting window. The points entry references the correct store region. The total balance reflects the addition accurately.

---

### TC-04B — Loyalty points credit when a purchase crosses a midnight regional boundary

- **Category:** Edge
- **Priority:** P3 — Moderate impact
- **In-Scope Surface:** Cross-region loyalty points credit

**Preconditions:**
- Customer places a Click & Collect order in one region but collects it in a store that is in a time zone where the calendar date rolls over to the next day during the transaction.
- The loyalty points engine uses transaction date to determine the earning period.

**Steps:**
1. Customer initiates Click & Collect checkout at 23:58 in the store's local time zone.
2. The order confirmation occurs just after midnight in the store's local time zone.
3. Customer collects the order and the status updates to "Collected".
4. Customer reviews the loyalty points history page.

**Expected Result:**
Exactly one points transaction is posted for the purchase. The earning period is attributed to a single calendar date (the date of collection) and no duplicate or split points entries are created.

---

### TC-04C — Loyalty points not credited when inventory becomes zero between reservation and pickup

- **Category:** Regression
- **Priority:** P2 — High business impact
- **In-Scope Surface:** Cross-region loyalty points credit

**Preconditions:**
- Customer holds a confirmed Click & Collect reservation.
- Between reservation and pickup, SAP inventory for the reserved SKU reaches zero (phantom-stock cancellation scenario).
- The order is cancelled at pickup and the customer does not receive the item.

**Steps:**
1. Store associate scans the customer's QR code on the POS terminal.
2. SAP inventory validation returns zero stock for the reserved SKU.
3. The POS cancels the pickup and notifies the customer that the order cannot be fulfilled.
4. Customer reviews the loyalty points history page.

**Expected Result:**
No loyalty points are credited to the customer's account for the cancelled order. If a points deduction was pre-posted at reservation time, it is reversed. The customer's balance reflects the state prior to the failed reservation.

---

## Surface 5 — POS QR-Code Pickup Confirmation

---

### TC-05A — Successful QR-code scan and pickup confirmation at POS

- **Category:** Critical-Path
- **Priority:** P1 — Blocks Italy pilot
- **In-Scope Surface:** POS QR-code pickup confirmation

**Preconditions:**
- Customer holds a confirmed Click & Collect reservation with a valid, unexpired QR code.
- SAP inventory confirms stock availability.
- POS terminal is online and the store associate is authenticated.

**Steps:**
1. Customer presents the QR code (on device or printed) to the store associate.
2. Associate scans the QR code on the POS terminal.
3. POS validates the reservation and displays the order summary.
4. Associate confirms handover; customer signs or acknowledges receipt.

**Expected Result:**
The POS confirms the pickup, the order status in the Meridian web account updates to "Collected", and the customer receives a collection receipt notification. Loyalty points credit is triggered.

---

### TC-05B — Pickup attempted with incorrect government ID presented

- **Category:** Edge
- **Priority:** P2 — High business impact
- **In-Scope Surface:** POS QR-code pickup confirmation

**Preconditions:**
- Customer holds a confirmed Click & Collect reservation with a valid QR code.
- The order is flagged for ID verification at pickup (e.g., age-restricted product or high-value item).
- Customer presents an ID document that does not match the name on the order.

**Steps:**
1. Associate scans the customer's QR code; the POS flags the order for ID verification.
2. Customer presents a government-issued ID that does not match the registered name on the Meridian account.
3. Associate enters the mismatch into the POS workflow.
4. Associate and customer observe the result.

**Expected Result:**
The POS blocks the handover and displays an ID mismatch warning. The order remains in "Ready for Collection" status. The associate is instructed to direct the customer to the store manager. No inventory is decremented and no loyalty points are credited.

---

### TC-05C — PSD2 Strong Customer Authentication failure at POS terminal

- **Category:** Regression
- **Priority:** P1 — Blocks Italy pilot
- **In-Scope Surface:** POS QR-code pickup confirmation

**Preconditions:**
- Customer holds a confirmed Click & Collect reservation with a valid QR code.
- Store is in an EU market where PSD2 Strong Customer Authentication applies at the POS confirmation step.
- The SCA provider stub is configured to return an authentication failure response.

**Steps:**
1. Associate scans the customer's QR code on the POS terminal.
2. POS initiates the PSD2 SCA challenge and redirects to the authentication step.
3. The SCA provider returns a failure response (authentication rejected or timed out).
4. Associate and customer observe the result on the POS screen.

**Expected Result:**
The POS displays a clear SCA failure message without completing the pickup. The order remains in "Ready for Collection" status. Inventory is not decremented. The customer is informed that collection cannot proceed and is given a reference number to contact Meridian Customer Services. No loyalty points are credited for the incomplete pickup.

---

## Report

- **Total test cases generated:** 15
- **Negative cases:** 6 (TC-01C, TC-02C, TC-03C, TC-04C, TC-05B, TC-05C)
- **Priority-1 cases:** 8 (TC-01A, TC-01C, TC-02A, TC-02C, TC-03A, TC-03C, TC-05A, TC-05C)
- **Files modified:** `artefacts/600-wide/01-test-cases.md` (created)
