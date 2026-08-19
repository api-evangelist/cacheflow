---
name: Load usage onto a subscription and invoice it
description: >-
  Meter a usage-priced Cacheflow subscription — find the active usage billing schedule,
  post usage records before the deadline, verify what landed, and generate the invoice.
api: openapi/cacheflow-api-openapi.yml
operations:
  - getContract
  - getUsageSchedule
  - createUsage
  - getBillingScheduleUsage
  - getBillingSchedule
  - generateInvoice
  - getInvoice
  - listInvoices
  - getInvoiceActivity
generated: '2026-08-13'
method: generated
source: openapi/_original/cacheflow-openapi.json
status: historical
---

# Load usage onto a subscription and invoice it

> **This API is retired.** `api.getcacheflow.com` returns NXDOMAIN as of 2026-08-13.
> Cacheflow was folded into HubSpot Commerce / CPQ after the 2024 acquisition. Reference
> only.

## Before you start

- Base `https://api.getcacheflow.com/api/latest`, bearer token, tenant `Host` header.
- **This is the flow most exposed by the missing idempotency contract.** `createUsage` has
  no `Idempotency-Key`. A retry after a timeout double-bills the customer. Always read back
  with `getBillingScheduleUsage` before re-posting.

## Steps

1. **Confirm the subscription meters usage.**
   `getContract` for the subscription, then `getUsageSchedule` — "Retrieve the active usage
   billing schedules for a subscription". If it returns nothing, the products on this
   subscription are not usage-priced and `createUsage` will fail with
   `USAGE_PRODUCT_NOT_USAGE`.

2. **Post the usage.**
   `createUsage` against the billing schedule. Timing is enforced at both ends:
   - `USAGE_TOO_EARLY` — the period has not opened yet.
   - `USAGE_DEADLINE_PAST` — the window for this period has closed.
   - `USAGE_INVOICE_EXISTS` — an invoice has already been cut for this period; usage is
     locked.
   Post continuously through the period rather than in one batch at the end, or a late
   batch is rejected outright.

3. **Verify what landed.**
   `getBillingScheduleUsage` lists the usage recorded against the schedule. Reconcile your
   own totals against this before generating the invoice — this is the only defence against
   the retry problem in step 2.

4. **Read the schedule.**
   `getBillingSchedule` shows the period, amounts and issue date.
   `PREVIOUS_SCHEDULE_NOT_INVOICED` means an earlier period is still open and must be
   settled first.

5. **Generate the invoice.**
   `generateInvoice` creates and syncs the invoice for the billing schedule. It also pushes
   to the connected accounting system, so expect integration-side failures here, not just
   validation ones (`INTEGRATION_ERROR`, `QUICKBOOKS_CUSTOM_TXN_NUMBERS_DISABLED`,
   `TAX_CALCULATION_ERROR`).

6. **Confirm.**
   `getInvoice` for the document, `getInvoiceActivity` for its audit trail, `listInvoices`
   with filters to sweep a period. `INVOICE_CLOSED_BOOKS` means the accounting period is
   shut and the invoice can no longer be altered.

## Guardrail for agents

Usage posting is a **write with financial consequence and no idempotency guarantee**. An
agent driving this flow should treat `createUsage` as requiring an explicit read-back
confirmation step, not a fire-and-forget call, and should never retry it automatically on a
network timeout.
