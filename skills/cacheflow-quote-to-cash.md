---
name: Create a customer, build a proposal, and get it accepted
description: >-
  The Cacheflow marquee flow: create a customer and billing contact, create a proposal,
  add priced items from the catalog, run it through approval if required, share it with
  the buyer, and confirm acceptance.
api: openapi/cacheflow-api-openapi.yml
operations:
  - createCustomer
  - createCustomerContact
  - listActiveProducts
  - createProposal
  - addProposalItem
  - updateProposalItems
  - requestApproval
  - getApprovalRequests
  - inviteBillingContact
  - getProposal
  - getProposalEvents
  - acceptProposal
generated: '2026-08-13'
method: generated
source: openapi/_original/cacheflow-openapi.json
status: historical
---

# Create a customer, build a proposal, and get it accepted

> **This API is retired.** Cacheflow was acquired by HubSpot in 2024. As of 2026-08-13
> `api.getcacheflow.com` returns NXDOMAIN. This skill documents the contract Cacheflow
> published; do not attempt to call it. It is kept as a reference for anyone migrating a
> Cacheflow integration to HubSpot Commerce / CPQ.

## Before you start

- **Base URL**: `https://api.getcacheflow.com` (sandbox: `https://api.sandbox.getcacheflow.com`).
  Every path is prefixed `/api/latest`.
- **Auth**: `Authorization: Bearer <api-token>`, minted in the app under Settings -> API.
- **Tenant routing is mandatory**: also send `Host: <your-org-flow-domain>.api.getcacheflow.com`.
  The token alone will not route your call.
- **No idempotency.** There is no `Idempotency-Key` on any operation. A retried `POST`
  creates a second object. Read back with `getProposals` / `listCustomers` before retrying.
- **Errors** come back as `{ errorCode, message, detail }` where `errorCode` is one of 353
  named string codes — see `errors/cacheflow-error-codes.yml`.

## Steps

1. **Find or create the customer.**
   Call `listCustomers` with `search` to check first, or `getCustomerByExternalId` if you
   carry a CRM id. If absent, call `createCustomer`.
   Watch for `CUSTOMER_EMAIL_ALREADY_EXIST`, `CUSTOMER_NAME_REQUIRED`,
   `CUSTOMER_ALREADY_EXISTS_FOR_EXTERNAL_SOURCE`.

2. **Add a billing contact.**
   `createCustomerContact` against the customer id. A proposal cannot be shared without
   one — `PROPOSAL_CONTACT_SET_WITH_NO_CUSTOMER` and `MISSING_SIGNING_CONTACT` are the
   failure modes.

3. **Pick catalog products.**
   `listActiveProducts` (not `listProducts` — inactive products will be rejected with
   `CANT_USE_NON_ACTIVE_PRODUCTS`). Use `getLatestByRoot` when you track a product family
   by root id rather than by version.

4. **Create the proposal.**
   `createProposal`. Set the term, start date and currency here. `PROPOSAL_START_DATE_IN_PAST`
   and `PRODUCT_MISSING_PROPOSAL_CURRENCY` are the common rejections.

5. **Add priced items.**
   `addProposalItem` per line, or `updateProposalItems` to add, update and remove in one
   batch. Recurring and usage items require item schedules; expect
   `PROPOSAL_ITEM_REQUIRES_ITEM_SCHEDULES`, `PROPOSAL_ITEM_REQUIRES_QUANTITY` and the
   `ITEM_SCHEDULE_*` family when the schedule shape is wrong.

6. **Run approvals if your org configured them.**
   `requestApproval` on the proposal, then poll `getApprovalRequests`. Approvers act via
   `createApprovalAction`; an approver client lists its own queue with
   `getMyApproverRequests`. `APPROVAL_USER_NOT_AUTHORIZED` and `APPROVAL_INVALID_TRANSITION`
   are the guards.

7. **Share it with the buyer.**
   `inviteBillingContact` sends the deal-room link. The buyer lands on
   `https://{org}.checkout.getcacheflow.com/p/{proposalId}` to e-sign and set up payment.
   `PROPOSAL_FIELD_REQUIRED_TO_ACTIVATE` means the proposal is not complete enough to send.

8. **Track it to acceptance.**
   Poll `getProposal` for status, or read the audit trail with `getProposalEvents`. Better:
   register a webhook (see the `cacheflow-webhook-subscriptions` skill) and react to the
   `status_changed` / `proposal` notification instead of polling.

9. **Accept programmatically only if you must.**
   `acceptProposal` exists for flows where the buyer accepts outside the deal room. It will
   refuse with `PROPOSAL_REQUIRES_SIGNING_BEFORE_ACCEPT` or
   `PROPOSAL_REQUIRES_PAYMENT_METHOD_ACCEPT` when signing or payment setup is outstanding,
   and `PROPOSAL_ALREADY_ACCEPTED` on a repeat.

## After acceptance

An accepted proposal becomes a subscription. Continue with the
`cacheflow-subscription-change` skill to amend it, or `cacheflow-usage-billing` to meter it.
