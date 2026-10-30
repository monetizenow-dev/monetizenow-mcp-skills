---
name: monetizenow-sync-migration
description: Migrate Salesforce Accounts and their Contacts into MonetizeNow by enabling sync on the records a user selects, at any data volume. Use when someone asks to sync/migrate/onboard Salesforce accounts and contacts to MonetizeNow, enable MonetizeNow sync for a set of customers, prepare accounts so quotes can be created in MonetizeNow, backfill records that were never synced, or fix records that are stuck or showing a sync error. Picks the right Salesforce API for the volume, paces the work to stay inside governor limits, and reports what landed in MonetizeNow and what did not. Requires a Salesforce MCP connection or the sf CLI; uses the MonetizeNow MCP connection for verification when available.
---

# Migrate Salesforce Accounts & Contacts into MonetizeNow

The MonetizeNow managed package syncs a Salesforce record to MonetizeNow when
`monetize__EnableSync__c` is set to true on it. This skill turns "sync these customers"
into a safe, paced, fully-reported migration — typically run right before someone starts
building quotes in MonetizeNow, so the accounts and contacts they need are already there.

Your job: take the user's selection criteria, work out the real volume, pick the right API
for that volume, flip sync on without silently dropping records, wait for it to land, and
report exactly what made it into MonetizeNow.

Read `references/how-sync-works.md` before improvising. The sync path, the eligibility
rules, and the limits that actually bind are all there, and several of them are
counterintuitive.

## The three things that will bite you

1. **A contact cannot sync until its account has landed in MonetizeNow.** Not "account
   enabled" — account *synced*, with `monetize__Id__c` populated. Always do accounts first,
   wait for them to drain, then contacts.
2. **Flipping more than ~10,000 records in one transaction silently drops syncs.** Past 50
   queueables the package logs a debug and moves on; the update reports success and those
   records never sync. Batching at 200 is what prevents this — not an optimization.
3. **A record that is already `EnableSync = true` will not re-sync on a plain re-save.**
   Sync fires on a *tracked field change*. Re-driving a stuck record means toggling off,
   then on, in two separate transactions.

## Procedure

Phases in order. Never write before the user has approved the plan in phase 3.

### Phase 1 — Preflight

1. **Confirm the target org** and state it back to the user before any write. Prefer the
   Salesforce MCP connection; `sf org display --target-org <alias>` otherwise.
2. **Detect the namespace.** In a customer org the package is namespaced, so fields are
   `monetize__EnableSync__c`. Probe with a `LIMIT 1` query; if the field errors, retry
   unprefixed (`EnableSync__c`) — an unpackaged org. Use the detected prefix everywhere;
   never hardcode it.
3. **Read the settings:**
   `SELECT Id, monetize__DisableTriggers__c, monetize__SyncAllNewRecords__c, monetize__TenantId__c, monetize__Domain__c FROM monetize__MonetizeNowSettings__c`
   - `DisableTriggers__c = true` → **stop.** Every sync trigger is bypassed; flipping the
     checkbox would do nothing. Ask the user to clear it first.
   - Confirm `TenantId__c` / `Domain__c` point at the MonetizeNow tenant the user expects.
     If the MonetizeNow MCP connection is available, check it resolves to the same tenant —
     syncing a customer's data into the wrong tenant is not quietly undoable.
4. **Prove the path works on one record** before mass-flipping anything. Pick one account
   that matches the criteria, enable sync, wait, and confirm `monetize__Id__c` populates
   (and, via the MonetizeNow MCP, that the account is visible there). If this fails, a
   migration just produces N copies of the same error.

### Phase 2 — Turn the user's criteria into a scope, then size it

The user supplies the selection criteria. Common shapes: accounts with a closed-won
opportunity, a named account list, a record type, an owner or region, a customer-status
field, a created/modified window. Restate the criteria as a concrete SOQL `WHERE` clause
and **show it to the user** — a misread criterion is the most expensive mistake available
here, and it is cheap to catch now.

