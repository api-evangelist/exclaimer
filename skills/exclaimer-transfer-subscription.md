---
name: exclaimer-transfer-subscription
description: >-
  Move an Exclaimer Cloud subscription between partners — initiate the transfer to
  get a transfer token, hand it to the receiving partner who claims it, and cancel
  a transfer that was not claimed.
generated: '2026-08-13'
method: generated
source: >-
  Grounded in openapi/_original/exclaimer-cloud-api-openapi.json (fetched from
  https://cloudapi.exclaimer.com/openapi.json 2026-08-13). The published
  specification declares no operationIds; steps name the verified METHOD + PATH.
api: Exclaimer Cloud API
base_url: https://cloudapi.exclaimer.com/exclaimerapi
operations:
- POST /1.0/subscriptions/{SubscriptionID}/transfer
- POST /1.0/subscriptions/transfer/claim
- PUT /1.0/subscriptions/{SubscriptionID}/transfer/cancel
- GET /1.0/subscriptions/{SubscriptionID}
---

# Transfer an Exclaimer Cloud subscription between partners

This is a **two-party, token-mediated** flow. The current holder issues a token;
the receiving partner redeems it. Commercial ownership of a live customer tenant
moves when the claim succeeds — treat it as irreversible.

## 1. Initiate (current holder)

```
POST /1.0/subscriptions/{SubscriptionID}/transfer   # (initiateSubscriptionTransfer)
```

Returns `InitiateSubscriptionTransferResponse { TransferToken }`.

Deliver the `TransferToken` to the receiving partner out of band. It is the only
credential needed to claim the subscription — handle it like a secret.

After this call, `GET /1.0/subscriptions/{SubscriptionID}` reports
`PendingTransfer: true`.

## 2. Claim (receiving partner)

```
POST /1.0/subscriptions/transfer/claim   # (claimSubscriptionTransfer)
```

Required: `TransferToken`. Optional, and worth setting at claim time because it
saves a follow-up write:

- `SubscriptionID` — your own reference for the subscription after transfer. If
  omitted, or if one with that reference already exists on your account, a value
  is generated.
- `ResellerID` **or** a `Reseller` object — associate the subscription with a
  reseller record on your account.
- `EndUserID` **or** an `EndUser` object — associate it with an end-user record.

Supply either the id of an existing record or the object to create one, not both.

## 3. Cancel (current holder, before it is claimed)

```
PUT /1.0/subscriptions/{SubscriptionID}/transfer/cancel   # (cancelSubscriptionTransfer)
```

Invalidates the outstanding token. Confirm with
`GET /1.0/subscriptions/{SubscriptionID}` that `PendingTransfer` has returned to
`false`.

## Cautions

- **No idempotency key.** `POST .../transfer` is not idempotent — a retry issues a
  second token. Read the subscription and check `PendingTransfer` before retrying.
- A `403` on any of these means the token is valid but the caller lacks the
  permission, or the action would leave the resource in a problem state.
- A `404` on claim means the token is unknown, already used, or cancelled.
- Error bodies carry no published schema; branch on the HTTP status.
