---
name: Subscribe to Cacheflow events with webhooks
description: >-
  Register, test and operate a Cacheflow webhook subscription so a system reacts to
  proposal and subscription status changes instead of polling.
api: openapi/cacheflow-api-openapi.yml
operations:
  - addToken
  - addWebhook
  - getHooks
  - getWebhooksMetadata
  - testHook
  - testEvent
  - getHookEvents
  - updateWebhook
  - removeHook
  - getProposal
generated: '2026-08-13'
method: generated
source: openapi/_original/cacheflow-openapi.json
status: historical
---

# Subscribe to Cacheflow events with webhooks

> **This API is retired.** `api.getcacheflow.com` returns NXDOMAIN as of 2026-08-13; no
> webhook can be registered or delivered. This records the contract Cacheflow published.

## What Cacheflow sends

A **thin notification**, delivered as an HTTP **PUT** to your endpoint:

```json
{
  "event_type": "status_changed",
  "reference_type": "proposal",
  "id": "<id of object>"
}
```

Two consequences worth planning around:

1. **The payload carries no state.** You get an identity and must call back —
   `getProposal`, `getContract`, `getInvoice` — to learn what actually changed. Budget for
   that read amplification.
2. **PUT, not POST.** Endpoints that only accept POST will silently reject every delivery.

## Steps

1. **Have an API token.**
   `addToken` (Settings -> API in the app). `getAllTokens` to list, `removeToken` to revoke.

2. **Your endpoint must be HTTPS.**
   Registration fails with `APIHOOK_REQUIRES_HTTPS` otherwise. This is enforced, not advisory.

3. **Register the subscription.**
   `addWebhook` with your URL and the event/reference type you want.
   `APIHOOK_INVALID_EVENT_TYPE` is the rejection when the type is not recognised. Note that
   Cacheflow never published the full event-type vocabulary — `status_changed` on
   `proposal` is the only pair shown in the provider's own example, so discover the rest
   empirically against sandbox before relying on them.

4. **Test before you trust it.**
   `testHook` on the registered hook id sends a probe delivery. `testEvent` synthesises an
   event so you can exercise your handler end to end.

5. **Operate it.**
   - `getHooks` / `getWebhooksMetadata` — what is registered.
   - `getHookEvents` — the delivery log for one hook. **This is your only retry visibility:
     Cacheflow published no retry or backoff policy, so treat the delivery log as the
     source of truth and reconcile against it.**
   - `updateWebhook` to change the URL or event type, `removeHook` to unsubscribe.

6. **Handle it idempotently on your side.**
   Cacheflow offers no delivery signature and no documented dedupe key. Key your handler on
   `(reference_type, id, event_type)` plus the state you read back, and make replays
   harmless — the API will not do that work for you.

## Alternative: poll the event logs

Where webhooks are not viable, the contract exposes read-only event surfaces:
`getProposalEvents`, `getInvoiceActivity`, `getSyncEvents`, `getEvents`. They are heavier
but they are pull-based and require no public endpoint.
