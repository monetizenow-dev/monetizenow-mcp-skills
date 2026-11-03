# Execution templates

Every template is bounded and resumable. Run them as anonymous Apex through the Salesforce
MCP connection, or via `sf apex run --file <path> --target-org <alias>`.

`<SCOPE>` stands for the user's selection criteria as a SOQL predicate — see
`soql-cohorts.md`. The examples use a closed-won filter purely as an illustration; replace
it with whatever the user actually asked for.

Each emits a single `MN_RESULT: {...}` line — parse that from the debug log rather than
scraping prose, and append it to your `state.json` ledger after every batch.

## 1. Backfill — enable sync on never-enabled Accounts (cohort A1)

Keyset pagination on `Id`. Run once per batch, feeding `lastId` forward from the previous
`MN_RESULT`. Start at `'000000000000000000'`. Stop when `count` is 0.

```apex
Id lastId = '000000000000000000';   // <-- from the previous batch's MN_RESULT.lastId
Integer batchSize = 200;            // 200 = exactly one queueable. Never exceed 2000.

List<Account> accounts = [
    SELECT Id
    FROM Account
    WHERE monetize__EnableSync__c = FALSE
      AND monetize__Id__c = NULL
      AND Id > :lastId
      AND Id IN (SELECT AccountId FROM Opportunity WHERE StageName = 'Closed Won')   // <-- <SCOPE>
    ORDER BY Id
    LIMIT :batchSize
];

for (Account a : accounts) {
    a.monetize__EnableSync__c = true;
}

List<String> failures = new List<String>();
Integer ok = 0;
if (!accounts.isEmpty()) {
    Database.SaveResult[] results = Database.update(accounts, false);   // partial success
    for (Integer i = 0; i < results.size(); i++) {
        if (results[i].isSuccess()) {
            ok++;
        } else {
            String msg = '';
            for (Database.Error e : results[i].getErrors()) {
                msg += e.getStatusCode() + ': ' + e.getMessage() + '; ';
            }
            failures.add(accounts[i].Id + ' | ' + msg);
        }
    }
}

System.debug('MN_RESULT: ' + JSON.serialize(new Map<String, Object>{
    'count'     => accounts.size(),
    'succeeded' => ok,
    'failed'    => failures.size(),
    'failures'  => failures,
    'lastId'    => accounts.isEmpty() ? String.valueOf(lastId) : String.valueOf(accounts[accounts.size() - 1].Id),
    'queueables'=> Limits.getQueueableJobs(),
    'cpuMs'     => Limits.getCpuTime()
}));
```

Swap the `WHERE` clause for the Contact cohort to backfill contacts — but only **after**
the account pass has drained:

```apex
List<Contact> contacts = [
    SELECT Id
    FROM Contact
    WHERE monetize__EnableSync__c = FALSE
      AND monetize__Id__c = NULL
      AND Email != NULL
      AND Account.monetize__Id__c != NULL     -- parent must already be in MonetizeNow
      AND Id > :lastId
    ORDER BY Id
    LIMIT :batchSize
];
```

## 2. Repair — toggle off, then back on (cohorts A2/A3/A4, C2/C3/C4)

Two passes, **two separate transactions**. A single transaction that flips off and on
again produces no net field change and therefore no sync.

### Pass 1 — off (cheap: publishes no events)

`EnableSync = false` makes the record ineligible, so no sync events fire. This pass can use
larger batches — up to the 10,000 DML row limit — or Bulk API.

