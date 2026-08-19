---
name: icontact-create-and-send-message
description: >-
  Compose an email message in iContact and dispatch it to lists or segments.
  IRREVERSIBLE — creating a send delivers real email to real people, and
  iContact provides no sandbox, no dry-run and no idempotency key. Use when
  sending or scheduling an email campaign through the iContact API.
api: icontact:rest-api
base_url: https://app.icontact.com/icp
generated: '2026-08-13'
method: generated
grounded_in: documentation
destructive: true
operations:
  - 'GET /a/{accountId}/c/{clientFolderId}/campaigns'
  - 'POST /a/{accountId}/c/{clientFolderId}/campaigns'
  - 'POST /a/{accountId}/c/{clientFolderId}/messages'
  - 'POST /a/{accountId}/c/{clientFolderId}/sends'
  - 'GET /a/{accountId}/c/{clientFolderId}/sends'
sources:
  - https://help.icontact.com/customers/s/article/Campaigns-iContact-API
  - https://help.icontact.com/customers/s/article/Messages-iContact-API
  - https://help.icontact.com/customers/s/article/Sends-iContact-API
  - https://help.icontact.com/customers/s/article/Code-Library-iContact-API
---

# Create and send an iContact message

## Stop and read this first

There is **no sandbox and no dry-run**. iContact's own Getting Started guide
says to test against a free *production* account and warns: "All changes in
contacts, messages, sends or any other resource will modify your live data."

There is **no idempotency key**. If `POST /sends` times out, you cannot tell
whether the send fired, and retrying may double-send. Always resolve a timeout
by reading `GET /sends` before retrying.

Do not call this flow autonomously without an explicit, scoped human approval
that names the message and the recipient lists.

## Headers

```
Accept: application/json
Content-Type: application/json
API-Version: 2.2
API-AppId: <app id>
API-Username: <username>
API-Password: <application password>
```

## Step 1 — pick or create a campaign (the sender identity)

A campaign is iContact's sender property: from-name, from-address, footer
address, tracking settings. A message cannot exist without one.

```
GET /a/{accountId}/c/{clientFolderId}/campaigns
```

To create one, all of these are required:

```
POST /a/{accountId}/c/{clientFolderId}/campaigns
[{
  "name": "Campaign to increase sales",
  "fromName": "Bob Smith",
  "fromEmail": "smith@example.com",
  "subscriptionManagement": 0,
  "clickTrackMode": 1,
  "useAccountAddress": 1,
  "archiveByDefault": 0
}]
```

- `subscriptionManagement` selects the unsubscribe wording: `0` Manage your
  subscription, `1` update/change profile, `2` to be removed click here,
  `3` Spanish, `4` French.
- `clickTrackMode` — `1` enables click tracking; without it you get no click
  statistics later.
- `useAccountAddress` — `1` takes the physical mailing address from the account,
  `0` from the campaign's own `street`/`city`/`state`/`zip`/`country`. iContact
  notes a mailing address must appear at the bottom of every email to comply
  with CAN-SPAM.
- `archiveByDefault` — `1` publishes the email to the iContact community
  archive. Set `0` unless you intend that.

## Step 2 — create the message

```
POST /a/{accountId}/c/{clientFolderId}/messages
[{
  "campaignId": 248,
  "messageType": "normal",
  "subject": "Symposium speakers sought!",
  "messageName": "September VIP sale",
  "htmlBody": "<![CDATA[<p>…</p>]]>",
  "textBody": "…",
  "previewText": "This is preview text."
}]
```

- Required: `campaignId`, `messageType`, `subject`.
- `messageType` is `normal` (an email) or `confirmation` (a subscription
  confirmation request). `welcome` and `autoresponder` are **deprecated** —
  use Automations instead.
- `htmlBody` must be well-formed XML. Wrap it in `<![CDATA[ … ]]>` if you are
  not certain, as the docs recommend.
- Do not send `messageId` on a create. Read it from the response.
- Messages cannot be deleted through the API — `DELETE` is documented as not
  supported. A message you create is permanent.

To edit a draft afterwards use `POST /messages/{messageId}`, which merges only
the fields you supply.

## Step 3 — send

```
POST /a/{accountId}/c/{clientFolderId}/sends
[{
  "messageId": 395872,
  "includeListIds": "1245625,235876",
  "excludeListIds": "234757",
  "includeSegmentIds": "273856",
  "excludeSegmentIds": "2357682",
  "scheduledTime": "2026-09-15T16:45:00-04:00"
}]
```

- Required: `messageId`, and `includeListIds` — **except** when sending only to
  a segment, which is the one documented case where a list id is not required.
- Id fields are comma-separated integer strings.
- Exclusions are subtractive: send to list A excluding list B, and anyone on
  both is dropped.
- `scheduledTime` must be **exactly** `YYYY-MM-DDTHH:MM:SS-04:00` in Eastern
  Time. This is stricter than the ISO 8601 handling elsewhere in the API — do
  not send a UTC offset, and do not send a date-only value.
- Omit `scheduledTime` to send immediately.
- Do not send `sendId` on a create.

## Step 4 — confirm

The send response and subsequent `GET /sends` return receive-only fields:

- `recipientCount` — how many subscribers the message will go to. **Check this
  before it releases.** It is the only pre-flight number the API gives you.
- `status` — `pending`, `sending`, `released` or `cancelled`.
- `releasedTime` — when iContact actually sent it.

A scheduled send sits at `pending` until its `scheduledTime`, which is the only
window in which anything can be reconsidered.

## After the send

Engagement data is poll-only — there is no send, delivery, bounce, open or
click webhook. See the `icontact-read-engagement` skill.

## Failure handling

| Status | Meaning | What to do |
|---|---|---|
| 400 | Invalid data | Most often a malformed `scheduledTime` or non-XML-safe `htmlBody` |
| 402 | Payment required | Unpaid account; the send will not go out |
| 403 | Forbidden | Permissions on the client folder, or not an Agency admin |
| 500 | Server error | **Do not blind-retry a send.** `GET /sends` first to see whether one was created |
| 503 | Service unavailable | Check https://status.icontact.com/ — it has an "API System" component |
| 507 | Insufficient space | Plan storage quota exhausted (250 MB / 500 MB / 1 GB by plan) |

iContact acknowledges it enforces rate limiting but publishes no limit, no
window and no response header, and documents no 429. Throttling is therefore
undetectable from headers — pace writes conservatively.
