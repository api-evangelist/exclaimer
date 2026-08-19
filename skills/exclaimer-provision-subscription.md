---
name: exclaimer-provision-subscription
description: >-
  Provision a new Exclaimer Cloud tenant subscription for a customer as an
  Exclaimer distribution partner — pick the SKU and data centre from reference
  data, create the subscription, add the customer's admin user, and confirm the
  seat count.
generated: '2026-08-13'
method: generated
source: >-
  Grounded in openapi/_original/exclaimer-cloud-api-openapi.json (fetched from
  https://cloudapi.exclaimer.com/openapi.json 2026-08-13) and the Usage section
  of https://cloudapi.exclaimer.com/. The published specification declares NO
  operationIds, so every step names the verified METHOD + PATH; the suggested
  operationIds in parentheses come from overlays/exclaimer-cloud-api-overlay.yaml
  and are ours, not Exclaimer's.
api: Exclaimer Cloud API
base_url: https://cloudapi.exclaimer.com/exclaimerapi
sandbox_url: https://sandbox-cloudapi.exclaimer.com
operations:
- GET /1.0/reference/skus
- GET /1.0/reference/tiers
- GET /1.0/reference/data-centers
- POST /v2.0/subscriptions
- GET /1.0/subscriptions/{SubscriptionID}
- POST /1.0/subscriptions/{SubscriptionID}/users
- GET /1.0/subscriptions/{SubscriptionID}/mailbox-count
---

# Provision an Exclaimer Cloud subscription

Access is restricted to the Exclaimer distributor network. You need a partner
`ExApiToken` issued by Exclaimer and the path prefix you were given (the docs use
`exclaimerapi`; yours may be `yourcompany`). Develop against
`https://sandbox-cloudapi.exclaimer.com` before touching live.

Every request carries `ExApiToken: <your token>` and `Content-Type: application/json`.
Requests accept PascalCase or camelCase; responses are always PascalCase.

## 1. Read the reference data first

Do not hard-code a SKU, a tier or a data centre — they are served.

```
GET /1.0/reference/skus          # SKU code + description  (getSkus)
GET /1.0/reference/tiers         # tier code + description (getTiers)
GET /1.0/reference/data-centers  # Location, Office365Code, GSuiteCode (getDataCenters)
```

`SKU` on the V2 create determines product, tier and NFR status in one field, so
resolve it here rather than guessing.

## 2. Create the subscription

Use the **V2** endpoint. `POST /1.0/subscriptions` was deprecated on 2025-02-17
and new consumers calling it receive `400` or `404`.

```
POST /v2.0/subscriptions          # (addSubscriptionV2)
```

Required body fields: `EmailAddress`, `MailboxCount`, `CountryCode`, `SKU`, and
an `EndUser` object whose `Domain` is required. For non-marketplace subscriptions
`Reseller.Domain` is also required.

Useful optional fields:

- `SubscriptionID` / `UserID` — your own references. Omit them and Exclaimer
  generates values, which you must then store from the response.
- `DataCenter` — omit and a default is chosen from `CountryCode`.
- `CurrencyCode` — **cannot be changed after creation**. Get it right the first time.
- `TrialDays` / `SkipTrial` — `SkipTrial: true` makes the subscription active
  immediately with no trial.
- `SendWelcomeEmailToCustomer` — whether Exclaimer emails the customer to log in.
- `BillingGroup` — optional grouping label.

The response is `AddSubscriptionResponse { SubscriptionID, UserID }`. **Persist both.**

> There is no idempotency key on this API. A retried `POST /v2.0/subscriptions`
> can create a second billable subscription. If the call fails ambiguously, do
> **not** blind-retry — go to step 3 and look first.

## 3. Confirm before you retry anything

```
GET /1.0/subscriptions/{SubscriptionID}   # (getSubscription)
```

Returns `Status`, `InTrial`, `TrialEndDate`, `Tier`, `SKU`, `NFR`,
`NumberOfUsers`, `BillableUsers`, `Overage`, `PendingTransfer`.

If you did not supply a `SubscriptionID`, list instead and filter:

```
GET /1.0/subscriptions?ResellerID=...&ActiveOnly=true   # (getSubscriptions)
```

## 4. Add the customer's admin user

```
POST /1.0/subscriptions/{SubscriptionID}/users   # (addSubscriptionUser)
```

Required: `UserID` (upper/lower letters, numbers, `-` and `_` only),
`EmailAddress`, and `Roles` — an array; the user must have at least one role and
may hold both. `SendWelcomeEmailToCustomer` controls the login invitation.

## 5. Confirm the seat count

```
GET /1.0/subscriptions/{SubscriptionID}/mailbox-count   # (getMailboxCount)
```

Returns `MailboxCount` and `ClientSideMailboxCount`. Adjust later with
`PUT /1.0/subscriptions/{SubscriptionID}/mailbox-count` — note new accounts have a
minimum value restriction.

## Error handling

| Status | Meaning | Do |
|---|---|---|
| 400 | Missing Field / Bad Request | Read the returned JSON for the offending fields. Also returned to new consumers calling deprecated endpoints. |
| 401 | Token missing or invalid | Check the `ExApiToken` header. |
| 403 | Valid token, insufficient permission — or an action that would leave a resource broken | Check your partner type and role. |
| 404 | Bad path, wrong version segment, or unknown id | Verify the tenant prefix and the `SubscriptionID`. |
| 415 | Unsupported media type | Send `Content-Type: application/json`. |
| 500 | Exclaimer-side fault | Retry safe calls with backoff; report persistent failures to support. |

Error bodies have **no published schema**. Branch on the status code; treat the
body as human-readable detail only.
