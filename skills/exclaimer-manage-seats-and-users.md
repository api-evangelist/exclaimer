---
name: exclaimer-manage-seats-and-users
description: >-
  Keep an existing Exclaimer Cloud subscription correct over its life — adjust the
  billable mailbox count, add and remove portal users, change roles, and move
  tier, SKU or NFR status without breaking billing.
generated: '2026-08-13'
method: generated
source: >-
  Grounded in openapi/_original/exclaimer-cloud-api-openapi.json (fetched from
  https://cloudapi.exclaimer.com/openapi.json 2026-08-13). The published
  specification declares no operationIds; steps name the verified METHOD + PATH.
api: Exclaimer Cloud API
base_url: https://cloudapi.exclaimer.com/exclaimerapi
operations:
- GET /1.0/subscriptions/{SubscriptionID}/mailbox-count
- PUT /1.0/subscriptions/{SubscriptionID}/mailbox-count
- GET /1.0/mailbox-allocation
- GET /1.0/subscriptions/{SubscriptionID}/users
- GET /1.0/subscriptions/users
- POST /1.0/subscriptions/{SubscriptionID}/users
- PUT /1.0/subscriptions/{SubscriptionID}/users/{UserID}/roles
- DELETE /1.0/subscriptions/{SubscriptionID}/users/{UserID}
- PUT /1.0/subscriptions/{SubscriptionID}/change-tier
- PUT /1.0/subscriptions/{SubscriptionID}/change-sku
- PUT /1.0/subscriptions/{SubscriptionID}/nfr
- POST /1.0/subscriptions/{SubscriptionID}/change-owner
- GET /1.0/subscriptions/{SubscriptionID}/history
---

# Manage seats, users and commercial state on an Exclaimer subscription

All of these are **PUT** except adding a user, changing the owner, and reading —
which means most of them are naturally idempotent and safe to retry. The `POST`s
are not.

## Seats

```
GET /1.0/subscriptions/{SubscriptionID}/mailbox-count   # (getMailboxCount)
PUT /1.0/subscriptions/{SubscriptionID}/mailbox-count   # (updateMailboxCount)
```

`UpdateMailboxCount` requires `MailboxCount` — the **total** for the subscription,
not a delta. New accounts have a minimum-value restriction. The read returns both
`MailboxCount` and `ClientSideMailboxCount`.

Across your whole book:

```
GET /1.0/mailbox-allocation   # (getMailboxAllocation)
```

## Portal users

```
GET  /1.0/subscriptions/{SubscriptionID}/users            # (getSubscriptionUsers)
GET  /1.0/subscriptions/users                             # (getAllSubscriptionUsers) — across every subscription, paged
POST /1.0/subscriptions/{SubscriptionID}/users            # (addSubscriptionUser)
PUT  /1.0/subscriptions/{SubscriptionID}/users/{UserID}/roles   # (updateSubscriptionUserRoles)
DELETE /1.0/subscriptions/{SubscriptionID}/users/{UserID} # (deleteSubscriptionUser)
```

- Adding requires `UserID`, `EmailAddress` and `Roles`. `UserID` accepts only
  letters, digits, `-` and `_`.
- A user must hold at least one role and may hold more than one.
- `PUT .../roles` requires `Roles` and **replaces** the set.
- **Deleting the subscription owner returns 403.** Move ownership first with
  `POST /1.0/subscriptions/{SubscriptionID}/change-owner`, then delete.

`GET /1.0/subscriptions/users` is paged — `ContinuationToken` / `Page` /
`PageSize`, default 50.

## Commercial state

```
PUT /1.0/subscriptions/{SubscriptionID}/change-tier   # (changeSubscriptionTier)
PUT /1.0/subscriptions/{SubscriptionID}/change-sku    # (changeSubscriptionSku)
PUT /1.0/subscriptions/{SubscriptionID}/nfr           # (changeSubscriptionNotForResale)
```

Resolve valid values from `GET /1.0/reference/tiers` and `GET /1.0/reference/skus`
before writing — do not hard-code codes.

`CurrencyCode` is fixed at creation and has no change endpoint.

## Audit what you changed

```
GET /1.0/subscriptions/{SubscriptionID}/history   # (getSubscriptionHistory)
```

Returns `TierHistory[]` and `SkuHistory[]`, each entry carrying `Date`, the new
value and the previous value. This is the record to reconcile a billing dispute
against.

## Rules that bite

- No idempotency keys anywhere. `POST .../users` retried after a timeout can
  create a duplicate — `GET` the user list first.
- Responses are always PascalCase; requests accept either casing, but **path
  parameters and JSON values are case-sensitive**.
- `415` means you forgot `Content-Type: application/json`.
- Error bodies have no published schema. Branch on the status code.
