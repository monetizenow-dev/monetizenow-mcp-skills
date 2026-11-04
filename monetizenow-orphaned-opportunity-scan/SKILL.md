---
name: monetizenow-orphaned-opportunity-scan
description: Scan Salesforce for opportunities orphaned from MonetizeNow — SFDC opportunities that were synced with a MonetizeNow opportunity but no longer have a working two-way link to it, whether the MonetizeNow customId was re-pointed elsewhere, cleared, or the SFDC monetize__Id__c was wiped. Pairs the Salesforce MCP connector (SOQL) with the MonetizeNow MCP connector, read-only, and reports each broken link with its likely cause. Use it whenever someone asks to find orphaned, stale, mismatched, duplicated or broken opportunity links between Salesforce and MonetizeNow, to audit the SFDC↔MN opportunity sync, or to check the fallout of changing an opportunity's custom id.
---

# Scanning for orphaned opportunities

The opportunity link is held on **both** sides, and nothing keeps the two halves in step:

| Side | Field | Holds |
|---|---|---|
| Salesforce `Opportunity` | `monetize__Id__c` | the MonetizeNow opportunity id (`opp_…`) |
| MonetizeNow `opportunity` | `customId` (filter: `custom_id`) | the Salesforce opportunity id (`006…`) |

A link is healthy only when both halves agree:

```
SFDC.Id ──monetize__Id__c──▶ opp_X      and      opp_X.customId == SFDC.Id
```

**Orphans usually have both halves blank, not mismatched.** When a `customId` is changed, the sync
clears `monetize__Id__c` on the SFDC opportunity it leaves, and may clear `customId` on the MN
side as well. So you can't find orphans by looking only at linked SFDC records. You find them from
what the sync leaves behind:

- `monetize__LastSyncDate__c` stays set on an SFDC opportunity that once synced.
- `monetize__Quote__c` records stay under the old SFDC opportunity, and each one carries its MN
  quote id in `monetize__Id__c`.

The scan runs four passes. Passes B and C usually find the orphans; pass A catches the rarer
half-broken link.

This skill is **read-only**. It never updates either system; fixing what it finds is a separate,
user-approved step.

## Pass A — linked SFDC opportunities that are not linked back

Pull every linked SFDC opportunity, paging with a keyset on `Id` (SOQL `OFFSET` caps at 2,000):

```sql
SELECT Id, Name, AccountId, StageName, monetize__Id__c, monetize__LastSyncDate__c,
       monetize__SyncError__c, LastModifiedDate
FROM Opportunity
WHERE monetize__Id__c != null AND Id > '<last Id seen>'
ORDER BY Id
LIMIT 2000
```

Start with `Id > '000000000000000'` and stop when a page comes back short. Then run the duplicate
check; every group it returns contains at least one orphan:

```sql
SELECT monetize__Id__c, COUNT(Id) n FROM Opportunity
WHERE monetize__Id__c != null
GROUP BY monetize__Id__c HAVING COUNT(Id) > 1
```

MonetizeNow opportunity search can't filter on its own `id`, but it can filter on `custom_id` with
`in:`. Resolve the SFDC ids in batches of about 50, and pass each id in **both** its 18- and
15-character forms (100 values, matching `page_size: 100`). Check `pagination.total`. Key the
results on the first 15 characters of the SFDC id; never lowercase it, because 15-character ids are
case-sensitive.

| Search returned for this SFDC id | Classification |
|---|---|
| one MN opp whose `id` == `monetize__Id__c` | OK |
| one MN opp with a different `id` | **SPLIT** |
| several MN opps | **MULTI-CLAIM** |
| nothing | retrieve `monetize__Id__c`: `customId` set to another SFDC id → **RE-POINTED**; `customId` empty → **MN-UNLINKED**; `Resource not found by Id` → **DANGLING** |

## Pass B — SFDC opportunities that synced once and lost their link