Then run the assessment queries in `references/soql-cohorts.md` (`SELECT COUNT()` — never
pull rows to count them) and classify:

**Accounts in scope**
- `A1 ready to enable` — `EnableSync = false`, `Id__c = null` → the migration target
- `A2 stalled` — `EnableSync = true`, `Id__c = null`, no error → enqueued and dropped, or
  still in flight. Recoverable by toggling off/on.
- `A3 errored` — `EnableSync = true`, `Id__c = null`, `SyncError__c` populated → MonetizeNow
  rejected it. **Show the user the actual messages.** Most are data problems (missing
  billing country, invalid currency) that no amount of retrying fixes.
- `A4 already synced` — `Id__c` populated. Nothing to do, unless `SyncError__c` is also
  set, meaning the last update failed.

**Contacts under those accounts**
- Same four cohorts, plus two that are report-only:
- `C5 no email` — contact sync requires `Email != null`. These will **never** sync. Count
  them, name them in the report, do not flip them.
- `C6 waiting on the account` — parent account not yet in MonetizeNow. Expected at this
  point; they become eligible once the account pass drains.

Also check the async backlog so you are not piling onto a saturated queue:
`SELECT COUNT() FROM AsyncApexJob WHERE JobType = 'Queueable' AND Status IN ('Queued','Processing')`

### Phase 3 — Pick the approach for the volume, then confirm

Choose the read and write strategy from the total record count (accounts + the contacts
that will follow them). `references/api-selection.md` has the full decision table and the
reasoning; the short version:

| In-scope records | Reads | Writes | Driving |
| --- | --- | --- | --- |
| < 2,000 | plain SOQL via MCP | chunked anonymous Apex, 200/batch | drive batches inline; a run is minutes |
| 2,000 – 50,000 | SOQL with keyset pagination | chunked anonymous Apex, 200/batch, gated on queue depth | write a ledger, resume on interruption, report progress as you go |
| > 50,000 | Bulk API 2.0 query (`sf data query --bulk`) for listings; `COUNT()` for sizing | same 200/batch for the sync-enabling pass; Bulk API only for passes that publish no sync events | run it as a background job with a ledger; plan for multiple sessions and a 24h API-limit ceiling |

The batch size for the **enabling** pass is 200 at every volume. Volume changes how you
*read*, how you *pace*, and how you *drive* the run — it never buys you a bigger write
batch, because the failure a bigger batch causes is silent.

Present to the user and get an explicit go-ahead:

- The `WHERE` clause you derived from their criteria, and the count it matches.
- Cohort counts for accounts and contacts.
- **The callout estimate** — accounts to enable, plus the contacts that will fan out from
  them. This is how many API calls MonetizeNow receives.
- Batch size, pacing, and a rough wall-clock estimate (~75 callouts per queueable, chained).
- For `A3`/`C3`, the top error messages verbatim and your read on whether a retry can help.
- Whether to include re-driving stalled/errored records, or only enable the fresh ones.

### Phase 4 — Execute

Templates in `references/apex-templates.md`. Rules that do not bend:

1. **Every query bounded** — `ORDER BY Id LIMIT :batchSize` with keyset pagination
   (`Id > :lastId`). Never an unbounded select over the scope.
2. **One DML per batch, 200 records** — exactly one trigger invocation, one queueable.
3. **`Database.update(records, false)`** — partial success, so one bad row never costs you
   the batch. Collect the per-record failures.
4. **Ledger after every batch** — write `state.json` to a working directory
   (`{ phase, object, cohort, whereClause, lastId, processed, succeeded, failed[], startedAt }`).
   An interrupted run resumes from `lastId` instead of starting over.
5. **Gate on queue depth** — pause while queued+processing queueables exceed ~100.
6. **Accounts fully drained before contacts.** Poll until the queue is empty, then re-run
   the contact cohort counts: many `C6` contacts will have become eligible, and some will
   already have synced on their own through the package's account→contact fan-out.

**Re-driving stalled or errored records (A2/A3/A4, C2/C3/C4)**