```apex
Id lastId = '000000000000000000';
Integer batchSize = 2000;

List<Account> accounts = [
    SELECT Id
    FROM Account
    WHERE monetize__EnableSync__c = TRUE
      AND monetize__Id__c = NULL          -- stalled + errored; drop this line to include A4
      AND Id > :lastId
    ORDER BY Id
    LIMIT :batchSize
];
for (Account a : accounts) { a.monetize__EnableSync__c = false; }
Database.SaveResult[] results = accounts.isEmpty()
    ? new List<Database.SaveResult>()
    : Database.update(accounts, false);

System.debug('MN_RESULT: ' + JSON.serialize(new Map<String, Object>{
    'pass'   => 'off',
    'count'  => accounts.size(),
    'lastId' => accounts.isEmpty() ? String.valueOf(lastId) : String.valueOf(accounts[accounts.size() - 1].Id)
}));
```

**Record the Ids you turned off.** Persist them in your ledger — after pass 1 those records
no longer match the "stalled/errored" predicate, so pass 2 must target them explicitly.

### Pass 2 — on (expensive: this is what re-fires the sync)

Chunk at 200, exactly like the backfill, driving off the captured Id list:

```apex
Set<Id> targetIds = new Set<Id>{ /* 200 ids captured in pass 1 */ };

List<Account> accounts = [SELECT Id FROM Account WHERE Id IN :targetIds];
for (Account a : accounts) { a.monetize__EnableSync__c = true; }
Database.SaveResult[] results = Database.update(accounts, false);

System.debug('MN_RESULT: ' + JSON.serialize(new Map<String, Object>{
    'pass'       => 'on',
    'count'      => accounts.size(),
    'queueables' => Limits.getQueueableJobs()
}));
```

If a run dies between passes, records are left with `EnableSync = false` — recoverable, but
tell the user, and resume pass 2 from the ledger.

## 3. Queue drain check

Between batches, and before declaring the run finished:

```apex
Integer pending = [
    SELECT COUNT()
    FROM AsyncApexJob
    WHERE JobType = 'Queueable' AND Status IN ('Queued', 'Processing')
];
System.debug('MN_RESULT: ' + JSON.serialize(new Map<String, Object>{ 'pending' => pending }));
```

Pause when `pending > 100`. Consider it drained only when it reads 0 **twice, 90s apart** —
the Account→Contact fan-out enqueues a delayed wave one minute behind each account batch.

## 4. Verification — did the records actually land?

Flipping the checkbox is not success; `Id__c` being populated is.

```apex
List<AggregateResult> rows = [
    SELECT COUNT(Id) total
    FROM Account
    WHERE monetize__EnableSync__c = TRUE AND monetize__Id__c = NULL
];
```

Then pull the residual errors for the report:

```sql
SELECT Id, Name, monetize__SyncError__c, monetize__LastSyncDate__c
FROM Account
WHERE monetize__EnableSync__c = TRUE AND monetize__SyncError__c != NULL
ORDER BY monetize__LastSyncDate__c DESC NULLS LAST
LIMIT 500
```

## 5. sf CLI (no Salesforce MCP, or high volume)

```bash
# size the scope
sf data query --query "SELECT COUNT() FROM Account WHERE monetize__EnableSync__c = false AND monetize__Id__c = null" --target-org <alias>

# list more than 50k rows (Bulk API 2.0 query) — for large scopes and failure listings
sf data query --query "<soql>" --bulk --wait 10 --result-format csv --target-org <alias> > accounts.csv

# run one batch
sf apex run --file ./batch.apex --target-org <alias>

# pass 1 (off) only — safe in bulk because it publishes no sync events.
# NEVER use bulk for the "on" pass: batch sizes are server-chosen and can blow past
# the 50-queueable-per-transaction ceiling, silently dropping syncs.
sf data update bulk --sobject Account --file ./disable.csv --target-org <alias> --wait 10
```

## Non-negotiables

- Bounded query (`ORDER BY Id LIMIT :batchSize`) + keyset pagination. Always.
- One DML per batch, 200 records by default, 2,000 absolute ceiling.
- `Database.update(records, false)` — never all-or-nothing.
- Ledger written after every batch, so any interruption resumes instead of restarting.
- Accounts drained before Contacts.
- Never widen the batch size to "go faster" — the failure it causes is silent.
