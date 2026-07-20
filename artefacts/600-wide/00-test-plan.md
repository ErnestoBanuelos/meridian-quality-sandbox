---
case: Meridian Retail Group — Click & Collect
feature: Cross-channel Click & Collect (Phase 1)
date: 2026-07-20
author: Ernesto Bañuelos
---

# Test Plan — Cross-channel Click & Collect (Phase 1)

## In Scope

- Web cart reservation flow
- Customer identity stitch (existing loyalty account + web account)
- SAP inventory validation during pickup confirmation
- Cross-region loyalty points credit
- POS QR-code pickup confirmation

## Out of Scope

- **SAP ECC inventory ground-truth correctness** — owned by Finance and covered by their own controls.
- **Legacy Shopify storefronts being retired during Phase 1** — outside the rollout scope.

## Top 3 Risks

**Risk 1 — Phantom-stock cancellation at pickup**
- *Risk:* SAP inventory cache is not invalidated in real time; a reservation succeeds against stock that has already sold through a parallel channel, causing the order to be cancelled when the associate scans the QR code at the POS.
- *User impact:* Customer travels to store and leaves empty-handed; estimated current rate is ~7% of Click & Collect orders.
- *Business impact:* Increased contact-centre volume, loyalty churn, and potential regulatory exposure if refund SLAs are breached.

**Risk 2 — Identity collision when merging loyalty accounts**
- *Risk:* The identity-stitch service may merge two distinct loyalty profiles that share an email address (e.g., a household sharing an inbox), inadvertently combining purchase history and points balances under a single identity.
- *User impact:* Customers see incorrect points balances and may access another person's order history, constituting a personal-data breach.
- *Business impact:* GDPR Article 5 violation requiring mandatory DPA notification; reputational damage and potential fines up to 4% of global annual turnover.

**Risk 3 — PSD2 SCA failure blocking EU pickup confirmation**
- *Risk:* The POS QR-code confirmation step triggers a payment re-authentication challenge in EU markets under PSD2 Strong Customer Authentication; if the SCA redirect fails or times out, the pickup cannot be completed at the terminal.
- *User impact:* Customer is unable to collect a paid order without store staff manually overriding the POS, creating queues and friction.
- *Business impact:* Regulatory non-compliance with PSD2 RTS Article 97 in EU markets; potential fines and forced remediation sprint delaying Phase 2 rollout.

## Entry Criteria

- Phase 1 build deployed successfully to the QA environment.
- SAP sandbox populated with realistic inventory deltas.
- Identity-provider stub configured for merged customer accounts.

## Exit Criteria

- Critical-path pass rate ≥ 95%.
- Zero Priority-1 phantom-stock failures.
- Formal sign-off completed by David Park and Sarah Chen.
