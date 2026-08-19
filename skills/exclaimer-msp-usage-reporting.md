---
name: exclaimer-msp-usage-reporting
description: >-
  Pull per-sender imprint usage for an MSP-billed Exclaimer Cloud subscription
  over a date range, and reconcile it against subscription status — the
  billing-evidence flow for managed service providers.
generated: '2026-08-13'
method: generated
source: >-
  Grounded in openapi/_original/exclaimer-cloud-api-openapi.json (fetched from
  https://cloudapi.exclaimer.com/openapi.json 2026-08-13). The published
  specification declares no operationIds; steps name the verified METHOD + PATH.
api: Exclaimer Cloud API
base_url: https://cloudapi.exclaimer.com/exclaimerapi
operations:
- GET /1.0/msp/subscriptions
- GET /1.0/msp/subscriptions/{SubscriptionID}
- GET /1.0/msp/subscriptions/{SubscriptionID}/status
- GET /1.0/msp/subscriptions/{SubscriptionID}/senders/json
---

# Report MSP usage from the Exclaimer Cloud API

The MSP operations return `403 Forbidden` unless the API user is of **MSP type**.
If you get a 403 on every call here with a token that works elsewhere, that is the
reason — not a permissions misconfiguration on the subscription.

## 1. Enumerate MSP subscriptions

```
GET /1.0/msp/subscriptions   # (getMspSubscriptions)
```

Page it. Call with `ContinuationToken=0`, then keep passing back the
`ContinuationToken` from the previous response until the property is **absent**
from the response — not empty, absent. Default page size is 50.

Each `MspSubscription` carries `SubscriptionID`, `Name`, `EndUserID`,
`ResellerID` and `TotalMailboxes`.

## 2. Check the tenant is actually billable

```
GET /1.0/msp/subscriptions/{SubscriptionID}/status   # (getMspSubscriptionStatus)
```

Returns `Status` — one of `Active`, `Suspended`, `Cancelled`, `Ended`, `Unknown` —
and `Onboarded`, whether the tenant finished onboarding. A tenant that is not
`Onboarded` will have no meaningful sender usage.

## 3. Pull the sender usage export

```
GET /1.0/msp/subscriptions/{SubscriptionID}/senders/json?DateFrom=...&DateTo=...
                                                          # (getMspSenders)
```

`SubscriptionID` is a required path parameter; `DateFrom` and `DateTo` are
optional query parameters — set both to bound a billing period rather than
pulling the default window.

Each `MspSenderUsage` row: `UserId`, `Sender`, `Total`, and the breakdown
`ImprintedServer`, `ImprintedCloud`, `ImprintedOnPremises`, plus `Outlook` and
`Gmail`. `Total` is the billable imprint count; the rest tell you which
deployment path produced it.

## 4. Handle the rate limit — this is the one operation that has one

`GET .../senders/json` is the **only** operation in the entire API that declares a
`429 Too Many Requests`. Exclaimer publishes no limit value, no window, and
returns no `Retry-After` or `RateLimit-*` header, so:

- Schedule the export (daily or per billing cycle). Do not poll it.
- On `429`, back off exponentially with jitter. There is no header telling you how
  long to wait.
- Fetch one subscription at a time; do not fan out the export across your whole
  MSP book concurrently.

## 5. Reconcile

Cross-check `Total` against `TotalMailboxes` from step 1 and, if you also hold the
non-MSP view, against `BillableUsers` / `Overage` on
`GET /1.0/subscriptions/{SubscriptionID}`.

## Errors

| Status | Meaning |
|---|---|
| 401 | `ExApiToken` missing or invalid |
| 403 | API user is not an MSP type, or lacks permission for this subscription |
| 404 | Unknown `SubscriptionID`, or wrong tenant path prefix |
| 429 | Rate limited on the sender export — back off |
| 500 | Exclaimer-side fault; retry with backoff |
