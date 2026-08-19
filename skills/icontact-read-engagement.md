---
name: icontact-read-engagement
description: >-
  Read engagement for a sent iContact message — aggregate statistics plus the
  per-contact opens, clicks, bounces and unsubscribes. All engagement is
  poll-only and scoped to one message; iContact publishes no engagement
  webhook and no account-level feed. Use when reporting on campaign
  performance or syncing engagement into another system.
api: icontact:rest-api
base_url: https://app.icontact.com/icp
generated: '2026-08-13'
method: generated
grounded_in: documentation
read_only: true
operations:
  - 'GET /a/{accountId}/c/{clientFolderId}/messages/{messageId}/statistics'
  - 'GET /a/{accountId}/c/{clientFolderId}/messages/{messageId}/opens'
  - 'GET /a/{accountId}/c/{clientFolderId}/messages/{messageId}/clicks'
  - 'GET /a/{accountId}/c/{clientFolderId}/messages/{messageId}/bounces'
  - 'GET /a/{accountId}/c/{clientFolderId}/messages/{messageId}/unsubscribes'
  - 'GET /a/{accountId}/c/{clientFolderId}/contacts/{contactId}/actions'
sources:
  - https://help.icontact.com/customers/s/article/Statistics-iContact-API
  - https://help.icontact.com/customers/s/article/Message-Opens-iContact-API
  - https://help.icontact.com/customers/s/article/Message-Clicks-iContact-API
  - https://help.icontact.com/customers/s/article/Message-Bounces-iContact-API
  - https://help.icontact.com/customers/s/article/Unsubscribes-iContact-API
  - https://help.icontact.com/customers/s/article/Contact-History-iContact-API
---

# Read iContact engagement

## Shape of the problem

Engagement in iContact is **poll-only and per-message**. There is no
account-level engagement feed, and the webhook surface carries only four
contact-lifecycle events (`contact_created`, `contact_updated`,
`contact_subscribed`, `contact_unsubscribed`) — nothing about sends, delivery,
bounces, opens, clicks or complaints. To keep an external system current you
must enumerate messages and poll each one.

## Headers

```
Accept: application/json
API-Version: 2.2
API-AppId: <app id>
API-Username: <username>
API-Password: <application password>
```

## Start with the aggregate

```
GET /a/{accountId}/c/{clientFolderId}/messages/{messageId}/statistics
```

Returns a single object — one call, all headline numbers:

```json
{"statistics":{
  "bounces":3,"delivered":26,"unsubscribes":0,
  "opens":{"unique":14,"total":17},
  "clicks":{"unique":3,"total":3},
  "forwards":2,"comments":0,"complaints":0
}}
```

`opens` and `clicks` are nested objects with `unique` and `total`. Everything
else is a flat integer. Use this before paging any detail collection.

Click statistics only exist if the message's campaign had
`clickTrackMode: 1` set at send time. If clicks are zero, check the campaign
before concluding nobody clicked.

## Per-contact detail

Four sibling collections, all GET-only, all paginated:

| Path | Returns |
|---|---|
| `/messages/{messageId}/opens` | Open events |
| `/messages/{messageId}/clicks` | Link click data |
| `/messages/{messageId}/bounces` | `contactId`, `bounceTime` |
| `/messages/{messageId}/unsubscribes` | `contactId`, `unsubscribeTime` |

Bounces and unsubscribes return only an integer `contactId` and an ISO 8601
timestamp — no email address. Resolve the person with
`GET /contacts/{contactId}`.

## Pagination — the trap

**If you omit `limit`, you get 20 rows.** That is the documented default for
*every* GET on this API. A message with 5,000 opens will silently return 20
unless you page it.

```
GET …/opens?limit=100&offset=0
```

Every collection response carries sibling `limit`, `offset` and `total`
members, so read `total` from the first page and loop until
`offset + limit >= total`. No maximum `limit` is published; raise it
cautiously.

Sorting and filtering work here as everywhere else: `?orderby=field:desc`,
per-field query filters, `*` wildcards, and comparison operators via a
`{field}SearchType` parameter (`gt`, `gte`, `lt`, `lte`, `bet`, `eq`).

## Per-contact history

For one person's full timeline rather than one message's audience:

```
GET /a/{accountId}/c/{clientFolderId}/contacts/{contactId}/actions
```

GET-only. Returns the actions that affected a contact — when they were added
and what changed about their subscription.

## Keeping suppression state in sync

If you are mirroring opt-outs into another system, note the documented gap:
the `contact_unsubscribed` webhook fires on list unsubscribes, but iContact
states plainly that **"Do Not Contact" events cannot be monitored**. A contact
moved to `donotcontact` status is invisible to webhook consumers. Poll
`GET /contacts?status=total` and inspect the `status` field to catch those.

Contact status values: `normal`, `bounced`, `donotcontact`, `pending`,
`invitable`, `deleted`.

## Pacing

iContact acknowledges rate limiting but publishes no limit, no window and no
response header, and documents no 429 — so a polling loop cannot detect
throttling at runtime. Its own guidance is to avoid redundant requests. Read
the aggregate `statistics` object first and page detail collections only when
you need per-contact rows.
