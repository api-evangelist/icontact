---
name: icontact-search-contacts
description: >-
  Query the iContact contacts collection correctly. Two undocumented-looking
  defaults silently under-report — a 20-row page cap and a list-membership
  filter — and both will make an agent report a wrong count. Use when
  searching, exporting, counting or reconciling iContact contacts.
api: icontact:rest-api
base_url: https://app.icontact.com/icp
generated: '2026-08-13'
method: generated
grounded_in: documentation
read_only: true
operations:
  - 'GET /a/{accountId}/c/{clientFolderId}/contacts'
  - 'GET /a/{accountId}/c/{clientFolderId}/contacts/{contactId}'
  - 'GET /a/{accountId}/c/{clientFolderId}/lists'
sources:
  - https://help.icontact.com/customers/s/article/Contacts-iContact-API
  - https://help.icontact.com/customers/s/article/Advanced-Users-iContact-API
---

# Search iContact contacts

## The two defaults that will make you wrong

1. **`limit` defaults to 20.** Not to "all", and not to 100. If you omit
   `limit`, you get 20 contacts even when the account holds 50,000.
2. **`GET /contacts` returns only contacts that are on a list.** Contacts
   created but never subscribed are excluded by default.

Both are silent. Neither raises an error. An agent that omits both parameters
will confidently report a contact count that is wrong twice over.

## The corrected baseline call

```
GET /a/{accountId}/c/{clientFolderId}/contacts?status=total&limit=100&offset=0
```

`status` on the query string selects the population:

| Value | Population |
|---|---|
| *(omitted)* | Only contacts on at least one list |
| `unlisted` | Only contacts not subscribed to any list |
| `total` | All contacts |

Read `total` from the response envelope and page until
`offset + limit >= total`. Every collection response carries sibling `limit`,
`offset` and `total` members alongside the contacts array. No maximum `limit`
is published.

## Filtering

Any documented field becomes a query parameter:

```
?status=total&firstName=James
?status=total&listId=123456
?status=total&city=Durham&state=NC
```

Commonly searchable: `listId`, `firstName`, `lastName`, `street`, `city`,
`state`, `postalCode`, `email`.

- **Field names are case sensitive.** `listId` works; `listID` returns nothing
  and does not error.
- **An empty value returns nothing.** `?listId=` returns zero rows rather than
  being ignored — never build a query string with an unset variable.
- Some fields accept comma-separated multiple values:
  `?[term]=[value1],[value2]`.

### Wildcards

Use `*` for a partial match:

```
?status=total&email=j*
```

### Comparison operators

Pass a companion `{field}SearchType` parameter:

```
?createDate=2026-01-01&createDateSearchType=gte
```

| Keyword | Meaning |
|---|---|
| `eq` | equals (the default) |
| `gt` | greater than |
| `gte` | greater than or equal |
| `lt` | less than |
| `lte` | less than or equal |
| `bet` | inclusive between — supply two comma-separated values |

`bet` takes both bounds in the value:

```
?createDate=2026-01-01,2026-06-30&createDateSearchType=bet
```

### Timestamps

ISO 8601, `YYYY-MM-DD[THH:MM:SS[±HH:MM]]`. All of `2026-09-16T14:30:00-06:00`,
`2026-09-16T14:30:00` and `2026-09-16` are valid.

To align a time-window query with the server clock, read it first — this is the
one endpoint on the API that needs no authentication:

```
GET https://app.icontact.com/icp/time
→ {"time":"2026-08-13T13:30:19-04:00","timestamp":1786642219}
```

## Sorting

```
?orderby=lastName:desc,firstName
```

Comma-separated field list, ascending by default, `:desc` per field. `:asc` is
accepted but redundant.

## Reading a single contact

```
GET /a/{accountId}/c/{clientFolderId}/contacts/{contactId}
```

Returns the full record. Receive-only fields worth noting: `createDate` and
`bounceCount`. `status` is one of `normal`, `bounced`, `donotcontact`,
`pending`, `invitable`, `deleted` — check it before treating anyone as
mailable.

## Pacing an export

iContact's own guidance: "mind the servers … if your API is making a lot of
redundant requests … you become more likely to be impacted by our rate
limiting." No limit, window or response header is published, and no 429 is
documented, so throttling is undetectable at runtime. For a full export, page
with a large `limit`, do not run parallel pagers against the same client
folder, and back off on any 503.

## Errors

`{"errors":["prose message"]}` — no machine-readable code. A 401 means the
three auth headers are wrong or missing; a 501 usually means a bad
`API-Version` (use 2.0, 2.1 or 2.2). See `errors/icontact-problem-types.yml`.