Two passes, two separate transactions:

- **Pass 1 — off.** `EnableSync = false` publishes no sync events, so this is cheap and can
  use large batches or Bulk API. **Record the Ids you turned off** — afterwards they no
  longer match the "stalled" predicate, so pass 2 must target them explicitly.
- **Pass 2 — on.** The expensive pass. Chunk at 200 exactly like a fresh migration.

Two side effects to state before running it: re-driving an already-synced account sends a
PUT to MonetizeNow (an update, not a duplicate, but real traffic), and enabling an account
also re-syncs its unsynced eligible contacts — a 500-account repair can be several thousand
callouts.

### Phase 5 — Verify

1. Wait for the queue to drain. Treat it as drained only when it reads zero **twice, 90
   seconds apart** — the account→contact fan-out enqueues a delayed wave a minute behind
   each account batch.
2. Re-run the phase 2 cohort counts and diff against the start.
3. **Confirm on the MonetizeNow side.** `monetize__Id__c` being populated is the Salesforce
   view of success; if the MonetizeNow MCP connection is available, spot-check that those
   accounts and contacts are actually present and usable for quoting. Disagreement between
   the two sides matters more than either alone.
4. Anything still stalled after a drained queue was dropped — re-drive it once. If it
   survives two clean passes, escalate rather than looping.

### Phase 6 — Report

Always produce **both**:

1. **Files on disk** — `report.md` (narrative) and `failures.csv`
   (`Id,Name,Object,Cohort,SyncError,DmlError`) so the user can work the exceptions in a
   spreadsheet or load them back into Salesforce.
2. **A shareable page**, when the environment supports publishing one (an Artifact, or
   whatever the host offers) — publish the report as an HTML page and hand the user the
   link. If nothing like that is available, say so and point at the files instead.

The report must answer, without the user having to infer anything:

- The criteria used, as the literal `WHERE` clause, and how many records matched.
- How many accounts and contacts **reached MonetizeNow** (`Id__c` populated) — not how many
  checkboxes were flipped.
- What failed, grouped by error message with counts, and for each group whether it is
  retryable, a data fix, or a configuration problem.
- What was deliberately skipped and why — contacts with no email, contacts whose account
  never landed, non-retryable errors.
- What the user should do next, per failure group.
- If the run was partial: where it stopped, and how to resume.

## Recovery

Match the error in `references/troubleshooting.md`. The ones that matter most:

| Error | Recovery |
| --- | --- |
| `Too many SOQL query rows` / `Too many DML rows` | A bounded-query or batch-size rule was broken. Re-chunk. |
| `Apex CPU time limit exceeded` | Halve the batch size, resume from `lastId`. |
| `UNABLE_TO_LOCK_ROW` | Retry that batch up to 3× with backoff (5s, 15s, 45s), then halve the batch size. Usually a concurrent process or a shared parent record. |
| `REQUEST_LIMIT_EXCEEDED` | Org API limit hit. **Stop.** Report remaining work and the reset time; do not retry in a loop. |
| Records enabled but never synced, queue empty | The enqueue was dropped. Re-drive with off/on at a smaller batch size. |
| Callout timeouts | 120s cumulative callout budget per transaction. Reduce batch size; if it persists, MonetizeNow is responding slowly — pause and raise it. |
| Auth / configuration / 4xx from MonetizeNow | **Zero retries.** One failure means every record fails the same way. Stop and fix. |
| Validation rules, required fields, missing data | Per-record data problems. They belong in `failures.csv`, not a retry loop. |

Never retry more than 3 times, never retry a non-retryable class at all, and never run the
same repair over the same records more than twice.

## Reference

- `references/how-sync-works.md` — the package's sync path, eligibility rules, real limits
- `references/api-selection.md` — which Salesforce API to use at which volume, and why
- `references/soql-cohorts.md` — every assessment query
- `references/apex-templates.md` — chunked enabling, off/on re-drive, drain polling
- `references/troubleshooting.md` — full error → cause → recovery matrix
