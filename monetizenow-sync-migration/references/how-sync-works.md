# How MonetizeNow sync actually works

Everything here is read off the MonetizeNow managed package source (v0.59). Field names are shown with the
`monetize__` namespace prefix as they appear in a customer org.

## The path a record takes

1. **DML on Account/Contact** → `AccountTrigger` / `ContactTrigger` → handler.
2. **`TriggerHandler.validateRun()`** — bails out entirely if
   `MonetizeNowSettings__c.DisableTriggers__c = true` or the handler was bypassed in-transaction
   (`AutomationControl.disable()`).
3. **`afterInsert` / `afterUpdate`** → `ObjectSyncService.publishSyncRecordEvents(...)`.
   - Skipped if the transaction originated from MonetizeNow itself (the connector always
     sends `mnSync=true` on the request URL) — this is the loop-prevention guard.
   - Filters to eligible records (below), then on **update** further filters to records
     where at least one *tracked sync field* changed.
4. **`System.enqueueJob(new ObjectSyncQueueable(events), delay)`** — one queueable per
   trigger invocation, i.e. per 200-record DML chunk. Guarded by
   `Limits.getQueueableJobs() < Limits.getLimitQueueableJobs()`; **when that guard fails the
   events are dropped with only a `System.debug`** — no exception, no error field, nothing
   user-visible. This is the silent failure mode at volume.
5. **`ObjectSyncQueueable.execute`** — splits into chunks of `MAX_EVENTS_PER_JOB = 75`
   (the 100-callouts-per-transaction limit), syncs the first 75, then chains a new
   queueable for the remainder.
6. **`ObjectSyncService.syncRecords`** — one HTTP callout per record.
   `POST /connector/crm/salesforce/connect/accounts` when `Id__c` is null, otherwise
   `PUT .../accounts/{Id__c}`. Contacts use `.../contacts`. Timeout 120s per callout.
7. **Write-back** — on success: `Id__c` (and other MN fields) populated, `SyncError__c`
   cleared, `LastSyncDate__c` stamped. On failure: `SyncError__c` set to
   `key: details` (truncated to 1000 chars). Results are committed at the end of the
   queueable via `Database.update(records, false)` with `syncEnabled = false` so the
   write-back does not re-trigger sync.

## Eligibility rules (verbatim from the selectors)

**Account** — `AccountSelector.isRecordEligibleToSync`:
```
EnableSync__c == true
```

**Contact** — `ContactSelector.isRecordEligibleToSync`:
```
EnableSync__c == true  AND  AccountSyncEnabled__c == true  AND  Email != null
```
`AccountSyncEnabled__c` is a formula: `AND(Account.EnableSync__c, NOT(ISBLANK(Account.Id__c)))`.

Two consequences that drive the whole procedure:
- **A contact cannot sync until its account has landed in MonetizeNow** (`Id__c` populated,
  not merely `EnableSync = true`). Always do accounts first and let them drain.
- **A contact with no email will never sync.** Flipping the checkbox on it is a no-op.

## On update, a *tracked field* must change

`publishSyncRecordEvents` for updates runs the eligible records through
`Utils.findChangedRecs(..., syncFields)`. The tracked field list is
`EnableSync__c`, `Name`, the billing/shipping address fields (Account) or the
name/title/email/phone/mailing fields (Contact) — **plus every field configured in
`FieldMapping__c`**, so it varies per org.

This is why **re-saving a stuck record does nothing** and why the repair must be
*off, then on, in two separate transactions* — the off→on transition is itself the tracked
change that re-fires the sync.

## The Account→Contact fan-out

`AccountTriggerHandler.syncContactsIfAccountSyncNewlyEnabled` runs on every
`EnableSync__c: false → true` transition. It finds contacts under those accounts where
`Account.EnableSync__c = true AND EnableSync__c = true AND Id__c = null AND Email != null`,
and enqueues them — immediately if the account already has an `Id__c`, otherwise on a
**1-minute delay** to give the account time to land in MonetizeNow.

So the true callout volume of a backfill is *accounts + their unsynced eligible contacts*,
and the delayed batch is a second wave of queueables a minute later. Size the estimate
accordingly, and expect the queue to refill once after you think it has drained.

## Limits that actually bind

| Limit | Value | Where it bites |
| --- | --- | --- |
| Queueables enqueued per transaction | 50 | >10,000 records in one DML ⇒ silent drop |
| Callouts per transaction | 100 | package self-limits to 75/queueable |
| Cumulative callout timeout per transaction | 120s | slow MonetizeNow responses ⇒ batch failure |
| SOQL query rows | 50,000 | any unbounded query over the scope |
| DML rows | 10,000 | any unbounded `update` over the scope |
| Apex CPU | 10s sync / 60s async | wide queries + many field mappings |
| Heap | 6MB sync / 12MB async | wide `SELECT` over many rows |
| Async executions | 250,000 / 24h (or 200 × licenses) | very large backfills |

## Settings worth reading first

`monetize__MonetizeNowSettings__c` (org defaults):
- `DisableTriggers__c` — **enforced**, via `AutomationControl.triggersEnabled()`. If true,
  nothing syncs. Check this first.
- `SyncAllNewRecords__c` — when true, `ObjectSyncService.enableSyncOnRecord` sets
  `EnableSync__c = true` on every new Account/Contact at insert. Tells you whether the
  backlog will keep growing.
- `DisableSync__c` — surfaced in the package config UI; it is **not** read by the trigger
  or sync service in this version, so do not rely on it to stop a sync.
- `DefaultCurrency__c` — applied as `BillingCurrency__c` when the account has none.
- `TenantId__c` / `Domain__c` — which MonetizeNow tenant the org points at. Confirm this
  matches the customer's expectation before mass-syncing.

## Fields

Account and Contact both carry: `EnableSync__c`, `Id__c` (MonetizeNow id),
`ExternalId__c` (unique mirror of `Id__c`), `SyncError__c` (long text, 1000),
`LastSyncDate__c`, `BillingStatus__c`, `MonetizeNowLink__c`.
Contact adds `AccountSyncEnabled__c` (formula) and `IsPrimary__c`.
Account adds `BillingCurrency__c`, `LatestContract__c`, `PrimaryBillingContact__c`.

`SyncError__c` is the single source of truth for "sync ran and failed". Empty `SyncError__c`
with empty `Id__c` on an enabled record means the sync **never ran** — a dropped enqueue or
a still-queued job, not a rejection.
