---
name: corva-find-well-and-read-data
description: >-
  Resolve a Corva well to its asset_id on the Platform API, then read that well's operational
  dataset records from the Data API. This is Corva's documented two-API flow and the foundation of
  almost every other Corva task.
api: Corva Platform API + Corva Data API
generated: '2026-09-05'
method: generated
source: >-
  Grounded in operations verified present in openapi/_original/ and in Corva's own documentation at
  https://dc-docs.corva.ai/docs/API/overview and /docs/API/choose-the-right-api
operations:
  - 'GET https://api.corva.ai/v2/wells'
  - 'GET https://api.corva.ai/v2/assets'
  - 'GET https://data.corva.ai/api/v1/dataset/company/'
  - 'GET https://data.corva.ai/api/v1/data/{provider}/{dataset}/'
---

# Find a well and read its data

Corva splits its surface across two hosts. You will need both.

| What you need | Host |
| --- | --- |
| Which well is this, and what is its id | `https://api.corva.ai` (Platform API) |
| What happened on that well | `https://data.corva.ai` (Data API) |

## 1. Authenticate

Use one of two headers. There is no OAuth flow.

```
Authorization: API YOUR_API_KEY
```

The literal `API ` prefix **including the trailing space** is required. If you do not have a key,
note that Corva does not enable key creation for most users by default — it is requested through a
Corva representative.

Or mint a JWT with Corva account credentials:

```
POST https://api.corva.ai/v1/user_token
Content-Type: application/json

{"auth": {"email": "...", "password": "..."}}
```

Use the `jwt` field from the response as `Authorization: Bearer <jwt>`. It expires in 7 days by
default — read the token's `exp` claim rather than assuming. Refresh by POSTing
`{"auth": {"refresh_token": "..."}}` to the same endpoint, and **replace both** the stored jwt and
the stored refresh_token from the response. On a 401 from refresh, discard both and re-authenticate;
never resubmit a failed refresh token.

## 2. Find the well and take its asset_id

```
GET https://api.corva.ai/v2/wells?search=<name>&per_page=25
```

Read `attributes.asset_id` from the result.

Use `GET /v2/assets?types[]=well` instead when you need mixed asset types, parent/child hierarchy,
or an exact API-number lookup. **Watch the depth difference**: `/v2/assets` returns the id as the
top-level `id`, while `/v2/wells` nests it under `attributes.asset_id`. Reading the wrong one is
the most common client bug in this flow.

## 3. Find out which datasets you can read

```
GET https://data.corva.ai/api/v1/dataset/company/
```

Datasets are addressed as `provider` + `name`, for example provider `corva`, dataset `corva#wits`.
Each dataset has a `data_type` of `time`, `depth`, `reference` or `timeseries`.

To learn a dataset's field shape before you query it:

```
GET https://data.corva.ai/api/v1/dataset/{provider}/{name}/
```

This matters because the record payload is an open object — the contract does not describe what is
inside any given dataset. This call is how you find out.

## 4. Read the records

```
GET https://data.corva.ai/api/v1/data/{provider}/{dataset}/
  ?query={"asset_id":12345,"timestamp":{"$gt":1710000000}}
  &sort={"timestamp":1}
  &limit=1000
  &fields=timestamp,asset_id,data.hole_depth
```

Rules that are enforced, not advisory:

- `limit` is **required** and must be between 1 and 10,000. Out of range returns 422.
- `query` and `sort` are JSON objects passed as query-string parameters. Encode them with your HTTP
  library's parameter encoder — do not concatenate them into a URL by hand.
- Filter on an indexed field (`asset_id`, `company_id`, time). Sorting on an unindexed field can
  time out.
- Use `fields` to project. It is the cheapest performance win available.

## 5. Handle failure the way Corva documents

| Status | Do |
| --- | --- |
| 400 | Fix malformed JSON or unsupported parameters. Do not retry unchanged. |
| 401 | Check the header, key status, and token expiry. Do not retry unchanged. |
| 403 | Authenticated but not authorized — a valid key still 403s on resources outside its company, owner or permission scope. Do not retry unchanged. |
| 404 | Verify endpoint, provider, dataset and asset ids. |
| 422 | Correct a missing or out-of-range parameter, usually `limit` or a malformed `query`. |
| 429 | Back off. No `Retry-After` or rate-limit header is returned, so use exponential backoff with jitter. |
| 5xx | Retry a bounded number of times, then stop and preserve diagnostics. |

Log the endpoint, status, elapsed time and non-sensitive error detail. **Never** log API keys,
bearer tokens, refresh tokens or passwords.

There is no request-id response header, so correlation is entirely client-side — record your own.
