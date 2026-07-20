---
case: Meridian Retail Group — Click & Collect
feature: Cross-channel Click & Collect (Phase 1)
date: 2026-07-20
author: Ernesto Bañuelos
---

# Data Generation Method — 02-test-data.json

- **Tool used:** Codemie Code (Claude Sonnet 4.6) — manual synthetic generation; no external data-generation library.
- **Source artefact consumed:** `artefacts/600-wide/01-test-cases.md` — all 15 test cases and their preconditions.
- **Fields obfuscated:** All personally identifiable fields (customer names, email addresses, phone numbers, payment tokens, street addresses) are fully synthetic and carry no relationship to real individuals or accounts.
- **Variety dimensions exercised:** Market/country (IT, DE, JP, GB, US), payment method (Postepay, Satispay, Klarna, Klarna Split-Pay, Credit Card, Debit Card), loyalty tier (Standard, Silver, Gold, Platinum), identity merge state (stitched, conflict, unstitched), reservation age (1–72 h), SAP inventory level (0–55), order size (1–50), multilingual character sets (Latin with accents, German ß, Japanese kana, Arabic script), and null/empty optional fields.
- **PII approach:** Synthetic-only values; IDs follow a structured fictional pattern (e.g., `CUST-IT-NNNNN`). No real names, addresses, emails, phone numbers, or payment tokens are present anywhere in the file.
- **Out-of-scope data intentionally omitted:** SAP ECC ground-truth inventory ledger entries; legacy Shopify storefront records; real payment-gateway credentials.
- **Reproducibility notes:** All records are deterministic and static; re-running generation with the same source artefact and instructions will produce an equivalent dataset. No random seed is required.
