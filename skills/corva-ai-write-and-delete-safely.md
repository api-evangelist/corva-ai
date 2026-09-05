---
name: corva-write-and-delete-safely
description: >-
  Write records into a Corva dataset and delete them without duplicating or destroying data, on an
  API that provides no idempotency mechanism and no undo. Read this before any Corva write.
api: Corva Data API
generated: '2026-09-05'
method: generated
source: >-
  https://dc-docs.corva.ai/docs/API/Core%20Concepts/limits-and-performance and the write operations
  declared in openapi/_original/corva-ai-data-api-openapi-original.json
operations:
  - 'POST https://data.corva.ai/api/v1/data/{provider}/{dataset}/'
  - 'PATCH https://data.corva.ai/api/v1/data/{provider}/{dataset}/{id}/'
  - 'DELETE https://data.corva.ai/api/v1/data/{provider}/{dataset}/{id}/'
  - 'DELETE https://data.corva.ai/api/v1/data/{provider}/{dataset}/'
  - 'GET https://data.corva.ai/api/v1/data/{provider}/{dataset}/count/'
---

# Write and delete Corva data safely

Two facts govern everything below, and both are absences rather than features.

1. **There is no idempotency mechanism.** No `Idempotency-Key` header, no client request id, no
   replay window, no conflict semantics — across all 26 mutating Data API operations and all 451
   mutating Platform API operations.
2. **There is no undo.** No restore, trash, soft-delete or point-in-time recovery operation exists
   anywhere in the Corva API.

## Writing

```
POST https://data.corva.ai/api/v1/data/{provider}/{dataset}/
Authorization: API YOUR_API_KEY
Content-Type: application/json

[ { "asset_id": 12345, "timestamp": 1710000000, "company_id": 67, "data": { ... } } ]
```

- 1 to 1,000 records per request. Outside that range you get 422.
- The record envelope is `company_id`, `asset_id`, `version`, `provider`, `collection`,
  `timestamp`, and a free-form `data` object.
- You can only write to datasets your key is permitted to write; a dataset's `permission_workflow`
  defaults to `request` for read, write and delete.

### The retry problem

If a POST times out, you do not know whether it committed. Retrying may insert up to 1,000
duplicate records; not retrying may lose them. Nothing in the API distinguishes the two.

Mitigations, in order of preference:

1. **Make your payload naturally unique and check before retrying.** Query for a record with the
   same `asset_id` + `timestamp` (+ your own marker field inside `data`) before re-POSTing.
2. **Write in small, replayable batches** and record which batch committed. Smaller batches shrink
   the blast radius of an ambiguous failure.
3. **Stamp your own correlation id inside `data`.** The API will not give you one — there is no
   request-id response header — so put one in the payload where you can query it back.

Do not assume a retry is safe because the method is a POST. It is not.

## Deleting

```
DELETE https://data.corva.ai/api/v1/data/{provider}/{dataset}/{id}/     # one record
DELETE https://data.corva.ai/api/v1/data/{provider}/{dataset}/          # MANY records by query
```

The second form is the most dangerous call in the Corva API: it removes every record matching the
query, in one request, permanently.

**Always rehearse first.** Corva provides no dry-run parameter, but it does provide a counter:

```
GET https://data.corva.ai/api/v1/data/{provider}/{dataset}/count/?query=<the exact same query>
```

Run the count with the identical query object you are about to delete with. If the number is not
what you expect, stop. This is the only rehearsal available, and it is a separate call against a
separate code path, so treat it as a strong indicator rather than a guarantee.

Checklist before any bulk delete:

- [ ] Ran `/count/` with the byte-identical `query` object.
- [ ] The count matches expectation.
- [ ] The `query` includes `asset_id` and/or `company_id` — never delete on an unscoped filter.
- [ ] Exported the matching records first, if they are recoverable at all.
- [ ] Confirmed you are on the intended `{provider}/{dataset}` — dataset names differ only by
      suffix in many Corva namespaces.

## Updating

`PATCH /api/v1/data/{provider}/{dataset}/{id}/` updates one record;
`PATCH .../batch_update/` and `.../batch_update_with_object/` update many. The same no-idempotency
rule applies, but PATCH-to-a-known-state is naturally re-runnable — prefer it over
delete-then-insert wherever the shape allows.

## Reversal elsewhere in the platform

Some Platform API surfaces *are* reversible — subscription `cancel`, API key `deactivate`, alert
definition `enable`/`disable`, and `PATCH /v2/partial_reruns/{id}/cancel`. None of them publish a
window, so do not assume how long you have to change your mind. Dataset records have no reversal
path at all.
