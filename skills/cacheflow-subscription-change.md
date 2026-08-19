---
name: Change an existing subscription
description: >-
  Amend a live Cacheflow subscription — change quantity, add or remove items, or renew —
  by creating a change proposal against the subscription rather than editing it in place.
api: openapi/cacheflow-api-openapi.yml
operations:
  - listContracts
  - getContract
  - getCustomerContracts
  - createContractProposal
  - updateProposalItems
  - getProposal
  - inviteBillingContact
  - getContractProposals
  - updateContract
generated: '2026-08-13'
method: generated
source: openapi/_original/cacheflow-openapi.json
status: historical
---

# Change an existing subscription

> **This API is retired.** Cacheflow was acquired by HubSpot in 2024 and
> `api.getcacheflow.com` no longer resolves (checked 2026-08-13). This documents the
> published contract for migration reference only.

## The rule that governs this whole flow

A Cacheflow subscription (`contract` in the API) is **not edited directly**. Every change
is expressed as a **change proposal** that the customer accepts, exactly like the original
quote. That is why the operation is `createContractProposal` and not `patchContract`.

## Before you start

- Base `https://api.getcacheflow.com/api/latest`, `Authorization: Bearer <token>`, plus the
  tenant `Host` header. See `authentication/cacheflow-authentication.yml`.
- No idempotency support. Do not blind-retry `createContractProposal` — read back with
  `getContractProposals` first, or you will strand a second draft change proposal.

## Steps

1. **Locate the subscription.**
   `listContracts` with `search`/`page`/`size`, or `getCustomerContracts` to scope to one
   customer. `getContract` for the full record including current items and billing
   schedules.

2. **Check nothing is already in flight.**
   `getContractProposals` lists proposals against this subscription.
   `CONTRACT_PROPOSAL_ALREADY_IN_PROGRESS` is the error when you skip this step.

3. **Create the change proposal.**
   `createContractProposal` against the subscription id. The subscription must be active —
   `CHANGE_PROPOSAL_CANT_CREATE_FROM_NON_ACTIVE_SUBSCRIPTION` otherwise — and the change
   must start within the previous term's bounds
   (`CHANGE_PROPOSAL_CANT_START_BEFORE_PREVIOUS`, `CHANGE_PROPOSAL_CANT_START_AFTER_PREVIOUS`).

4. **Adjust the lines.**
   `updateProposalItems` for a batch add/update/remove. Change proposals are more
   constrained than new ones — these will refuse you:
   - `CHANGE_PROPOSAL_CANT_REMOVE_ONETIME`
   - `CHANGE_PROPOSAL_CANT_DECREASE_ONETIME_QUANTITY`
   - `CHANGE_PROPOSAL_CANT_CHANGE_ONETIME`
   - `CHANGE_PROPOSAL_CANT_ADD_SCHEDULED_ITEM`
   - `CHANGE_PROPOSAL_CANT_EDIT_FIELD`
   One-time items are effectively frozen once the subscription is live.

5. **Send it for acceptance.**
   `inviteBillingContact`, then track with `getProposal` or a webhook.
   `CHANGE_RENEWAL_PROPOSAL_NEEDS_PREVIOUS_ACCEPTED` means an earlier change is still open.

6. **Renewals are the same shape.**
   A renewal is a proposal type, not a separate endpoint. `RENEWAL_PROPOSAL_CANT_EDIT_FIELD`
   and `PROPOSAL_MUST_BE_RENEWAL` guard it; `AUTO_RENEWAL_DISABLED` means the org turned
   auto-renewal off.

7. **`updateContract` is for administrative fields only.**
   Use it for metadata and status transitions the org is authorised to make — not for
   pricing. `CONTRACT_NOT_AUTHORIZED_TO_CHANGE_STATUS` and `CONTRACT_IS_CANCELLED` apply.

## Reading the result

`getContract` after acceptance reflects the new items and schedules. Billing schedules are
regenerated — read them with `getBillingSchedule`, and see the `cacheflow-usage-billing`
skill if the subscription meters usage.
