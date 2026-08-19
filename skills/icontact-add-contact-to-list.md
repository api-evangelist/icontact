---
name: icontact-add-contact-to-list
description: >-
  Create a contact in iContact and subscribe it to a list. Creating a contact
  does NOT put it on a list — a separate subscription must be created. Use when
  adding a new subscriber, syncing a signup, or importing a person into an
  iContact list.
api: icontact:rest-api
base_url: https://app.icontact.com/icp
generated: '2026-08-13'
method: generated
grounded_in: documentation
operations:
  - 'POST /a/{accountId}/c/{clientFolderId}/lists'
  - 'POST /a/{accountId}/c/{clientFolderId}/contacts'
  - 'POST /a/{accountId}/c/{clientFolderId}/subscriptions'
  - 'GET /a/{accountId}/c/{clientFolderId}/contacts'
sources:
  - https://help.icontact.com/customers/s/article/Contacts-iContact-API
  - https://help.icontact.com/customers/s/article/Lists-iContact-API
  - https://help.icontact.com/customers/s/article/Subscriptions-iContact-API
  - https://help.icontact.com/customers/s/article/Code-Library-iContact-API
---

# Add a contact to an iContact list

## Before you start

Send these headers on every request:

```
Accept: application/json
Content-Type: application/json
API-Version: 2.2
API-AppId: <your app id>
API-Username: <your iContact username>
API-Password: <your API application password>
```

`API-Password` is the **application** password set when the integration was
registered under Settings and Billing → iContact Integrations → Custom API
Integrations. It is not the iContact login password.

You also need `accountId` and `clientFolderId`. Read them from Settings and
Billing → iContact Integrations → View Details → Account Information, or
discover them: `GET /a/` returns accounts, `GET /a/{accountId}/c/` returns
client folders.

## The thing that catches everyone

**Creating a contact does not subscribe it to anything.** Contact-to-list is
many-to-many through a `subscriptions` join resource. A contact created and
never subscribed will not appear in a default `GET /contacts` at all, because
that call returns only contacts that are on a list.

## Step 1 — ensure a list exists

If you already have a `listId`, skip to step 2.

```
POST /a/{accountId}/c/{clientFolderId}/lists
[{ "name": "Bakery Newsletter", "publicname": "Bakery Newsletter" }]
```

`name` is the only required field. Do **not** send `listId` on a create.
`description` is internal; `publicname` is what contacts see.

Welcome-message fields on this resource are deprecated — welcome mail is now
handled by Automations.

## Step 2 — create the contact

```
POST /a/{accountId}/c/{clientFolderId}/contacts
[{ "email": "smith@example.com", "firstName": "Mary", "lastName": "Smith" }]
```

- `email` is the only required field, and it must be unique in the client folder.
- Do **not** send `contactId` on a create.
- Optional fields: `prefix`, `firstName`, `lastName`, `suffix`, `street`,
  `street2`, `city`, `state` (max 10 chars), `postalCode`, `phone`, `fax`,
  `business`, `status`.
- `createDate` and `bounceCount` are receive-only.

Read `contactId` out of the response. Because POST on this collection means
"create *or* update", posting an email that already exists updates the existing
contact rather than failing — treat this as an upsert, not a create.

## Step 3 — subscribe the contact to the list

```
POST /a/{accountId}/c/{clientFolderId}/subscriptions
[{ "contactId": 148525, "listId": 137582, "status": "normal" }]
```

- Required on create: `status`, `listId`, `contactId`.
- `status` is one of `normal`, `pending`, `unsubscribed`. Use `pending` if you
  want a confirmed opt-in; supply `confirmationMessageId` to choose the
  confirmation email.
- The resulting `subscriptionId` is a composite: `{listId}_{contactId}` —
  e.g. `137582_148525`. You can construct it rather than storing it.

## Step 4 — verify

```
GET /a/{accountId}/c/{clientFolderId}/contacts?status=total&email=smith@example.com
```

Pass `status=total` or you will only see list-subscribed contacts.

## Moving a contact to a different list

```
PUT /a/{accountId}/c/{clientFolderId}/subscriptions/{listId}_{contactId}
[{ "listId": <new list id> }]
```

This requires `API-Version: 2.2`. Only `listId` is required on the PUT.

## Rules that bite

- **Never PUT a contact to update it.** `PUT /contacts/{contactId}` "deletes all
  data and stores supplied data" — every field you omit is erased. Use
  `POST /contacts/{contactId}` for a merge update.
- **Status transitions are one-way.** You cannot change a contact's status if
  it is currently `donotcontact`, `pending` or `invitable`. Do not try to
  re-subscribe someone who opted out.
- **No idempotency key exists.** If a POST times out, you cannot safely retry it.
  Re-read by `email` first to check whether the write landed.
- **Field names are case sensitive.** `listId` works; `listID` returns nothing.

## Errors

Errors come back as `{"errors":["prose message"]}` with no machine-readable
code — you have to string-match. Common ones:

| Status | Meaning | What to do |
|---|---|---|
| 400 | Payload could not be parsed, or invalid data | Check required fields; make sure you did not send a primary key on a create |
| 401 | Not authorized | All three auth headers required; check you are using the app password |
| 402 | Payment required | The account's bill is unpaid — not an API fault |
| 403 | Forbidden | Check per-client-folder permissions; on an Agency account you must be the admin |
| 501 | Not implemented | Usually a bad `API-Version` — use 2.0, 2.1 or 2.2 |

See `errors/icontact-problem-types.yml` for the full catalog.