```sql
SELECT Id, Name, AccountId, StageName, CreatedDate, monetize__LastSyncDate__c,
       (SELECT Id, monetize__Id__c FROM monetize__Quotes__r)
FROM Opportunity
WHERE monetize__Id__c = null AND monetize__LastSyncDate__c != null
ORDER BY Id
```

Every row here is an orphan candidate: it was synced, and it no longer names an MN opportunity.
Two shapes:

- **Has `monetize__Quotes__r` rows.** Retrieve the MN quote named by a child's `monetize__Id__c` and
  read its opportunity. Then read that MN opportunity's `customId`:
  - empty → **BOTH-UNLINKED**: the SFDC opp still has the quote, and the MN opp still exists, but
    neither side points at the other. Pass C finds the same pair from the MN side.
  - another SFDC id → **RE-POINTED**: the quote's MN opp now belongs to that SFDC opp.
- **No quote rows.** → **EMPTY SHELL**: the sync created it and the quote then moved away. Look for
  the quote that moved: a `monetize__Quote__c` whose `CreatedDate` is within a few seconds of this
  opp's `CreatedDate` but whose parent is now another opportunity. When one matches, name that
  opportunity as the likely re-point target. Flag it as inferred from timestamps, not read from a
  field.

`monetize__SyncError__c` can't be used in a SOQL `WHERE`; select it instead.

## Pass C — MN opportunities with no customId

```
monetizenow_search_objects  object_type: "opportunity"
filters: {"custom_id": "eq:null", "page_size": 100}
```

For each, find the SFDC quote record for its `primaryQuote.id`:

```sql
SELECT Id, monetize__Id__c, monetize__Opportunity__c, monetize__Opportunity__r.monetize__Id__c
FROM monetize__Quote__c WHERE monetize__Id__c IN ('quot_…', …)
```

A hit whose parent opportunity has a blank `monetize__Id__c` is the BOTH-UNLINKED pair from pass B.
Don't report it twice. No hit means an MN-native opportunity that was never in Salesforce, which is
out of scope.

## Pass D — MN customIds that don't exist in this org

Take every non-empty `customId` from an unfiltered opportunity search (paged) and query
`SELECT Id FROM Opportunity WHERE Id IN (…)`. Ids that come back missing are **NO-SFDC-TARGET**.
A block of them sharing a prefix other than this org's (`006KX…` vs `006al…`) usually means the
tenant was once connected to a different Salesforce org. That isn't the re-pointing bug, so report
those ids separately.

## Report

Lead with counts: SFDC opportunities scanned, linked and OK, then each failure class. Then one table
per non-empty class, most actionable first: RE-POINTED, BOTH-UNLINKED, EMPTY SHELL, DANGLING,
SPLIT/MULTI-CLAIM, NO-SFDC-TARGET.

| SFDC opp | Name | MN opp | MN `customId` now | MN quote(s) | Created | Last sync | Evidence |
|---|---|---|---|---|---|---|---|

Fill every cell from a tool result; leave it blank rather than guessing. When a link was inferred
from timestamps, say so in **Evidence**. For more than about 50 findings, also write a CSV to the
scratchpad and give its path. A clean run reports its scanned counts so it can't be mistaken for a
scan that didn't run.

## What not to do

- **Don't fix anything during the scan.** Re-linking or clearing either side changes what the next
  sync does and can strand a quote. Present the findings and let the user decide per record.
- **Don't stop at pass A.** A clean pass A with no duplicates says nothing about orphans: the usual
  orphan has a blank `monetize__Id__c` and never appears in it.
- **Don't treat every SFDC opportunity without `monetize__Id__c` as orphaned.** Without a
  `monetize__LastSyncDate__c` or a `monetize__Quote__c` child, it was never synced.
- **Don't retrieve every MN opportunity one by one.** Batched `custom_id` searches clear the healthy
  majority; retrieve only the rows that fail.

## Related skills

`monetizenow-data-retrieval` for reading empty and rejected MonetizeNow searches.
`monetizenow-data-analysis` for tenants with reporting access, where the MonetizeNow side can be
pulled in one SQL query instead of batched searches.
