# Error → cause → recovery

## Governor limits (the batch failed loudly)

| Error | Cause | Recovery |
| --- | --- | --- |
| `System.LimitException: Too many query rows: 50001` | Unbounded `SELECT` over the cohort | Add `ORDER BY Id LIMIT :batchSize` + keyset pagination. Retryable immediately. |
| `System.LimitException: Too many DML rows: 10001` | Single `update` over the whole cohort | Chunk to 200. Retryable immediately. |
| `System.LimitException: Apex CPU time limit exceeded` | Batch too wide, or many `FieldMapping__c` entries inflating the tracked-field comparison | Halve the batch size, resume from `lastId`. If it recurs at 50, narrow the `SELECT` to `Id` only. |
| `System.LimitException: Apex heap size too large` | Wide `SELECT` over many rows | Select `Id` only; the trigger re-queries what it needs. |
| `Too many queueable jobs added to the queue: 51` | >50 queueables in one transaction | Batch size ≤ 2,000. Retryable after re-chunking. |

## Silent failures (nothing raised — these are the ones that bite)

| Symptom | Cause | Recovery |
| --- | --- | --- |
| `EnableSync = true`, `Id__c = null`, `SyncError__c = null`, queue empty | The enqueue was skipped — `publishSyncRecordEvents` checks `Limits.getQueueableJobs() < Limits.getLimitQueueableJobs()` and, when it fails, logs a debug and drops the events | Toggle off/on in 200-record batches. This is cohort A2/C2 and the main reason the repair pass exists. |
| DML succeeded but nothing ever syncs, org-wide | `MonetizeNowSettings__c.DisableTriggers__c = true` | Clear the setting, then re-run the repair. Everything flipped while it was set is stalled. |
| Contact flipped but never syncs | `Email = null`, or `Account.Id__c` still null (`AccountSyncEnabled__c` is false) | No retry helps. Fix the email, or sync the account first. Report as skipped. |
| Record re-saved, no sync | No *tracked* field changed — `EnableSync__c` was already `true` | Must be off→on across two transactions. |
| Sync appears to run but nothing reaches MonetizeNow | Transaction originated from the MonetizeNow connector (`mnSync=true` on the request URL), so `syncEnabled` is false | Not applicable to anonymous Apex; if seen, the work is being driven through the connector — drive it directly instead. |

## Transient / concurrency

| Error | Recovery |
| --- | --- |
| `UNABLE_TO_LOCK_ROW` | Retry the batch up to 3× with backoff (5s, 15s, 45s). Then halve the batch size. Common when many records share a parent, or another job/integration is touching the same rows. Run the backfill during a quiet window. |
| `ENTITY_IS_LOCKED` | Record is in an approval process or locked. Not retryable — send to `failures.csv`. |
| `Read timed out` / callout timeout in the queueable | 120s cumulative callout budget per transaction; MonetizeNow is responding slowly. Reduce batch size and pace the run. If it persists across batches, stop and raise it — retrying just generates more load. |
| `REQUEST_LIMIT_EXCEEDED` | Org's 24h API limit. **Stop the run.** Report progress and remaining work; resume after the limit resets. Do not retry in a loop. |
| `CONCURRENT_REQUESTS_LIMIT_EXCEEDED` | Too many long-running requests. Pause 60s, resume at a smaller batch size. |

## MonetizeNow rejections (surfaced in `SyncError__c`)

These are data problems. Toggling off/on will reproduce the identical error — fix the data
or report it; do not loop.

| Pattern | Meaning | Action |
| --- | --- | --- |
| `Unauthorized` / auth error text | Package auth (`MonetizeNowAuth`) is broken or the tenant is wrong | **Stop the whole run.** Every record will fail identically. Fix auth, verify with one record, restart. |
| Invalid configuration | Missing/invalid `Domain__c`/`TenantId__c` in `MonetizeNowSettings__c` | Stop; fix settings. |
| Not found (404) | Endpoint or tenant mismatch | Stop; verify the org points at the right MonetizeNow tenant. |
| Validation-style `key: details` (missing country, invalid currency, duplicate email…) | Per-record data problem | Report grouped by message in `failures.csv`. Retry only after the data is corrected. |
| `Unexpected error …` | Unhandled exception during sync | Retry once. If it repeats, escalate with the record Ids and `LastSyncDate__c`. |

`SyncError__c` is truncated to 1000 characters — the message may be cut off. Group by the
first ~120 characters when tallying.

## Salesforce-side DML rejections during the flip

Validation rules, required fields, triggers from other packages, and field-level security
can all reject the `EnableSync__c` update itself. Because the templates use
`Database.update(records, false)`, these come back per-record — collect them into
`failures.csv` with their `StatusCode`, and report them as "never attempted", distinct from
"attempted and rejected by MonetizeNow". The two need very different follow-up.

## Retry policy

- Transient (`UNABLE_TO_LOCK_ROW`, `CONCURRENT_REQUESTS_LIMIT_EXCEEDED`, one-off timeouts):
  max 3 attempts, exponential backoff.
- Limit errors: re-chunk and retry once — if the smaller batch fails the same way, stop and
  report; something else is wrong.
- Auth / configuration / 4xx from MonetizeNow: **zero** retries. One failure means all of
  them fail.
- Data validation: zero retries. Report.

Never leave a retry loop unbounded, and never let a repair run more than twice over the
same records — a record that is still stalled after two clean passes needs a human.
