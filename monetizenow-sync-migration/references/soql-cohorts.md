# Assessment queries

All read-only. Run the `COUNT()` form first — never pull rows to count them.
Namespace prefix shown as `monetize__`; drop it in an unpackaged org.

## Scope filter

`<SCOPE>` below is the user's selection criteria expressed as a SOQL predicate on Account.
Derive it from what they asked for, and show it back to them before running anything.
Common shapes:

```sql
-- has a closed-won opportunity
Id IN (SELECT AccountId FROM Opportunity WHERE StageName = 'Closed Won')

-- a named list
Id IN ('001...', '001...')          -- or  Name IN ('Acme', 'Globex')

-- by record type, owner, or segment field
RecordType.DeveloperName = 'Customer'
OwnerId = '005...'
Type = 'Customer - Direct'

-- by date window
CreatedDate >= 2024-01-01T00:00:00Z

-- everything
TRUE
```

Semi-joins (`Id IN (SELECT ...)`) are capped by the platform, so on very large orgs prefer
a direct field filter on Account where one exists.

## Account cohorts

```sql
-- A1 never-enabled — the backfill target
SELECT COUNT() FROM Account
WHERE monetize__EnableSync__c = FALSE AND monetize__Id__c = NULL AND <SCOPE>

-- A2 stalled — enabled, never reached MonetizeNow, no error recorded
SELECT COUNT() FROM Account
WHERE monetize__EnableSync__c = TRUE AND monetize__Id__c = NULL
  AND monetize__SyncError__c = NULL AND <SCOPE>

-- A3 errored — sync ran, MonetizeNow rejected it
SELECT COUNT() FROM Account
WHERE monetize__EnableSync__c = TRUE AND monetize__Id__c = NULL
  AND monetize__SyncError__c != NULL AND <SCOPE>

-- A4 synced-but-degraded — landed once, last update failed
SELECT COUNT() FROM Account
WHERE monetize__Id__c != NULL AND monetize__SyncError__c != NULL AND <SCOPE>

-- A5 healthy — for the denominator
SELECT COUNT() FROM Account
WHERE monetize__Id__c != NULL AND monetize__SyncError__c = NULL AND <SCOPE>
```

Error breakdown (sample, then group client-side — `SyncError__c` is a long text area and
cannot be used in `GROUP BY`):

```sql
SELECT Id, Name, monetize__SyncError__c, monetize__LastSyncDate__c
FROM Account
WHERE monetize__SyncError__c != NULL AND <SCOPE>
ORDER BY monetize__LastSyncDate__c DESC NULLS LAST
LIMIT 200
```

## Contact cohorts

```sql
-- C1 never-enabled and ready (parent account already in MonetizeNow)
SELECT COUNT() FROM Contact
WHERE monetize__EnableSync__c = FALSE AND monetize__Id__c = NULL
  AND Email != NULL AND Account.monetize__Id__c != NULL

-- C2 stalled
SELECT COUNT() FROM Contact
WHERE monetize__EnableSync__c = TRUE AND monetize__Id__c = NULL
  AND monetize__SyncError__c = NULL
  AND Email != NULL AND monetize__AccountSyncEnabled__c = TRUE

-- C3 errored
SELECT COUNT() FROM Contact
WHERE monetize__EnableSync__c = TRUE AND monetize__Id__c = NULL
  AND monetize__SyncError__c != NULL

-- C4 synced-but-degraded
SELECT COUNT() FROM Contact
WHERE monetize__Id__c != NULL AND monetize__SyncError__c != NULL

-- C5 ineligible: no email — will NEVER sync, report only, do not flip
SELECT COUNT() FROM Contact
WHERE Email = NULL AND Account.monetize__EnableSync__c = TRUE

-- C6 blocked by account — parent not in MonetizeNow yet; re-check after the account pass
SELECT COUNT() FROM Contact
WHERE monetize__Id__c = NULL AND Email != NULL
  AND Account.monetize__Id__c = NULL AND Account.monetize__EnableSync__c = TRUE
```

## Fan-out estimate

How many contacts the account pass will drag along (this is added callout volume, and the
delayed wave arrives ~1 minute after each account batch):

```sql
SELECT COUNT() FROM Contact
WHERE monetize__EnableSync__c = TRUE AND monetize__Id__c = NULL AND Email != NULL
  AND AccountId IN (SELECT Id FROM Account
                    WHERE monetize__EnableSync__c = FALSE AND monetize__Id__c = NULL AND <SCOPE>)
```

## Queue depth — the throttle signal

```sql
SELECT COUNT() FROM AsyncApexJob
WHERE JobType = 'Queueable' AND Status IN ('Queued','Processing')

-- package jobs specifically
SELECT Status, COUNT(Id) FROM AsyncApexJob
WHERE JobType = 'Queueable' AND ApexClass.Name = 'ObjectSyncQueueable'
  AND CreatedDate = TODAY
GROUP BY Status
```

Gate each write batch on this: if queued+processing is above ~100, wait before continuing.
Treat the queue as drained only after it reads zero **twice, 90 seconds apart** — the
Account→Contact fan-out enqueues a delayed wave one minute later.

## Settings and config

```sql
SELECT Id, monetize__DisableTriggers__c, monetize__DisableSync__c,
       monetize__SyncAllNewRecords__c, monetize__DefaultCurrency__c,
       monetize__TenantId__c, monetize__Domain__c
FROM monetize__MonetizeNowSettings__c

-- which fields count as "changed" for sync, beyond the built-ins
SELECT Id, monetize__ObjectAPIName__c, monetize__FieldAPIName__c
FROM monetize__FieldMapping__c
```
