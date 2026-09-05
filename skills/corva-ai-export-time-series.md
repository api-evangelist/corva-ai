---
name: corva-export-time-series
description: >-
  Page a large Corva time-series export safely using timestamp-cursor pagination instead of an
  unbounded skip, the way Corva's own query-controls documentation prescribes.
api: Corva Data API
generated: '2026-09-05'
method: generated
source: https://dc-docs.corva.ai/docs/API/Core%20Concepts/query-controls
operations:
  - 'GET https://data.corva.ai/api/v1/data/{provider}/{dataset}/'
  - 'GET https://data.corva.ai/api/v1/data/{provider}/{dataset}/count/'
---

# Export a large time series from Corva

Corva caps a single read at 10,000 records, so any real export is a loop. Corva explicitly
recommends a **timestamp cursor** rather than an increasing `skip`, because deep offsets degrade
and can time out.

## 1. Size the job first

```
GET https://data.corva.ai/api/v1/data/{provider}/{dataset}/count/
  ?query={"asset_id":12345,"timestamp":{"$gte":1710000000,"$lt":1712592000}}
```

Knowing the count up front tells you how many pages to expect and lets you detect a truncated run.

## 2. Page with a cursor, not an offset

```
GET .../api/v1/data/{provider}/{dataset}/
  ?query={"asset_id":12345,"timestamp":{"$gt":<last_timestamp>}}
  &sort={"timestamp":1}
  &limit=10000
  &fields=timestamp,asset_id,data.hole_depth,data.bit_depth
```

The loop:

1. Sort by `timestamp` ascending.
2. Request up to 10,000 records.
3. Record the last `timestamp` you received.
4. Put `timestamp: {"$gt": last_timestamp}` into the next query.
5. Stop when the response is an empty array.

**Collision hazard.** If two records can share a timestamp — and in one-second operational data
they can — a pure `$gt` cursor will silently drop or duplicate rows at the page boundary. Corva
says to add a second stable field to *both* the sort and the cursor. Do that, or verify your row
count against step 1.

## 3. Checkpoint

Persist the last successful cursor after every page. Corva publishes no rate limit and no
`Retry-After`, so a long export can be interrupted by a 429 you cannot predict; a checkpoint means
you resume rather than restart.

## 4. Backoff

Retry transient failures with exponential backoff and jitter. Do not retry 400, 401, 403 or 422
without changing the request — those will fail identically forever.

## Why not `skip`

`skip` exists and defaults to 0, with no documented ceiling. Corva advises against it for large
exports. Deep offset paging makes the database walk and discard everything before the offset, which
gets slower every page, and there is a platform request timeout whose duration Corva does not
publish — so you cannot calculate the offset at which you will start failing. The cursor has none
of these properties.
